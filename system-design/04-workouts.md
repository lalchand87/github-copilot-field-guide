# Workouts — Think in systems

> 50 full system designs, each worked through in 9 stages (requirements, scale, API, data model, architecture, deep dives, failures, trade-offs, interview questions) plus the staff-level view — and 24 reusable end-to-end recipes.

## Contents

- **Foundational** (5): [URL shortener](#url-shortener) · [Distributed rate limiter](#distributed-rate-limiter) · [Distributed cache](#distributed-cache) · [Unique ID service](#unique-id-service) · [Durable job scheduler](#durable-job-scheduler)
- **Social** (5): [Social news feed](#social-news-feed) · [Photo sharing](#photo-sharing) · [Follow graph](#follow-graph) · [Threaded comments](#threaded-comments) · [Ephemeral stories](#ephemeral-stories)
- **Messaging** (5): [Direct and group chat](#direct-and-group-chat) · [Notification platform](#notification-platform) · [Email service](#email-service) · [Webhook delivery](#webhook-delivery) · [Publish-subscribe event bus](#publish-subscribe-event-bus)
- **Media** (5): [YouTube-style video platform](#youtube-style-video-platform) · [Live streaming platform](#live-streaming-platform) · [Music streaming](#music-streaming) · [Video conferencing](#video-conferencing) · [Podcast platform](#podcast-platform)
- **Storage** (5): [Cloud file synchronization](#cloud-file-synchronization) · [Distributed object storage](#distributed-object-storage) · [Backup and point-in-time restore](#backup-and-point-in-time-restore) · [Content-addressed blob store](#content-addressed-blob-store) · [Distributed filesystem](#distributed-filesystem)
- **Commerce** (5): [Commerce checkout](#commerce-checkout) · [Inventory reservation](#inventory-reservation) · [Payment ledger](#payment-ledger) · [Ticket booking](#ticket-booking) · [Marketplace orders](#marketplace-orders)
- **Real Time** (5): [Ride dispatch](#ride-dispatch) · [Collaborative document editor](#collaborative-document-editor) · [Presence and typing indicators](#presence-and-typing-indicators) · [Game leaderboard](#game-leaderboard) · [Fleet tracking](#fleet-tracking)
- **Search** (5): [Web search engine](#web-search-engine) · [Search autocomplete](#search-autocomplete) · [Product search](#product-search) · [Log ingestion and search](#log-ingestion-and-search) · [Vector retrieval service](#vector-retrieval-service)
- **Infrastructure** (5): [Global load balancer](#global-load-balancer) · [Authoritative DNS](#authoritative-dns) · [Metrics and alerting](#metrics-and-alerting) · [Feature flag platform](#feature-flag-platform) · [Service discovery and configuration](#service-discovery-and-configuration)
- **Advanced** (5): [Multi-region key-value store](#multi-region-key-value-store) · [Durable workflow engine](#durable-workflow-engine) · [Ad auction and budget pacing](#ad-auction-and-budget-pacing) · [Recommendation serving](#recommendation-serving) · [Streaming fraud detection](#streaming-fraud-detection)

- **Recipes** (24): [Read-Heavy URL Shortener](#read-heavy-url-shortener) · [Global Product Catalog](#global-product-catalog) · [Reliable Checkout](#reliable-checkout) · [Flash Sale Inventory](#flash-sale-inventory) · [Ticket Reservation](#ticket-reservation) · [Hybrid Social Feed](#hybrid-social-feed) · [Durable Chat](#durable-chat) · [Live Collaboration](#live-collaboration) · [Video On Demand](#video-on-demand) · [Live Video Broadcast](#live-video-broadcast) · [Resumable File Drive](#resumable-file-drive) · [Searchable Knowledge Base](#searchable-knowledge-base) · [Location-Based Ride Matching](#location-based-ride-matching) · [Notification Platform](#notification-platform) · [Webhook Delivery Platform](#webhook-delivery-platform) · [Multi-Tenant Rate Limiter](#multi-tenant-rate-limiter) · [Durable Job Scheduler](#durable-job-scheduler) · [Public API With Async Exports](#public-api-with-async-exports) · [Real-Time Analytics Dashboard](#real-time-analytics-dashboard) · [Data Warehouse Ingestion](#data-warehouse-ingestion) · [Metrics And Alerting](#metrics-and-alerting) · [Transactional Ledger](#transactional-ledger) · [Multi-Region SaaS](#multi-region-saas) · [Fraud Decision Pipeline](#fraud-decision-pipeline)

## Foundational

### URL shortener

*Create durable short links and serve redirects with a small latency budget.*

**1. Requirements**

- Custom aliases and expiry

- Redirect without synchronous analytics

- Prevent alias collisions

**2. Scale:** 10M daily users; 20 redirects each; 100:1 read/write ratio; redirect p99 target 80 ms.

**3. API**

- `POST /links {url,alias?,expiresAt?}`

- `GET /{code} -> 302`

- `DELETE /links/{code}`

**4. Data model**

- `links(code PK, target, owner, expires_at, version)`

- `click_events(event_id PK, code, timestamp)`

**5. Architecture:** `Browser` → `Edge cache` → `Redirect API` → `Redis hot links` → `Durable link store` → `Async click log`

**6. Deep dives**

- Cache the alias at edge and process level; coalesce misses and retain a stale value during refresh. Apply versioned invalidation for edits and keep click ingestion asynchronous.

- Use a unique primary key or conditional insert for the alias. Return conflict for the losing request; bind a client idempotency key to the winning request payload and result.

**7. Failures**

- An unavailable alias was cached as missing before its owner created it.

- Invalidate the negative entry after the durable insert, use a short negative TTL, and optionally bypass cache for the owner after creation. Track negative-hit rate and creation-to-visibility delay.

**8. Trade-offs**

- Long edge TTLs improve redirect latency but delay target changes; analytics can be eventually consistent while alias ownership cannot.

**9. Interview questions**

- One alias suddenly receives 200,000 redirects per second.

- An unavailable alias was cached as missing before its owner created it.

- Two create requests both observe that /launch is available.

- Model abuse takedown propagation separately from ordinary edits; establish a maximum revocation delay and test cache invalidation under partial regional failure.

**Staff-level view:** Model abuse takedown propagation separately from ordinary edits; establish a maximum revocation delay and test cache invalidation under partial regional failure.

**Reference design:** https://redis.io/docs/latest/develop/reference/eviction/

---

### Distributed rate limiter

*Enforce burst and sustained request budgets across API workers.*

**1. Requirements**

- Per-tenant limits

- Weighted expensive operations

- Return retry timing

**2. Scale:** 100M decisions/day; 1M tenants; 2 ms regional decision budget.

**3. API**

- `POST /decisions {tenant,cost,requestId}`

- `PUT /policies/{tenant}`

**4. Data model**

- `bucket(tenant, tokens, last_refill, policy_version)`

- `policy(tenant PK, rate, burst)`

**5. Architecture:** `API gateway` → `Policy cache` → `Regional limiter` → `Atomic bucket store` → `Usage metrics`

**6. Deep dives**

- Allocate bounded token leases from a regional owner to gateways. A lease bounds overshoot and reduces central operations; reclaim only with an expiry rule that prevents double spending.

- Partition the allowance into regional allocations with a single transfer authority, or route strict decisions to one owner. Quantify the maximum excess caused by outstanding leases before promising a global cap.

**7. Failures**

- The bucket datastore times out while the payment API remains healthy.

- Use conservative local emergency budgets for essential reads; reject expensive writes if their budget cannot be established. Avoid unlimited fail-open traffic and record every fallback decision.

**8. Trade-offs**

- A globally exact limit increases coordination latency; distributed leases buy availability with a calculable overshoot bound.

**9. Interview questions**

- A single tenant sends half of all requests to one token bucket.

- The bucket datastore times out while the payment API remains healthy.

- A tenant uses the full global allowance simultaneously in three regions.

- Define fairness during regional evacuation and prevent one tenant from monopolizing the emergency budget. Roll out policy changes with explicit version semantics.

**Staff-level view:** Define fairness during regional evacuation and prevent one tenant from monopolizing the emergency budget. Roll out policy changes with explicit version semantics.

**Reference design:** https://redis.io/docs/latest/develop/reference/eviction/

---

### Distributed cache

*Build a bounded cache in front of authoritative application data.*

**1. Requirements**

- TTL and eviction

- Horizontal redistribution

- Bound stale reads

**2. Scale:** 2B GETs/day; 200GB hot data; 1KB median values.

**3. API**

- `GET /cache/{key}`

- `PUT /cache/{key} {value,ttl,version}`

- `DELETE /cache/{key}`

**4. Data model**

- `entry(key, value, version, expires_at)`

- `ring(epoch, virtual_nodes)`

**5. Architecture:** `Application` → `Client routing map` → `Cache shards` → `Read-through loader` → `Source database`

**6. Deep dives**

- Move small hash ranges gradually, read old owners during handoff, and throttle warmup below database headroom. Use virtual nodes and request coalescing; adding nodes all at once can worsen overload.

- Store versioned values and use compare-and-set against an invalidation watermark or versioned namespace. If that complexity is excessive, bound staleness explicitly and route read-after-write requests to the source.

**7. Failures**

- A batch import gives ten million popular keys the same expiry.

- Add TTL jitter, single-flight refresh, and bounded stale-while-revalidate for permitted data. Gate refill concurrency against source capacity and shed cacheable traffic before overwhelming the database.

**8. Trade-offs**

- Aggressive eviction lowers memory cost but creates source traffic; serving stale data is acceptable only for explicitly chosen fields.

**9. Interview questions**

- Adding four nodes remaps keys and doubles database reads.

- A batch import gives ten million popular keys the same expiry.

- A delayed loader writes version 7 after an update invalidated version 8.

- Size for failover with one shard unavailable, including connection storms, allocator overhead, replication buffers, and the source load from a cold replacement.

**Staff-level view:** Size for failover with one shard unavailable, including connection storms, allocator overhead, replication buffers, and the source load from a cold replacement.

**Reference design:** https://redis.io/docs/latest/develop/reference/eviction/

---

### Unique ID service

*Issue unique sortable identifiers without a global transaction per ID.*

**1. Requirements**

- No duplicate IDs

- Predictable bit budget

- Tolerate worker restarts

**2. Scale:** 500M IDs/day; bursts of 200K IDs/s; IDs fit in signed 64-bit storage.

**3. API**

- `POST /ids:allocate {count}`

- `GET /workers/{id}/lease`

**4. Data model**

- `worker_lease(worker_id PK, epoch, expires_at)`

- `local_state(last_timestamp, sequence)`

**5. Architecture:** `Client library` → `Worker identity registry` → `Local timestamp and sequence generator` → `Lease monitor`

**6. Deep dives**

- Block until the next safe tick, distribute allocation across leased worker IDs, or revise the bit split. Benchmark peak per worker, and account for a reduced timestamp lifetime when increasing sequence bits.

- Ensure the identity-generation epoch is encoded in the uniqueness space or prevent reuse for the full uncertainty interval. Gate generation on conservative lease validity and prove behavior for a paused process that resumes late.

**7. Failures**

- A host clock steps backwards by three seconds after synchronization.

- Refuse or pause generation until time catches up, or use a persisted logical timestamp strategy with a documented skew budget. Alert on clock regressions; never reuse a timestamp-sequence pair.

**8. Trade-offs**

- Sortable IDs expose approximate creation time; larger worker or sequence fields shorten the timestamp horizon.

**9. Interview questions**

- A worker needs more IDs in one clock tick than its sequence bits allow.

- A host clock steps backwards by three seconds after synchronization.

- A partitioned old worker continues issuing IDs after its lease was reassigned.

- Prove uniqueness across clock rollback, restored VM snapshots, identity reuse, and disaster recovery. Include a documented migration before the timestamp field expires.

**Staff-level view:** Prove uniqueness across clock rollback, restored VM snapshots, identity reuse, and disaster recovery. Include a documented migration before the timestamp field expires.

**Reference design:** https://redis.io/docs/latest/develop/reference/eviction/

---

### Durable job scheduler

*Execute delayed and recurring jobs with retries and durable status.*

**1. Requirements**

- Schedule and cancel jobs

- Retry transient failures

- Isolate tenants

**2. Scale:** 50M jobs/day; 30-day scheduling horizon; execution delay p99 under 10 seconds.

**3. API**

- `POST /jobs {runAt,payload,idempotencyKey}`

- `POST /jobs/{id}:cancel`

- `GET /jobs/{id}`

**4. Data model**

- `jobs(job_id PK, run_at, state, attempt, lease_epoch)`

- `attempts(job_id, attempt, outcome)`

**5. Architecture:** `Schedule API` → `Time-partitioned job store` → `Due-job scanners` → `Ready queue` → `Leased workers` → `Result store`

**6. Deep dives**

- Bucket due jobs by time and tenant, introduce user-approved scheduling windows, and reserve urgent capacity. Dispatch with fair queues and backlog-age autoscaling; calculate drain time rather than scaling only on CPU.

- Define cancel as successful only before a compare-and-set transition to running. Return already-running otherwise, and offer cooperative cancellation for interruptible work. Track occurrence IDs separately for recurring jobs.

**7. Failures**

- The email side effect succeeded but the attempt acknowledgement was lost.

- Record a stable delivery key per job occurrence and pass it to an idempotent destination when available. Reconcile ambiguous outcomes before resending; represent unknown status if the provider cannot deduplicate.

**8. Trade-offs**

- At-least-once execution is recoverable but requires idempotent effects; at-most-once execution avoids repeats by accepting lost work.

**9. Interview questions**

- Three million reports become due at midnight for the same timezone.

- The email side effect succeeded but the attempt acknowledgement was lost.

- A user cancels a job while a worker has acquired its lease.

- Specify misfire behavior for missed recurring schedules, timezones, daylight-saving transitions, and prolonged outages; avoid replaying years of missed jobs on recovery.

**Staff-level view:** Specify misfire behavior for missed recurring schedules, timezones, daylight-saving transitions, and prolonged outages; avoid replaying years of missed jobs on recovery.

**Reference design:** https://redis.io/docs/latest/develop/reference/eviction/

---

## Social

### Social news feed

*Produce a personalized timeline from followed accounts.*

**1. Requirements**

- Publish posts

- Cursor pagination

- Enforce privacy at read time

**2. Scale:** 50M daily users; 20 feed reads/day; follower counts have a heavy tail.

**3. API**

- `POST /posts`

- `GET /feed?cursor=`

- `PUT /follows/{author}`

**4. Data model**

- `posts(post_id PK, author, created_at, visibility_version)`

- `inbox(user_id, rank_key, post_id)`

- `follows(user_id, author_id)`

**5. Architecture:** `Post API` → `Post store and outbox` → `Fanout workers` → `Feed inbox store` → `Ranking service` → `Feed API`

**6. Deep dives**

- Push posts for ordinary authors into inboxes, pull celebrity posts at read time, and merge by stable rank keys. Bound fanout work per author and measure active-follower coverage instead of writing dormant inboxes.

- Use an opaque cursor containing a ranking snapshot or stable boundary plus a deduplication window. Freeze candidate ordering for a bounded browsing session and refresh explicitly for newer posts.

**7. Failures**

- A cached inbox still references content removed by moderation.

- Filter candidates against current deletion and access state before rendering, propagate tombstones to caches, and backfill if filtering leaves a short page. Purge materialized entries asynchronously.

**8. Trade-offs**

- Fanout-on-write gives fast reads but amplifies celebrity writes; read-time merging saves work at the cost of more feed computation.

**9. Interview questions**

- An account with 80M followers posts during peak traffic.

- A cached inbox still references content removed by moderation.

- Scores change between page one and page two, causing duplicates and skips.

- Set separate SLOs for post acceptance, follower visibility, privacy revocation, and ranking freshness; a single availability number hides the important failures.

**Staff-level view:** Set separate SLOs for post acceptance, follower visibility, privacy revocation, and ranking freshness; a single availability number hides the important failures.

**Reference design:** https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html

---

### Photo sharing

*Upload photos, generate variants, and display profile galleries.*

**1. Requirements**

- Resumable upload

- Thumbnail generation

- Private albums and deletion

**2. Scale:** 8M uploads/day; 4MB original photos; 10 thumbnail views per upload.

**3. API**

- `POST /uploads`

- `POST /uploads/{id}:complete`

- `GET /users/{id}/photos?cursor=`

**4. Data model**

- `photos(photo_id PK, owner, object_key, state, visibility)`

- `variants(photo_id, transform_version, object_key)`

**5. Architecture:** `Mobile client` → `Upload signer` → `Object storage` → `Transform queue` → `Image workers` → `CDN` → `Gallery API`

**6. Deep dives**

- Allowlist transform recipes, key variants by recipe version, precompute popular sizes, and single-flight lazy generation. Rate-limit per original and serve a nearby existing size during backlog.

- Use authenticated delivery or short-lived signed access with a documented revocation bound; invalidate cached variants on urgent removal. Keep object origins private and check album permissions before issuing access.

**7. Failures**

- An object event arrives before image validation and the gallery publishes it.

- Validate format, dimensions, checksums, and required variants before atomically publishing ready metadata. Keep uploads staged and sweep abandoned objects; the gallery reads only ready records.

**8. Trade-offs**

- Precomputing variants increases storage but reduces user-visible transform latency; signed URL lifetime sets part of the access-revocation delay.

**9. Interview questions**

- A layout release asks for a previously uncached transform for every photo.

- An object event arrives before image validation and the gallery publishes it.

- A long-lived CDN URL continues to expose a photo after access is revoked.

- Coordinate deletion of originals, variants, caches, and backup-retention records; expose deletion progress without promising that a metadata delete erases every byte instantly.

**Staff-level view:** Coordinate deletion of originals, variants, caches, and backup-retention records; expose deletion progress without promising that a metadata delete erases every byte instantly.

**Reference design:** https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html

---

### Follow graph

*Maintain directed relationships, counts, and mutual connections.*

**1. Requirements**

- Follow and unfollow

- List followers

- Block accounts

**2. Scale:** 200M accounts; 20B edges; most users have fewer than 1,000 edges.

**3. API**

- `PUT /users/{id}/following/{target}`

- `DELETE /users/{id}/following/{target}`

- `GET /users/{id}/followers`

**4. Data model**

- `out_edges(source, target, version)`

- `in_edges(target, bucket, source)`

- `blocks(blocker, blocked)`

**5. Architecture:** `Relationship API` → `Authoritative outgoing edges` → `Change log` → `Incoming-edge projection` → `Count cache`

**6. Deep dives**

- Bucket inbound edges by follower hash and paginate across fixed buckets using an opaque cursor. Keep ordinary outgoing lists simple, and compute counts asynchronously from versioned edge events.

- Apply current block and visibility rules when returning results, even if graph projections are stale. Prioritize block invalidations and keep the blocked relationship authoritative independently of follower counts.

**7. Failures**

- Unfollow retries decrement counters even when no edge was removed.

- Make edge mutations conditional and emit a transition event only when state changes. Deduplicate projection events, reconcile counts from edge partitions, and clamp display only as a temporary presentation safeguard.

**8. Trade-offs**

- Duplicating inbound and outbound edges accelerates both directions but requires repairable projections and explicit count staleness.

**9. Interview questions**

- One account has 60M followers and its inbound row becomes unmanageable.

- Unfollow retries decrement counters even when no edge was removed.

- A blocked account appears in a mutual-friends result produced from lagging indexes.

- Design repair jobs that compare projection checkpoints and sample edge symmetry without loading the complete graph into memory.

**Staff-level view:** Design repair jobs that compare projection checkpoints and sample edge symmetry without loading the complete graph into memory.

**Reference design:** https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html

---

### Threaded comments

*Support nested discussion with moderation and stable pagination.*

**1. Requirements**

- Post and edit replies

- Sort by score or time

- Hide deleted content

**2. Scale:** 30M comments/day; one live event may receive 20K replies/s.

**3. API**

- `POST /posts/{id}/comments`

- `PATCH /comments/{id}`

- `GET /comments?parent=&cursor=`

**4. Data model**

- `comments(comment_id PK, root_id, parent_id, body, version, state)`

- `votes(user_id, comment_id, value)`

**5. Architecture:** `Comment API` → `Comment store` → `Moderation log` → `Thread index` → `Score projection` → `Thread cache`

**6. Deep dives**

- Shard comment ingestion by root and time/hash bucket, assign stable IDs, and merge bounded slices for reads. Cache the top-level page while separating live append traffic from score recomputation.

- Replace the parent body with a tombstone while retaining structural identifiers. Apply separate policies for subtree removal and author-content deletion; ensure descendants remain paginatable without exposing deleted text.

**7. Failures**

- An old body-edit request overwrites a newer hidden state.

- Use optimistic concurrency and field-specific updates; keep moderation state independently versioned and enforce it in rendering. Reject outdated edits and retain an audit trail of state transitions.

**8. Trade-offs**

- Materialized thread paths speed subtree reads but make moves expensive; restricting reparenting simplifies storage and moderation semantics.

**9. Interview questions**

- A match-final thread receives 20,000 comments per second.

- An old body-edit request overwrites a newer hidden state.

- A parent comment is deleted but hundreds of valid replies remain.

- Distinguish author deletion, moderator hiding, legal removal, and thread locking, since each has different visibility and audit requirements.

**Staff-level view:** Distinguish author deletion, moderator hiding, legal removal, and thread locking, since each has different visibility and audit requirements.

**Reference design:** https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html

---

### Ephemeral stories

*Publish short-lived media and track viewers without exposing expired stories.*

**1. Requirements**

- 24-hour visibility

- Audience rules

- View receipts

**2. Scale:** 15M stories/day; 150M daily story views; media expiry is strict at serving time.

**3. API**

- `POST /stories`

- `GET /stories?owner=`

- `PUT /stories/{id}/views/{viewer}`

**4. Data model**

- `stories(story_id PK, owner, expires_at, audience_version)`

- `views(story_id, viewer_id, first_seen_at)`

**5. Architecture:** `Story API` → `Story metadata` → `Media object store` → `Audience filter` → `CDN` → `View-event pipeline`

**6. Deep dives**

- Hash-bucket view receipts by viewer, maintain approximate or delayed public counts, and page exact lists only where the product requires them. Deduplicate with a stable story-viewer key.

- Version audience rules and revalidate when renewing access. Bound existing signed-token validity, invalidate where supported, and document whether already downloaded media can continue playing.

**7. Failures**

- A cleanup worker is late, leaving an expired object in storage and cache.

- Enforce expires_at when issuing playback access and use access lifetimes no longer than remaining story life. Purge asynchronously, but deny new serving authorization after expiry even when bytes still exist.

**8. Trade-offs**

- Exact receipts improve creator analytics but increase privacy and storage cost; delayed aggregates are cheaper for huge audiences.

**9. Interview questions**

- A public story accumulates 30M unique viewers in an hour.

- A cleanup worker is late, leaving an expired object in storage and cache.

- A creator switches from public to close friends while existing sessions are active.

- Separate the promised visibility deadline from physical deletion and backup expiration, and measure each boundary independently.

**Staff-level view:** Separate the promised visibility deadline from physical deletion and backup expiration, and measure each boundary independently.

**Reference design:** https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html

---

## Messaging

### Direct and group chat

*Deliver durable messages across online and offline devices.*

**1. Requirements**

- Conversation ordering

- Offline sync

- Delivery and read receipts

**2. Scale:** 50M daily users; 40 messages/user/day; groups up to 1,000 members.

**3. API**

- `POST /conversations/{id}/messages {clientMessageId,body}`

- `GET /sync?cursor=`

- `PUT /receipts`

**4. Data model**

- `messages(conversation_id, sequence, message_id, body)`

- `memberships(conversation_id,user_id,version)`

- `device_cursor(device_id,conversation_id,sequence)`

**5. Architecture:** `Client` → `WebSocket gateway` → `Conversation owner` → `Durable message log` → `Message store` → `Recipient fanout` → `Push fallback`

**6. Deep dives**

- Keep one sequencer per conversation but separate append from sharded recipient fanout. Batch deliveries, cap outstanding bytes per connection, and enforce a room traffic limit if one sequencer reaches its measured ceiling.

- Define whether membership is evaluated at send time or delivery time, record the relevant membership epoch, and check current eligibility for future delivery. Stop live fanout immediately and enforce the same rule in history sync.

**7. Failures**

- A gateway acknowledged receipt before durable storage and crashed.

- Acknowledge acceptance only after durable append; use clientMessageId for retry deduplication. On reconnect, fetch from the last durable sequence and deduplicate live deliveries against replay.

**8. Trade-offs**

- Per-conversation order is useful and affordable; a global order across all chats introduces coordination with little user benefit.

**9. Interview questions**

- A 1,000-member trading room generates 10,000 messages per second.

- A gateway acknowledged receipt before durable storage and crashed.

- Queued fanout includes a user whose membership was revoked.

- Specify device synchronization, end-to-end encryption metadata boundaries, abuse reporting, and multi-region ownership transfer without claiming that a WebSocket alone guarantees delivery.

**Staff-level view:** Specify device synchronization, end-to-end encryption metadata boundaries, abuse reporting, and multi-region ownership transfer without claiming that a WebSocket alone guarantees delivery.

**Reference design:** https://kafka.apache.org/design/

---

### Notification platform

*Route product events to email, push, SMS, and in-app inboxes.*

**1. Requirements**

- Preferences and quiet hours

- Per-channel retries

- Durable delivery status

**2. Scale:** 500M notification intents/day; provider quotas differ by region and channel.

**3. API**

- `POST /notifications {eventId,userId,templateId}`

- `PUT /preferences`

- `GET /notifications/{id}`

**4. Data model**

- `intents(event_id,user_id,template_version)`

- `deliveries(intent_id,channel,state,provider_key)`

- `preferences(user_id,version)`

**5. Architecture:** `Event producers` → `Intent log` → `Preference evaluator` → `Channel queues` → `Provider adapters` → `Callback processor`

**6. Deep dives**

- Reserve provider throughput for transactional traffic, use separate priority queues with starvation protection, and cap campaign admission. Autoscale workers only up to provider quotas and alert on oldest critical intent age.

- Recheck current marketing consent immediately before provider submission and record the evaluated preference version. Define a clear cutoff once submission has occurred and propagate suppression changes to all channels.

**7. Failures**

- A request timed out after the provider may have sent the SMS.

- Retry with the same provider idempotency key where supported and reconcile using delivery lookup or callbacks. Otherwise mark the attempt uncertain and apply a product-specific duplicate-versus-loss policy.

**8. Trade-offs**

- Provider failover improves availability but can duplicate ambiguous deliveries; status should preserve uncertainty instead of inventing certainty.

**9. Interview questions**

- A campaign backlog delays password reset messages by twenty minutes.

- A request timed out after the provider may have sent the SMS.

- The user disables promotional email after an intent was queued.

- Set channel-specific SLOs and per-tenant cost budgets, and rehearse a provider brownout with callbacks arriving after failover.

**Staff-level view:** Set channel-specific SLOs and per-tenant cost budgets, and rehearse a provider brownout with callbacks arriving after failover.

**Reference design:** https://kafka.apache.org/design/

---

### Email service

*Store mailboxes and reliably accept, route, and search messages.*

**1. Requirements**

- Durable acceptance

- Attachments

- Mailbox folders and search

**2. Scale:** 100M inbound messages/day; 80KB average body plus attachments; 1M active mailboxes.

**3. API**

- `POST /messages:send`

- `GET /mailboxes/{id}/messages?cursor=`

- `PATCH /messages/{id}/labels`

**4. Data model**

- `messages(message_id PK, envelope, blob_key)`

- `mailbox_entries(mailbox_id, uid, message_id, flags)`

- `delivery_attempts(message_id,destination)`

**5. Architecture:** `SMTP ingress` → `Durable spool` → `Spam scanning` → `Blob storage` → `Mailbox index` → `Search projection` → `Delivery workers`

**6. Deep dives**

- Enforce sender and mailbox byte quotas before acceptance, stream attachments to staged object storage, and apply bounded concurrency for scanners. Reject before durable acceptance when capacity is unavailable.

- Use version-checked or operation-based mailbox updates with stable UIDs. Separate independent flags from folder membership, return conflict for incompatible moves, and expose a change cursor for device reconciliation.

**7. Failures**

- The SMTP edge returns success before its local spool is replicated.

- Acknowledge only after the configured durable replicated write, then perform routing asynchronously. Recover from spool checkpoints and deduplicate delivery by message and recipient identifiers.

**8. Trade-offs**

- Scanning before acceptance reduces accepted malicious mail but lengthens SMTP latency; scanning after durable spooling requires quarantine and status handling.

**9. Interview questions**

- A few senders upload 100MB attachments and exhaust spool disk.

- The SMTP edge returns success before its local spool is replicated.

- Two devices move and mark a message read using stale mailbox state.

- Plan mailbox export, delivery-loop protection, per-recipient bounce state, and restore tests that verify both bodies and mailbox indexes.

**Staff-level view:** Plan mailbox export, delivery-loop protection, per-recipient bounce state, and restore tests that verify both bodies and mailbox indexes.

**Reference design:** https://kafka.apache.org/design/

---

### Webhook delivery

*Deliver signed events to customer endpoints with retry visibility.*

**1. Requirements**

- Durable events

- Endpoint-specific retries

- Replay and signing rotation

**2. Scale:** 200M deliveries/day; 100K endpoints; endpoints can be slow or unavailable.

**3. API**

- `POST /subscriptions`

- `GET /deliveries/{id}`

- `POST /events/{id}:replay`

**4. Data model**

- `events(event_id PK, tenant_id, payload_version)`

- `subscriptions(id,url,secret_version)`

- `attempts(event_id,subscription_id,number,status)`

**5. Architecture:** `Application outbox` → `Event log` → `Subscription router` → `Per-endpoint queues` → `HTTP workers` → `Retry scheduler`

**6. Deep dives**

- Use per-endpoint concurrency caps, separate tenant queues, bounded request deadlines, and a circuit breaker. Keep global workers available for healthy endpoints and give the customer clear backlog and replay controls.

- Include aggregate ID and monotonic resource version; receivers ignore superseded updates or fetch current state. Offer ordered per-resource delivery only with an explicit head-of-line blocking tradeoff.

**7. Failures**

- A receiver commits a database change and then its response path fails.

- Retry the same event ID with a new attempt ID, document at-least-once delivery, and provide signing over immutable event bytes plus timestamp. Receivers must deduplicate durable effects by event ID.

**8. Trade-offs**

- Strict delivery order simplifies receivers but lets one poison event block a stream; versioned events permit parallel recovery.

**9. Interview questions**

- An enterprise endpoint takes 25 seconds per request and blocks unrelated tenants.

- A receiver commits a database change and then its response path fails.

- The delete event is delivered before a delayed create event.

- Make replay preserve original event identity while recording a distinct replay request, and bound customer endpoint access to prevent internal-network targeting.

**Staff-level view:** Make replay preserve original event identity while recording a distinct replay request, and bound customer endpoint access to prevent internal-network targeting.

**Reference design:** https://kafka.apache.org/design/

---

### Publish-subscribe event bus

*Distribute durable events to independently checkpointed consumers.*

**1. Requirements**

- Partition ordering

- Replay retention

- Backpressure and tenant quotas

**2. Scale:** 1B events/day; 1KB mean event; 20 consumer groups.

**3. API**

- `POST /topics/{id}/events`

- `GET /topics/{id}/partitions/{p}?offset=`

- `PUT /groups/{id}/checkpoints`

**4. Data model**

- `log(topic,partition,offset,key,payload)`

- `group_offsets(group,topic,partition,offset)`

- `schema(subject,version)`

**5. Architecture:** `Producers` → `Partition routers` → `Replicated log brokers` → `Consumer groups` → `Sink adapters` → `Checkpoint store`

**6. Deep dives**

- Model aggregate broker egress and catch-up reads, batch and compress events, isolate heavy replays, and tier old data. Add partitions only after checking key-order implications and consumer parallelism.

- Use a durable event-ID uniqueness record in the same transaction as the sink mutation, then advance the broker checkpoint. Replays become no-ops; Kafka transactions alone do not make arbitrary database writes exactly once.

**7. Failures**

- A broken analytics consumer resumes after its oldest required offset was deleted.

- Detect offset-out-of-range explicitly, restore a consistent snapshot, and replay from its associated checkpoint. Do not silently skip to newest; publish data-loss or rebuild status to downstream owners.

**8. Trade-offs**

- Long retention enables recovery but increases storage and replay load; ordering by one hot key limits available parallelism.

**9. Interview questions**

- Twenty consumer groups each read the full billion-event stream.

- A broken analytics consumer resumes after its oldest required offset was deleted.

- The consumer writes to a database then crashes before committing its offset.

- Budget replay traffic separately, govern schema compatibility, and provide tenant-level kill switches that preserve other streams during poison-event incidents.

**Staff-level view:** Budget replay traffic separately, govern schema compatibility, and provide tenant-level kill switches that preserve other streams during poison-event incidents.

**Reference design:** https://kafka.apache.org/design/

---

## Media

### YouTube-style video platform

*Accept large video uploads, process adaptive renditions, and serve globally cached playback.*

**1. Requirements**

- Resumable multipart uploads

- HLS and DASH adaptive playback

- Search, subscriptions, and watch history

- Publish only validated playable assets

**2. Scale:** 20M daily viewers watch 30 minutes each at an assumed 3 Mb/s average: 13.5 PB/day of delivered video; 100K uploads/day averaging 600MB = 60TB/day of originals before renditions.

**3. API**

- `POST /videos/uploads {size,checksum,idempotencyKey} -> uploadId and signed part URLs`

- `POST /videos/{id}:complete {parts,checksum} -> processing`

- `GET /videos/{id}/status`

- `GET /videos/{id}/playback -> authorized manifest URL`

- `POST /watch-events {eventId,videoId,position}`

**4. Data model**

- `videos(video_id PK, owner, state, source_version, visibility_version)`

- `uploads(upload_id PK, video_id, parts, checksum, expires_at)`

- `jobs(video_id, source_version, recipe_version, rendition, state, lease_epoch)`

- `assets(video_id, recipe_version, rendition, segment, object_key, checksum)`

- `publications(video_id PK, manifest_version, published_at)`

**5. Architecture:** `Client multipart uploader` → `Signed upload API` → `Staging object storage` → `Completion validator` → `Durable workflow and queues` → `Probe and moderation stages` → `Transcode workers by codec and resolution` → `HLS/DASH packager` → `Asset validator` → `Atomic publication metadata` → `Origin shield and CDN` → `Adaptive player` → `QoE telemetry and watch-event pipeline`

**6. Deep dives**

- Use immutable versioned segment URLs, an origin shield, request collapsing, and prewarm initial segments for predicted hot videos. Keep signed authorization from fragmenting cache keys. Measure origin bytes, startup time, rebuffer ratio, and per-rendition cache hit rate; reduce initial bitrate during overload.

- Key each job by videoId, sourceVersion, recipeVersion, and rendition. Write attempt outputs to staging, verify checksums, and use a fenced compare-and-set to commit one successful asset set. Losing attempts are harmless and garbage-collected. A durable workflow records stage completion; at-least-once queue delivery is expected.

- Upload lifecycle: allocate a video ID and upload session before issuing narrowly scoped signed part URLs. The client uploads directly to staging storage with per-part retries. Completion verifies expected parts, whole-object checksum, owner, size limit, and upload expiry; a conditional transition from uploading to uploaded emits one durable workflow event. Incomplete multipart sessions are cleaned up after a grace period.

- Processing DAG: probe source codecs, dimensions, frame rate, and duration; validate and scan; generate thumbnails, captions where supported, and a content-appropriate bitrate ladder; encode resolution/codec variants in parallel; package and validate; publish. Keep moderation outcomes separate from technical readiness. Prioritize a baseline playable rendition before optional costly formats, and expose processing progress without promising a fixed encode time.

- HLS uses multivariant and media playlists; DASH uses an MPD to describe representations and segment timelines. Select packaging and codecs for target devices. Align keyframes and segment boundaries across adaptive renditions so a player can switch safely. Use immutable media segments and versioned VOD manifests, with cache policies different from frequently changing live manifests. See Apple HLS and DASH-IF guidance.

- Worker recovery: assume at-least-once delivery. A job key includes video ID, source version, recipe version, and rendition. Workers acquire bounded leases, heartbeat long encodes, write attempt-specific staging outputs, and atomically commit a validated asset set under a fencing epoch. Persist stage results, classify retryable versus corrupt-input errors, apply retry budgets, and send exhausted jobs to an inspectable dead-letter workflow.

- Playback control and media delivery: the API checks visibility, geography, and entitlement, then issues access to a manifest. The player estimates throughput and buffer occupancy to choose an adaptive rendition. CDN edges serve segments from immutable keys; an origin shield coalesces misses. Keep authorization tokens out of the cache identity only when the CDN still validates access correctly. QoE telemetry records startup delay, rebuffer time, chosen bitrate, and playback errors.

- Capacity walkthrough: 20M viewers x 1,800 seconds/day x 3,000,000 bits/second / 8 = 13.5PB/day (decimal) and 1.25Tb/s average delivered bandwidth. A 4x peak is 5Tb/s. A 97% byte-hit ratio would leave about 405TB/day of origin bytes before replication and retries. 100K x 600MB uploads = 60TB/day originals; add the measured sum of rendition sizes and retention, rather than assuming one universal storage multiplier.

- Publication and removal: validate that all manifest-referenced assets exist before conditionally promoting the visible publication version. Readers pin a complete version. Removal first blocks new authorization, then updates discovery and purges serving caches and derivative assets under a tracked workflow. Object deletion, backup expiry, and already-downloaded bytes have distinct boundaries.

**7. Failures**

- A packager updates the published manifest before all rendition segments are durable and validated.

- Write outputs under immutable video/source/recipe/rendition keys. Validate checksums, codec metadata, aligned segment timelines, and required assets, then conditionally promote a publication pointer. Keep the previous complete version available. For live playback, publish only segments that meet the availability contract and distinguish manifest TTL from immutable segment TTL.

**8. Trade-offs**

- More codecs and renditions improve device coverage and bandwidth efficiency but increase encoding time and storage; eager encoding helps popular uploads while demand-driven renditions reduce waste.

**9. Interview questions**

- A worker finishes a 4K encode after its lease was reassigned. Which exact keys and conditional transition prevent stale publication?

- Show the upload-to-publish state machine and identify when you acknowledge upload completion versus playback readiness.

- Estimate CDN and origin bandwidth for 20M viewers, 30 minutes/day, 3Mb/s, a 4x peak, and a 97% byte-hit ratio. Which assumptions break for a viral cold video?

- Explain how HLS/DASH manifests, aligned rendition segments, adaptive bitrate selection, and CDN cache keys interact during version rollout and takedown.

**Staff-level view:** Separate upload, processing, playback-start, rebuffering, and deletion SLOs. Compare regional encoding cost, CDN egress, and origin shielding; plan takedown propagation through authorization, manifests, caches, search, and derived clips.

**Reference design:** https://developer.apple.com/streaming/ | https://dashif.org/guidelines/iop-v5/ | https://kafka.apache.org/design/ | https://aws.amazon.com/s3/consistency/

---

### Live streaming platform

*Ingest live broadcasts and deliver adaptive playback with a bounded live delay.*

**1. Requirements**

- Authenticated ingest

- Live manifest updates

- DVR window and recording

**2. Scale:** 10K concurrent broadcasters; 5M peak viewers; target glass-to-glass latency 5 seconds.

**3. API**

- `POST /streams`

- `POST /streams/{id}:start`

- `GET /streams/{id}/playback`

- `GET /streams/{id}/health`

**4. Data model**

- `streams(stream_id PK, ingest_region, state, epoch)`

- `segments(stream_id, epoch, sequence, duration, key)`

- `manifests(stream_id,version)`

**5. Architecture:** `Broadcaster` → `Regional ingest` → `Transcoder` → `Live packager` → `Segment origin` → `CDN` → `Player` → `Recording assembler`

**6. Deep dives**

- Provision CDN delivery headroom, shield segment origin, collapse requests, and keep a carefully bounded manifest refresh cadence. Use player bitrate adaptation and cohort-based failover rather than redirecting every viewer simultaneously.

- Allocate a monotonic stream epoch through an authoritative coordinator and permit publication only for the current epoch. Old workers may finish objects but cannot advance the visible manifest.

**7. Failures**

- An encoder reconnects with a new timeline and existing players stall.

- Start a new ingest epoch, emit protocol-appropriate discontinuity or period boundaries, and preserve coherent sequence and timing metadata. Validate with players across codecs; keep DVR references tied to immutable segment identities.

**8. Trade-offs**

- Smaller segments reduce live delay but increase request overhead and sensitivity to jitter; longer buffers improve resilience while moving viewers further behind live.

**9. Interview questions**

- A match audience grows from 200K to 5M viewers in ninety seconds.

- An encoder reconnects with a new timeline and existing players stall.

- A broadcaster reconnects to another region while the old ingest keeps publishing.

- Treat live latency, playback availability, recording completeness, and rights enforcement as independent objectives; test failover mid-segment and mid-ad-break.

**Staff-level view:** Treat live latency, playback availability, recording completeness, and rights enforcement as independent objectives; test failover mid-segment and mid-ad-break.

**Reference design:** https://developer.apple.com/streaming/

---

### Music streaming

*Deliver licensed audio, maintain libraries, and synchronize playback state.*

**1. Requirements**

- Low-latency track start

- Playlist edits

- Territory and subscription rules

**2. Scale:** 20M daily listeners; 40 tracks/day; 4MB average delivered audio per track.

**3. API**

- `GET /tracks/{id}/playback`

- `PATCH /playlists/{id} {baseVersion,operations}`

- `PUT /sessions/{id}/position`

**4. Data model**

- `tracks(track_id PK, asset_version, rights_version)`

- `playlist_entries(playlist_id,position_id,track_id)`

- `sessions(user_id,device_id,position,version)`

**5. Architecture:** `Client` → `Entitlement API` → `Catalog metadata` → `Audio CDN` → `Playlist service` → `Playback-event log`

**6. Deep dives**

- Prewarm encrypted or access-controlled audio assets and cache catalog metadata separately from entitlement decisions. Stagger telemetry uploads and scale authorization by active sessions, not audio segment count.

- Use versioned edit operations on stable playlist entry IDs and return conflicts for ambiguous moves. Rebase independent additions and deletions; reserve a CRDT only if offline concurrent editing is a real requirement.

**7. Failures**

- A track loses rights in a territory while long-lived playback tokens remain valid.

- Issue short-lived playback authorization tied to rights version and region, recheck on renewal, and invalidate urgent removals. Keep immutable audio caching efficient while applying access controls at the serving boundary.

**8. Trade-offs**

- Frequent entitlement checks improve revocation latency but add startup latency and dependency load; token lifetime sets a measurable compromise.

**9. Interview questions**

- Ten million listeners start the same album immediately at release.

- A track loses rights in a territory while long-lived playback tokens remain valid.

- A phone adds tracks while a laptop reorders an old playlist snapshot.

- Maintain an auditable rights-decision version and reconcile royalty events by stable playback IDs without counting network retries as extra listens.

**Staff-level view:** Maintain an auditable rights-decision version and reconcile royalty events by stable playback IDs without counting network retries as extra listens.

**Reference design:** https://developer.apple.com/streaming/

---

### Video conferencing

*Route interactive audio and video between participants with adaptive quality.*

**1. Requirements**

- Join and leave rooms

- Screen share

- Recover from packet loss and reconnects

**2. Scale:** 500K concurrent users; average room size 6; target interactive latency below 300 ms.

**3. API**

- `POST /rooms`

- `POST /rooms/{id}/tokens`

- `WebRTC signaling offer/answer`

- `POST /rooms/{id}:record`

**4. Data model**

- `rooms(room_id PK, region, owner_epoch)`

- `participants(room_id,user_id,session_id,role)`

- `recordings(recording_id,state,object_key)`

**5. Architecture:** `Browser` → `Signaling service` → `Room allocator` → `Regional SFU` → `STUN/TURN support` → `Optional recording workers`

**6. Deep dives**

- Limit active senders, forward selected simulcast or scalable-video layers per subscriber, and move passive viewers to broadcast delivery when acceptable. Model SFU egress by subscribed tracks rather than participants squared by default.

- Bind credentials to room and participant session epochs, check revocation at join, and remove existing SFU subscriptions on kick. Set a short token lifetime and reauthorize reconnects.

**7. Failures**

- Signaling succeeds but participants see no media.

- Use ICE connectivity checks with provisioned TURN fallback, track relay selection and connection setup stages, and reduce bitrate on constrained paths. Show connection recovery status rather than retrying room creation.

**8. Trade-offs**

- An SFU saves endpoint uplink compared with full mesh but costs server egress; mixing simplifies clients at a substantial compute and latency cost.

**9. Interview questions**

- A 2,000-person meeting sends every camera to every participant.

- Signaling succeeds but participants see no media.

- A previously valid join credential is reused after host removal.

- Partition failure domains by room, quantify regional evacuation capacity, and make recording consent and membership policy explicit without putting media through the signaling service.

**Staff-level view:** Partition failure domains by room, quantify regional evacuation capacity, and make recording consent and membership policy explicit without putting media through the signaling service.

**Reference design:** https://developer.apple.com/streaming/

---

### Podcast platform

*Publish episodes from creator feeds and deliver resumable audio playback.*

**1. Requirements**

- Feed ingestion

- Episode deduplication

- Progress sync and downloads

**2. Scale:** 2M feeds; 200K updated episodes/day; 10M listeners.

**3. API**

- `POST /feeds`

- `GET /shows/{id}/episodes`

- `PUT /progress/{episodeId} {position,sessionVersion}`

**4. Data model**

- `feeds(feed_id PK, url, etag, next_poll_at)`

- `episodes(show_id,guid,asset_version)`

- `progress(user_id,episode_id,position,session_version)`

**5. Architecture:** `Feed scheduler` → `Conditional HTTP fetchers` → `Parser and validator` → `Episode catalog` → `Audio origin/CDN` → `Progress service`

**6. Deep dives**

- Use adaptive polling based on recent publication intervals, conditional requests with validators, and a minimum/maximum poll cadence. Allocate host-level concurrency budgets and prioritize recently active followed shows.

- Use session versions and explicit seek events, reject stale session writes, and let users choose which active session controls progress. Do not blindly take the maximum or last client wall-clock timestamp.

**7. Failures**

- A feed replaces audio under an existing identifier and users download mixed versions.

- Track GUID plus observed enclosure metadata and content version, validate changed media, and publish a new asset version deliberately. Keep existing range downloads pinned to their original immutable object.

**8. Trade-offs**

- Aggressive polling improves freshness for active shows but wastes network and publisher capacity; adaptive schedules require starvation bounds for quiet feeds.

**9. Interview questions**

- Two million feeds are polled every minute although most publish weekly.

- A feed replaces audio under an existing identifier and users download mixed versions.

- An old offline device overwrites a newer position with a smaller value.

- Isolate hostile or malformed feeds with byte, recursion, redirect, and fetch-time limits, and build explainable ingestion status for creators.

**Staff-level view:** Isolate hostile or malformed feeds with byte, recursion, redirect, and fetch-time limits, and build explainable ingestion status for creators.

**Reference design:** https://developer.apple.com/streaming/

---

## Storage

### Cloud file synchronization

*Synchronize file versions and folder changes across devices.*

**1. Requirements**

- Resumable uploads

- Offline edits

- Recoverable version history

**2. Scale:** 10M active users; 100M file mutations/day; typical file 2MB.

**3. API**

- `POST /files/{id}/uploads`

- `PATCH /files/{id} {baseVersion,manifest}`

- `GET /changes?cursor=`

**4. Data model**

- `files(file_id PK,parent_id,name,current_version)`

- `versions(file_id,version,chunk_manifest)`

- `changes(account_id,sequence,operation)`

**5. Architecture:** `Desktop client` → `Metadata API` → `Chunk object store` → `Version transaction` → `Change journal` → `Sync notifications`

**6. Deep dives**

- Provide paginated batched manifests, compressed metadata snapshots, and chunked directory traversal. Capture a consistent snapshot cursor then apply subsequent changes; limit parallel downloads per client and preserve resumability.

- Require baseVersion on commit; accept one successor and store the other as an explicit conflict version or conflict file. Use format-aware merge only for supported document types and keep recoverable history.

**7. Failures**

- The metadata pointer changes while some chunks are still missing.

- Stage chunks and verify their identities, then atomically commit a version manifest and current pointer. Readers see only complete versions; a garbage collector removes unreferenced staged chunks after a grace period.

**8. Trade-offs**

- Content-defined chunking reduces upload bytes for edits but adds client CPU, indexing complexity, and encryption/deduplication tradeoffs.

**9. Interview questions**

- Initial synchronization performs one request per file and takes days.

- The metadata pointer changes while some chunks are still missing.

- Both devices upload valid content based on version 12.

- Define rename atomicity across folders, deletion recovery, shared-folder permission revocation, and safe garbage collection in the presence of offline devices.

**Staff-level view:** Define rename atomicity across folders, deletion recovery, shared-folder permission revocation, and safe garbage collection in the presence of offline devices.

**Reference design:** https://aws.amazon.com/s3/consistency/

---

### Distributed object storage

*Store immutable or versioned blobs with replication and integrity checks.*

**1. Requirements**

- PUT/GET/range reads

- Checksums and versions

- Repair after disk loss

**2. Scale:** 10PB logical data; 50M objects; 20GB/s peak read traffic.

**3. API**

- `PUT /buckets/{b}/objects/{key}`

- `GET /objects/{key}?version=`

- `POST /multipart:complete`

**4. Data model**

- `object_index(bucket,key,version,manifest)`

- `fragments(object_id,stripe,location,checksum)`

- `repairs(fragment_id,state)`

**5. Architecture:** `Object gateway` → `Metadata quorum` → `Placement service` → `Storage nodes` → `Integrity scrubbers` → `Repair workers`

**6. Deep dives**

- Reserve repair bandwidth, prioritize under-redundant stripes, and throttle background scrubbing. Model correlated failure domains and rebuild duration; placement should spread fragments across racks before optimizing balance.

- Write versioned immutable fragments and atomically replace the key-to-manifest pointer under a defined write ordering. Range reads pin a version; conditional writes prevent unintended overwrite races.

**7. Failures**

- One disk returns wrong bytes without a read error.

- Verify end-to-end object or fragment checksums on write and read, retrieve a healthy replica or reconstruct from parity, and quarantine the bad copy. Schedule scrubbing so cold corruption is found before another failure.

**8. Trade-offs**

- Erasure coding lowers storage overhead but increases repair and small-write complexity; replication is simpler and often better for hot small objects.

**9. Interview questions**

- Losing a rack requires reconstructing petabytes while clients continue reading.

- One disk returns wrong bytes without a read error.

- Two uploads target the same key and a reader assembles pieces from both.

- State consistency separately for object writes, listings, multipart completion, and cross-region replication. Do not assume all object stores or regional copies share identical guarantees.

**Staff-level view:** State consistency separately for object writes, listings, multipart completion, and cross-region replication. Do not assume all object stores or regional copies share identical guarantees.

**Reference design:** https://aws.amazon.com/s3/consistency/

---

### Backup and point-in-time restore

*Create recoverable snapshots and continuous change archives.*

**1. Requirements**

- Point-in-time restore

- Encryption and retention

- Verified recovery

**2. Scale:** 500TB source data; 2TB changes/day; target RPO 5 minutes and RTO 4 hours.

**3. API**

- `POST /backups`

- `POST /restores {snapshotId,targetTime}`

- `GET /restores/{id}`

**4. Data model**

- `snapshots(snapshot_id PK, base_checkpoint, manifest, state)`

- `log_segments(start,end,checksum)`

- `restore_jobs(id,target,verified_at)`

**5. Architecture:** `Database snapshotter` → `Change-log archiver` → `Immutable backup storage` → `Catalog` → `Restore coordinator` → `Verification runner`

**6. Deep dives**

- Calculate transfer and replay lower bounds, parallelize across storage partitions, and keep warm or incremental recovery capacity if needed. Reduce the restored working set only with an explicit degraded-service contract.

- Track snapshot and log dependencies, acquire durable restore pins, and garbage-collect only unreferenced expired chains after a safety window. Use generation-based deletion with a final reference recheck.

**7. Failures**

- Snapshots completed successfully, yet a gap prevents replay to the requested time.

- Validate checkpoint-to-log continuity continuously, mark the recoverable time range precisely, and restore to the last verified point if a gap cannot be repaired. Run automated restore drills with application-level checks.

**8. Trade-offs**

- Longer retention and immutable backups improve recovery options but increase cost; low RTO usually needs preprovisioned capacity rather than a cheaper archive alone.

**9. Interview questions**

- A 500TB snapshot must be restored over a 10Gb/s link.

- Snapshots completed successfully, yet a gap prevents replay to the requested time.

- A cleanup task deletes an incremental base while a restore is reading it.

- Test compromised credentials, key loss, and region loss as recovery scenarios. A backup success dashboard must show verified recovery points and measured restore time.

**Staff-level view:** Test compromised credentials, key loss, and region loss as recovery scenarios. A backup success dashboard must show verified recovery points and measured restore time.

**Reference design:** https://aws.amazon.com/s3/consistency/

---

### Content-addressed blob store

*Deduplicate immutable content while preserving tenant isolation and safe collection.*

**1. Requirements**

- Hash-verified uploads

- Reference tracking

- Safe garbage collection

**2. Scale:** 3PB logical content; expected 40% duplicate bytes; 100M references/day.

**3. API**

- `POST /blobs {digest,size}`

- `PUT /blobs/{digest}/content`

- `POST /references`

- `DELETE /references/{id}`

**4. Data model**

- `blobs(digest PK,size,verified,state)`

- `references(tenant,reference_id,digest)`

- `gc_candidates(digest,generation)`

**5. Architecture:** `Upload API` → `Tenant authorization` → `Digest verifier` → `Blob object storage` → `Reference database` → `Mark-and-sweep collector`

**6. Deep dives**

- Shard reference records and compute liveness through indexed existence or periodic marking instead of a single synchronous global counter. Batch accounting and partition work by digest prefix while keeping reference identity unique.

- Require proof of content possession or scope deduplication by tenant, and authorize references independently of the digest. Avoid existence endpoints that reveal cross-tenant content and assess encryption requirements before global dedupe.

**7. Failures**

- An unreferenced blob is selected for deletion just before a new tenant links it.

- Mark candidates, delay physical removal, and atomically recheck the reference generation before final deletion. New references either cancel deletion or require a verified re-upload; never trust a stale zero count.

**8. Trade-offs**

- Global deduplication saves more storage but complicates isolation, encryption, accounting, and deletion guarantees compared with tenant-scoped deduplication.

**9. Interview questions**

- A common dependency receives millions of reference additions per minute.

- An unreferenced blob is selected for deletion just before a new tenant links it.

- A tenant can probe whether another tenant uploaded a known file.

- Prove collection safety under concurrent references, delayed replicas, interrupted sweeps, and hash-verification failures; measure storage saved after accounting for metadata and replication.

**Staff-level view:** Prove collection safety under concurrent references, delayed replicas, interrupted sweeps, and hash-verification failures; measure storage saved after accounting for metadata and replication.

**Reference design:** https://aws.amazon.com/s3/consistency/

---

### Distributed filesystem

*Expose a hierarchical namespace over replicated file chunks.*

**1. Requirements**

- Atomic namespace operations

- Chunk replication

- Client cache coherence

**2. Scale:** 1B files; 5PB data; 100K concurrent clients; most operations are metadata reads.

**3. API**

- `OPEN /path`

- `READ /file/{id}?offset=&length=`

- `RENAME /path {destination}`

- `APPEND /file/{id}`

**4. Data model**

- `inodes(inode_id PK,parent,name,version)`

- `chunk_map(inode_id,index,chunk_id)`

- `leases(inode_id,client,epoch)`

**5. Architecture:** `Filesystem client` → `Namespace service` → `Metadata consensus group` → `Chunk placement` → `Data nodes` → `Lease and repair services`

**6. Deep dives**

- Measure inode and directory-index overhead, partition namespace metadata by stable subtree or inode ranges, and batch client metadata operations. Define the behavior of cross-partition rename before distributing the namespace.

- Use a recoverable cross-shard transaction or durable rename intent with an authoritative namespace state machine. Alternatively restrict atomic rename to one namespace partition and expose the restriction clearly.

**7. Failures**

- A paused writer resumes after a new client became the file owner.

- Attach a monotonic fencing epoch to writes and have data nodes reject epochs older than the current file lease. Recover or truncate uncommitted tails according to the documented append contract.

**8. Trade-offs**

- Fine-grained metadata sharding increases capacity but makes atomic rename and directory listings harder; central metadata is simpler until measured limits require partitioning.

**9. Interview questions**

- A billion tiny files fit on disks but exhaust metadata servers.

- A paused writer resumes after a new client became the file owner.

- Moving a directory between shards must not expose it twice or lose it.

- Write down the exact POSIX subset, append semantics, client-cache invalidation rules, and behavior during metadata quorum loss before drawing data-node boxes.

**Staff-level view:** Write down the exact POSIX subset, append semantics, client-cache invalidation rules, and behavior during metadata quorum loss before drawing data-node boxes.

**Reference design:** https://aws.amazon.com/s3/consistency/

---

## Commerce

### Commerce checkout

*Convert a cart into an order with payment and inventory coordination.*

**1. Requirements**

- Stable checkout retries

- Inventory reservation

- Payment reconciliation

**2. Scale:** 5M orders/day; flash-sale peak 20K checkout attempts/s.

**3. API**

- `POST /checkouts {cartVersion,idempotencyKey}`

- `GET /orders/{id}`

- `POST /orders/{id}:cancel`

**4. Data model**

- `orders(order_id PK,state,amount,currency)`

- `reservations(order_id,sku,quantity,expires_at)`

- `payment_attempts(order_id,key,provider_state)`

**5. Architecture:** `Checkout API` → `Order transaction and outbox` → `Inventory reservation` → `Payment adapter` → `Fulfillment queue` → `Reconciliation worker`

**6. Deep dives**

- Admit attempts through a bounded SKU queue, reserve stock atomically, and cap per-user attempts. Keep unrelated products on independent partitions and return sold-out promptly once allocatable stock is exhausted.

- Define a deadline and durable state machine: either extend a valid hold before capture, or compensate a late capture through refund when stock cannot be reacquired. Serialize state transitions and make refund requests idempotent.

**7. Failures**

- The provider captured funds while the checkout process crashed before recording success.

- Persist a payment attempt with a stable provider idempotency key before calling the provider. Reconcile callbacks and provider lookup into a monotonic order state machine; never create a fresh charge merely because the client retried.

**8. Trade-offs**

- Long inventory holds improve checkout completion but reduce availability to other buyers; short holds require well-defined late-payment compensation.

**9. Interview questions**

- Most checkouts target one scarce SKU while the rest of the store is healthy.

- The provider captured funds while the checkout process crashed before recording success.

- The inventory hold expires while a delayed payment success arrives.

- Measure unknown payments, reservation age, refund backlog, and reconciliation lag alongside checkout success. Write operational playbooks that preserve evidence for ambiguous money movement.

**Staff-level view:** Measure unknown payments, reservation age, refund backlog, and reconciliation lag alongside checkout success. Write operational playbooks that preserve evidence for ambiguous money movement.

**Reference design:** https://www.postgresql.org/docs/18/transaction-iso.html

---

### Inventory reservation

*Maintain available stock and bounded holds across warehouses.*

**1. Requirements**

- Prevent overselling

- Expiring reservations

- Warehouse allocation

**2. Scale:** 20M SKUs; 50M stock mutations/day; a hot SKU may receive 50K attempts/s.

**3. API**

- `POST /reservations {sku,quantity,orderId}`

- `POST /reservations/{id}:commit`

- `POST /reservations/{id}:release`

**4. Data model**

- `stock(warehouse,sku,on_hand,reserved,version)`

- `reservations(id,order_id,sku,qty,state,expires_at)`

- `stock_ledger(event_id,delta)`

**5. Architecture:** `Reservation API` → `SKU owner` → `Stock database` → `Reservation expiry queue` → `Warehouse events` → `Reconciliation`

**6. Deep dives**

- Allocate bounded stock quotas to independent reservation buckets or regions and replenish through a single authority. Preserve the invariant that allocated quotas plus central stock never exceed real stock; reject when a local quota is exhausted.

- Record a unique warehouse event ID with the stock ledger mutation and reject repeated application. Reconcile physical counts through explicit adjustment entries rather than overwriting balances silently.

**7. Failures**

- A hold is committed while an expiry worker also releases it.

- Use a transactional compare-and-set from held to committed or expired, with stock counters updated in that same transaction. Exactly one transition wins; retries return the recorded result.

**8. Trade-offs**

- Regional quota allocation reduces contention and latency but can strand stock in idle regions; rebalancing must preserve ownership and prevent double allocation.

**9. Interview questions**

- Thousands of buyers contend for the last thousand units.

- A hold is committed while an expiry worker also releases it.

- A duplicated shipment event deducts stock twice.

- Define invariants for damaged goods, returns, transfers in flight, backorders, and manual adjustments instead of treating stock as one mutable integer.

**Staff-level view:** Define invariants for damaged goods, returns, transfers in flight, backorders, and manual adjustments instead of treating stock as one mutable integer.

**Reference design:** https://www.postgresql.org/docs/18/transaction-iso.html

---

### Payment ledger

*Record balanced financial entries with immutable history and reconciled external transfers.*

**1. Requirements**

- Balanced double-entry transactions

- Idempotent posting

- Auditable reversals

**2. Scale:** 10M ledger transactions/day; 7-year business retention assumption; exact integer minor units.

**3. API**

- `POST /ledger/transactions {key,entries}`

- `GET /accounts/{id}/balance`

- `POST /ledger/transactions/{id}:reverse`

**4. Data model**

- `transactions(tx_id PK,idempotency_key UNIQUE,state)`

- `entries(tx_id,account_id,currency,amount_minor)`

- `account_balances(account_id,currency,version)`

**5. Architecture:** `Ledger API` → `Validation` → `Transactional ledger store` → `Outbox` → `Balance projections` → `Provider reconciliation`

**6. Deep dives**

- Append balanced entries transactionally without forcing all traffic through one synchronously updated aggregate row. Partition settlement subaccounts and reconcile to a parent total; retain strict checks where available-funds authorization requires them.

- Represent amounts as integer minor units with currency metadata, require zero-sum entries within each currency, and record FX legs, rates, and rounding accounts explicitly. Reverse through new linked entries instead of editing history.

**7. Failures**

- A client retries after a lost response and creates duplicate debit-credit pairs.

- Use a durable unique idempotency key and compare a canonical payload hash on reuse. Return the original transaction for identical retries and reject mismatched payloads; keep the dedupe window aligned with actual retry and replay behavior.

**8. Trade-offs**

- Synchronous balance projections simplify immediate reads but add contention; asynchronous projections need freshness markers and cannot authorize overspend alone.

**9. Interview questions**

- Every merchant payment updates the same platform settlement balance.

- A client retries after a lost response and creates duplicate debit-credit pairs.

- A multi-currency operation sums decimal approximations across entries.

- Separate ledger truth, available balance, provider settlement, and bank reconciliation. Treat recovery, access control, and immutable audit evidence as design requirements rather than afterthoughts.

**Staff-level view:** Separate ledger truth, available balance, provider settlement, and bank reconciliation. Treat recovery, access control, and immutable audit evidence as design requirements rather than afterthoughts.

**Reference design:** https://www.postgresql.org/docs/18/transaction-iso.html

---

### Ticket booking

*Sell assigned seats with temporary holds and durable booking outcomes.*

**1. Requirements**

- One owner per seat

- Hold expiry

- Fair admission under bursts

**2. Scale:** 100K seats for one event; 3M buyers arrive in a minute.

**3. API**

- `POST /events/{id}/holds {seats,key}`

- `POST /holds/{id}:confirm`

- `GET /events/{id}/seats`

**4. Data model**

- `seats(event_id,seat_id,state,hold_id,version)`

- `holds(hold_id,user_id,expires_at,state)`

- `bookings(booking_id,hold_id,payment_id)`

**5. Architecture:** `Waiting room` → `Seat map cache` → `Hold service` → `Transactional seat store` → `Payment workflow` → `Booking confirmation`

**6. Deep dives**

- Serve static geometry via CDN and bounded availability deltas via polling or push cohorts. Rate-limit refresh, cache approximate availability briefly, and validate exact seat ownership only inside the hold transaction.

- Reserve the full seat set in one transaction using a deterministic lock order, or return failure without partial holds. If multi-shard seat groups are unavoidable, define an explicit short-lived coordination protocol and cleanup.

**7. Failures**

- An old hold-expiry task runs after payment confirmation.

- Condition expiry on matching hold ID, held state, and expected version. Confirm and expire compete atomically; late expiry becomes a no-op and cannot free a sold seat.

**8. Trade-offs**

- Strict queue fairness can reduce throughput; opportunistic admission improves utilization but must not undermine the advertised purchase policy.

**9. Interview questions**

- Three million waiting users poll the full seat map every second.

- An old hold-expiry task runs after payment confirmation.

- Each group obtains some seats and neither can complete the requested block.

- Model bot resistance, waiting-room token replay, queue position continuity during failover, and the user-visible policy for ambiguous payment completion.

**Staff-level view:** Model bot resistance, waiting-room token replay, queue position continuity during failover, and the user-visible policy for ambiguous payment completion.

**Reference design:** https://www.postgresql.org/docs/18/transaction-iso.html

---

### Marketplace orders

*Coordinate multi-seller orders, shipments, returns, and seller payouts.*

**1. Requirements**

- Split fulfillment

- Partial cancellation

- Auditable payout eligibility

**2. Scale:** 2M orders/day; average 3 sellers/order; fulfillment can take weeks.

**3. API**

- `POST /orders`

- `POST /order-items/{id}:cancel`

- `POST /shipments`

- `GET /seller-payouts`

**4. Data model**

- `orders(id,buyer,total)`

- `order_items(id,seller,state,amount)`

- `shipments(id,item_id,status_version)`

- `payout_items(item_id,state)`

**5. Architecture:** `Order API` → `Order database` → `Saga coordinator` → `Seller fulfillment adapters` → `Shipment events` → `Returns service` → `Payout ledger`

**6. Deep dives**

- Run per-seller or per-item fulfillment state machines and aggregate order status. Isolate adapter queues and show partial progress; keep discounts, shipping allocation, and refunds tied to stable item-level amounts.

- Use a transactional payout-eligibility record linked to item and return state. Freeze eligible amounts before payout submission and compensate late returns with explicit seller receivables rather than editing settled ledger entries.

**7. Failures**

- A delayed in-transit callback arrives after delivered and triggers another payout hold.

- Deduplicate carrier events, validate allowed state transitions, and preserve raw evidence. Ignore superseded updates while escalating genuine corrections through a separate correction workflow.

**8. Trade-offs**

- Independent item workflows improve resilience but make totals, customer communication, and compensation more complex than a single order status.

**9. Interview questions**

- One seller integration is unavailable while other items are ready to ship.

- A delayed in-transit callback arrives after delivered and triggers another payout hold.

- A returned item becomes refundable just as its seller payout is released.

- Define operational ownership for stuck states and disputed deliveries, and make seller-specific failures visible without exposing unrelated customer data.

**Staff-level view:** Define operational ownership for stuck states and disputed deliveries, and make seller-specific failures visible without exposing unrelated customer data.

**Reference design:** https://www.postgresql.org/docs/18/transaction-iso.html

---

## Real Time

### Ride dispatch

*Match riders with nearby available drivers and track trip state.*

**1. Requirements**

- Fresh driver locations

- Exclusive driver assignment

- Recoverable trip state

**2. Scale:** 2M active drivers; location updates every 5 seconds; dispatch target under 2 seconds.

**3. API**

- `PUT /drivers/{id}/location {sequence,lat,lon}`

- `POST /rides`

- `POST /offers/{id}:accept`

**4. Data model**

- `driver_state(driver_id PK,availability,trip_id,version)`

- `locations(cell,driver_id,sequence,observed_at)`

- `trips(trip_id,state,rider,driver)`

**5. Architecture:** `Driver app` → `Location ingest` → `Geospatial index` → `Dispatch matcher` → `Assignment authority` → `Trip store` → `Rider updates`

**6. Deep dives**

- Subdivide hot cells, cap candidate sets, and apply queueing or ranked batches for airport dispatch. Keep driver assignment conditional by driver ID even when candidate search fans across multiple spatial partitions.

- Conditionally transition the driver from available to offered with a unique offer ID and expiry. Acceptance references that offer; late or duplicate acceptances cannot claim a different trip.

**7. Failures**

- A driver went offline but remains in the nearby-driver index.

- Require monotonic per-device sequences and server-observed freshness bounds; filter stale candidates and expire presence. Request confirmation before assignment and monitor stale-candidate rejection and pickup ETA error.

**8. Trade-offs**

- Frequent location updates improve matching but consume battery and ingest capacity; adaptive frequency can depend on trip state and movement.

**9. Interview questions**

- Thousands of riders and drivers concentrate in one geospatial bucket.

- A driver went offline but remains in the nearby-driver index.

- Two matchers choose the same available driver from a stale index.

- Prove assignment safety across retries and regional ownership changes, and distinguish dispatch latency from location freshness and driver acceptance rate.

**Staff-level view:** Prove assignment safety across retries and regional ownership changes, and distinguish dispatch latency from location freshness and driver acceptance rate.

**Reference design:** https://etcd.io/docs/v3.5/learning/api_guarantees/

---

### Collaborative document editor

*Merge concurrent document edits while supporting offline work and durable history.*

**1. Requirements**

- Convergent edits

- Reconnect and replay

- Access revocation

**2. Scale:** 1M concurrent editors; 5 operations/s per active editor; documents up to 10MB.

**3. API**

- `POST /documents`

- `WebSocket /documents/{id}/operations`

- `GET /documents/{id}/snapshot?version=`

**4. Data model**

- `documents(id,permission_version,snapshot_version)`

- `operations(document_id,op_id,causal_metadata,payload)`

- `snapshots(document_id,version,object_key)`

**5. Architecture:** `Editor client` → `Authenticated room gateway` → `Document operation authority` → `Durable operation log` → `Snapshot compactor` → `Collaboration fanout`

**6. Deep dives**

- Create versioned snapshots with a replay checkpoint, bound startup replay, and compact only metadata no longer needed by supported offline clients. Define a maximum offline horizon and resync protocol for older clients.

- Authenticate every resumed session and validate current write permission before durable acceptance. Reject unauthorized operations with a recoverable local export path; never merge first and remove later.

**7. Failures**

- Two edits at the same position produce different text depending on arrival order.

- Use a specified OT or CRDT algorithm with stable operation IDs and causal/version metadata, rather than last-write-wins on whole text. Reproduce the operation schedule and test that all valid delivery orders converge.

**8. Trade-offs**

- CRDTs help offline convergence but carry metadata and semantic complexity; OT can be compact but relies on carefully managed transformation context.

**9. Interview questions**

- Opening a document replays five million edits and freezes the client.

- Two edits at the same position produce different text depending on arrival order.

- A former collaborator reconnects with edits made while disconnected.

- Specify supported edit semantics, undo behavior across collaborators, offline horizon, and snapshot compatibility before choosing a merge algorithm.

**Staff-level view:** Specify supported edit semantics, undo behavior across collaborators, offline horizon, and snapshot compatibility before choosing a merge algorithm.

**Reference design:** https://etcd.io/docs/v3.5/learning/api_guarantees/

---

### Presence and typing indicators

*Show approximate online and typing state with bounded freshness.*

**1. Requirements**

- Per-device heartbeats

- Multi-device aggregation

- Automatic expiry

**2. Scale:** 30M connected devices; heartbeat every 30 seconds; typing expiry after 5 seconds.

**3. API**

- `PUT /presence/{deviceId} {sequence,state}`

- `POST /typing {conversationId,expiresAt}`

- `SUBSCRIBE /presence`

**4. Data model**

- `device_presence(user_id,device_id,session_epoch,last_seen)`

- `subscribers(user_id,watcher_id)`

- `typing(conversation_id,user_id,expires_at)`

**5. Architecture:** `Client` → `Connection gateway` → `Presence shards` → `Subscription fanout` → `Recipient gateways`

**6. Deep dives**

- Use exponential backoff with jitter, admission control, and staggered initial heartbeats. Drop redundant typing signals and batch presence transitions; protect durable chat traffic with separate resource pools.

- Condition disconnect cleanup on the matching device session epoch. Maintain independent per-device state and report the user online while any unexpired authorized device session exists.

**7. Failures**

- A mobile device loses network without closing its socket.

- Derive online status from expiring heartbeats and aggregate across active device sessions. Use server time for expiry, expire silent devices, and display approximate last-seen state rather than claiming exact liveness.

**8. Trade-offs**

- Short heartbeat intervals improve freshness but increase battery and traffic; presence accuracy should be bounded and approximate by product contract.

**9. Interview questions**

- A regional network outage ends and 10M devices reconnect at once.

- A mobile device loses network without closing its socket.

- A delayed close event for session A marks session B offline.

- Budget fanout by watchers and privacy rules, not just connected users, and isolate presence outages from message delivery.

**Staff-level view:** Budget fanout by watchers and privacy rules, not just connected users, and isolate presence outages from message delivery.

**Reference design:** https://etcd.io/docs/v3.5/learning/api_guarantees/

---

### Game leaderboard

*Rank players by validated scores with season boundaries and tie rules.*

**1. Requirements**

- Top K and user rank

- Idempotent score events

- Season rollover

**2. Scale:** 50M players; 200M score events/day; top-100 reads dominate.

**3. API**

- `POST /scores {eventId,playerId,delta,season}`

- `GET /leaderboards/{id}?top=100`

- `GET /players/{id}/rank`

**4. Data model**

- `score_events(event_id PK,player,season,delta)`

- `scores(season,player,total,version)`

- `season(id,state,final_checkpoint)`

**5. Architecture:** `Game server` → `Score validation` → `Event log` → `Score aggregator` → `Ordered ranking index` → `Leaderboard API`

**6. Deep dives**

- Partition player scores and merge shard top-K lists for global leaders. Serve exact rank through a separate periodically built order-statistics view or accept bounded approximation; declare the freshness and accuracy separately.

- Assign the season from authoritative match metadata, keep a bounded grace window, and finalize at a recorded checkpoint. Route later corrections through an explicit revision process rather than silently changing awarded rankings.

**7. Failures**

- A consumer replay applies the same match result twice.

- Deduplicate the event in the same durable update as the score mutation, or rebuild scores from unique validated results. Keep an auditable event log so incorrect projections can be discarded and recomputed.

**8. Trade-offs**

- Exact live global rank requires expensive coordination or indexing; top-K and approximate personal rank can scale independently.

**9. Interview questions**

- Fifty million players and constant writes overload one ranking shard.

- A consumer replay applies the same match result twice.

- A valid match ends before midnight but arrives after the season is finalized.

- Define tie-breaking, anti-cheat review, prize finality, and late correction policy as part of ranking correctness.

**Staff-level view:** Define tie-breaking, anti-cheat review, prize finality, and late correction policy as part of ranking correctness.

**Reference design:** https://etcd.io/docs/v3.5/learning/api_guarantees/

---

### Fleet tracking

*Ingest device positions, display moving fleets, and detect geofence transitions.*

**1. Requirements**

- Out-of-order GPS handling

- Current location and history

- Geofence alerts

**2. Scale:** 5M devices; update every 10 seconds while moving; 90-day history.

**3. API**

- `POST /telemetry {deviceId,sequence,observedAt,lat,lon}`

- `GET /fleets/{id}/positions`

- `GET /devices/{id}/history`

**4. Data model**

- `telemetry(device_id,time_bucket,sequence,observed_at,point)`

- `latest(device_id,sequence,point)`

- `geofence_state(device_id,fence_id,state,version)`

**5. Architecture:** `Devices` → `Regional ingest` → `Partitioned telemetry log` → `Latest-position processor` → `Time-series archive` → `Geofence processor` → `Fleet dashboard`

**6. Deep dives**

- Serve spatial clusters or tiles at low zoom, fetch individual positions only within a bounded viewport, and stream deltas for visible devices. Separate latest-state queries from historical scans and enforce per-tenant query budgets.

- Store all accepted historical points but condition latest-position updates on a newer device epoch/sequence or validated event-time policy. Handle clock anomalies explicitly and prevent stale replay from triggering current-location alerts.

**7. Failures**

- A parked truck near a boundary generates dozens of geofence transitions.

- Use entry/exit buffers, minimum dwell time, and accuracy-aware filtering. Deduplicate alert transitions by fence state version and preserve raw points for diagnosis instead of treating every crossing as certain.

**8. Trade-offs**

- Keeping every raw point aids audits and analytics but increases storage; downsampling must preserve the fidelity required for billing or route evidence.

**9. Interview questions**

- A large customer opens a world map containing one million vehicles.

- A parked truck near a boundary generates dozens of geofence transitions.

- A device reconnects and uploads yesterday’s points after a fresh live update.

- Define clock-drift tolerance, device identity reset, tenant isolation, and the difference between real-time safety alerts and best-effort business notifications.

**Staff-level view:** Define clock-drift tolerance, device identity reset, tenant isolation, and the difference between real-time safety alerts and best-effort business notifications.

**Reference design:** https://etcd.io/docs/v3.5/learning/api_guarantees/

---

## Search

### Web search engine

*Crawl public pages, build an inverted index, and rank query results.*

**1. Requirements**

- Polite crawling

- Freshness and deduplication

- Low-latency ranked retrieval

**2. Scale:** 10B pages; 100M queries/day; p95 search target 300 ms.

**3. API**

- `GET /search?q=&cursor=`

- `POST /crawl-seeds`

- `GET /crawl-status/{url}`

**4. Data model**

- `urls(url_id PK,canonical_url,next_fetch_at,content_hash)`

- `documents(doc_id,version,terms,signals)`

- `postings(term,shard,doc_id,positions)`

**5. Architecture:** `URL frontier` → `Host-aware crawlers` → `Parser and canonicalizer` → `Document store` → `Inverted-index builders` → `Query fanout` → `Ranker`

**6. Deep dives**

- Use tiered indexes, posting-list pruning, bounded candidate retrieval, and cached popular queries. Apply per-shard deadlines and partial-result policy; rank a limited candidate set with expensive features.

- Record durable removal tombstones with versions, apply them to serving filters and future index builds, and atomically swap validated index generations. Distinguish temporary fetch errors from confirmed removals.

**7. Failures**

- A site generates unlimited distinct URLs with equivalent low-value pages.

- Cap host and pattern crawl budgets, canonicalize known query parameters, detect content duplicates, and stop low-yield URL families. Respect crawl policies and avoid letting one host consume the frontier.

**8. Trade-offs**

- Fresh indexing consumes resources that could improve query latency; tier pages by change rate and query value instead of recrawling everything uniformly.

**9. Interview questions**

- A short query matches billions of postings and exhausts tail latency.

- A site generates unlimited distinct URLs with equivalent low-value pages.

- The source is gone but an old index generation still serves it.

- Set budgets for crawling, indexing freshness, query tail latency, and removal propagation, and plan index-generation rollback with durable tombstone preservation.

**Staff-level view:** Set budgets for crawling, indexing freshness, query tail latency, and removal propagation, and plan index-generation rollback with durable tombstone preservation.

**Reference design:** https://www.elastic.co/docs/manage-data/data-store/near-real-time-search

---

### Search autocomplete

*Return fast prefix suggestions with popularity, locale, and safety filtering.*

**1. Requirements**

- Prefix lookup

- Fresh trending suggestions

- Remove unsafe suggestions

**2. Scale:** 500M keystroke queries/day; p99 server budget 30 ms; 50 locales.

**3. API**

- `GET /suggest?q=&locale=&context=`

- `POST /suggestion-events`

**4. Data model**

- `suggestions(locale,prefix,rank,candidate_id)`

- `candidates(id,text,policy_version)`

- `query_counts(term,time_bucket,count)`

**5. Architecture:** `Client debounce` → `Edge suggestion cache` → `Prefix index` → `Policy filter` → `Async query aggregation` → `Snapshot builder`

**6. Deep dives**

- Debounce input, enforce a minimum useful prefix, cancel obsolete requests, and cache prefix-locale results at edge. Return bounded top-K candidates and suppress stale responses on the client using request sequence IDs.

- Cache only public prefix candidates globally, retrieve private history under user-scoped authorization, and merge at the final response boundary. Never place personalized responses in a shared cache without correct private cache controls.

**7. Failures**

- A moderation update removes a candidate but old prefix snapshots still include it.

- Maintain a small versioned deny filter in serving, invalidate affected hot prefixes, and rebuild snapshots asynchronously. Check policy after retrieval so old index generations cannot resurrect the candidate.

**8. Trade-offs**

- Precomputed suggestions are fast but stale between builds; online ranking improves freshness while increasing latency and cache fragmentation.

**9. Interview questions**

- Fast typists create six requests for one intended search.

- A moderation update removes a candidate but old prefix snapshots still include it.

- Suggestions cached only by prefix include a previous user’s private searches.

- Specify minimum cohort thresholds, retention, locale normalization, and emergency removals before using raw search logs as a suggestion source.

**Staff-level view:** Specify minimum cohort thresholds, retention, locale normalization, and emergency removals before using raw search logs as a suggestion source.

**Reference design:** https://www.elastic.co/docs/manage-data/data-store/near-real-time-search

---

### Product search

*Search a catalog with facets while respecting current price and availability.*

**1. Requirements**

- Text relevance and filters

- Facets and sorting

- Search freshness

**2. Scale:** 50M products; 80M queries/day; 10M product updates/day.

**3. API**

- `GET /products/search?q=&filters=&sort=`

- `GET /products/{id}`

**4. Data model**

- `products(product_id PK,price,stock_version,catalog_version)`

- `search_docs(product_id,index_version,terms,facets)`

- `price_history(product_id,version)`

**5. Architecture:** `Catalog database` → `Transactional outbox` → `Search indexer` → `Search cluster` → `Availability and price enrichment` → `Results API`

**6. Deep dives**

- Allowlist facets, cap bucket counts and candidate sets, precompute common category facets, and impose query time/memory budgets. Avoid unrestricted scripts and expose approximate counts when exact aggregation is too costly.

- Build a new generation from a consistent snapshot, replay changes from its checkpoint, and reject older per-product versions. Switch the alias only after lag and validation checks pass, retaining rollback capability.

**7. Failures**

- Index lag leaves an old sale price visible after the catalog changed.

- Display enrichment from current price for top results where feasible, show bounded freshness, and always reprice at checkout with user-visible confirmation. Monitor source-to-index lag and replay missing catalog events.

**8. Trade-offs**

- Denormalizing catalog fields accelerates queries but adds update fanout and staleness; current checkout validation protects the purchase invariant.

**9. Interview questions**

- A request aggregates millions of distinct seller and attribute combinations.

- Index lag leaves an old sale price visible after the catalog changed.

- A slow full reindex writes version 40 after live ingestion wrote version 44.

- Define a freshness budget per field: descriptions can lag longer than price or availability. Measure zero-result rate and business correctness separately from query latency.

**Staff-level view:** Define a freshness budget per field: descriptions can lag longer than price or availability. Measure zero-result rate and business correctness separately from query latency.

**Reference design:** https://www.elastic.co/docs/manage-data/data-store/near-real-time-search

---

### Log ingestion and search

*Collect high-volume logs and support bounded operational queries.*

**1. Requirements**

- Durable ingestion

- Tenant isolation

- Time-range search and retention

**2. Scale:** 100TB raw logs/day; 30-day hot retention; 10K tenants.

**3. API**

- `POST /logs:batch`

- `POST /queries {timeRange,filter}`

- `GET /queries/{id}`

**4. Data model**

- `log_events(tenant,event_id,event_time,service,body)`

- `partitions(tenant,time_bucket,location)`

- `query_jobs(id,budget,state)`

**5. Architecture:** `Agents` → `Ingest gateway` → `Durable buffer` → `Parsing workers` → `Time-partitioned search store` → `Query planner` → `Archive storage`

**6. Deep dives**

- Apply tenant/service byte quotas, sample repeated low-priority events, and preserve error fingerprints with counts. Buffer within strict limits and publish dropped-byte metrics so reduced visibility is explicit.

- Define an explicit lateness policy: archive old events, reject them with feedback, or route to a late-arrival partition. Keep event and ingest timestamps and ensure retention enforcement cannot be bypassed by future-dated events.

**7. Failures**

- User-supplied JSON creates a new field name for every request ID.

- Limit indexed field counts, normalize dynamic keys, and store arbitrary attributes in a controlled representation. Quarantine offending streams and reprocess from the durable buffer after fixing the schema.

**8. Trade-offs**

- Indexing every field improves arbitrary search but multiplies storage and ingest cost; selective indexing plus compressed scans is often more economical.

**9. Interview questions**

- A failing service emits the same stack trace millions of times per second.

- User-supplied JSON creates a new field name for every request ID.

- An offline agent uploads events older than the hot-retention boundary.

- Budget bytes, cardinality, query CPU, and result size per tenant, and provide operationally useful degradation during the exact incidents when logs surge.

**Staff-level view:** Budget bytes, cardinality, query CPU, and result size per tenant, and provide operationally useful degradation during the exact incidents when logs surge.

**Reference design:** https://www.elastic.co/docs/manage-data/data-store/near-real-time-search

---

### Vector retrieval service

*Retrieve semantically similar documents while enforcing metadata filters and permissions.*

**1. Requirements**

- Embedding versioning

- Hybrid retrieval

- Access-aware results

**2. Scale:** 500M chunks; 768-dimensional embeddings; 50M searches/day.

**3. API**

- `POST /documents`

- `POST /search {query,filters,topK}`

- `DELETE /documents/{id}`

**4. Data model**

- `chunks(chunk_id PK,document_id,text_version,embedding_version,acl_version)`

- `vectors(chunk_id,embedding)`

- `index_generations(id,model,checkpoint)`

**5. Architecture:** `Document ingest` → `Chunker` → `Embedding workers` → `Vector index` → `Lexical index` → `Hybrid candidate merge` → `ACL filter` → `Reranker`

**6. Deep dives**

- Partition by tenant where practical, use supported prefiltering or filter-aware search, and adapt candidate overfetch within a budget. Benchmark filtered recall against exact search samples and fall back for small candidate sets.

- Recheck authoritative permissions before fetching or returning chunk text, include ACL versioning, and propagate deletions to all index generations. Avoid logging unauthorized candidate contents and test cross-tenant adversarial queries.

**7. Failures**

- New documents use a different model while queries still use the old model.

- Build a separate model-versioned index, generate matching query embeddings, backfill and validate retrieval quality, then switch routing atomically. Dual-serve during evaluation and retain a rollback generation.

**8. Trade-offs**

- Approximate indexes improve speed with a recall tradeoff; aggressive post-filtering and compression can further reduce useful recall.

**9. Interview questions**

- Approximate search retrieves nearby vectors that are mostly outside the requested tenant or category.

- New documents use a different model while queries still use the old model.

- The vector index contains an old ACL and passes private text to a downstream model.

- Evaluate semantic quality, filtered recall, freshness, deletion propagation, and per-query cost independently; never treat similarity score as proof of factual correctness.

**Staff-level view:** Evaluate semantic quality, filtered recall, freshness, deletion propagation, and per-query cost independently; never treat similarity score as proof of factual correctness.

**Reference design:** https://www.elastic.co/docs/manage-data/data-store/near-real-time-search

---

## Infrastructure

### Global load balancer

*Route traffic to healthy capacity while controlling regional failover.*

**1. Requirements**

- Health-aware routing

- Connection draining

- Tenant and regional policy

**2. Scale:** 10M requests/s global peak; three active regions; long-lived connections included.

**3. API**

- `PUT /routes/{service}`

- `GET /backends/health`

- `POST /backends/{id}:drain`

**4. Data model**

- `routes(service,version,policy)`

- `backends(id,region,capacity,health_epoch)`

- `health_samples(backend,time,result)`

**5. Architecture:** `DNS or anycast entry` → `Regional edge proxies` → `Service routing table` → `Backend pools` → `Active/passive health evaluators`

**6. Deep dives**

- Route only admitted load according to available capacity, shed lower-priority traffic, and ramp failover by cohorts. Reserve disaster capacity or define a degraded-mode contract; retries must stay within the same global budget.

- Stop new assignments, allow a bounded drain window, notify clients to reconnect with jitter, and recover sessions through durable cursors. Force-close only after the documented deadline and measure lost in-flight requests.

**7. Failures**

- A shared dependency makes a deep health endpoint fail on all otherwise useful servers.

- Separate process health from dependency readiness, require evidence across samples, and limit simultaneous ejection. Maintain a carefully defined last-resort pool for safe operations and expose dependency-specific degradation.

**8. Trade-offs**

- Fast health reaction shortens outages but risks flapping; slower convergence is steadier but routes more requests to failing capacity.

**9. Interview questions**

- A failed region’s entire traffic is redirected to a region with only 30% spare capacity.

- A shared dependency makes a deep health endpoint fail on all otherwise useful servers.

- A deployment removes a backend while WebSocket sessions still rely on it.

- Define overload policy before failover, test correlated health-check failures, and include long-lived connection migration in regional evacuation drills.

**Staff-level view:** Define overload policy before failover, test correlated health-check failures, and include long-lived connection migration in regional evacuation drills.

**Reference design:** https://etcd.io/docs/v3.5/learning/api_guarantees/

---

### Authoritative DNS

*Serve authoritative records globally with controlled update propagation.*

**1. Requirements**

- Fast lookup

- Zone versioning

- Signed and audited configuration changes

**2. Scale:** 5M queries/s; 10M zones; records range from 30-second to 1-day TTLs.

**3. API**

- `PUT /zones/{id}/records`

- `POST /zones/{id}:publish`

- `GET /zones/{id}/versions`

**4. Data model**

- `zones(zone_id PK,owner,active_version)`

- `records(zone_id,version,name,type,value,ttl)`

- `publications(zone_id,version,regions)`

**5. Architecture:** `Zone API` → `Validation and signing` → `Versioned zone store` → `Regional publication pipeline` → `Authoritative anycast servers`

**6. Deep dives**

- Serve validated zone snapshots from memory at distributed edges, apply response-rate limiting where appropriate, and provision packet-processing capacity. Separate zone control-plane quotas from public query traffic.

- Explain and measure the remaining cache lifetime; plan migrations by lowering TTL in advance. Avoid promising instant global revocation and account for negative caching as well as positive records.

**7. Failures**

- A configuration update accidentally removes an essential record.

- Validate zone invariants, canary the publication, compare critical answers, and roll back the active version quickly. Preserve the prior signed generation and audit who changed which records.

**8. Trade-offs**

- Low TTL speeds planned changes but increases query load and dependency on authoritative availability; high TTL improves resilience while slowing propagation.

**9. Interview questions**

- One popular or attacked name attracts millions of queries per second.

- A configuration update accidentally removes an essential record.

- A record is corrected but recursive resolvers still return the previous answer.

- Test DNSSEC signing rollover where used, negative-cache behavior, delegation errors, and recovery from a control-plane compromise without disrupting stable serving.

**Staff-level view:** Test DNSSEC signing rollover where used, negative-cache behavior, delegation errors, and recovery from a control-plane compromise without disrupting stable serving.

**Reference design:** https://etcd.io/docs/v3.5/learning/api_guarantees/

---

### Metrics and alerting

*Collect time-series measurements and evaluate actionable alerts.*

**1. Requirements**

- High-volume ingestion

- Range queries

- Reliable alert state

**2. Scale:** 50M active series; scrape every 15 seconds; 30-day raw retention.

**3. API**

- `POST /metrics:write`

- `GET /query_range`

- `POST /alert-rules`

- `GET /alerts`

**4. Data model**

- `samples(series_id,timestamp,value)`

- `series(label_set_hash,labels)`

- `rules(rule_id,expression,for_duration)`

- `alert_state(rule_id,labels,state,since)`

**5. Architecture:** `Collectors` → `Ingest distributor` → `Time-series shards` → `Compactor` → `Query frontend` → `Rule evaluators` → `Notification router`

**6. Deep dives**

- Reject or drop high-cardinality labels at ingestion, enforce per-tenant series budgets, and use logs or traces for request IDs. Roll back the instrumentation and track created-series rate as well as sample rate.

- Preserve no-data as a distinct state, require sufficient samples for ratios, and configure explicit absent-series alerts for critical services. Report ingestion lag and evaluation data age alongside the computed value.

**7. Failures**

- Ingestion works but no rules are evaluated for twenty minutes.

- Alert on evaluation heartbeat and staleness from an independent path, fail over rule ownership with deduplication, and expose gaps. Evaluate missed windows according to a defined catch-up policy rather than fabricating continuous health.

**8. Trade-offs**

- Fine scrape intervals improve resolution but multiply samples; downsampling reduces cost while losing short-lived spikes and some aggregation fidelity.

**9. Interview questions**

- An instrumentation change adds request_id as a metric label.

- Ingestion works but no rules are evaluated for twenty minutes.

- A network partition makes an error-rate alert falsely report healthy traffic.

- Design alert ownership, deduplication, inhibition, and evaluation continuity during regional failover; monitor the monitor through an independent failure domain.

**Staff-level view:** Design alert ownership, deduplication, inhibition, and evaluation continuity during regional failover; monitor the monitor through an independent failure domain.

**Reference design:** https://etcd.io/docs/v3.5/learning/api_guarantees/

---

### Feature flag platform

*Distribute versioned configuration for safe progressive product rollouts.*

**1. Requirements**

- Stable targeting

- Fast local evaluation

- Emergency rollback

**2. Scale:** 100K applications; 10B evaluations/day; configuration updates are comparatively rare.

**3. API**

- `PUT /flags/{id}`

- `POST /flags/{id}:rollout`

- `GET /sdk-config?version=`

**4. Data model**

- `flags(flag_id PK,version,rules,default)`

- `segments(segment_id,version,members)`

- `audit(change_id,actor,before,after)`

**5. Architecture:** `Admin API` → `Validated config store` → `Publication log` → `SDK streaming/polling` → `Local evaluator` → `Exposure analytics`

**6. Deep dives**

- Distribute signed or integrity-checked versioned snapshots to SDKs and evaluate locally. Keep the last-known-good version and bounded refresh; send sampled exposure events asynchronously with privacy-aware context.

- Hash a stable subject key with flag or experiment salt into a fixed bucket space. Increasing rollout percentage expands a stable cohort; version deliberate rebucketing and keep subject identity consistent across devices when required.

**7. Failures**

- A malformed rule crashes an old SDK version.

- Validate configurations against supported SDK schemas, canary by client version, and retain a fallback snapshot. Reject unsupported operators or safely use the flag default; provide a minimal emergency-disable path.

**8. Trade-offs**

- Local evaluation improves availability and latency but makes immediate revocation depend on SDK refresh and connectivity; critical safety controls may need a separate path.

**9. Interview questions**

- Ten billion evaluations per day call the flag server synchronously.

- A malformed rule crashes an old SDK version.

- A 10% rollout uses a random number on every evaluation.

- Define stale-config limits, defaults during startup, audit and approval ownership, and the maximum supported offline period for each class of flag.

**Staff-level view:** Define stale-config limits, defaults during startup, audit and approval ownership, and the maximum supported offline period for each class of flag.

**Reference design:** https://etcd.io/docs/v3.5/learning/api_guarantees/

---

### Service discovery and configuration

*Maintain service endpoints and deliver coherent configuration revisions.*

**1. Requirements**

- Lease-based registration

- Watch and resync

- Quorum-safe updates

**2. Scale:** 1M service instances; 50K watch clients; endpoint churn during deployments.

**3. API**

- `PUT /services/{name}/instances/{id} {lease}`

- `GET /services/{name}?revision=`

- `WATCH /config?fromRevision=`

**4. Data model**

- `instances(service,id,address,lease_id)`

- `config(key,value,revision)`

- `watch_checkpoints(client,revision)`

**5. Architecture:** `Service agents` → `Registration API` → `Consensus store` → `Watch distribution` → `Client caches` → `Health-aware callers`

**6. Deep dives**

- Subscribe by service or namespace, batch updates, and distribute through regional watch relays. Rate-limit registrations and reconnects; clients retain a valid cached view while catching up from a revision.

- Accept configuration from committed consensus revisions and reject stale epochs. Keep writes unavailable without quorum if required for safety, while data-plane clients continue with last-known-good configuration within a documented staleness bound.

**7. Failures**

- A disconnected client requests events older than retained history.

- Return an explicit resync-required result, fetch a consistent snapshot with its revision, then watch subsequent changes. Do not silently continue from current time and leave missing endpoint updates undetected.

**8. Trade-offs**

- Strict configuration consistency can reduce control-plane availability during partitions; cached data-plane operation limits the user impact.

**9. Interview questions**

- Restarting 100K instances sends endpoint updates to every client.

- A disconnected client requests events older than retained history.

- A partitioned former leader continues sending updates after a new leader is elected.

- Test quorum loss, watch compaction, mass reconnects, and certificate rotation together; control-plane recovery must not trigger a larger data-plane outage.

**Staff-level view:** Test quorum loss, watch compaction, mass reconnects, and certificate rotation together; control-plane recovery must not trigger a larger data-plane outage.

**Reference design:** https://etcd.io/docs/v3.5/learning/api_guarantees/

---

## Advanced

### Multi-region key-value store

*Serve partitioned keys across regions with an explicit conflict and consistency model.*

**1. Requirements**

- Regional reads

- Defined write ownership

- Recovery and anti-entropy

**2. Scale:** 20TB hot data; 200K writes/s; three regions with 80-180 ms inter-region RTT.

**3. API**

- `GET /keys/{key}?consistency=`

- `PUT /keys/{key} {value,expectedVersion}`

- `GET /conflicts/{key}`

**4. Data model**

- `records(key,version,value,tombstone)`

- `ownership(range,epoch,region)`

- `replication_checkpoints(region,range,position)`

**5. Architecture:** `Client router` → `Regional replicas` → `Per-range write authority` → `Replication log` → `Anti-entropy repair` → `Conflict resolver`

**6. Deep dives**

- Split hot ranges and distribute independent tenant entities with hashed suffixes. For one hot value, redesign as sharded components or a single-owner log; adding replicas does not automatically parallelize conflicting writes.

- Choose per-field merge semantics or explicit conflict records for mergeable data; use single ownership or quorum transactions for nonmergeable invariants. Preserve causal/version metadata and tombstones long enough for repair.

**7. Failures**

- The primary region fails before its replication backlog reaches the standby.

- Quantify the missing log range, fence the old primary, and promote only under the accepted RPO policy. If zero acknowledged-write loss is required, use synchronous cross-region quorum before acknowledgement and accept its latency.

**8. Trade-offs**

- Low-latency independent regional writes require conflict handling or weaker invariants; synchronous global coordination trades latency and partition availability for stronger ordering.

**9. Interview questions**

- One tenant key prefix generates 40% of global write traffic.

- The primary region fails before its replication backlog reaches the standby.

- Two regions update the same shopping preference while disconnected.

- State the permitted anomaly for each operation, prove fencing during failover, and test restoration of an old region without resurrecting deleted data.

**Staff-level view:** State the permitted anomaly for each operation, prove fencing during failover, and test restoration of an old region without resurrecting deleted data.

**Reference design:** https://kafka.apache.org/design/

---

### Durable workflow engine

*Run long-lived business processes with durable timers, activities, and compensation.*

**1. Requirements**

- Crash-resilient progress

- Versioned workflow logic

- Human approval and timers

**2. Scale:** 100M active workflows; 20M activity completions/day; workflows may last a year.

**3. API**

- `POST /workflows {type,key,input}`

- `POST /workflows/{id}/signals`

- `GET /workflows/{id}/history`

**4. Data model**

- `histories(workflow_id,event_sequence,type,payload)`

- `tasks(queue,task_id,lease_epoch)`

- `timers(bucket,workflow_id,fire_at)`

**5. Architecture:** `Workflow API` → `History store` → `Deterministic workflow executor` → `Activity queues` → `External services` → `Durable timer service`

**6. Deep dives**

- Checkpoint supported state or continue as a new linked execution with a bounded history. Move high-frequency telemetry outside workflow history and keep only business decisions needed for deterministic recovery.

- Persist compensation state, retry idempotently with bounded backoff, and escalate to an operator queue after a policy threshold. Expose compensating and unresolved states rather than marking the original workflow simply rolled back.

**7. Failures**

- New workflow code takes a different branch while replaying an old history.

- Use workflow version markers or compatible code paths for existing histories; record external activity results and avoid uncontrolled wall-clock or random calls in deterministic execution. Canary replay against real sanitized histories.

**8. Trade-offs**

- Durable orchestration centralizes recoverability but introduces history storage and versioning complexity; choreography distributes ownership while making global progress harder to inspect.

**9. Interview questions**

- A workflow has accumulated a million events and replay takes minutes.

- New workflow code takes a different branch while replaying an old history.

- A booking workflow needs to refund payment after hotel reservation fails, but refunds are unavailable.

- Design version retirement, operator intervention, workflow cancellation, and data retention for processes that outlive several application releases.

**Staff-level view:** Design version retirement, operator intervention, workflow cancellation, and data retention for processes that outlive several application releases.

**Reference design:** https://kafka.apache.org/design/

---

### Ad auction and budget pacing

*Select eligible ads under tight latency while respecting advertiser budgets.*

**1. Requirements**

- Candidate eligibility

- Auction scoring

- Budget accounting and pacing

**2. Scale:** 5B auctions/day; p99 decision budget 100 ms; 1M campaigns.

**3. API**

- `POST /auctions {requestId,context}`

- `POST /impressions {eventId,auctionId}`

- `PUT /campaigns/{id}/budget`

**4. Data model**

- `campaigns(id,budget,targeting,version)`

- `budget_allocations(campaign,region,epoch,remaining)`

- `auction_events(auction_id,winner,price)`

- `billable_events(event_id,amount)`

**5. Architecture:** `Ad request` → `Eligibility filter` → `Candidate retrieval` → `Bid and quality scoring` → `Budget lease check` → `Auction result` → `Impression log` → `Billing reconciliation`

**6. Deep dives**

- Lease spend quotas to regions or serving shards and debit locally within those limits. Refill conservatively, account for in-flight impressions, and compute the worst-case budget excess before accepting a pacing design.

- Use a canonical billable-event identity tied to auction, placement, and event type, validate eligibility, and deduplicate within the ledger transaction. Keep raw events for fraud review without counting every raw delivery as billable.

**7. Failures**

- The system allocates new ads before earlier impressions are reported as billable.

- Reserve estimated spend at auction or delivery according to billing rules, reconcile actual billable events, and release unused reservations with a bounded timeout. Include outstanding reservations in pacing and monitor reporting lag.

**8. Trade-offs**

- Tight pacing reduces overspend but can underspend during partitions; larger local allocations improve serving availability while increasing in-flight exposure.

**9. Interview questions**

- One campaign participates in millions of auctions across regions.

- The system allocates new ads before earlier impressions are reported as billable.

- A client retries impression tracking after a lost response.

- Separate auction latency, relevance, billable-event accuracy, advertiser budget safety, and fraud detection. Explain which accounting decisions are provisional.

**Staff-level view:** Separate auction latency, relevance, billable-event accuracy, advertiser budget safety, and fraud detection. Explain which accounting decisions are provisional.

**Reference design:** https://kafka.apache.org/design/

---

### Recommendation serving

*Retrieve and rank personalized candidates with fresh features and safe fallbacks.*

**1. Requirements**

- Low-latency recommendations

- Feature freshness

- Experiment attribution

**2. Scale:** 100M daily users; 30 recommendations requests/day; 100ms p95 serving budget.

**3. API**

- `GET /recommendations?surface=&cursor=`

- `POST /interactions {eventId,itemId,type}`

- `POST /models/{id}:activate`

**4. Data model**

- `features(entity_id,feature_version,event_time,values)`

- `embeddings(entity_id,model_version,vector)`

- `models(id,schema,artifact,active_epoch)`

- `exposures(request_id,model,items)`

**5. Architecture:** `Interaction log` → `Stream and batch features` → `Feature store` → `Candidate retrieval` → `Ranker` → `Policy and diversity filter` → `Response cache`

**6. Deep dives**

- Set per-feature deadlines, precompute high-value slow features, and use explicit missing-value defaults trained into the model. Compare quality lift against latency and cost; fall back to a simpler ranker when dependencies degrade.

- Record assignment separately from actual rendered exposure, deduplicate both by stable request/item identity, and define the experiment analysis unit. Preserve model and feature versions for reproducibility.

**7. Failures**

- Offline evaluation improves while production recommendations regress.

- Share or validate transformation definitions, version feature schemas with the model, and compare sampled online vectors to point-in-time-correct offline recomputation. Roll back the model-feature bundle rather than only model weights.

**8. Trade-offs**

- Fresh online features improve responsiveness but add dependency and consistency costs; precomputed candidates are resilient but may miss immediate intent.

**9. Interview questions**

- The ranker waits 120ms for a rarely useful feature on every request.

- Offline evaluation improves while production recommendations regress.

- A request times out but its assigned treatment is logged as if the user saw the recommendations.

- Evaluate quality, latency, feature freshness, policy compliance, experiment validity, and cost together; maintain a useful fallback that can run when the feature platform is unavailable.

**Staff-level view:** Evaluate quality, latency, feature freshness, policy compliance, experiment validity, and cost together; maintain a useful fallback that can run when the feature platform is unavailable.

**Reference design:** https://kafka.apache.org/design/

---

### Streaming fraud detection

*Evaluate transaction risk using fresh event features and reviewable decisions.*

**1. Requirements**

- Low-latency decisioning

- Late-event handling

- Auditable model and rule versions

**2. Scale:** 100M payment attempts/day; 50ms feature budget; events arrive up to 10 minutes late.

**3. API**

- `POST /risk-decisions {transactionId,features}`

- `POST /risk-events`

- `GET /decisions/{id}/explanation`

**4. Data model**

- `events(event_id PK,entity,event_time,type)`

- `windows(entity,window_start,feature_version,state)`

- `decisions(transaction_id,rule_version,model_version,outcome)`

**5. Architecture:** `Transaction API` → `Online feature lookup` → `Rules and model scorer` → `Decision store` → `Event log` → `Windowed stream processors` → `Review queue`

**6. Deep dives**

- Use hierarchical or salted partial aggregates and merge bounded windows, cap per-key work, and include network context. Avoid treating shared-IP volume as a standalone fraud verdict; compare feature utility on legitimate cohorts.

- Preserve the original decision with its feature snapshot, compute a linked updated risk assessment, and trigger an explicit review or downstream action when policy allows. Use watermarks and a lateness policy for aggregates without rewriting past approvals.

**7. Failures**

- A deployment resets rolling velocity counts and risky transactions are approved.

- Restore a consistent state checkpoint with its source offsets, replay the gap, and mark features stale until caught up. Use a defined degraded decision policy and monitor event-time lag rather than CPU alone.

**8. Trade-offs**

- Waiting for more events can improve signal completeness but violates real-time latency; the design needs an explicit policy for uncertain or stale features.

**9. Interview questions**

- A mobile carrier NAT causes millions of users to share one IP feature bucket.

- A deployment resets rolling velocity counts and risky transactions are approved.

- An old device event arrives after the original transaction was approved.

- Separate detection quality from financial loss and customer friction; document model versions, fallback decisions, appeals, and the operational response to a stale feature pipeline.

**Staff-level view:** Separate detection quality from financial loss and customer friction; document model versions, fallback decisions, appeals, and the operational response to a stale feature pipeline.

**Reference design:** https://kafka.apache.org/design/

---

## Recipes — reusable end-to-end design flows

### Read-Heavy URL Shortener

*A small write path that allocates unique codes durably, and a redirect path so heavily cached that the database is barely involved.*

**Flow:** `Create API` → `Code allocator` → `URL database` → `Redirect cache` → `Redirect service` → `Analytics stream`

**The brief**

Users submit a long URL and receive a short one. Anyone following the short link is redirected to the original. The system must also report how many times each link was followed, and allow links to expire or be disabled.

The shape of the workload is what makes this interesting: creations are rare and must be durable and unique, while redirects are enormous in volume, trivially cacheable, and must be fast enough that the redirect is imperceptible.

- **Create**: accept a long URL, return a short code, never issue the same code twice.
- **Redirect**: resolve a code to a URL and redirect, in single-digit milliseconds at the edge.
- **Analytics**: count follows per link, accurate enough for reporting rather than for billing.
- **Lifecycle**: links can expire, be disabled, or be removed on abuse reports.
- **Scale**: assume 100 million links, 10,000 redirects per second at peak, 100 creations per second.

> **The read and write paths share almost nothing**  
> Creation needs uniqueness, durability and validation — a database problem with modest volume. Redirection needs a key-value lookup and nothing else, at a hundred times the rate. Treating them as one service with one datastore forces the redirect path to inherit constraints it does not need, and the entire design follows from separating them.

**Scale**

**Capacity arithmetic**

```text
TRAFFIC
  redirects  10,000/s peak, ~3,000/s average
  creations     100/s peak
  ratio       ~100:1 read to write

STORAGE
  100,000,000 links
  code 7 B + url ~200 B + metadata ~50 B ≈ 260 B
  -> ~26 GB of link data
  -> trivially fits one database; this is not a
     sharding problem

CACHE
  hot set: the top 10% of links serve ~90% of
  redirects
  10,000,000 entries x ~220 B ≈ 2.2 GB
  -> fits comfortably in memory
  -> 95%+ hit rate is achievable

DATABASE LOAD AFTER CACHING
  10,000/s x 5% miss = 500 reads/s
  + 100 writes/s
  -> a single well-provisioned instance handles this
     with enormous headroom

ANALYTICS
  10,000 events/s x 86,400 = 864,000,000/day
  -> must NOT be a synchronous database write
  -> stream it; aggregate in the background

CODE SPACE
  base62, 7 characters = 62^7 ≈ 3.5 x 10^12
  -> 100 million links is 0.003% of the space
  -> random collisions are vanishingly rare
```

| Metric | Value | Note |
|---|---|---|
| Redirect p99 | < 20 ms | **edge-served** |
| Cache hit rate | > 95% | hot set is small |
| Database load | ~600 ops/s | after caching |
| Analytics | 864 M/day | asynchronous |

The arithmetic says something important: at this scale nothing is hard except the redirect latency and the analytics volume. The link data fits on one machine, the write rate is trivial, and the working set fits in memory. Designs that begin by sharding the database are solving a problem this system does not have.

> **Writing an analytics row synchronously on every redirect makes the redirect path as slow as the database**  
> A counter incremented inside the redirect turns a cached lookup into a database write, multiplying the write load by a hundred and coupling redirect availability to the analytics store. The redirect must respond from cache and emit an event asynchronously — and if the event is lost during an incident, the count is slightly wrong, which is the correct thing to sacrifice.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Code generation | Random, counter-encoded, hash of URL | Random with collision retry | No coordination, unguessable, no hot partition |
| Redirect status | 301 permanent, 302 temporary | 302 | 301 is cached by browsers forever, so analytics and expiry stop working |
| Cache strategy | Cache-aside, read-through, write-through | Cache-aside with warm-on-create | Simple, and new links are hot immediately |
| Analytics path | Synchronous, queue, log stream | Log stream with batch aggregation | Volume is too high for synchronous writes |
| Custom aliases | Allowed, not allowed | Allowed, separate namespace check | A product requirement with a uniqueness cost |
| Storage | Relational, key-value | Relational | 26 GB with a uniqueness constraint is exactly its strength |

1. **Generate a random 7-character code and insert with a unique constraint** — Collisions are rare and the database detects them; retry on conflict.
2. **Validate and normalise the URL at creation** — Rejecting malformed and dangerous targets is cheaper here than at redirect time.
3. **Write the link, then warm the cache** — A newly created link is usually followed immediately.
4. **Serve redirects from cache, falling back to the database** — The miss path must exist and should be rare.
5. **Emit a redirect event to a stream, never a synchronous write** — Analytics volume is a hundred times the creation volume.
6. **Check expiry and disabled status at redirect time** — Both are cheap fields on the cached record.

The choice of 302 over 301 deserves emphasis because it is the decision most often made wrongly. A permanent redirect is cached by browsers and intermediaries indefinitely, which is excellent for latency and destroys every other requirement: follows stop reaching the service so analytics undercount, and a disabled or expired link continues redirecting from caches nobody controls. The small latency saving is not worth losing control of the link.

> **Ask before building it**  
> Do links need to be revocable, and does anyone depend on the click counts? If both answers are no, a permanently cached redirect served entirely from a CDN is simpler and faster than anything described here. Almost always at least one of them is yes, and that single answer determines the whole design.

**Deep dive**

The system divides cleanly into three subsystems with different characteristics: allocation, resolution and measurement.

**Code allocation strategies**

```text
RANDOM + UNIQUE CONSTRAINT  (recommended)
  generate 7 random base62 characters
  INSERT ... ON CONFLICT -> retry
  at 100 M links in 3.5 x 10^12 space:
    P(collision) per insert ≈ 0.003%
    -> roughly 1 retry per 35,000 creations
  + no coordination between instances
  + codes are unguessable, so links cannot be
    enumerated
  - requires a uniqueness check on insert

COUNTER + ENCODING
  take the next id from a sequence, encode base62
  + no collisions by construction
  + shortest possible codes early on
  - codes are sequential and enumerable: anyone can
    walk the entire link set
  - the sequence is a coordination point
  -> use only where enumeration is acceptable

HASH OF THE URL
  code = first 7 chars of hash(url)
  + the same URL always yields the same code
  - different users shortening the same URL share a
    link, so analytics and revocation collide
  - collisions still require handling
  -> rarely what is wanted

PRE-ALLOCATED BLOCKS
  each instance claims a block of 10,000 codes
  + no per-insert coordination, no collisions
  - unused codes in a block are lost on restart
  -> good when creation rate is very high; overkill
     here
```

**Random codes with a uniqueness constraint are the right default because enumerability is a real problem.** Sequential codes let anyone walk the entire set of links, exposing every URL anyone has shortened — which for a system where people shorten private documents and internal links is a serious disclosure. The random approach costs an occasional retry and removes the entire class of issue.

**The redirect path should not reach the application at all in the common case.** A cached mapping is small and immutable for the lifetime of the link, which makes it ideal for edge caching — but only where expiry and revocation can be honoured. The practical arrangement is a short edge cache lifetime measured in seconds plus an in-memory cache in the redirect service, which keeps latency low while bounding how long a revoked link stays live.

**Redirect request path**

```text
REQUEST /abc1234

1  EDGE CACHE  (short TTL, e.g. 30 s)
     hit  -> redirect immediately, ~5 ms
     miss -> forward to the redirect service

2  REDIRECT SERVICE
     in-memory cache lookup
       hit  -> respond
       miss -> shared cache lookup
         hit  -> respond, populate local
         miss -> database read
           found    -> respond, populate caches
           missing  -> 404

3  ALWAYS (non-blocking)
     emit {code, timestamp, referrer, user agent}
     to the analytics stream
     -> fire and forget; never awaited

CHECKS ON THE CACHED RECORD
  expired?    -> 410 Gone
  disabled?   -> 410 Gone
  -> both fields travel with the cached value, so
     neither costs a lookup

REVOCATION
  delete from caches on disable
  edge TTL bounds the worst case
  -> "revoked within 30 seconds" is the honest
     guarantee, and it should be stated
```

**Analytics must be a stream, and the accuracy target must be stated honestly.** At ten thousand events per second, synchronous writes are a hundred times the creation load and put the analytics database in the redirect path. Emitting to a log and aggregating in the background decouples them entirely, at the cost of occasionally losing events during an incident. For click counts that is acceptable; if the counts were used for billing, the answer would be different and more expensive.

**Abuse handling is a product requirement with architectural consequences.** Short links hide their destination, which makes them attractive for phishing, so the system needs URL validation at creation, a list of blocked destinations, a reporting path and the ability to disable a link quickly. The last of these is what rules out permanent redirects and long cache lifetimes — the design must be able to stop a link, and any caching decision that prevents that is not available.

**Ten times the traffic changes very little, which is worth saying explicitly.** At a hundred thousand redirects per second the cache tier grows and the analytics stream needs more partitions; the database still sees a few thousand operations per second and the data still fits comfortably. The binding constraint remains cache capacity and edge distribution, not storage or write throughput — which means the scaling story is adding read capacity, and that is the easiest kind of scaling there is.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Cache tier unavailable | Every redirect hits the database: 10,000/s against a tier sized for 600 | Request collapsing on misses; a small in-process cache as a second line |
| Database unavailable | Cached links keep working; creations fail; misses 500 | Extend cache lifetimes during the incident; fail creation cleanly |
| Analytics stream backed up | Counts fall behind; redirects unaffected | Buffer and drop oldest; never block the redirect |
| Code collision on insert | Duplicate key error | Retry with a new code, bounded attempts |
| A link goes viral | One key dominates cache traffic | Local per-instance caching absorbs it entirely |
| Malicious link reported | Users being sent to a harmful destination | Disable, purge caches, accept the edge TTL window |
| Bot traffic inflating counts | Analytics meaningless for affected links | Filter by user agent and rate; report separately |

> **A cache outage is the only genuine capacity risk in this system**  
> Every other component is provisioned with enormous headroom because caching removes almost all the load. If the cache disappears, the database receives the full redirect volume — roughly twenty times what it was sized for — and fails, taking every uncached link with it. The mitigations are a second cache layer inside the redirect service, request collapsing so that a thousand simultaneous misses for one key become one database read, and load shedding that protects the database rather than letting it collapse.

The pattern here is worth generalising: when caching improves a system by two orders of magnitude, the system's real availability depends on the cache rather than on the store, and the cache must therefore be designed with the care usually reserved for the database.

**The staff-level view**

What distinguishes a strong answer is recognising how little of this system is actually hard, and spending the time on the parts that are.

- **Separate the paths immediately.** Creations and redirects share a data model and nothing else; the design follows from treating them as two systems.
- **Do the storage arithmetic out loud.** Twenty-six gigabytes and a hundred writes per second tells everyone that sharding is not the problem, which redirects the conversation to what is.
- **Choose 302 and explain why.** Permanent redirects trade away revocation and analytics for a small latency gain, and knowing that trade is the signal.
- **Argue for random codes on enumerability grounds**, not on collision grounds — the security property is the real reason.
- **Name the cache as the availability risk** and describe the specific protections: request collapsing, a local second tier, and shedding.
- **State the revocation guarantee honestly**, including the edge cache window, rather than implying links stop instantly.

> **The question that raises the level**  
> What is the accuracy requirement for click counts, and who consumes them? If they drive a customer-facing dashboard, eventual consistency and occasional loss are fine; if they drive revenue share to publishers, the analytics path needs the durability of the outbox pattern and the cost rises substantially. Asking this before designing the analytics path is what separates a system that fits its purpose from one that is either over-engineered or quietly wrong.

---

### Global Product Catalog

*Serve product data worldwide with fast reads and correct prices, by separating the authoring path from a denormalised, cached read model.*

**Flow:** `Authoring service` → `Catalog database` → `Change stream` → `Denormalised read store` → `Edge cache` → `Search index`

**The brief**

A catalogue of several million products must be browsable and searchable from anywhere in the world in tens of milliseconds. Each product page combines base attributes, regional pricing, availability, images, reviews and recommendations — data owned by different teams and updated at very different rates.

Merchants update products continuously, and some of those updates — price in particular — must take effect quickly and correctly everywhere, because showing a stale price is not a cosmetic problem.

- **Read**: product pages in under 100 milliseconds globally, at 50,000 requests per second.
- **Write**: merchants update products; changes visible within a minute, prices within seconds.
- **Correctness**: the price shown must be the price charged, in every region.
- **Search and browse**: filter by category, attributes and availability, ranked sensibly.
- **Scale**: 5 million products, 20 regions, 10 to 50 attributes per product.

> **The catalogue is one dataset with two entirely different shapes**  
> Authoring needs normalised, validated, transactional data with clear ownership per field. Reading needs one denormalised document per product per region that can be fetched in a single operation and cached anywhere. Trying to serve both from the same representation produces either slow pages assembled from joins, or an authoring model contorted to suit reads — the resolution is to derive the second from the first.

**Scale**

**Capacity arithmetic**

```text
READS
  50,000 product page views/s at peak
  each page = 1 product document
  -> 50,000 document reads/s if uncached

CACHING CHANGES EVERYTHING
  product popularity is extremely skewed
  top 1% of products ≈ 60-80% of views
  edge cache hit rate 85-95% achievable
  -> origin sees 2,500-7,500 reads/s

STORAGE
  5,000,000 products
  x 20 regions (price, availability, locale)
  base document ~8 KB, regional overlay ~1 KB
  -> base: 40 GB
  -> regional: 100 GB
  -> ~150 GB total for the read model

WRITES
  merchant updates ~500/s average, 5,000/s during
  bulk imports
  each update fans out to up to 20 regional documents
  -> 10,000-100,000 document writes/s during imports
  -> the write amplification is the real scaling
     problem, not the reads

PRICE PROPAGATION
  target: under 10 s from change to visible
  -> excludes long edge cache lifetimes for price
  -> which is why price is often fetched separately

SEARCH INDEX
  5 M documents, ~2 KB indexed each -> ~10 GB
  reindex on attribute changes only, not on price
```

| Metric | Value | Note |
|---|---|---|
| Page latency | < 100 ms | **globally** |
| Cache hit rate | 85-95% | popularity is skewed |
| Read model | ~150 GB | denormalised per region |
| Write fan-out | up to 20× | the real constraint |

The write amplification is what sizing must respect. A single price change to a product sold in twenty regions becomes twenty document updates, and a bulk import of a hundred thousand products becomes two million. The pipeline that materialises the read model therefore needs to be sized for import bursts rather than for the steady-state edit rate, and it needs backpressure so that a large import does not delay the price change that has to propagate in seconds.

> **Caching a product page including its price couples page caching to price correctness**  
> A fully assembled page cached for ten minutes is fast and will show a price that changed nine minutes ago. Either the cache lifetime drops to seconds — losing most of the benefit — or price is excluded from the cached document and fetched separately. The second is almost always correct, and it means the page is not actually one document at render time.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Read model | Join at read time, denormalised documents | Denormalised per product per region | One read per page, cacheable, no fan-out at request time |
| Materialisation | Dual write, change data capture | Change data capture from the authoring store | No dual-write divergence; the log is the source of ordering |
| Price handling | In the document, fetched separately | Separate, short-lived | Price freshness requirements differ by orders of magnitude |
| Regional data | One document with all regions, one per region | One per region | Pages read one region; a combined document is 20× too large |
| Search | Query the database, dedicated index | Dedicated index fed by the same stream | Faceting and ranking are not database operations |
| Edge caching | Full page, fragments | Fragments with different lifetimes | Stable attributes cache for hours; price cannot |

1. **Author into a normalised, validated store with per-field ownership** — Pricing, inventory and content teams update different fields independently.
2. **Capture changes from the database log rather than emitting events in application code** — The log cannot disagree with the data.
3. **Materialise one document per product per region** — The read path should never join or fan out.
4. **Keep price and availability out of the long-lived cached document** — Their freshness requirements are incompatible with caching the rest.
5. **Feed the search index from the same change stream** — One pipeline keeps the index and the read model consistent with each other.
6. **Make the read model rebuildable from the authoring store** — It is derived data, and reprocessing is the recovery mechanism.

Choosing change data capture over application-emitted events matters more than it appears. The catalogue is updated by several services, admin tools, bulk importers and occasional manual corrections, and any path that writes directly to the database without emitting an event produces a read model that silently diverges. Reading the log captures every change regardless of how it arrived, which is the only approach that holds up in a system with this many writers.

> **Ask before building it**  
> How fresh must each field be? Grouping fields by freshness requirement — descriptions in hours, availability in minutes, price in seconds — decides the caching strategy, the fragmentation of the read model and the shape of the whole delivery path. A single answer of everything must be current forces the design towards no caching at all, which is both unnecessary and unaffordable.

**Deep dive**

Three subsystems carry the weight: the materialisation pipeline, the composition of the page, and the handling of price.

**From authoring change to visible page**

```text
1  AUTHORING WRITE
     merchant updates a description
     -> normalised tables, validated, transactional

2  CHANGE CAPTURE
     the database log emits the row change
     -> ordered, complete, no application code
        involved

3  MATERIALISER
     reads the change
     loads the product's full current state
     writes one document per affected region
     -> idempotent: writing the same state twice is
        harmless
     -> partitioned by product id, so updates to one
        product stay ordered

4  READ STORE
     key: product id + region
     value: the full document
     -> single-key read, no joins

5  EDGE
     cached fragments with per-fragment lifetimes
       attributes and media   hours
       availability           ~60 s
       price                  not cached, or seconds

LAG BUDGET
  capture      < 1 s
  materialise  < 5 s
  cache expiry  varies by fragment
  -> "description updates visible within a minute"
     is achievable and should be stated as the SLA
```

**Partitioning the materialiser by product identifier is what keeps updates correct.** Two changes to the same product must be applied in order, or a stale version can overwrite a newer one; changes to different products have no relationship and can proceed in parallel. Keying the stream by product gives exactly that guarantee, and it is the same reasoning that makes partitioned logs the standard transport for this kind of pipeline.

**The read model must be rebuildable, and that capability gets used.** Schema changes, materialiser bugs and new derived fields all require regenerating documents from the authoring store, so the pipeline needs a backfill mode that can reprocess the entire catalogue without disturbing live traffic. Systems that treat the read model as precious rather than as derived end up patching it in place, which accumulates inconsistencies nobody can explain.

**Composing a page with mixed freshness**

```text
PRODUCT PAGE
  ┌─ attributes, images, description ──── cached 6 h
  ├─ category and related products ────── cached 1 h
  ├─ availability ─────────────────────── cached 60 s
  ├─ price ────────────────────────────── live, or 5 s
  └─ reviews summary ──────────────────── cached 15 m

WHY NOT CACHE THE WHOLE PAGE
  the page is only as cacheable as its least
  cacheable component
  -> one fragment needing 5 s freshness caps the
     whole page at 5 s
  -> fragmenting recovers the caching benefit for
     the 95% of bytes that are stable

WHERE COMPOSITION HAPPENS
  at the edge  fragments assembled close to the user
  in a BFF     one request per page, fan-out server
               side
  in the client stable shell cached, price fetched
               after render
  -> all three are used in practice; the client
     option gives the best perceived performance
     and the worst first-paint correctness

THE PRICE RULE
  the price displayed must equal the price charged
  -> checkout re-validates the price server-side
  -> a stale displayed price becomes a visible
     correction at checkout, not a wrong charge
```

**Price is the field where correctness and performance genuinely conflict, and the resolution is to enforce correctness at checkout.** No caching strategy guarantees that a displayed price is current at the moment of purchase — the user may leave the page open for an hour. Re-validating the price server-side when the order is placed makes the displayed price advisory and the charged price authoritative, which is both correct and legally necessary in most jurisdictions; the user sees a clear correction rather than an incorrect charge.

**Search is a separate system fed by the same stream, not a query against the catalogue.** Faceted filtering, relevance ranking and attribute search are not operations a document store performs well, and attempting them against the read model produces either slow queries or an unmaintainable set of secondary indexes. Feeding a dedicated index from the same change stream keeps it consistent with the read model to within a second or two, which is the right consistency target for search.

**Bulk imports are the operational event this system is actually judged on.** A merchant uploading a hundred thousand products generates millions of document writes and can saturate the materialiser, delaying the price change that needed to propagate in seconds. The remedy is a priority distinction in the pipeline — small interactive edits ahead of bulk work — plus throttling on imports so that catalogue freshness for existing products is not held hostage to someone's spreadsheet.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Materialiser falls behind | Edits not visible; price stale | Alert on pipeline lag; prioritise price changes |
| Bulk import saturates the pipeline | All updates delayed, including urgent ones | Priority lanes; throttle imports |
| Read store unavailable in a region | Pages fail in that region | Serve from another region with higher latency |
| Stale price displayed | User sees one price, is charged another | Re-validate at checkout; show the correction |
| Search index diverges | Products missing from results | Rebuild from the authoring store |
| Change stream gap | Some edits never materialise | Periodic reconciliation against the source |
| Cache purge fails | Stale content persists past its lifetime | Short lifetimes rather than reliance on purging |

> **Divergence between the authoring store and the read model is silent by nature**  
> Nothing errors when a materialised document is missing an update — the page renders, the product looks normal, and only the specific changed field is wrong. Because the failure produces no signal, it is discovered by a merchant complaining weeks later, which means the system needs active reconciliation: periodically comparing a sample of source records against their materialised documents and alerting on mismatches, rather than assuming the pipeline is correct because nothing has failed.

The same reasoning applies to the search index, which diverges in a more visible way — products simply do not appear in results — but is equally undetectable from inside the system.

**The staff-level view**

The strong version of this design treats freshness as a per-field property and builds the delivery path around it.

- **Split authoring from reading explicitly**, and derive the read model rather than dual-writing it.
- **Group fields by freshness requirement** and let that drive fragmentation and cache lifetimes, instead of choosing one lifetime for the page.
- **Choose change data capture** and justify it by the number of writers, which is what makes application-emitted events unreliable here.
- **Make the price guarantee explicit**: displayed price is advisory, checkout re-validates, and the user sees a correction rather than a wrong charge.
- **Raise bulk imports as the operational risk** and propose priority lanes, because it is the scenario that actually degrades the product.
- **Design reconciliation from the start**, since materialisation gaps produce no errors and are found by customers.

> **The question that raises the level**  
> What happens to a page when the read store in a region is unavailable? Answering serve from another region sounds obvious and has consequences worth stating — higher latency, possibly stale regional pricing, and a decision about whether to show a page with degraded data or an error. Systems that have not decided this in advance discover during the incident that showing the wrong region's price was the worst available option.

---

### Reliable Checkout

*Turn an ambiguous multi-service purchase into an idempotent, resumable order whose state is always explainable to the customer.*

**Flow:** `Checkout request` → `Idempotency store` → `Order record` → `Payment saga` → `Inventory reservation` → `Fulfilment events`

**The brief**

A customer submits an order. The system must reserve inventory, charge a card, create a fulfilment record and send a confirmation — across four services with separate databases, over a network that drops responses, to a client that retries and a user who double-clicks.

Every failure here is expensive in a way that ordinary system failures are not. A duplicate charge is a refund and a support conversation; a charge with no order is a dispute; an order with no charge is lost revenue; and inventory reserved for an order that does not exist is unsellable stock.

- **Exactly-once effect**: one submission produces one order and one charge, regardless of retries.
- **Resumable**: a crash partway through must not leave the order in an unknown state.
- **Explainable**: at any moment, the customer and support must be able to see where the order is.
- **Bounded**: every order reaches a terminal state — complete or failed — within a defined time.
- **Scale**: 500 orders per second at peak, 5,000 during a sale event.

> **The hard requirement is not atomicity, it is accountability**  
> A purchase cannot be atomic across four services, and pretending otherwise leads to designs that fail in unexplainable ways. What it can be is fully accounted for: every operation carries an identifier, every state transition is recorded, and no combination of failures produces a state nobody can name. Once that holds, the remaining problems are ordinary retries and compensations.

**Scale**

**Capacity and timing arithmetic**

```text
TRAFFIC
  500 orders/s steady, 5,000/s during a sale
  each order: ~6 service calls, ~10 database writes
  -> 5,000/s peak means ~50,000 writes/s across the
     estate

THE PAYMENT CALL DOMINATES LATENCY
  inventory reservation   ~20 ms
  payment authorisation   300-3,000 ms (external)
  order persistence       ~10 ms
  -> the external call is 90%+ of the elapsed time
  -> and it is the one that can time out ambiguously

CONCURRENCY DURING A SALE
  5,000 orders/s x 1.5 s average payment latency
  = 7,500 concurrent in-flight payments
  -> thread-per-request models cannot do this
  -> asynchronous calls, or the order goes async
     entirely

IDEMPOTENCY STORE
  500/s x 86,400 x 7 days retention
  = ~300 M keys
  key + request hash + response ≈ 1 KB
  -> ~300 GB, or shorter retention
  -> 24 h covers automatic retries; 7 days covers
     support-driven resubmission

TERMINAL STATE BUDGET
  happy path            < 5 s
  with payment retries  < 60 s
  stuck, needs a human  flagged within 15 min
  -> every order must be in a terminal state or
     flagged; none may sit silently
```

| Metric | Value | Note |
|---|---|---|
| Peak | 5,000 orders/s | sale events |
| Latency driver | payment call | **90% of elapsed time** |
| Idempotency keys | ~300 M | at 7-day retention |
| Stuck orders | flagged in 15 min | never silent |

The concurrency figure is the one that shapes the implementation. Seven and a half thousand simultaneous in-flight payments is not a load problem in throughput terms — it is a problem of holding that many operations open at once, which rules out any model that dedicates a thread to each. Either the calls are asynchronous, or checkout returns immediately with a pending order and the work continues in the background.

> **Sale events do not scale the load uniformly**  
> During a sale the order rate rises tenfold while the payment provider's latency also rises, because everyone's traffic increases at once. Concurrent in-flight payments therefore grow faster than the order rate — potentially twenty or thirty times the baseline — which is why capacity planning based on orders per second alone underestimates the peak requirement substantially.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Duplicate prevention | Client-side guard, idempotency key | Idempotency key from the client | The server is the only place that knows what already happened |
| Coordination | Two-phase commit, saga | Orchestrated saga | Services have separate databases; 2PC couples their availability |
| Step ordering | Charge first, reserve first | Reserve inventory first, charge last | The least reversible step goes last |
| Execution model | Synchronous, asynchronous | Synchronous with an async fallback | Users expect an answer; slow payments must not hold connections |
| Payment ambiguity | Retry, query | Query the provider by idempotency key | A timed-out charge may have succeeded |
| State model | Implicit from flags, explicit state machine | Explicit, persisted, with timestamps | Every order must be explainable at any moment |

1. **Accept an idempotency key with the order submission** — Generated once by the client, reused on every retry, persisted so it survives a reload.
2. **Claim the key atomically and record the order in one transaction** — The order's existence and its key must be inseparable.
3. **Reserve inventory before charging** — A reservation is cheap to release; a charge is not cheap to refund.
4. **Pass the same key to the payment provider** — It makes the provider able to deduplicate and, crucially, to be queried.
5. **Persist every state transition with a timestamp** — This is what makes the order explainable during an incident.
6. **Time out every step and define its compensation** — An order with no deadline is an order that can sit forever.

Reserving inventory before charging is the ordering that most reduces customer harm, and the reasoning generalises. Payment failures are far more common than inventory failures, so charging first means routine declines produce refunds the customer sees on their statement. Reserving first means the common failure — a declined card — releases a reservation nobody knew about and produces no financial trace at all.

> **Ask before building it**  
> What happens if the payment provider times out and never tells you the outcome? Every serious design for this system is shaped by that answer, and a design that has not confronted it will either double-charge or lose orders when it occurs — which it will, regularly, at this volume.

**Deep dive**

Three mechanisms carry the correctness of this system: the idempotency claim, the saga with its compensations, and the resolution of ambiguous payments.

**The order lifecycle**

```text
STATES
  created      idempotency key claimed, order
               persisted
  reserved     inventory held
  authorising  payment call in flight
  paid         authorisation succeeded
  confirmed    fulfilment created, customer notified
  failed       compensated, terminal
  stuck        neither forward nor back; needs a
               human

TRANSITIONS
  created -> reserved -> authorising -> paid ->
  confirmed
  any step failing -> compensate backwards -> failed
  compensation failing -> stuck

WHAT MAKES IT EXPLAINABLE
  every transition writes {state, timestamp, reason}
  -> support can answer "where is my order" exactly
  -> and "why did it fail" without reading logs

TIMEOUTS PER STATE
  reserved      2 min  -> release, fail the order
  authorising   60 s   -> resolve the ambiguity
  paid          5 min  -> retry fulfilment
  stuck         0      -> alert immediately

THE RULE
  no state without a timeout
  -> an order sitting in 'authorising' for a day is
     a bug that a timeout would have surfaced in a
     minute
```

**The idempotency claim must be in the same transaction as the order record.** If the key is claimed first and the order written second, a crash between them leaves a key that blocks every retry for an order that does not exist — the customer cannot reorder and the system reports success for nothing. Claiming and creating together means the key's presence is exactly equivalent to the order's existence, which is the property every subsequent retry depends on.

**Resolving an ambiguous payment**

```text
THE SITUATION
  POST /charge with idempotency key K
  -> timeout, no response
  -> the charge may have succeeded, may not

WHAT NOT TO DO
  retry blindly           -> risks a second charge
  assume failure          -> risks losing a real
                             charge and the order
  assume success          -> risks confirming an
                             unpaid order

WHAT TO DO
  1  GET the charge by idempotency key K
       found, succeeded -> treat as paid, proceed
       found, failed    -> compensate, fail cleanly
       not found        -> safe to retry with K
  2  if the query itself fails
       remain in 'authorising'
       retry the query with backoff
       after the timeout -> mark stuck, alert
  -> the key is what makes the query possible; this
     is the practical reason to pass it downstream

RECONCILIATION
  daily: compare provider settlements against orders
  -> charges with no order      -> refund
  -> orders marked paid with no
     charge                     -> investigate
  -> this catches everything the live path missed
```

**Passing the idempotency key to the payment provider is what converts an unresolvable state into a query.** Without it, a timed-out charge leaves no way to ask what happened — the system knows only that it sent a request. With it, the provider can be asked directly, and the answer is authoritative. This single practice removes the majority of ambiguous-payment incidents and is the reason payment APIs expose idempotency keys in the first place.

**Daily reconciliation is not optional at this scale.** Even with careful handling, some fraction of orders will end in states the live path could not resolve, and the only authoritative record of what money moved is the provider's settlement file. Comparing it against orders catches charges without orders and orders marked paid without charges, and the volume of discrepancies found is a direct measure of how well the live path is working.

**Going asynchronous is the answer to sale-event concurrency, and it changes the product.** Returning an order identifier immediately with a pending status, and notifying the customer when it completes, removes the requirement to hold thousands of connections open through slow payment calls. It also means the customer leaves the checkout page without knowing whether their order succeeded, which is a product decision — acceptable for a scheduled drop where everyone expects a queue, and jarring for an ordinary purchase.

**The stuck state must exist explicitly and must be small.** Orders that can move neither forward nor backward are genuinely unresolvable by code — a charge succeeded and the refund failed, or the provider is unreachable and has been for an hour. Giving that condition a name, alerting on it immediately, and providing support tooling to resolve it by hand is what prevents those orders from sitting invisibly in a table. The count of stuck orders per day is one of the best health metrics this system has.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Client retries after a timeout | Risk of duplicate order and charge | Idempotency key returns the original result |
| Payment times out ambiguously | Unknown whether money moved | Query the provider by key; never blind-retry |
| Charge succeeds, fulfilment fails | Customer paid, nothing shipping | Retry fulfilment; refund if it cannot be created |
| Refund fails during compensation | Money taken, order failed | Bounded retries, then stuck and alert |
| Inventory service unavailable | Cannot reserve | Fail fast before charging; nothing to compensate |
| Sale event exhausts payment concurrency | Checkout requests queue and time out | Async checkout; shed low-value traffic |
| Order stuck in a non-terminal state | Customer sees processing indefinitely | Per-state timeouts; alert on age |

> **The order taken for money that produces nothing is the failure that defines this system**  
> A charge that succeeds followed by a compensation that fails leaves the customer paid and unserved, with no automatic path to resolution. It cannot be designed away — the effects live in different systems — so it must be designed for: bounded retries on the refund, an explicit stuck state, immediate alerting, tooling that lets support resolve it, and reconciliation that catches any instance the live path missed entirely. A system without those four things will have this failure and will not know.

Every other failure in the table is recoverable by mechanism. This one is recoverable only by a person, which is why the design's job is to make sure a person finds out quickly and has enough recorded context to act.

**The staff-level view**

What distinguishes a strong answer is treating the ambiguous payment as the central problem rather than as an edge case.

- **Open with the idempotency key**, including where it is generated and that it must survive a client restart.
- **Justify the step ordering** by compensation cost: reserve first, charge last, because declines are common and refunds are visible.
- **Describe the ambiguous timeout resolution** — query by key rather than retry — as the design's central mechanism.
- **Give every state a timeout** and name the stuck state explicitly, with alerting and support tooling.
- **Include daily reconciliation** against provider settlements, and say what discrepancies it is expected to find.
- **Address sale-event concurrency** with asynchronous checkout, and acknowledge the product consequence rather than hiding it.

> **The question that raises the level**  
> What is the business's tolerance for a duplicate charge versus a lost order? They are not symmetrical: a duplicate charge is detectable, refundable and generates a support contact, while a lost order is usually invisible to everyone including the customer, who simply believes the purchase failed. Most businesses, asked directly, prefer the recoverable failure — and knowing that answer determines how aggressively the system retries.

---

### Flash Sale Inventory

*Sell a fixed quantity to an enormous simultaneous crowd without overselling, by making the scarce decision atomic and everything else disposable.*

**Flow:** `Waiting room` → `Admission control` → `Atomic decrement` → `Reservation hold` → `Checkout` → `Release on expiry`

**The brief**

Ten thousand units go on sale at a fixed moment. Two million people are waiting, and essentially all of them arrive within the same few seconds. Every one of them attempts the same operation against the same counter.

Overselling is unacceptable — it means cancelling confirmed orders, refunding customers and absorbing the reputational cost — and so is underselling, which leaves stock unsold while people were refused. The system must sell exactly the available quantity, quickly, under the most concentrated load it will ever see.

- **Never oversell**: the number of confirmed orders must not exceed the stock.
- **Sell out**: stock held by abandoned checkouts must return for sale.
- **Survive the spike**: two million requests in seconds against one logical counter.
- **Be fair enough**: the ordering must be defensible, even if it is not strictly first-come.
- **Fail clearly**: users who miss out must be told quickly, not left waiting.

> **Almost every request is going to be rejected, so rejection must be the cheapest path**  
> With ten thousand units and two million people, 99.5% of requests fail. A design that does authentication, cart creation and a database transaction before discovering there is no stock spends all of its capacity on work that produces nothing. The scarce decision has to happen as early and as cheaply as possible, and everything expensive must occur only for the small number of requests that have already won.

**Scale**

**The arithmetic of a concentrated spike**

```text
THE CROWD
  2,000,000 waiting
  arrival window ~10 s
  -> 200,000 requests/s at the front door
  -> versus a normal peak of perhaps 5,000/s
  -> a 40x spike, arriving instantly

THE COUNTER
  10,000 units
  one logical value that every request must touch
  -> cannot be sharded by key: there is one key
  -> this is a hot key by construction

WHAT AN ATOMIC COUNTER CAN DO
  a single in-memory counter: ~100,000 ops/s
  -> still below 200,000/s
  -> so the counter must be protected by admission
     control, not exposed to the full crowd

SPLITTING THE COUNTER
  10,000 units as 10 buckets of 1,000
  each request picks a bucket at random
  -> 10x the throughput
  -> but a request hitting an empty bucket while
     others have stock is wrongly refused
  -> fix: on empty, try one or two other buckets
  -> imperfect, and usually acceptable

RESERVATION HOLDS
  10,000 winners x 10 min hold
  expected abandonment 20-40%
  -> 2,000-4,000 units return to sale
  -> the release mechanism decides whether you sell
     out
```

| Metric | Value | Note |
|---|---|---|
| Front door | 200,000 req/s | **40× normal** |
| Rejection rate | 99.5% | make it cheap |
| Counter | one key | hot by construction |
| Abandonment | 20-40% | must be recovered |

The two numbers that matter most are the rejection rate and the abandonment rate. The first says that the system's primary job is refusing people efficiently; the second says that selling out depends entirely on reclaiming holds from people who did not complete. A design that handles the spike beautifully and never releases abandoned reservations will end the sale with stock unsold and customers who were refused.

> **The counter cannot be scaled by adding servers**  
> Every request must consult the same value, so no amount of horizontal capacity increases how fast that value can be decremented. Throughput at the counter is bounded by one atomic operation rate, which means the design must reduce how many requests reach it — through a waiting room, admission control and early rejection — rather than trying to make it faster.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Crowd handling | Let everyone through, waiting room | Waiting room with admitted tokens | The counter cannot serve 200,000/s; nothing else will fix that |
| Decrement mechanism | Database row, atomic counter | In-memory atomic counter, durably backed | A database row under this contention serialises everything |
| Counter splitting | Single, split into buckets | Split with fallback probing | Throughput matters more than perfect exhaustion |
| Reservation | Immediate sale, hold then checkout | Hold with a short expiry | Payment takes time; stock must not be lost to slow users |
| Fairness | First-come, lottery, queue position | Queue position from the waiting room | Defensible, and it decouples arrival timing from success |
| Overselling posture | Optimistic with reconciliation, strictly prevented | Strictly prevented | Cancelling confirmed orders is worse than refusing a sale |

1. **Put everyone in a waiting room before the sale opens** — It converts an instantaneous spike into a controlled admission rate.
2. **Admit users at a rate the counter can serve** — Admission control is the mechanism that makes the rest possible.
3. **Decrement an atomic counter as the single point of truth** — One operation decides the outcome; everything else follows from it.
4. **Create a reservation with a short expiry on success** — The winner needs time to pay without holding stock forever.
5. **Release expired reservations back to the counter** — Abandonment is 20-40%; without release, the sale does not sell out.
6. **Confirm the sale only when payment succeeds** — The reservation is a hold, not a sale.

The waiting room is the decision that makes everything else tractable, and it is often treated as a user-experience feature rather than as the load-bearing mechanism it is. By issuing positions before the sale opens and admitting people at a controlled rate, it converts two million simultaneous arrivals into a steady stream the counter can actually serve — and it gives users a clear, honest answer about their position instead of an error.

> **Ask before building it**  
> Is strict first-come ordering a requirement, or merely an assumption? Enforcing it precisely requires a globally ordered queue, which is itself a bottleneck at this scale. A waiting room that admits in approximate arrival order is far cheaper, and users generally accept it — but only if the product does not promise something stricter.

**Deep dive**

Three mechanisms determine whether this works: admission control, the atomic decision, and the reclamation of unconfirmed holds.

**The request path, from crowd to sale**

```text
BEFORE THE SALE
  users join a waiting room
  each receives a signed token with a position
  -> this happens minutes in advance, spreading the
     load

AT SALE TIME
  admission controller releases tokens in batches
    e.g. 2,000 admitted per second
  -> the counter sees 2,000/s, not 200,000/s
  -> everyone else waits, with a position shown

ADMITTED REQUEST
  1  validate the token (signature, not a lookup)
  2  atomic decrement of a stock bucket
       success -> continue
       zero    -> probe 1-2 other buckets
       all zero -> sold out, respond immediately
  3  create a reservation with a 10 min expiry
  4  return the reservation to the client

CHECKOUT
  the user pays within the hold window
  payment succeeds -> confirm the order
  window expires   -> release the unit

WHY TOKEN VALIDATION IS A SIGNATURE CHECK
  200,000/s of database lookups to validate tokens
  would be its own bottleneck
  -> a signed token validates with CPU alone
```

**The counter must be a single atomic operation, and the choice of where it lives decides the ceiling.** A database row under this contention serialises every request behind one lock and delivers perhaps a few thousand operations per second. An in-memory atomic counter delivers a hundred thousand, and splitting it into buckets multiplies that further. The trade is durability: an in-memory counter must be reconciled against durable reservation records so that a restart does not lose or invent stock.

**Splitting the counter, and what it costs**

```text
SINGLE COUNTER
  stock = 10,000
  every request: DECR stock
  -> perfectly exhausts the stock
  -> throughput capped at one key's operation rate

SPLIT INTO 10 BUCKETS
  bucket[0..9] = 1,000 each
  request picks a bucket by hash or at random
  -> 10x throughput

THE EXHAUSTION PROBLEM
  late in the sale, buckets empty unevenly
  bucket 3 empty, bucket 7 has 40 units
  a request landing on 3 is told sold out
  -> under-selling, and users who should have won
     are refused

MITIGATION: PROBE
  on empty, try 2 other buckets at random
  -> nearly eliminates false sold-out at the cost of
     2 extra operations on the rare path
  -> the remaining error is small and bounded

RECONCILIATION
  a durable record is written for each reservation
  periodically: counter + reservations + confirmed
  should equal the original stock
  -> drift means the counter is wrong, and the
     durable records win
```

**Reservation expiry is what turns a sale into a sell-out.** With a third of winners abandoning checkout, thousands of units sit held by people who will never pay. Releasing them promptly returns them to the counter for a second wave of buyers, and the length of the hold is a direct trade: too short and genuine buyers lose their place while entering card details, too long and the stock is idle while demand is still present. Ten minutes is a common compromise, with the release happening on expiry rather than on a periodic sweep.

**Fairness is a product decision that should be made explicitly and communicated.** Strict first-come ordering rewards fast connections and bots; a lottery among everyone who joined the waiting room rewards nobody unfairly and frustrates people who queued early; position-based admission sits between them. None is objectively correct, and users accept any of them provided the rule was stated in advance — what they do not accept is discovering that the rule was different from what they assumed.

**Bot traffic is a substantial fraction of the crowd and must be addressed before the counter.** Automated buyers arrive faster, retry harder and consume a disproportionate share of admissions, so the waiting room needs identity binding, rate limiting per account and per network, and enough friction to make automation unattractive. Doing this at the front door is essential; doing it after the decrement means bots have already taken stock, and reversing that means cancelling orders.

**Everything except the decrement should be considered disposable.** Session state, analytics, recommendations and personalisation all add load during the spike and none of them affects whether the right number of units is sold. Shedding them aggressively — serving a static waiting page, deferring all non-essential writes, disabling optional features for the duration — preserves capacity for the one operation that must be correct, and is far more effective than optimising any individual component.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Counter throughput exceeded | Requests queue, users see timeouts | Admission control; split the counter further |
| Overselling | Confirmed orders exceed stock | Atomic decrement before any confirmation; never decrement twice |
| Under-selling | Stock unsold while users are refused | Bucket probing; prompt release of expired holds |
| Counter lost on restart | Stock count wrong in either direction | Rebuild from durable reservation records |
| Waiting room bypassed | Full crowd reaches the counter | Signed tokens required; reject unadmitted requests |
| Bots taking most of the stock | Real customers refused | Identity binding and rate limits at the front door |
| Payment provider saturated | Winners cannot complete | Extend holds; queue payments; communicate clearly |
| Expired holds not released | Sale ends with unsold stock | Release on expiry; reconcile at the end |

> **Overselling is recoverable only by cancelling confirmed orders, which is why it must be prevented rather than detected**  
> Once a customer has a confirmation, retracting it costs a refund, a support contact and public annoyance out of proportion to the value of the item. This is the reason the design accepts a strictly serialised decision point and builds admission control around it, rather than taking an optimistic approach with reconciliation afterwards — the cost asymmetry between under-selling and overselling is large enough to decide the architecture.

Under-selling, by contrast, is recoverable: unsold stock can go on sale again, and the customers who were refused are no worse off than if the item had sold out a second earlier.

**The staff-level view**

The strong answer recognises that this is a load-shaping problem with a small atomic core, not a database problem.

- **Lead with the rejection ratio.** When 99.5% of requests fail, making rejection cheap is the primary design goal.
- **Name the counter as an unshardable hot key** and explain that admission control, not horizontal scaling, is the answer.
- **Propose the waiting room as load-bearing infrastructure**, not as a user-experience nicety.
- **Discuss counter splitting with its cost** — false sold-out responses — and the probing mitigation.
- **Make reservation expiry central**, since abandonment of a third means release is what produces a sell-out.
- **Justify strict prevention of overselling** by the asymmetry of recovery costs, rather than by preference.

> **The question that raises the level**  
> What does the business want to optimise: selling out, fairness, or the experience of the people who lose? These conflict. Optimising for sell-out favours aggressive release and re-offering, which means people refused earlier see items return; optimising for fairness favours a lottery that frustrates the quick; optimising for the losing experience favours telling people early and definitively that they will not succeed. Choosing one deliberately produces a coherent system, and most flash-sale designs fail because they were never asked to choose.

---

### Ticket Reservation

*Hold specific seats for a limited time while a buyer completes payment, without ever selling the same seat twice.*

**Flow:** `Seat map` → `Availability cache` → `Hold with expiry` → `Payment` → `Confirmation` → `Release sweeper`

**The brief**

A venue has assigned seats, and buyers choose specific ones rather than a quantity. Two people looking at the same seat map must not both be able to buy seat 14F, and a buyer who selects seats needs a few minutes to enter payment details without losing them.

Unlike a flash sale, the units are not interchangeable: a buyer wants those seats, and being given different ones is a failure. The seat map must also stay visibly accurate, because a map showing available seats that cannot actually be bought is worse than one showing fewer options.

- **Uniqueness**: each seat is sold at most once, ever.
- **Holds**: a selected seat is reserved for a bounded period while payment completes.
- **Visible accuracy**: the seat map reflects holds and sales within a second or two.
- **Release**: abandoned holds return to availability automatically.
- **Scale**: venues up to 100,000 seats; peaks of 50,000 concurrent buyers when a popular event opens.

> **A hold is a lease on a specific resource, and every difficulty follows from that**  
> The seat is identified rather than counted, so the mechanism is not a counter but a per-seat claim with an expiry. That makes the problem a leasing problem: who holds what, until when, and what happens when the holder vanishes — which is a much better-understood shape than trying to treat seats as inventory units.

**Scale**

**Capacity and contention arithmetic**

```text
VENUE
  100,000 seats
  seat record: id, section, row, number, state,
  hold_expiry, order_id ≈ 100 B
  -> 10 MB per event; trivial storage

OPENING SPIKE
  50,000 concurrent buyers
  each viewing the seat map, refreshing every few
  seconds
  -> 25,000 map reads/s
  -> map must be cached and diffed, not re-queried

HOLD ATTEMPTS
  peak ~5,000 hold attempts/s
  each touching ONE seat row
  -> contention is per seat, not global
  -> unlike a flash sale, this shards naturally

CONTENTION IS CONCENTRATED
  the best 5% of seats attract 60% of attempts
  -> ~3,000/s against 5,000 rows
  -> individual popular seats see dozens of
     simultaneous attempts
  -> the per-row atomic claim is what resolves it

HOLD DURATION
  7 minutes typical
  50,000 buyers, 30% abandonment
  -> steady release traffic throughout the sale
  -> release must be prompt or the map is wrong

MAP FRESHNESS
  target: under 2 s
  -> push updates rather than polling where possible
  -> a stale map produces failed holds and
     frustration
```

| Metric | Value | Note |
|---|---|---|
| Seats | 100,000 | 10 MB |
| Contention | per seat | **shards naturally** |
| Hold | 7 minutes | then released |
| Map freshness | < 2 s | or holds fail |

The crucial structural difference from a flash sale is that contention distributes across seat rows rather than concentrating on one counter. A hundred thousand independent rows means the database can serve the claim operations directly, with contention only on individually popular seats — which a conditional update resolves without any global coordination.

> **A stale seat map converts into failed hold attempts, which multiply load**  
> If the map lags by ten seconds, buyers select seats that were taken nine seconds ago, the hold fails, and they immediately try again — producing several attempts per successful hold and a frustrating experience. Map freshness is therefore not a cosmetic concern: it directly determines how much wasted contention the claim path receives.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Claim mechanism | Application lock, conditional update | Conditional update in the database | The row itself arbitrates; no external lock to expire |
| Hold representation | Separate hold table, state on the seat | State and expiry on the seat row | One row, one atomic transition, no join to check availability |
| Release | Sweeper job, expiry checked on read | Both | Reads must never see an expired hold as active |
| Map delivery | Polling, push | Push with periodic full reconciliation | Thousands of buyers polling a large map is its own load |
| Multi-seat selection | Independent claims, all-or-nothing | All-or-nothing in one transaction | Buyers want adjacent seats; partial success is useless |
| Best-available | Client picks, server picks | Offer both | Most buyers want good seats, not specific ones |

1. **Store seat state and hold expiry on the seat row** — Availability is then a single-row read with no joins.
2. **Claim with a conditional update that checks current state and expiry** — The database arbitrates concurrent attempts atomically.
3. **Claim multiple seats in one transaction** — Adjacent-seat requests must succeed or fail together.
4. **Treat an expired hold as available on read** — A sweeper alone leaves a window where seats look taken but are not.
5. **Push map changes to connected clients** — Polling a hundred-thousand-seat map at scale is avoidable load.
6. **Confirm only after payment succeeds** — The hold is a lease, and the sale is a separate transition.

Checking expiry on read as well as running a sweeper is a small detail with a real effect. A sweeper alone means a seat whose hold lapsed thirty seconds ago still appears unavailable until the next sweep, so buyers are refused seats that are genuinely free. Making the read treat an expired hold as available closes that window entirely, and the sweeper becomes a tidying mechanism rather than the mechanism of correctness.

> **Ask before building it**  
> Do buyers choose exact seats, or the best available in a price tier? The second is a substantially easier problem — closer to a counter with a selection heuristic — and many venues actually want it. Building exact-seat selection when the product needs best-available adds contention, map complexity and failure modes that nobody required.

**Deep dive**

Three mechanisms determine correctness here: the atomic claim, the treatment of expiry, and the delivery of an accurate map.

**Claiming seats atomically**

```text
SEAT ROW
  {id, event, section, row, number,
   state: available | held | sold,
   hold_id, hold_expires_at, order_id}

SINGLE-SEAT CLAIM
  UPDATE seats
  SET state = 'held', hold_id = ?,
      hold_expires_at = now() + interval '7 minutes'
  WHERE id = ?
    AND (state = 'available'
         OR (state = 'held'
             AND hold_expires_at < now()))
  -> 1 row updated  = claimed
  -> 0 rows updated = someone else has it
  -> no lock, no external coordination, no
     possibility of two winners

MULTI-SEAT CLAIM  (adjacent seats)
  BEGIN
    claim each seat with the same condition
    if any returns 0 rows -> ROLLBACK
  COMMIT
  -> all or nothing
  -> order seat ids consistently to avoid deadlocks
     between concurrent multi-seat attempts

WHY THE EXPIRY CLAUSE MATTERS
  it makes an expired hold claimable immediately
  -> no dependence on a sweeper for correctness
  -> the sweeper exists only to keep the map clean
```

**Ordering seat identifiers within a multi-seat transaction prevents deadlocks.** Two buyers attempting overlapping sets of seats in opposite orders will each hold what the other needs, and the database will resolve it by aborting one — which is correct but produces avoidable failures under load. Sorting the identifiers before claiming means concurrent transactions always acquire rows in the same sequence, and one simply waits for the other.

**Keeping the map accurate**

```text
THE MAP IS LARGE
  100,000 seats, ~20 B each rendered
  -> ~2 MB of state
  -> sending it repeatedly to 50,000 clients is
     untenable

DELIVERY
  1  initial load: full map, compressed, cached at
     the edge (seat layout is static)
  2  then: a stream of changes
       {seat_id, state} deltas
       -> a few bytes per change
       -> tens of KB/s for a busy event

  3  periodic reconciliation
       every 60 s, clients request a checksum or a
       compact state summary
       -> corrects any missed delta
       -> essential, because delta streams drop
          messages

WHY PUSH RATHER THAN POLL
  25,000 map reads/s against a large document is a
  significant load for information that changes
  slowly per seat
  -> deltas reduce it by orders of magnitude

DEGRADATION
  if the stream fails, fall back to polling a
  summarised map at a low rate
  -> stale but usable, and holds still arbitrate
     correctly
```

**The map is advisory and the claim is authoritative, which should be reflected in the interface.** However fresh the map, two buyers can select the same seat within the same instant, and one will lose. The design's job is to make that outcome rare through freshness and graceful through presentation — telling the buyer the seat was just taken and offering the nearest equivalent, rather than returning an error that sends them back to an unchanged map.

**Hold duration is a direct trade between conversion and availability.** Short holds return seats quickly and lose buyers who are slower with payment details; long holds protect buyers and leave seats idle while others are waiting. Seven minutes is a common choice, and extending it when a buyer reaches the payment step — rather than making it uniformly longer — gets most of the benefit with less idle inventory.

**Best-available selection deserves first-class support because most buyers want it.** Choosing exact seats is a feature for enthusiasts; most people want two seats together in a price range, and serving that with a server-side selection from a maintained availability structure avoids the whole cycle of examining the map, selecting, failing and retrying. It also concentrates contention where the system can manage it, since the server can choose seats that are not currently contested.

**Resale and transfer change the model more than they appear to.** A seat that returns to sale after being sold has a history, may have a different price, and must not be resold while a transfer is in flight — which turns a two-state lifecycle into something closer to an ownership record. Systems that add resale to a design built for one-time sales typically discover that the seat row needs to become an ownership ledger, and retrofitting that is considerably harder than designing for it.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Two buyers claim one seat | Double sale | Conditional update makes it impossible; one gets zero rows |
| Expired holds not released | Seats unsellable | Expiry checked on read; sweeper as cleanup |
| Map stale during the opening rush | Repeated failed holds, frustration | Push deltas; reconcile periodically |
| Payment fails after the hold expires | Buyer paid for released seats | Validate the hold before charging; extend at payment step |
| Deadlocks on multi-seat claims | Transactions aborting under load | Order seat ids consistently |
| Delta stream drops messages | Client map silently wrong | Periodic checksum reconciliation |
| Bot buyers taking the best seats | Real customers excluded | Identity binding, rate limits, queueing |

> **Charging for seats whose hold has already expired is the failure that produces the worst customer outcome**  
> If the payment step does not re-validate the hold, a buyer who took slightly too long can be charged for seats that were released and resold in the interim — leaving them paid and seatless, with someone else holding their tickets. The remedy is to check the hold before authorising payment and to extend it when the buyer reaches that step, so the window between validation and charge is as small as possible.

The general rule is the same one that governs any lease: acting on a lease without confirming it is still held is unsafe, and the confirmation must be as close as possible to the action.

**The staff-level view**

The distinguishing content is the atomic claim expressed precisely, and the treatment of expiry as a read-time concern.

- **Give the conditional update in full**, including the expired-hold clause, since that single statement is the correctness mechanism.
- **Contrast with a flash sale**: contention distributes across seat rows here, so this is not an unshardable hot key problem.
- **Handle expiry on read as well as by sweeper**, and explain the window a sweeper alone leaves open.
- **Make multi-seat claims all-or-nothing** and order the identifiers to avoid deadlocks.
- **Design the map as deltas with reconciliation**, because streams drop messages and a silently wrong map is worse than a slow one.
- **Re-validate the hold at payment time**, since charging for released seats is the worst outcome available.

> **The question that raises the level**  
> Will these tickets be resold or transferred? If yes, the seat is not a two-state resource but an ownership record with a history, and designing it that way from the start costs little — while retrofitting ownership onto a sold flag after launch means migrating live inventory and reconciling every historical sale. It is the single most consequential question about the data model, and it is rarely asked early.

---

### Hybrid Social Feed

*Precompute timelines for ordinary accounts, merge large accounts at read time, and rank the combined candidates before serving.*

**Flow:** `Post ingestion` → `Fanout service` → `Per-user timelines` → `Large-account store` → `Merge and rank` → `Feed API`

**The brief**

Users follow accounts and see a feed of their posts. The follower distribution spans six orders of magnitude — most accounts have hundreds of followers, a few have tens of millions — and the following distribution is similarly skewed, with some users following thousands of accounts.

The feed is also ranked rather than chronological, which means retrieval produces candidates and a separate scoring step decides what the user actually sees.

- **Feed load**: under 200 milliseconds for the first page, at 100,000 requests per second.
- **Freshness**: a post from someone you follow should be eligible within seconds.
- **Scale**: 500 million users, 500 million posts per day, median 200 followers, maximum 100 million.
- **Ranking**: the feed is ordered by predicted relevance, not by time.
- **Correctness**: blocked, deleted and private content must never appear.

> **Neither fanout strategy is bounded, so the system must use both**  
> Writing a post into every follower's timeline costs work proportional to follower count, which is unbounded; assembling a feed by querying every followed account costs work proportional to following count, which is also unbounded. Applying each strategy only where it is cheap — precompute for ordinary accounts, read-time merge for the rare enormous ones — bounds both paths, and the reason it works is that large accounts are rare enough that any one user follows only a few of them.

**Scale**

**Fanout and read arithmetic**

```text
POSTS
  500,000,000 posts/day ≈ 5,800/s average,
  ~20,000/s peak

FANOUT COST
  median author: 200 followers -> 200 inserts
  p99 author: 50,000 followers -> 50,000 inserts
  celebrity: 100,000,000 followers
    -> 100,000,000 inserts for ONE post
    -> at 50,000 inserts/s that is 33 minutes
    -> and it blocks everyone else's posts

THRESHOLD
  fan out below 100,000 followers
  above it, do not fan out
  -> accounts above the threshold: ~0.01% of accounts
  -> a user following 2,000 accounts follows perhaps
     10-30 of them

READ COST WITH THE HYBRID
  1 timeline read (precomputed)
  + 1 batched read for ~20 large accounts
  -> two round trips regardless of following size

TIMELINE STORAGE
  500 M users x 800 entries x 16 B
  = ~6.4 TB
  -> only for ACTIVE users; inactive users need no
     precomputed timeline
  -> active fraction ~20% -> ~1.3 TB

RANKING
  ~500 candidates per feed request
  scored in under 50 ms
  -> feature lookup dominates, not the model
```

| Metric | Value | Note |
|---|---|---|
| Fanout threshold | 100,000 followers | **0.01% of accounts** |
| Read path | 2 round trips | regardless of following |
| Timelines | ~1.3 TB | active users only |
| Candidates | ~500 per request | then ranked |

Restricting precomputation to active users is the single largest storage saving available and is frequently overlooked. Most accounts on a large platform have not opened the application in months, and maintaining their timelines consumes both storage and a share of every fanout — writing into inboxes nobody will read. Materialising on first return, from the read-time path, costs one slower feed load for a returning user and removes the majority of the work.

> **Fanout volume is driven by the tail of the follower distribution, not by the median**  
> Median authors are irrelevant to capacity: two hundred inserts is nothing. The pipeline is sized by accounts in the tens of thousands of followers, which are numerous enough to matter and below the threshold, so their posts still fan out. Capacity planning based on average follower counts underestimates the requirement by a wide margin.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Feed construction | Push, pull, hybrid | Hybrid with a follower threshold | Both pure strategies are unbounded in a real graph |
| Timeline contents | Post identifiers, full posts | Identifiers, hydrated at read | Edits propagate; storage stays small |
| Who gets a timeline | All users, active users | Active users only | Most accounts never read; fanout to them is wasted |
| Ranking stage | At write, at read | At read, over merged candidates | Scores depend on time and viewer context |
| Filtering | At write, at read | At read | Blocks and deletions after fanout are prohibitive to apply at write |
| Timeline depth | Unbounded, capped | Capped at ~800 entries | Users rarely scroll past a few hundred |

1. **Classify authors by follower count, with hysteresis at the threshold** — An account oscillating across the boundary causes repeated backfills and duplicates.
2. **Fan out ordinary posts asynchronously to active followers** — The author's request must not wait for the writes.
3. **Store large-account posts once, keyed by author** — They are merged at read time instead.
4. **Cache each user's list of followed large accounts** — Computing it per feed request reintroduces the read-time cost.
5. **Merge, deduplicate and then rank** — Deduplication matters because an account crossing the threshold appears in both paths.
6. **Apply blocks, privacy and deletions during hydration** — Applying them at write time means touching millions of timelines per change.

Filtering at read time rather than at write time is the decision that makes the system operable. A user blocking someone, an account going private, or a post being deleted would otherwise require removing entries from every timeline that contains them — work equal to the original fanout, triggered by an action that should be instantaneous. Checking during hydration costs a small amount on every read and makes those operations free.

> **Ask before building it**  
> Is the feed ranked or chronological? A chronological feed is a merge and nothing more, and the precomputed timeline is the finished product. A ranked feed makes retrieval a candidate-generation step feeding a scoring system with its own features, latency budget and infrastructure — which is a substantially larger system wearing the same description.

**Deep dive**

Three subsystems carry the design: fanout, read-time merge, and ranking over the merged candidates.

**The write path**

```text
POST CREATED
  1  persist the post (source of truth)
  2  classify the author
       followers < 100,000 -> FANOUT
       followers >= 100,000 -> NO FANOUT
  3a FANOUT PATH
       load active followers
       append {post_id, timestamp} to each timeline
       trim each timeline to ~800 entries
       -> asynchronous, through a queue
       -> partitioned so one large author cannot
          block others
  3b NO-FANOUT PATH
       append to the author's own recent-posts list
       -> that is all; readers will find it

PRIORITISATION
  fanout jobs are not equal
    200 followers   -> milliseconds
    80,000 followers -> seconds, and it occupies
                        workers
  -> separate lanes by follower count
  -> otherwise a 90,000-follower post delays
     thousands of ordinary ones

ACTIVE FOLLOWERS ONLY
  "active" = opened the app in the last 30 days
  -> typically 20-30% of followers
  -> a 3-5x reduction in fanout volume
  -> inactive users get their feed from the
     read-time path when they return
```

**Separating fanout into lanes by follower count is what keeps ordinary posts fast.** Without it, a post to eighty thousand followers occupies workers for seconds while thousands of small posts queue behind it, and feed freshness becomes a function of what large accounts happen to be doing. Routing by expected work — small posts in one lane, large ones in another with its own capacity — keeps the common case immediate.

**The read path**

```text
FEED REQUEST from user U

1  read U's precomputed timeline        (1 read)
     -> ~800 post ids, ranked by time

2  read U's followed-large-accounts list (cached)
     -> typically 10-30 accounts

3  batched read of those accounts' recent posts
     -> 1 round trip, ~20 lists

4  MERGE
     combine both sources
     DEDUPLICATE by post id
       -> essential: an account that recently crossed
          the threshold appears in both
     -> ~500 candidates

5  HYDRATE and FILTER
     fetch post content (cached)
     drop: deleted, blocked author, private,
           already seen
     -> filtering here is why write-time is free

6  RANK
     score candidates with viewer features
     -> return the top 20

LATENCY BUDGET  (200 ms)
  timeline read        10 ms
  large-account read   20 ms
  hydration            40 ms
  filtering            10 ms
  ranking              50 ms
  overhead             20 ms
  -> ~150 ms, leaving headroom
```

**Deduplication at the merge is not an edge case; it is a routine occurrence.** Accounts grow past the threshold continuously, and while an account is crossing it, its older posts sit in timelines while its newer ones arrive through the read path. Without deduplication by post identifier, followers of growing accounts see items twice — a visible defect that appears only for some users and is hard to reproduce.

**Ranking turns the retrieved set into the product, and it dominates the latency budget.** Scoring several hundred candidates requires features about the viewer, the author, the relationship and the post, most of which are lookups rather than computation — which is why feature-store latency, not model inference, is usually the binding constraint. Keeping the candidate set to a few hundred rather than a few thousand is the most effective lever on feed latency.

**The precomputed timeline is a candidate source, not a feed.** This reframing matters: once ranking exists, the timeline's job is to supply a good set of possibilities cheaply, and it does not need to be complete or perfectly ordered. That relaxation is what permits capping depth at a few hundred entries, restricting fanout to active users, and tolerating the small inconsistencies that any asynchronous pipeline produces.

**New follows need an answer, and the cheapest one is usually to do nothing.** Following someone leaves their existing posts absent from the precomputed timeline, and backfilling is expensive. Because the read path already merges large accounts, extending it to merge recently followed accounts for a short window covers the gap at read time — and for most products, the simplest acceptable behaviour is that new follows appear in the feed from their next post onward.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Celebrity post fans out | Pipeline saturated for minutes | Threshold classification; never fan out above it |
| Fanout backlog grows | Posts appear minutes late | Lane separation by follower count; scale workers |
| Duplicate posts in a feed | Same item twice | Deduplicate by post id at the merge |
| Deleted post still displayed | Content that should be gone | Filter at hydration, not at write |
| Blocked user visible | Serious trust failure | Filter at hydration; treat as correctness, not cosmetics |
| Ranking service slow | Feed latency exceeds budget | Fall back to chronological ordering of candidates |
| Timeline store unavailable | No precomputed candidates | Serve from the read-time path only, degraded |
| Account crosses the threshold | Gaps or duplicates for its followers | Hysteresis plus deduplication |

> **Filtering failures for blocks are a trust failure, not a bug**  
> If a blocked account's content appears in someone's feed, the product has failed at something users explicitly asked it to do, and the harm is not proportional to the frequency — a single occurrence is memorable and reportable. This is why block filtering belongs at hydration, where it is applied on every read against current state, rather than at write time where it depends on a fanout having correctly excluded someone who may have been blocked afterwards.

The same reasoning applies to privacy transitions: an account that becomes private must stop appearing to non-followers immediately, and only read-time evaluation can guarantee that.

**The staff-level view**

The strong answer arrives at the hybrid by showing both pure strategies failing, then spends its time on ranking and filtering.

- **Show the arithmetic for both extremes** — a hundred million inserts for one post, two thousand queries for one feed — before proposing the hybrid.
- **Explain why the read-time merge stays small**: large accounts are rare, so any user follows only a few of them.
- **Restrict fanout to active users** and quantify the saving, since it is the largest and least-mentioned optimisation here.
- **Separate fanout into lanes** so that mid-sized accounts do not delay everyone else's posts.
- **Filter at hydration and say why**: blocks, deletions and privacy changes are read-time correctness concerns.
- **Reframe the timeline as a candidate source** once ranking exists, which justifies capping depth and tolerating imprecision.

> **The question that raises the level**  
> What is the acceptable staleness for a post appearing in a follower's feed? Seconds implies a fanout pipeline sized for peak with priority lanes; a minute permits batching and much cheaper infrastructure. The answer also determines whether new follows need backfill, and most products, asked directly, find that a minute is entirely acceptable for everything except the author's own view of their post.

---

### Durable Chat

*Deliver messages in order within a conversation, persist them before acknowledging, and let clients resume exactly where they left off.*

**Flow:** `Client connection` → `Connection registry` → `Message store` → `Fanout to members` → `Delivery receipts` → `History and sync`

**The brief**

Users exchange messages in one-to-one and group conversations. Messages must arrive quickly, appear in the same order for everyone in a conversation, survive the server restarting, and be retrievable on a device that has been offline for a week.

Clients are unreliable by nature — phones change networks, applications are killed, connections drop silently — so the difficult part is not sending a message but knowing what each device has actually received.

- **Ordering**: all participants see a conversation's messages in the same sequence.
- **Durability**: an acknowledged message is never lost.
- **Sync**: a client returning after any interval receives exactly what it missed.
- **Latency**: under 200 milliseconds from send to delivery for connected recipients.
- **Scale**: 50 million users, 5 million concurrent connections, 100,000 messages per second.

> **Per-conversation sequence numbers turn every hard problem into a cursor comparison**  
> If each message receives a monotonically increasing number within its conversation, then ordering is defined, gaps are detectable, sync is a range request, and read receipts are a single integer per participant. Almost everything that seems difficult about chat — ordering across devices, resumption, unread counts, missed messages — collapses into comparing positions in a sequence.

**Scale**

**Connection and message arithmetic**

```text
CONNECTIONS
  5,000,000 concurrent
  ~30 KB memory per connection
  -> 150 GB across the fleet
  -> at 100,000 connections per instance: 50
     instances
  -> connection count, not message rate, sizes the
     fleet

MESSAGES
  100,000/s at peak
  average group size 8
  -> 800,000 deliveries/s
  -> fanout is 8x the ingest rate

STORAGE
  100,000/s x 86,400 = 8.6 B messages/day
  ~400 B each with metadata
  -> ~3.4 TB/day
  -> partition by conversation, archive cold
     conversations

SYNC BURSTS
  a client offline for a week in a busy group
  -> potentially 50,000 missed messages
  -> must be paginated, not delivered as one payload

DELIVERY RECEIPTS
  every message x every recipient x 2 states
  (delivered, read)
  -> 1,600,000 receipt events/s at peak
  -> this is MORE traffic than the messages
  -> receipts must be batched and coalesced

RECONNECTION STORM
  one instance restarts -> 100,000 clients reconnect
  -> without jitter they arrive together
  -> with 50 instances, a rolling deploy repeats it
     50 times
```

| Metric | Value | Note |
|---|---|---|
| Connections | 5 M concurrent | **sizes the fleet** |
| Fanout | 8× ingest | group size |
| Receipts | 16× messages | batch and coalesce |
| Storage | 3.4 TB/day | partition and archive |

The receipt volume is the number that surprises people. Acknowledging delivery and reads for every recipient of every message generates an order of magnitude more events than the messages themselves, and sending each one individually to every participant makes a busy group conversation unusable. Coalescing receipts — sending the highest position seen rather than one event per message — reduces it to a trickle.

> **Rolling deploys are the largest recurring load event this system has**  
> Each instance restart disconnects its entire client population, and they all reconnect within seconds. With fifty instances the cycle repeats fifty times, and if reconnection is not jittered and drained gradually, every deploy becomes a self-inflicted overload — which teams frequently mistake for a capacity problem rather than a deployment one.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Ordering | Client timestamps, server timestamps, sequence per conversation | Sequence per conversation | Clocks disagree; a sequence is unambiguous and gap-detectable |
| Transport | WebSocket, long polling, push only | WebSocket with fallbacks | Bidirectional, low per-message overhead |
| Persistence timing | Acknowledge then store, store then acknowledge | Store then acknowledge | An acknowledged message must exist |
| Cross-server routing | Broadcast, registry, pub/sub | Pub/sub per conversation | Scales without every server knowing every connection |
| Sync model | Push missed messages, client pulls by cursor | Client pulls by cursor | The client knows what it has; the server should not have to track it |
| Receipts | Per message, high-water mark | High-water mark | Reduces receipt traffic by an order of magnitude |

1. **Assign a per-conversation sequence number on persist** — This is the ordering authority and the basis of sync.
2. **Persist before acknowledging to the sender** — An acknowledged message that was never stored is the worst failure available.
3. **Publish to a per-conversation channel for fanout** — Servers subscribe for the conversations their clients are in.
4. **Let clients resume with their last known sequence** — Sync becomes a range query with pagination.
5. **Send receipts as a high-water mark, batched** — Per-message receipts generate more traffic than messages.
6. **Deduplicate by client-generated message identifier** — Retries after an ambiguous send are certain.

Choosing a client-generated identifier alongside the server sequence solves a problem that otherwise has no clean answer. A client that sends a message and loses the connection before the acknowledgement does not know whether it was delivered; retrying with the same identifier lets the server recognise the duplicate and return the original sequence number, so the message appears once and the client learns its position.

> **Ask before building it**  
> Is history stored on the server or only on devices? End-to-end encryption with device-local history is a different system: the server cannot search, cannot assist sync with content, and a lost device means lost history. That decision changes the storage layer, the sync protocol and the product, and it cannot be retrofitted.

**Deep dive**

Three mechanisms carry this system: the sequence, the connection layer, and the sync protocol that reconciles them.

**Sending a message**

```text
CLIENT
  generate client_msg_id (uuid, persisted locally)
  send {conversation, client_msg_id, body}
  show it as 'sending'

SERVER
  1  authorise: is the sender a member?
  2  deduplicate: has this client_msg_id been seen?
       yes -> return the original sequence number
  3  persist:
       assign seq = next for this conversation
       write {conversation, seq, sender, body, ts}
       -> this is the durability point
  4  acknowledge to the sender with seq
  5  publish to the conversation channel

FANOUT
  each server subscribed to that conversation
  delivers to its connected members
  -> members offline: nothing to do; they will sync

WHY THE SEQUENCE IS ASSIGNED AT PERSIST
  it must be unique and gapless per conversation
  -> a single writer per conversation, or an atomic
     increment
  -> partitioning by conversation makes this cheap:
     different conversations never contend
```

**Persisting before acknowledging is the rule that cannot be relaxed.** A client shown a delivered message that the server never stored is the one failure users will not forgive, because they proceed on the assumption that it arrived. Acknowledging first and persisting asynchronously improves latency by a few milliseconds and introduces a window in which crashes lose messages that were confirmed — a trade no chat product should make.

**Sync and reconnection**

```text
CLIENT STATE
  per conversation: last_seq received
  plus a list of unacknowledged sends

ON RECONNECT
  for each conversation:
    GET /messages?conv=X&since=last_seq&limit=200
    -> paginated; a week offline may be thousands
  resend unacknowledged messages with their
  client_msg_ids
    -> deduplicated server-side

GAP DETECTION
  received seq 104 but last_seq was 101
  -> 102 and 103 are missing
  -> request them explicitly
  -> this is why gapless sequences matter: the
     client can see what it lacks without asking

HIGH-WATER RECEIPTS
  instead of "delivered 101, 102, 103, 104"
  send "delivered up to 104"
  -> one event replaces many
  -> and it is idempotent: resending is harmless

RECONNECTION BACKOFF
  exponential with jitter, always
  -> a deploy disconnects 100,000 clients per
     instance
  -> synchronised reconnection overwhelms the
     remaining fleet
```

**Gap detection is the property that makes delivery reliable without server-side tracking.** Because sequence numbers are contiguous within a conversation, a client that receives 104 after 101 knows precisely what it is missing and can ask for it. The server need not remember what each of five million devices has seen — the client carries its own cursor, which is both simpler and more robust than any server-side delivery table.

**Group size changes the cost model non-linearly and needs a policy.** A conversation with ten members is eight deliveries per message; one with five thousand is a broadcast, with receipt traffic to match. Large groups typically need different treatment — no per-member delivery receipts, possibly fanout on read rather than push — and defining the threshold at which behaviour changes is better than discovering it when a large group degrades the service for everyone on the same servers.

**Presence is a separate system that is easy to underestimate.** Online status, typing indicators and last-seen timestamps generate continuous traffic proportional to the number of connections and conversations rather than to messages, and they are ephemeral — nobody needs last week's typing indicators. Keeping presence in a separate, non-durable path with aggressive coalescing prevents it from consuming the capacity that message delivery needs.

**Media messages should not travel through the message path.** A photograph is uploaded directly to object storage using a presigned URL, and the chat message carries a reference. Routing megabytes through the connection layer inflates memory per connection, delays text messages behind large transfers, and makes the delivery path responsible for content it is not designed to handle.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Server restarts | Thousands of clients disconnect at once | Drain gradually; jittered reconnection |
| Acknowledgement lost | Client resends the same message | Deduplicate by client message id |
| Client offline for a week | Large sync payload | Paginate by sequence range |
| Gap in received sequence | Missing messages the client can detect | Explicit range request for the gap |
| Pub/sub unavailable | Messages persisted but not delivered live | Clients sync on reconnect; nothing is lost |
| Receipt volume overwhelming | More receipt traffic than messages | High-water marks, batched |
| Very large group | Fanout and receipts dominate a server | Different policy above a size threshold |
| Slow client | Send buffer grows on the server | Bound the buffer; disconnect and let it sync |

> **A message acknowledged but not persisted is the failure users remember**  
> Everything else in this system degrades visibly — messages arrive late, the application reconnects, sync takes a moment — and users tolerate all of it. A message shown as sent that simply does not exist breaks the product's central promise, and because the sender saw confirmation they will act as though it was received. This is why persistence precedes acknowledgement even at a latency cost, and why the durability of the message store is the one property not to compromise.

The corresponding client-side rule is that a message is not shown as sent until the server's acknowledgement carrying its sequence number arrives.

**The staff-level view**

The distinguishing content is the sequence number and everything it makes easy, plus honest treatment of connection management.

- **Introduce per-conversation sequences early** and show that ordering, gap detection, sync and receipts all follow from them.
- **Insist on persist-then-acknowledge**, and say plainly why the reverse is unacceptable in this product.
- **Put the cursor on the client**, so the server does not track what five million devices have seen.
- **Coalesce receipts into high-water marks**, since per-message receipts exceed message volume by an order of magnitude.
- **Treat deploys as the main load event** and require jittered reconnection and gradual draining.
- **Keep media out of the message path**, using direct uploads with references in the message.

> **The question that raises the level**  
> What is the largest group the product will support? The answer changes the architecture rather than the configuration: ten thousand members means per-message fanout and receipts are untenable, and the conversation behaves more like a broadcast channel with read-time assembly. Designing for small groups and discovering large ones later usually means building a second delivery path under pressure.

---

### Live Collaboration

*Let many people edit one document simultaneously, converging on the same result without locking and without losing anyone's work.*

**Flow:** `Client edits` → `Local apply` → `Operation stream` → `Server ordering` → `Rebroadcast` → `Snapshot and history`

**The brief**

Several people edit the same document at the same time. Each sees their own changes immediately, sees others' changes within a moment, and everyone ends up with an identical document — even though edits are made concurrently against slightly different views of the text.

The difficulty is that two people editing the same region produce operations whose meaning depends on what else has been applied. An insertion at position 20 means something different once someone else has inserted five characters at position 10.

- **Convergence**: all clients reach the same document state, always.
- **Responsiveness**: local edits appear instantly, with no round trip.
- **Intent preservation**: the result should reflect what each person meant, not merely be consistent.
- **Offline tolerance**: a client that disconnects can continue editing and reconcile on return.
- **Scale**: documents with up to 50 simultaneous editors, hundreds of operations per second per document.

> **Local changes must apply immediately, which is what creates the entire problem**  
> If every keystroke waited for the server, the editor would be unusable — so clients apply their own edits optimistically against a state the server has not yet seen. That optimism is the source of divergence, and the system's whole job is to reconcile it: reordering, transforming or merging operations so that a set of concurrent edits produces one agreed result.

**Scale**

**Operation and state arithmetic**

```text
OPERATION RATE
  50 editors, active typing ~5 chars/s each
  -> 250 operations/s per document
  -> batched into ~20 ms windows: ~50 messages/s
  -> batching is essential; per-keystroke messages
     are 5x the traffic for no benefit

FANOUT
  every operation goes to all other editors
  250 ops/s x 49 recipients = ~12,000 deliveries/s
  for ONE document
  -> documents must be sharded by document, and one
     server should own each active document

DOCUMENT STATE
  typical document 100 KB
  operation history unbounded if never compacted
  -> snapshot periodically, discard prior operations
  -> without snapshots, opening a year-old document
     replays millions of operations

SNAPSHOT CADENCE
  every 1,000 operations or 5 minutes
  -> opening cost = snapshot + a few hundred ops
  -> storage = snapshots + recent operations

OFFLINE RECONCILIATION
  a client offline for an hour may have 1,000 local
  operations and 5,000 remote ones
  -> transformation cost grows with the product
  -> bound how far behind a client may be before it
     reloads from a snapshot instead
```

| Metric | Value | Note |
|---|---|---|
| Ops per document | ~250/s | at 50 editors |
| Batching | 20 ms windows | **5× traffic saved** |
| Snapshots | every 1,000 ops | bounds load time |
| Offline limit | reload beyond N | transformation cost |

The concentration is what distinguishes this from other real-time systems: all the traffic for one document converges on whatever component orders it. That makes the document — not the user — the unit of partitioning, and it means a single very active document is a hot partition that cannot be split, because splitting it would destroy the ordering the algorithm depends on.

> **Unbounded operation history makes documents progressively slower to open**  
> If every operation is retained and replayed, a document edited over months takes longer to load each week, and eventually becomes unusable. Snapshotting at a regular cadence and discarding superseded operations bounds the load cost, and the absence of that cadence is a defect that only manifests after the product has been in use long enough for someone to have a large document.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Concurrency algorithm | Locking, operational transformation, CRDT | Server-ordered operational transformation for text | Locking destroys the experience; OT with a server is simpler than peer CRDTs |
| Ordering authority | Peer to peer, central server | Central server per document | One ordering removes the hardest transformation cases |
| Local application | Wait for server, apply optimistically | Optimistic with reconciliation | Responsiveness is the product |
| History | Full operation log, periodic snapshots | Snapshots plus recent operations | Bounds load time and storage |
| Offline editing | Block, queue operations | Queue with a bounded window | Beyond a threshold, reload rather than transform |
| Presence and cursors | In the operation stream, separate channel | Separate, ephemeral | Cursor updates are frequent and worthless to persist |

1. **Assign each active document a single ordering authority** — One server owns the sequence; everything else follows from it.
2. **Apply local edits immediately and mark them pending** — The user must never wait for a round trip.
3. **Transform incoming operations against pending local ones** — This is the reconciliation that produces convergence.
4. **Acknowledge with a sequence number so clients can retire pending operations** — A client needs to know when its edit is canonical.
5. **Snapshot regularly and prune superseded operations** — Load time must not grow with document age.
6. **Keep cursors and presence off the durable path** — They are high-frequency and valueless a second later.

Choosing a central ordering authority per document rather than a peer-to-peer algorithm removes most of the difficulty. With one server deciding the canonical sequence, each client only needs to transform against operations it has not yet seen, in a known order — whereas peer-to-peer convergence requires handling arbitrary orderings and is where operational transformation acquired its reputation for subtle, hard-to-reproduce bugs.

> **Ask before building it**  
> Is the content plain text, rich text, or a structured document? Text has well-understood algorithms; rich text with nested formatting has substantially harder transformation rules; arbitrary structured data may be better served by a CRDT designed for it. Choosing the algorithm before knowing the data model is how projects end up with an implementation that converges for typing and diverges for tables.

**Deep dive**

Two mechanisms determine whether this works: the transformation that reconciles concurrent edits, and the session management that lets clients join, leave and return.

**The concurrency problem, concretely**

```text
DOCUMENT: "Hello world"

SIMULTANEOUSLY
  A inserts "beautiful " at position 6
  B deletes 5 characters at position 6 ("world")

NAIVE APPLICATION
  A's view: "Hello beautiful world"
  B's view: "Hello "
  A receives B's delete at position 6
    -> deletes "beaut"
    -> "Hello iful world"      WRONG
  B receives A's insert at position 6
    -> "Hello beautiful "      DIFFERENT
  -> divergence

WITH TRANSFORMATION
  B's delete is transformed against A's insert:
    the insert added 10 characters at position 6
    -> the delete's position shifts to 16
  A applies delete(16, 5) -> "Hello beautiful "
  A's insert transformed against B's delete:
    position 6 is before the deleted range
    -> unchanged
  B applies insert(6, "beautiful ")
    -> "Hello beautiful "
  -> CONVERGENCE

THE RULE
  every operation must be transformed against every
  concurrent operation it has not yet seen
  -> the server's ordering defines what "concurrent"
     means, which is why central ordering simplifies
     this enormously
```

**Convergence and intent preservation are different goals, and only the first is guaranteed.** The algorithm ensures everyone ends with the same document; whether that document reflects what each person meant is a separate question with no universal answer. Two people replacing the same word concurrently converge on a result that one of them did not intend, and no transformation rule fixes that — which is why good editors show presence and cursors prominently, to prevent the collision rather than to resolve it.

**Session lifecycle**

```text
JOINING
  1  fetch the latest snapshot          (state + seq)
  2  fetch operations after that seq
  3  apply them locally
  4  subscribe to the live stream from that point
  -> the client is now current and can edit

EDITING
  local edit -> apply immediately, queue as pending
  send to server with the last seen sequence
  server: order it, assign seq, broadcast
  client: on acknowledgement, retire the pending op

RECEIVING
  incoming op with seq N
    transform against all still-pending local ops
    apply
    advance last-seen to N

RECONNECTING
  reconnect with last-seen sequence
    gap small  -> replay the missing operations
    gap large  -> discard local state, reload
                  snapshot, replay pending local
                  operations against it
  -> the threshold matters: transformation cost
     grows with the number of concurrent operations,
     so reloading is cheaper past some point

LEAVING
  pending operations must be flushed or they are
  lost
  -> warn on close while unacknowledged edits exist
```

**The reconnection threshold is a practical decision with a large effect.** Transforming a thousand local operations against five thousand remote ones is expensive and increases the chance of hitting an edge case in the transformation rules; reloading from a snapshot and reapplying only the pending local edits is cheap and predictable. Setting that threshold low — a few hundred operations — trades a moment of loading for a great deal of reliability.

**Presence and cursors carry more traffic than edits and must be handled separately.** A cursor position update per keystroke from fifty editors is thousands of messages per second, all of them worthless a moment later. Sending them on a separate ephemeral channel, coalesced to a few updates per second per editor and never persisted, keeps them from competing with the operations that actually matter.

**Access control must be enforced on every operation, not at session start.** Permissions change while a session is open — a user is removed from a document, a link is revoked — and a client that connected with access should not retain it indefinitely. Checking on each operation is inexpensive relative to the transformation work and closes a window that is otherwise open for as long as the browser tab stays open.

**The undo model is harder than it looks in a collaborative setting.** Undo should reverse the user's own last action, not the document's last change, which means each client maintains its own undo stack of operations that must themselves be transformed against everything that happened since. Products that implement undo as document-level rollback surprise users by reverting other people's work, and the correction is architectural rather than cosmetic.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Divergence between clients | Different documents, silently | Periodic checksum comparison; force resync on mismatch |
| Ordering server for a document fails | Edits stop being accepted | Reassign ownership; clients reconnect and resync |
| Client offline too long | Transformation cost explodes | Threshold to reload from snapshot instead |
| Operation history unbounded | Documents slow to open over time | Regular snapshots with pruning |
| Presence traffic dominates | Cursor updates crowding out edits | Separate channel, coalesced, ephemeral |
| Unacknowledged edits on close | The user loses recent work | Warn on close; flush before disconnect |
| Permission revoked mid-session | Edits accepted from a removed user | Authorise every operation |
| Very active single document | One hot partition | Cannot be split; scale vertically and batch harder |

> **Silent divergence is the failure this system must actively detect**  
> If transformation has a bug in a rare case, two clients end up with different documents and neither knows — both are editing happily, both believe they are correct, and the difference surfaces when someone reloads and sees unexpected content. Because nothing errors, the only defence is active detection: periodically exchanging a checksum of the document state and forcing a resynchronisation on mismatch, which converts an invisible corruption into a brief and recoverable reload.

This is the collaborative equivalent of replica divergence, and it deserves the same treatment: assume the reconciliation logic is imperfect and build a mechanism that finds out.

**The staff-level view**

The strong answer explains why optimistic local application creates the problem, then addresses convergence detection and session lifecycle rather than only the algorithm.

- **Start from responsiveness**: local edits must apply instantly, which is precisely what makes reconciliation necessary.
- **Work a concrete transformation example**, since the position-shifting is what makes the mechanism understandable.
- **Choose a central ordering authority per document** and justify it by how much transformation complexity it removes.
- **Separate convergence from intent preservation**, and say that presence indicators prevent collisions the algorithm cannot resolve.
- **Bound offline reconciliation** with a threshold beyond which the client reloads rather than transforms.
- **Propose active divergence detection** by checksum, because silent divergence is undetectable otherwise.

> **The question that raises the level**  
> What happens to a document with one extremely active editing session — fifty people in a live workshop? It is an unsplittable hot partition, since the ordering must be central, so the only levers are batching harder, coalescing more aggressively, and vertical capacity. Knowing that the architecture has no horizontal answer for this case, and saying so, is more useful than implying that sharding solves it.

---

### Video On Demand

*Ingest large uploads reliably, transcode into an adaptive ladder, and deliver segments from the edge so playback adapts to every connection.*

**Flow:** `Resumable upload` → `Object storage` → `Transcode pipeline` → `Segment and manifest` → `CDN delivery` → `Playback analytics`

**The brief**

Creators upload video files of arbitrary size from unreliable connections. Viewers watch on everything from a phone on a congested mobile network to a television on fibre, and expect playback to start within a couple of seconds and never stall.

Between those two points sits a transcoding pipeline whose cost and duration are proportional to the content, and a delivery network whose economics dominate the entire system's bill.

- **Upload**: multi-gigabyte files, resumable across network changes and application restarts.
- **Processing**: available for playback within minutes of upload for typical content.
- **Playback**: start within 2 seconds, adapt to bandwidth, rebuffer rarely.
- **Reach**: global delivery with no origin involvement in the common case.
- **Scale**: 100,000 uploads per day, 10 million viewing hours per day.

> **Segmentation turns video delivery into static file serving**  
> Once content is encoded at several bitrates and split into short aligned segments, every one of those segments is an immutable file that a CDN caches like any image. There is no streaming protocol at the origin, no per-viewer server state, and no connection to maintain — which is why global video delivery runs on the same infrastructure that serves web pages.

**Scale**

**Storage, compute and delivery arithmetic**

```text
INGEST
  100,000 uploads/day, average 500 MB
  -> 50 TB/day of source material
  -> upload bandwidth must not touch application
     servers

TRANSCODING
  one hour of source -> 5 renditions
  roughly 2-5x real time on CPU, faster on hardware
  100,000 uploads averaging 10 minutes
  = ~16,700 source hours/day
  at 2x real time with 500 parallel workers:
    16,700 x 2 / 500 = ~67 hours of wall clock
  -> 500 workers is not enough; needs ~1,500-2,000
  -> transcoding is the dominant compute cost

STORAGE
  source 50 TB/day (keep for reprocessing)
  renditions ≈ 2.3x the top bitrate
  10 min at 5 Mbps top ≈ 375 MB
    x 2.3 = ~860 MB per video
  100,000/day -> 86 TB/day of output
  -> tiering matters enormously: most videos are
     watched in their first week

DELIVERY
  10,000,000 viewing hours/day
  average 2 Mbps -> 900 MB/hour
  = 9,000 TB/day = 9 PB/day of egress
  -> at typical CDN pricing this dominates every
     other cost combined
  -> cache hit rate is the single most valuable
     metric in the system
```

| Metric | Value | Note |
|---|---|---|
| Transcode | dominant compute | ~2,000 workers |
| Output storage | 86 TB/day | tier aggressively |
| Egress | 9 PB/day | **dominates cost** |
| Lever | cache hit rate | worth more than anything |

The egress figure reframes the whole system. Compute and storage are significant and manageable; delivery bandwidth is an order of magnitude larger and is charged per byte. Every percentage point of cache hit rate, every reduction in delivered bitrate through better encoding, and every avoided re-download translates directly into money in a way that no backend optimisation does.

> **Storing every rendition of every video is affordable only with aggressive tiering**  
> Viewing is extremely skewed: a small fraction of videos accounts for most watch time, and most uploads are barely watched after their first week. Keeping full ladders in hot storage indefinitely multiplies the storage bill for content nobody requests, while moving cold content to cheaper tiers — or transcoding it on demand when it is requested — reduces it substantially at the cost of occasional first-view latency.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Upload path | Through the API, direct to storage | Direct with presigned multipart | Upload bandwidth must not size the application tier |
| Transcoding trigger | Synchronous, event-driven | Event-driven from storage notification | The client must not wait, and the event is authoritative |
| Ladder | Fixed, per-title | Fixed initially, per-title for popular content | Per-title analysis costs compute and pays off only at volume |
| Segment duration | 2 s, 4-6 s, 10 s | 4-6 seconds | Balances adaptation speed against request overhead |
| Delivery | Origin, CDN | CDN with origin shielding | Egress economics make anything else untenable |
| Cold content | Keep all renditions, transcode on demand | Tier cold, retain source | Most content is watched briefly and then never |

1. **Upload directly to object storage using presigned multipart URLs** — Resumable, parallel, and entirely off the application path.
2. **Trigger transcoding from a storage event, not a client callback** — Clients disappear; the storage event does not.
3. **Transcode into an aligned ladder with keyframes at segment boundaries** — Misalignment makes switching visible or impossible.
4. **Publish a manifest only when all renditions are complete** — A partially available ladder produces playback failures.
5. **Serve every segment through the CDN with long cache lifetimes** — Segments are immutable, so they can be cached indefinitely.
6. **Retain the source for reprocessing** — Codec improvements and ladder changes require re-encoding.

Triggering from a storage event rather than a client confirmation is a small decision with an outsized reliability effect. Clients close tabs, lose connectivity and crash after the final part uploads, and any pipeline depending on them to say the upload finished will silently lose videos — which the creator discovers, and the platform does not.

> **Ask before building it**  
> How quickly must a video be watchable after upload? Minutes permits a straightforward batch pipeline; seconds requires transcoding to begin while the upload is still arriving and to publish renditions progressively, which is a considerably more complex pipeline for a benefit most products do not need.

**Deep dive**

Three subsystems define this platform: ingest, the transcoding pipeline, and delivery economics.

**From upload to playable**

```text
1  UPLOAD
     client requests presigned multipart URLs
     uploads 8-16 MB parts in parallel, resumable
     completes the multipart upload
     -> bytes never touch application servers

2  STORAGE EVENT
     object store notifies the pipeline
     -> the authoritative signal that ingest finished

3  VALIDATION AND PROBE
     container, codec, duration, resolution
     reject unsupported or corrupt input early
     -> a failed transcode two hours in is expensive

4  TRANSCODE  (parallel by segment)
     split the source into chunks
     encode each chunk at each rendition, in parallel
     -> a 60-minute video becomes hundreds of
        independent tasks
     -> wall-clock time falls from hours to minutes
     reassemble and verify keyframe alignment

5  PACKAGE
     segment each rendition, write the manifest
     -> publish only when every rendition is done

6  PUBLISH
     mark the video available
     notify the creator
     warm the CDN for content expected to be popular

FAILURE AT ANY STAGE
  retry the failed chunk, not the whole video
  -> chunk-level granularity is what makes long
     videos tractable
```

**Parallel chunk transcoding is what makes processing time acceptable.** Encoding a one-hour video serially takes hours; splitting it into hundreds of independently encodable chunks and distributing them across workers reduces wall-clock time to minutes, and it makes failure cheap — a failed chunk is retried alone rather than restarting the whole job. The constraint is that chunk boundaries must align with keyframes so the reassembled renditions remain switchable.

**Delivery and its economics**

```text
SEGMENT REQUEST PATH
  player requests segment N at rendition R
  -> edge cache hit: served locally, ~10 ms
  -> miss: shield tier -> origin (object storage)

WHY THE HIT RATE DOMINATES
  9 PB/day of egress
  90% hit rate -> 0.9 PB/day from origin
  99% hit rate -> 0.09 PB/day
  -> a 9-point improvement reduces origin egress
     tenfold
  -> and origin egress is typically the most
     expensive byte in the system

WHAT DRIVES HIT RATE
  popularity skew (helps enormously)
  segment immutability (cache forever)
  consistent URLs (no query parameters that vary)
  pre-warming for anticipated demand

BITRATE IS THE OTHER LEVER
  better codecs deliver the same perceived quality
  at 30-50% fewer bytes
  -> directly proportional saving on 9 PB/day
  -> at the cost of more transcoding compute and
     limited device support
  -> which is why platforms encode multiple codec
     families and serve by device capability

STARTUP
  the first segment should be small and low bitrate
  -> fast first frame matters more to abandonment
     than initial quality does
```

**Codec choice is a direct trade of compute against bandwidth, and at this volume bandwidth wins.** A newer codec that reduces delivered bytes by a third saves three petabytes a day, which vastly exceeds the additional encoding cost — but only for devices that support it. The practical arrangement is encoding popular content into several codec families and selecting by device capability, which is more compute and more storage in exchange for a large and continuous delivery saving.

**Digital rights management, where required, adds startup latency and operational weight.** Encrypted segments need a licence acquired before playback begins, which is an extra round trip on the critical path to first frame, plus a licence service that must be highly available or nothing plays. It is a business requirement rather than a technical one, and its cost should be attributed accordingly rather than absorbed silently into the playback budget.

**Reprocessing is a first-class requirement, which is why the source must be retained.** Codec improvements, ladder changes and encoding bugs all require re-encoding existing content, and a platform that discarded sources after transcoding cannot take advantage of any of them. Source storage is a meaningful cost and it buys the ability to improve the entire catalogue's delivery economics later.

**Quality of experience metrics are the ones that matter, and they are client-side.** Startup time, rebuffering ratio and average delivered bitrate describe what viewers actually experience, and none of them is visible from server metrics — a perfectly healthy origin is compatible with terrible playback. Collecting these from players, segmented by device, region and network type, is what reveals whether the ladder and the delivery configuration are correct.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Upload abandoned mid-transfer | Orphaned parts consuming storage | Lifecycle rule to abort incomplete uploads |
| Client never confirms completion | Video uploaded but never processed | Trigger from storage events, not the client |
| Transcode chunk fails | Video stuck in processing | Retry the chunk; fail the job after bounded attempts |
| Manifest published early | Playback fails on missing renditions | Publish only when all renditions are complete |
| Keyframes misaligned | Switching glitches or fails | Verify alignment before publishing |
| Popular video not cached | Origin egress spike and cost | Pre-warm anticipated content; shield tier |
| Rebuffering on mobile | Viewers abandon | Add lower rungs; tune player switching |
| Storage costs growing faster than revenue | Full ladders retained for cold content | Tier aggressively; transcode on demand for the tail |

> **Publishing a manifest before every rendition exists produces failures that look like player bugs**  
> A player that requests a rendition listed in the manifest and receives a 404 may stall, fall back unpredictably, or fail entirely depending on its implementation — and the symptom appears on the client, in device-specific ways, for content that looks fine on the server. Making publication atomic, gated on every rendition being complete and verified, removes an entire category of irreproducible playback reports.

The same principle applies to any change: a new rendition added later should be published by replacing the manifest atomically, never by editing it in place while players are reading it.

**The staff-level view**

The strong answer is grounded in the cost structure, because delivery economics determine most of the important decisions.

- **Lead with the egress figure.** Nine petabytes a day makes cache hit rate and delivered bitrate the dominant optimisation targets.
- **Keep uploads off the application tier** with presigned multipart, and trigger processing from storage events rather than clients.
- **Describe chunk-parallel transcoding** as the mechanism that makes processing time and failure recovery acceptable.
- **Insist on atomic manifest publication** and keyframe alignment, since both produce client-side failures that are hard to diagnose.
- **Tier storage by popularity** and retain sources for reprocessing, which is what allows future codec gains.
- **Measure quality of experience from players**, because server metrics say nothing about whether playback is good.

> **The question that raises the level**  
> What fraction of watch time comes from the top one per cent of videos? That single number decides how aggressively to tier storage, whether per-title encoding is worth its compute, how much pre-warming matters, and whether transcoding the long tail on demand is viable. Most platforms have an extreme answer, and designing without knowing it means either over-provisioning for content nobody watches or under-serving the content everyone does.

---

### Live Video Broadcast

*Ingest a continuous stream, transcode it in real time, and distribute segments to millions of concurrent viewers with bounded latency.*

**Flow:** `Broadcaster ingest` → `Real-time transcode` → `Segment packager` → `Manifest updates` → `CDN distribution` → `Chat and interaction`

**The brief**

A broadcaster sends a continuous video stream. Millions of viewers watch it simultaneously, arriving and leaving at unpredictable times, and expect to be close enough to live that chat and reactions make sense in context.

Everything that makes on-demand video tractable — processing in advance, caching indefinitely, retrying failed work — is unavailable. The content does not exist until moments before it is watched, and there is no second chance at any step.

- **Latency**: viewers within a few seconds of live, with chat aligned to what they see.
- **Scale**: up to 5 million concurrent viewers on a single popular stream.
- **Reliability**: an ingest interruption must degrade rather than end the broadcast.
- **Adaptivity**: the same ladder of qualities as on-demand, produced in real time.
- **Recording**: the stream is also available afterwards as on-demand content.

> **Everything is the same as on-demand except that there is no time**  
> The pipeline shape is identical — ingest, transcode, segment, manifest, CDN — but each stage must complete in less time than the segment it is producing, permanently. That single constraint removes retries, removes buffering as a recovery mechanism, and makes every stage's worst case the system's worst case, which is what makes live broadcasting a different engineering problem rather than a variation.

**Scale**

**Latency budget and delivery arithmetic**

```text
LATENCY BUDGET  (target ~6 s to live)
  capture and encode at source   1-2 s
  network to ingest              0.5 s
  transcode ladder               1-2 s
  segment and publish            = segment duration
  CDN propagation                0.2 s
  player buffer                  2-3 segments
  -> with 2 s segments and 3 buffered: ~10 s
  -> with 2 s segments and 1 buffered: ~6 s, less
     resilient
  -> low-latency variants using partial segments
     reach 2-3 s

THE BUFFER TRADE
  deeper buffer  = more latency, fewer stalls
  shallow buffer = closer to live, stalls on any
                   hiccup
  -> this is a product decision, not a technical one

DELIVERY AT 5 M CONCURRENT
  average 3 Mbps
  5,000,000 x 3 Mbps = 15 Tbps
  -> only a CDN can serve this
  -> and every viewer requests the SAME segments at
     nearly the same moment

THE SYNCHRONISED REQUEST PATTERN
  a new segment becomes available
  -> millions of players request it within ~1 s
  -> cache miss at the edge means a stampede to the
     origin
  -> request collapsing at the edge is mandatory,
     not optional

TRANSCODE CAPACITY
  one stream, 5 renditions, real time
  -> must never fall behind; there is no catching up
  -> provision for peak, with failover encoders
```

| Metric | Value | Note |
|---|---|---|
| Latency | ~6 s achievable | **buffer is the dial** |
| Peak delivery | 15 Tbps | at 5 M viewers |
| Requests | synchronised | collapsing required |
| Transcode | real time, always | no catching up |

The synchronised request pattern is what distinguishes live delivery from on-demand at the CDN. With on-demand content, viewers are scattered across a catalogue and across time; with a live stream, every viewer requests the same segment within the same second, so a single cache miss at an edge location can generate millions of origin requests. Request collapsing — where an edge fetches once and serves everyone waiting — is the mechanism that makes this survivable.

> **There is no retry in a live pipeline**  
> A segment that fails to encode cannot be re-encoded later, because by then viewers have moved past it. Every stage must therefore be provisioned for its worst case rather than its average, with redundant encoders running in parallel rather than failover that takes seconds — since a few seconds of gap is visible to every viewer simultaneously.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Ingest protocol | RTMP, SRT, WebRTC | SRT or RTMP for broadcast, WebRTC for sub-second | Reliability over unmanaged networks versus latency |
| Segment duration | 1 s, 2 s, 4-6 s | 2 s, or partial segments | Directly sets the latency floor |
| Transcoding | Single encoder, redundant encoders | Redundant, running in parallel | Failover time is visible to everyone at once |
| Delivery | CDN, peer-assisted | CDN with request collapsing | 15 Tbps has no other answer |
| Latency target | Standard (~10 s), low (~3 s) | Standard by default, low where interaction requires it | Lower latency costs resilience |
| Recording | Separate capture, reuse segments | Reuse the live segments | The segments already exist; re-encoding wastes compute |

1. **Accept ingest on a protocol tolerant of unmanaged networks** — Broadcasters are not in data centres; packet loss is normal.
2. **Run redundant encoders in parallel from the same ingest** — Failover measured in seconds is visible to every viewer.
3. **Produce short segments, or partial segments for low latency** — Segment duration is the dominant term in the latency budget.
4. **Publish manifest updates atomically** — A manifest referencing a segment that is not yet available breaks players.
5. **Enable request collapsing and shielding at the edge** — Millions of synchronised requests otherwise reach the origin.
6. **Retain the live segments as the recording** — The on-demand version is the same files with a static manifest.

Running redundant encoders in parallel rather than relying on failover reflects the absence of retries. A standby encoder that takes five seconds to take over produces a five-second gap for every viewer simultaneously — one of the most visible failures the product can have — whereas two encoders producing the same output continuously means one can disappear without anyone noticing.

> **Ask before building it**  
> How interactive is the experience? A broadcast people watch passively is fine at ten seconds behind live and benefits from the resilience that a deeper buffer provides. Anything where viewers interact with the broadcaster in real time needs three seconds or less, which costs resilience, increases infrastructure complexity, and should be adopted only where the interaction genuinely requires it.

**Deep dive**

Three areas determine whether a live system works: the ingest and encoding path, the delivery of synchronised requests, and graceful degradation when something interrupts.

**The live pipeline**

```text
INGEST
  broadcaster -> nearest ingest point
  protocol tolerant of loss and jitter
  -> authenticate the stream key
  -> immediately replicate to a second region

ENCODE  (redundant, parallel)
  encoder A and encoder B both consume the ingest
  both produce the full ladder
  packager selects A, falls back to B seamlessly
  -> no failover gap

PACKAGE
  segment each rendition, aligned across the ladder
  write segments to origin storage
  update the manifest
  -> manifest update must be atomic and must never
     reference a segment that is not yet readable

DISTRIBUTE
  players poll the manifest every segment duration
  request the next segment
  -> edge serves from cache
  -> on miss, collapse concurrent requests into one
     origin fetch

RECORD
  the same segments, with a static manifest written
  at the end, become the on-demand asset
  -> no re-encoding, no separate capture path

WHAT MAKES THIS FRAGILE
  every stage is on the critical path, continuously
  -> there is no queue to absorb a hiccup
  -> and no opportunity to retry anything
```

**Manifest publication is the synchronisation point and the most common source of player failures.** Players poll it to discover new segments, so a manifest listing a segment that has not finished writing produces a burst of failed requests from every viewer at once. Writing the segment first, verifying it is readable, then updating the manifest atomically is the ordering that prevents it, and getting it wrong produces errors proportional to the audience.

**Handling interruptions**

```text
INGEST DROPS  (broadcaster's connection fails)
  0-5 s     player buffer covers it; nobody notices
  5-30 s    players exhaust the buffer and stall
            -> insert a holding slate or the last
               frame
            -> keep the manifest alive so players do
               not give up
  > 30 s    end the stream gracefully
            -> tell viewers; do not leave them
               staring at a frozen frame

ENCODER FAILS
  redundant encoder continues
  -> viewers see nothing

ORIGIN STORAGE SLOW
  segments published late
  -> players stall despite everything upstream
     working
  -> this is why origin write latency is monitored
     as tightly as encode latency

REGIONAL CDN PROBLEM
  viewers in one region stall
  -> steer to another region; latency rises but
     playback continues

THE PRINCIPLE
  degrade visibly and informatively
  -> a slate saying "reconnecting" is far better
     than a frozen frame, because it tells the
     viewer the situation is known
```

**Player behaviour during interruptions is part of the system design, not a client detail.** A player that gives up after two failed manifest requests will disconnect millions of viewers during a brief hiccup, and they will not all return. Keeping the manifest available with a holding segment, and instructing players to retry with backoff, turns a short interruption into a pause rather than the end of the broadcast.

**Chat and reactions must be aligned to the video timeline or they become confusing.** Viewers at different latencies see different moments, so a reaction sent by someone two seconds ahead arrives before the event it refers to for everyone behind them. Timestamping interactions against the video position rather than wall-clock time, and delivering them accordingly, keeps the experience coherent — and it is a requirement that often drives the decision to pursue lower latency in the first place.

**Recording should reuse the live segments rather than capturing separately.** The segments already exist, are already encoded at every rendition, and are already stored; producing the on-demand asset is a matter of writing a static manifest over them. Systems that run a parallel recording pipeline pay for encoding twice and introduce a second thing that can fail during the broadcast.

**Scale is bounded by the CDN, and the origin must be sized for the miss rate rather than the audience.** Five million viewers generate essentially the same request pattern as fifty thousand from the origin's perspective, provided collapsing works — so origin capacity is a function of edge locations and cache behaviour, not of audience size. Verifying that collapsing actually functions, before an event rather than during one, is the difference between a comfortable broadcast and an origin outage.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Broadcaster connection drops | Stream gaps; players stall | Holding slate; keep the manifest alive; end gracefully after a threshold |
| Encoder fails | Gap visible to everyone at once | Redundant encoders in parallel, not failover |
| Manifest references a missing segment | Mass request failures | Write segment, verify, then update atomically |
| Cache miss at segment publication | Millions of origin requests in one second | Request collapsing and shielding |
| Origin write latency spikes | Segments published late; players stall | Monitor write latency as a first-class metric |
| Player gives up too easily | Mass disconnection on a brief hiccup | Retry with backoff; keep the manifest served |
| Chat ahead of the video | Spoilers and confusion | Align interactions to video position |
| Low-latency mode on a poor network | Frequent stalls | Fall back to a deeper buffer automatically |

> **A failure in a live system affects every viewer simultaneously and cannot be retried away**  
> On-demand failures are individual and recoverable — one viewer retries, the segment is still there. A live failure hits the entire audience at the same instant, and the content that was missed no longer matters by the time anything could be fixed. This is why the design uses redundancy rather than retry at every stage, and why the operational posture during a broadcast is watching rather than responding.

It is also why holding slates and clear messaging matter more than they would elsewhere: when the system cannot recover the moment, the best remaining outcome is a viewer who understands what is happening and stays.

**The staff-level view**

The strong answer builds the latency budget explicitly and treats the synchronised request pattern as the distinguishing delivery problem.

- **Lay out the latency budget stage by stage**, and identify segment duration and buffer depth as the two dials.
- **Frame the buffer as a product decision**: closer to live means more stalls, and the right answer depends on interactivity.
- **Name the synchronised request pattern** and require request collapsing, since it is what separates live from on-demand delivery.
- **Use redundant encoders rather than failover**, because any gap is visible to the entire audience at once.
- **Order segment write before manifest update**, atomically, to avoid mass player failures.
- **Design the degradation** — holding slate, manifest kept alive, graceful end — rather than leaving it to the player.

> **The question that raises the level**  
> What does a viewer see during a ten-second ingest interruption? The answer reveals whether the design has been thought through past the happy path. A frozen frame, a spinner, a slate with an explanation and an automatically resumed stream are very different experiences, and the last one requires deliberate work at the packager, the manifest and the player — none of which happens unless someone asked this question first.

---

### Resumable File Drive

*Sync files across devices with content-addressed chunks, so only changed blocks move and interrupted transfers resume where they stopped.*

**Flow:** `Client watcher` → `Chunker and hasher` → `Metadata service` → `Chunk store` → `Sync protocol` → `Conflict resolution`

**The brief**

Users keep files in a folder that stays synchronised across their devices and is shareable with others. Files range from documents to multi-gigabyte videos, devices go offline for days, and the same file may be edited on two devices while both are disconnected.

Moving whole files is unaffordable: changing one paragraph in a large document would re-upload the entire thing, and doing that on every save across every device multiplies bandwidth and storage far beyond what the content warrants.

- **Efficiency**: only the changed portions of a file are transferred.
- **Resumability**: an interrupted transfer continues rather than restarting.
- **Convergence**: all devices eventually hold identical content.
- **Conflict handling**: concurrent edits are preserved, never silently discarded.
- **Scale**: 50 million users, 500 million files, 10 PB stored.

> **Content-addressed chunks make deduplication, resumption and sync the same mechanism**  
> If files are split into chunks named by the hash of their contents, then a device knows exactly which chunks it already has, the server knows which ones exist globally, and a transfer is simply the set difference. Deduplication across users, incremental sync, and resumption after interruption all fall out of that one representation rather than requiring three separate mechanisms.

**Scale**

**Chunking, storage and sync arithmetic**

```text
CHUNKING
  4 MB fixed chunks, or content-defined boundaries
  500,000,000 files averaging 2 MB
  -> ~1 billion chunks before deduplication

DEDUPLICATION
  typical cross-user duplication 20-40%
  (shared documents, common assets, re-uploads)
  -> 10 PB logical becomes 6-8 PB physical
  -> the saving is larger for organisations than for
     consumers

METADATA
  per file: path, size, version, chunk list
  a 2 MB file = 1 chunk reference (32 B)
  a 4 GB file = 1,000 chunk references (32 KB)
  500 M files -> metadata is tens of GB
  -> metadata operations vastly outnumber chunk
     transfers

EDIT EFFICIENCY
  4 GB video, 10 MB appended
  fixed chunks: 3 chunks change (boundary shift is
  contained because the change is at the end)
  document, 1 paragraph edited in the middle
    fixed chunks  -> every chunk after the edit
                     shifts: nearly the whole file
    content-defined -> ~1-2 chunks change
  -> content-defined chunking is what makes
     mid-file edits cheap

SYNC CHATTER
  50 M users x 10 devices-checks/hour
  -> metadata polling dominates request volume
  -> push notification of changes is essential
```

| Metric | Value | Note |
|---|---|---|
| Chunks | ~1 B | before dedup |
| Dedup saving | 20-40% | cross-user |
| Mid-file edit | 1-2 chunks | **content-defined** |
| Dominant traffic | metadata | not chunk bytes |

The distinction between fixed and content-defined chunk boundaries is the single largest efficiency decision. With fixed boundaries, inserting text in the middle of a file shifts every subsequent boundary and effectively rewrites the file; with boundaries chosen by content, the shift is absorbed within one or two chunks and everything after realigns. The cost is a rolling hash over the data, which is cheap relative to the bandwidth it saves.

> **Metadata operations, not file transfers, generate most of the load**  
> Clients check for changes far more often than files actually change, so a system that polls produces enormous request volume for information that is usually unchanged. Push notification of changes, with polling only as a fallback, moves the system from a request rate proportional to devices times frequency to one proportional to actual change — a difference of several orders of magnitude.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Chunk boundaries | Fixed size, content-defined | Content-defined | Mid-file edits otherwise rewrite everything after them |
| Chunk naming | Sequential ids, content hash | Content hash | Deduplication, integrity and resumption all follow |
| Metadata store | Same store as chunks, separate database | Separate, transactional | Metadata needs queries and consistency; chunks need volume |
| Change notification | Client polling, server push | Push with polling fallback | Polling volume dwarfs actual change |
| Conflict handling | Last write wins, preserve both | Preserve both as distinct versions | Silently discarding someone's work is unacceptable |
| Deduplication scope | Per user, global | Global, with care | Much larger saving; requires handling the privacy implications |

1. **Chunk with content-defined boundaries and name chunks by hash** — This one decision provides deduplication, integrity and resumability.
2. **Keep file metadata in a transactional store separate from chunk storage** — Versions and directory structure need consistency that object storage does not provide.
3. **Upload only chunks the server does not already have** — Ask first with hashes; transfer the difference.
4. **Notify devices of changes rather than having them poll** — Change notification is the difference between manageable and enormous request volume.
5. **Preserve both sides of a conflict as named versions** — Convergence must not come at the cost of losing work.
6. **Reference-count chunks before deleting** — A chunk may be shared by many files and many users.

Global deduplication is where efficiency and privacy meet, and it needs a deliberate answer. Storing one copy of a chunk shared by thousands of users is a large saving; it also means the server can determine that a user possesses a file identical to one it has seen elsewhere, and a naive implementation lets a client claim ownership of any file whose hash it knows. Requiring proof of possession — or restricting deduplication to within an account or organisation — resolves it, at a cost in savings.

> **Ask before building it**  
> Do users share folders with others, or only sync their own devices? Sharing changes the model substantially: permissions become per-folder and mutable, a change by one user must notify others, and conflicts arise between people rather than between one person's devices. Building single-user sync and adding sharing later usually means revisiting the metadata model entirely.

**Deep dive**

Three mechanisms carry this system: chunking and deduplication, the sync protocol, and conflict handling.

**Uploading a changed file**

```text
CLIENT DETECTS A CHANGE
  1  chunk the file with content-defined boundaries
  2  hash each chunk
  3  ask the server: which of these hashes do you
     already have?
       -> one request, list of hashes
  4  upload only the missing chunks
       -> multipart, resumable, parallel
  5  commit a new file version referencing the full
     chunk list

WHAT THIS BUYS
  one paragraph edited in a 50 MB document
    -> 1-2 chunks uploaded, a few MB
    -> versus 50 MB with whole-file sync
  a file identical to one already stored
    -> zero chunks uploaded; instant "upload"

RESUMPTION
  the chunk list is the transfer plan
  an interrupted upload resumes by asking again
  which chunks are missing
  -> no separate resumption protocol is needed

INTEGRITY
  the chunk name IS its hash
  -> corruption is detectable on read
  -> and a corrupted chunk can be re-fetched

VERSIONING
  each version is a list of chunk references
  -> old versions cost only the chunks that changed
  -> file history is nearly free
```

**File history comes almost free from this representation, which is worth exploiting.** Because a version is a list of chunk references and unchanged chunks are shared, keeping the previous ten versions of a document costs only the chunks that actually differed. Products that implement versioning separately, by copying files, pay far more for a feature the storage model already provides.

**Sync and conflicts**

```text
SYNC STATE PER DEVICE
  the highest change sequence it has applied
  -> the server keeps an ordered change log per
     account

ON RECONNECT
  GET /changes?since=<seq>
  -> ordered list of file changes
  -> apply in order; fetch missing chunks
  -> a device offline for a week gets a bounded,
     paginated catch-up

CONFLICT: the same file edited on two devices
  both offline, both edit document.txt
  both reconnect

  LAST WRITE WINS
    -> one person's work disappears silently
    -> unacceptable for a product people trust with
       their files

  PRESERVE BOTH
    document.txt                   (server version)
    document (Device B's copy).txt (conflicted copy)
    -> both users keep their work
    -> the human resolves it
    -> ugly, honest, and correct

DETECTING THE CONFLICT
  each version records its parent version
  if the incoming change's parent is not the current
  version -> concurrent edit -> conflict
  -> this is why versions are a chain, not a
     timestamp
```

**Parent-version tracking is what makes conflict detection reliable.** Comparing timestamps cannot distinguish a genuine concurrent edit from a device with a skewed clock, whereas recording which version each edit was based on makes concurrency explicit: an edit whose parent is not the current version was made without seeing the latest state. That is exactly the condition a conflict describes, and it needs no clock at all.

**Deletion requires reference counting and is easy to get wrong.** A chunk may belong to many files across many users, so removing a file cannot remove its chunks — only decrementing their references and collecting those that reach zero is safe. The dangerous failure is a race between a deletion collecting a chunk and a concurrent upload deduplicating against it, which produces a file referencing a chunk that no longer exists; resolving it requires either a grace period before collection or coordination between the two paths.

**Selective sync and large files force a distinction between metadata and content.** A device should be able to see the whole folder structure without storing every byte, which means metadata syncs completely while content syncs on demand. This makes the metadata store the component that must always be available and consistent, while chunk storage can be slower, cheaper and eventually consistent — a separation that also matches their very different access patterns.

**Bandwidth on the client is a resource the design must respect.** Uploading continuously at full speed makes the application unpleasant to run, so transfers need throttling, scheduling around active use, and sensitivity to metered connections. This is not a detail of polish: a sync client that saturates someone's connection is uninstalled, regardless of how efficient its chunking is.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Upload interrupted | Partial transfer | Resume by re-querying missing chunks |
| Concurrent edits on two devices | Divergent content | Detect by parent version; preserve both copies |
| Chunk collected while being deduplicated | File references a missing chunk | Grace period before collection; verify on commit |
| Metadata and chunks inconsistent | File appears but cannot be downloaded | Commit metadata only after chunks are durable |
| Device offline for weeks | Very large catch-up | Paginated change log; snapshot beyond a threshold |
| Client claims a chunk it does not have | Unauthorised access to content | Proof of possession, or scope deduplication |
| Sync loop | The same file uploaded repeatedly | Record the applied version; ignore self-originated changes |
| Client saturates the connection | Unusable device | Throttle; respect metered networks |

> **Committing file metadata before every chunk is durable creates files that cannot be downloaded**  
> If the version record is written first, a failure before the last chunk uploads leaves a file that appears in every device's listing and fails whenever anyone opens it. Because the failure is at read time and often on a different device, it is reported as a mysterious corruption rather than as an upload problem. Metadata must be committed only after all referenced chunks are confirmed durable, which makes the file's appearance and its availability the same event.

The same ordering argument applies to deletion: remove the metadata first and collect chunks afterwards, so no window exists in which a file references something already gone.

**The staff-level view**

The strong answer derives deduplication, resumption and integrity from one representation rather than treating them as separate features.

- **Lead with content-addressed chunking** and show that deduplication, resumption and integrity all follow from it.
- **Distinguish fixed from content-defined boundaries** with the mid-file edit example, since it is the largest efficiency decision.
- **Separate metadata from chunk storage**, and require metadata to be committed only after chunks are durable.
- **Detect conflicts by parent version**, not by timestamp, and preserve both sides rather than discarding work.
- **Replace polling with push notification**, because metadata chatter dominates request volume.
- **Raise deduplication's privacy implication** and propose proof of possession or scoped deduplication.

> **The question that raises the level**  
> What should happen when two people edit a shared document offline and both reconnect? Answering preserve both copies is correct and incomplete — the interesting follow-up is how the product communicates it, because a silently created conflicted copy that nobody notices is nearly as bad as losing the edit. The systems people trust with their files make conflicts visible and resolvable, which is a design requirement rather than an interface detail.

---

### Searchable Knowledge Base

*Index documents for both keyword and semantic retrieval, enforce permissions at query time, and rank results people actually find useful.*

**Flow:** `Document sources` → `Ingestion and chunking` → `Inverted index` → `Vector index` → `Hybrid retrieval` → `Permission filter and rank`

**The brief**

An organisation's documents — wikis, tickets, design records, chat archives — must be searchable by everyone who is allowed to see them. Users ask both precise questions containing exact terms and vague ones describing a concept in their own words.

Permissions are the complication that distinguishes this from public search. Documents have per-user and per-group access rules that change continuously, and a search result revealing the existence of a document someone may not see is itself a disclosure.

- **Relevance**: useful answers for both keyword and conceptual queries.
- **Authorisation**: results must respect current permissions, with no leakage through titles or snippets.
- **Freshness**: an edited document is searchable within a minute.
- **Scale**: 10 million documents, 50,000 employees, 500 queries per second.
- **Explainability**: users should understand why a result appeared.

> **Permissions must be evaluated against current state, not baked into the index**  
> Access changes far more often than documents do, and a permission encoded at index time is wrong the moment someone leaves a team. Retrieving candidates broadly and filtering against live authorisation immediately before returning results is the only approach that stays correct — and it makes over-fetching a design requirement, because filtering removes an unpredictable share of every result set.

**Scale**

**Index and query arithmetic**

```text
CORPUS
  10,000,000 documents
  average 5 KB of text -> 50 GB raw
  inverted index ≈ 30-50% of raw text -> ~20 GB
  -> comfortably fits; this is not a sharding
     challenge for keyword search

VECTOR INDEX
  documents chunked into ~500-token passages
  10 M docs -> ~40 M passages
  768-dimension embeddings, 4 B per dimension
  = ~3 KB per passage
  -> ~120 GB of vectors
  -> this, not the inverted index, drives memory
  -> quantisation reduces it 4-8x with modest recall
     loss

QUERIES
  500/s
  hybrid: keyword retrieval + vector retrieval +
  merge + permission filter + rank
  latency budget 300 ms
    keyword        30 ms
    vector         50 ms
    merge          5 ms
    permissions    40 ms
    rank           80 ms
    overhead       30 ms

OVER-FETCH FOR PERMISSIONS
  a user may be permitted to see 20% of the corpus
  want 10 results -> retrieve 200+ candidates
  -> over-fetch factor is the inverse of the
     permitted fraction
  -> for restrictive users this becomes very large

EMBEDDING COST
  10 M docs x 4 passages = 40 M embeddings
  re-embedding the corpus on a model change is a
  multi-day batch job
  -> model version is a migration, not a config
     change
```

| Metric | Value | Note |
|---|---|---|
| Inverted index | ~20 GB | not the constraint |
| Vector index | ~120 GB | **drives memory** |
| Over-fetch | inverse of access | can be large |
| Model change | days to re-embed | a migration |

The over-fetch factor is the number that causes trouble in practice. A user with access to a fifth of the corpus needs roughly five times the candidates to fill a page, and a user with access to one per cent needs a hundred times — at which point retrieving broadly and filtering becomes impractical and the system needs permission-aware retrieval instead, usually by restricting the candidate set to accessible collections before scoring.

> **Changing the embedding model invalidates the entire vector index**  
> Embeddings from different models are not comparable, so upgrading means re-embedding every passage and rebuilding the index — days of compute and a careful cutover where both indexes exist simultaneously. Treating the model version as a configuration value rather than as a schema version leads to mixed indexes that silently return poor results for the portion embedded with the old model.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Retrieval | Keyword, vector, hybrid | Hybrid with fused ranking | Each fails where the other succeeds |
| Permission enforcement | Indexed, filtered at query time | Filtered at query time | Access changes constantly; indexed permissions go stale |
| Chunking | Whole documents, passages | Passages with document context | Long documents dilute embeddings and hurt precision |
| Freshness | Batch reindex, incremental | Incremental from change events | A minute is achievable; a nightly rebuild is not |
| Ranking | Retrieval score only, reranking model | Rerank the top candidates | Retrieval scores are weak signals of usefulness |
| Snippets | From the index, from the source | From the source, after authorisation | Snippets can leak content the user may not see |

1. **Ingest from each source with its permission model captured as metadata** — Authorisation data must travel with the document.
2. **Chunk documents into passages and embed each** — Retrieval targets passages; results present documents.
3. **Index into both a keyword index and a vector index** — The two retrieve different things and complement each other.
4. **Retrieve generously from both, fuse the rankings** — Reciprocal rank fusion is simple and works well.
5. **Filter against live permissions before anything is returned** — Including titles, snippets and counts.
6. **Rerank the surviving candidates with a stronger model** — Retrieval scores order poorly; reranking is where quality comes from.

Filtering before generating snippets matters more than it appears. A snippet is content, so producing it for a document the user cannot see and then discarding the result still means the content was assembled — and in systems where snippets are cached or logged, it can escape. Authorising first and extracting text afterwards makes the boundary unambiguous.

> **Ask before building it**  
> How restrictive are permissions in practice? If most documents are visible to most people, broad retrieval with filtering works well. If access is highly compartmentalised, the over-fetch factor becomes unworkable and the retrieval layer itself must be permission-aware — a different and more constrained design that should be chosen deliberately rather than discovered under load.

**Deep dive**

Three areas determine quality here: hybrid retrieval, permission enforcement, and the ranking that turns candidates into answers.

**Why hybrid retrieval outperforms either half**

```text
KEYWORD SEARCH
  strong: exact terms, identifiers, error codes,
          names, rare words
  weak:   paraphrase, synonyms, conceptual questions
  query "SIGSEGV in payment worker"
    -> finds documents containing those exact tokens

VECTOR SEARCH
  strong: meaning, paraphrase, questions in natural
          language
  weak:   exact identifiers, rare tokens, negation
  query "why does the payment service crash"
    -> finds documents about crashes without those
       words

THE FAILURE EACH HAS ALONE
  keyword misses the user who does not know the
  terminology
  vector misses the user searching for an exact error
  string, because embeddings blur rare tokens

FUSION
  retrieve top 100 from each
  combine with reciprocal rank fusion:
    score(d) = sum over sources of 1 / (k + rank)
  -> no score normalisation needed between
     incomparable systems
  -> documents ranked well by either source surface
  -> documents ranked well by BOTH rise to the top

WHY NOT JUST VECTORS
  internal knowledge bases are full of identifiers,
  commands and codes that embeddings handle poorly
  -> keyword retrieval is not legacy; it is the half
     that gets those right
```

**Reciprocal rank fusion is the practical way to combine incomparable scoring systems.** Keyword relevance scores and vector similarity scores live on different scales with different distributions, so normalising them into a weighted sum requires tuning that does not transfer between queries. Fusing on rank position rather than score avoids the problem entirely and performs well without calibration, which is why it is the common default.

**Permissions at query time**

```text
THE REQUIREMENT
  no evidence of a document the user cannot see
  -> not in results
  -> not in titles or snippets
  -> not in result counts or facet totals

THE FLOW
  1  retrieve candidates (over-fetched)
  2  resolve the user's current access:
       groups, roles, per-document grants
       -> cached per user, short lifetime
  3  filter candidates against it
  4  if too few survive, retrieve more
  5  generate snippets ONLY for survivors
  6  rank and return

PERMISSION-AWARE RETRIEVAL  (for restrictive cases)
  attach an access token set to each indexed document
  include the user's token set as a filter in the
  retrieval query itself
  -> the index returns only permitted candidates
  -> much better over-fetch behaviour
  -> but the tokens must be kept current, which
     reintroduces staleness
  -> the usual compromise: coarse tokens in the
     index, fine-grained check afterwards

REVOCATION
  access removed -> the user's cached permissions
  must expire quickly
  -> short cache lifetimes, or explicit invalidation
     on change
```

**The coarse-plus-fine approach is the practical resolution for restrictive corpora.** Indexing a stable, coarse access dimension — the workspace, the team, the classification level — lets the retrieval query exclude most inaccessible documents cheaply, while the precise per-document check runs afterwards on a much smaller set. It accepts that the coarse dimension may be slightly stale, which is safe because the fine check is authoritative and runs against current state.

**Reranking is where result quality is won, and retrieval scores are a poor proxy for it.** A model that examines the query alongside each candidate passage produces far better ordering than any retrieval score, at a cost that is affordable because it runs on tens of candidates rather than millions. Systems that return retrieval order directly are leaving most of their achievable quality unclaimed.

**Passage-level chunking with document-level presentation is the arrangement that works.** Embedding an entire long document produces a vector that represents nothing in particular, so retrieval targets passages — but users think in documents, so results present the document with the matching passage highlighted. Getting this wrong in either direction produces either poor recall on long documents or a result list of disconnected fragments.

**Freshness comes from change events, not from rebuilds.** Each source emits changes, ingestion re-chunks and re-embeds only the affected passages, and both indexes are updated incrementally — which keeps the lag under a minute. A nightly full rebuild is simpler and produces a knowledge base that is always a day out of date, which for an organisation's working documents is the difference between a tool people use and one they stop trusting.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Permission leak through a snippet | Content disclosure | Authorise before generating snippets |
| Stale indexed permissions | Access after revocation | Filter against live state; short permission caches |
| Too few results after filtering | Empty pages for restricted users | Adaptive over-fetch; permission-aware retrieval |
| Embedding model changed in place | Mixed index, poor results for part of the corpus | Treat as a migration with dual indexes |
| Exact identifiers not found | Vector-only retrieval blurring rare tokens | Hybrid retrieval with a keyword half |
| Long documents ranking poorly | Whole-document embeddings dilute meaning | Passage-level chunking |
| Index diverges from sources | Missing or deleted documents in results | Periodic reconciliation against sources |
| Reranking latency | Query budget exceeded | Rerank fewer candidates; cache common queries |

> **Any disclosure of a document's existence is a permission failure, not a ranking imperfection**  
> Titles, snippets, result counts and facet totals all reveal information about documents the user may not see, and in a corporate knowledge base the existence of a document — a planned reorganisation, an acquisition memo — can be more sensitive than its contents. The requirement is therefore not that inaccessible documents rank low, but that they are removed entirely before anything derived from them reaches the response.

This is why permission filtering sits between retrieval and every subsequent stage, rather than being applied as a final adjustment to a result list that has already been assembled.

**The staff-level view**

The strong answer treats permissions as a first-class retrieval constraint rather than a post-processing step.

- **Argue for hybrid retrieval with concrete failure cases** — an error string that vectors blur, a paraphrase that keywords miss.
- **Fuse by rank rather than by normalised score**, since the two retrieval scores are not comparable.
- **Filter permissions against live state** and explain why indexed permissions go stale the moment someone changes team.
- **Quantify the over-fetch problem** and propose coarse tokens in the index with a fine check afterwards.
- **Authorise before generating snippets**, treating existence disclosure as the actual requirement.
- **Add reranking** and say plainly that retrieval scores are weak signals of usefulness.

> **The question that raises the level**  
> What does the system do when a user's query matches only documents they cannot see? Returning an empty result is correct and unhelpful; explaining that results exist but are inaccessible is helpful and a disclosure. Most organisations want the first with a route to request access through a channel that does not reveal what was found — and having thought about that before implementation is what separates a considered system from one that leaks by omission.

---

### Location-Based Ride Matching

*Track moving vehicles at scale, find nearby candidates in milliseconds, and assign each rider exactly one driver without double-booking.*

**Flow:** `Driver location stream` → `Spatial index` → `Request intake` → `Candidate search` → `Offer and acceptance` → `Trip state machine`

**The brief**

Hundreds of thousands of drivers report their position every few seconds. Riders request trips from anywhere in a city, and the system must find suitable nearby drivers, offer the trip, and confirm an assignment before the rider loses patience.

Two things must never happen: one driver assigned to two riders, and one rider assigned two drivers. Everything else — perfect optimality, exact distances, ideal fairness — is negotiable in service of speed and correctness.

- **Matching latency**: a driver assigned within a few seconds of the request.
- **Exclusivity**: each driver takes at most one trip; each request produces at most one assignment.
- **Freshness**: driver positions no more than a few seconds old.
- **Scale**: 500,000 active drivers, 100,000 location updates per second, 5,000 ride requests per second at peak.
- **Coverage**: dense city centres and sparse outer areas in the same system.

> **Location updates and matching decisions have completely different requirements**  
> Position reporting is enormous in volume, tolerant of loss, and needs no durability — a missed update is corrected three seconds later. Assignment is low volume and must be exactly correct. Treating them as one system forces the matching path to inherit the write load of tracking, and the tracking path to inherit the consistency requirements of matching; separating them lets each be built appropriately.

**Scale**

**Update volume and search arithmetic**

```text
LOCATION UPDATES
  500,000 drivers reporting every 4 s
  = 125,000 updates/s
  each ~100 B -> ~12 MB/s of raw ingest
  -> volume is high, value per update is low
  -> in-memory spatial structure, not a database

WHY NOT PERSIST EVERY UPDATE
  125,000 writes/s of data that is worthless in 4 s
  -> keep current positions in memory
  -> stream a sampled trail for audit and analytics

INDEX UPDATE OPTIMISATION
  most updates keep a driver in the same cell
  cell size ~500 m, typical movement per update ~50 m
  -> ~90% of updates require no index change
  -> only rewrite the index entry on cell change
  -> effective index writes: ~12,000/s

SEARCH
  5,000 requests/s at peak
  each: find drivers within ~3 km
  dense centre: 3 km radius may contain 2,000 drivers
  sparse suburb: may contain 2
  -> the same query returns wildly different
     candidate counts

CANDIDATE CAP
  rank by estimated time of arrival, keep top ~20
  -> bounded work regardless of density
  -> sparse areas widen the radius instead

OFFER ROUND
  offer to 1-3 drivers, 15 s to accept
  -> most accept; declines trigger the next round
  -> total matching time target: under 10 s
```

| Metric | Value | Note |
|---|---|---|
| Updates | 125,000/s | in memory |
| Index writes | ~12,000/s | **cell-change only** |
| Candidates | capped at ~20 | bounded work |
| Match target | < 10 s | including offers |

The cell-change optimisation is the difference between a viable system and an expensive one. Rewriting the spatial index on every position report means a hundred and twenty-five thousand index mutations per second; recognising that most reports leave the driver in the same cell reduces that tenfold, and the position itself is updated in a simple in-memory map that costs almost nothing.

> **Density variation of three orders of magnitude makes uniform parameters wrong everywhere**  
> A three-kilometre radius returns thousands of candidates downtown and none in a rural area, so a fixed search radius either wastes work or finds nobody. The search must adapt — start small and expand until enough candidates are found — which also bounds the work in dense areas naturally, because the first small ring already contains enough drivers.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Location storage | Database, in-memory index | In-memory, sharded by region | 125,000 writes/s of ephemeral data does not belong in a database |
| Spatial structure | Uniform grid, geohash, S2 or H3 cells | Hierarchical cells with level selection | Density varies enormously; uniform cells fail at both extremes |
| Search strategy | Fixed radius, expanding ring | Expanding ring with a candidate cap | Bounds work in dense areas, finds drivers in sparse ones |
| Assignment | Optimistic with reconciliation, atomic claim | Atomic claim on the driver | Double-booking is a real-world failure, not a data inconsistency |
| Offer model | Assign directly, offer and accept | Offer with a short acceptance window | Drivers decline; direct assignment strands riders |
| Sharding | By driver, by geography | By geography | Searches are spatial; a driver's shard is where they are |

1. **Ingest location updates into an in-memory index sharded by region** — Volume is high and the data is worthless within seconds.
2. **Update the spatial index only when the driver changes cell** — Most updates do not move a driver between cells.
3. **Search by expanding ring until enough candidates are found** — It handles both dense and sparse areas with the same code.
4. **Rank candidates by estimated arrival time, not straight-line distance** — A driver across a river is close and unreachable.
5. **Claim the driver atomically before offering** — Two concurrent requests must not offer the same driver.
6. **Release the claim if the offer is declined or expires** — A held driver who never accepts is unavailable to everyone.

Ranking by estimated arrival time rather than by distance is what makes matches sensible. Straight-line proximity ignores rivers, motorways, one-way systems and traffic, so the nearest driver by distance is regularly not the soonest to arrive — and riders judge the service by how long they wait, not by how many metres away the vehicle started.

> **Ask before building it**  
> Is the goal to match each request quickly or to optimise across all pending requests? Greedy per-request matching is simpler, faster and slightly worse globally; batched optimisation over a few seconds of requests produces better overall assignment and adds latency to every rider. Most products choose greedy, and the ones that batch do it deliberately at peak times only.

**Deep dive**

Three mechanisms carry this system: the location pipeline, the candidate search, and the exclusivity of assignment.

**The location pipeline**

```text
DRIVER APP
  reports {driver_id, lat, lng, heading, timestamp}
  every 4 s while available

INGEST  (sharded by region)
  1  update the current-position map
       O(1), in memory, no index work
  2  compute the driver's cell
  3  if the cell changed:
       remove from the old cell's set
       add to the new cell's set
  4  sample: every Nth update to a durable stream
       -> for audit, analytics, replay
       -> not for matching

WHAT LIVES WHERE
  current position     memory, authoritative for
                       matching
  cell membership      memory, the search structure
  position history     durable stream, sampled
  driver availability  memory, plus a durable record
                       of state transitions

FAILURE OF A REGION SHARD
  the in-memory index is lost
  -> drivers repopulate it within one reporting
     interval (4 s)
  -> this is why not persisting is acceptable: the
     data rebuilds itself almost immediately

STALE DRIVERS
  no update for 30 s -> mark unavailable
  -> a driver whose phone died must not receive
     offers
```

**The in-memory index being rebuildable in one reporting interval is what justifies its volatility.** Losing the entire spatial structure sounds catastrophic and is not: every driver reports within four seconds, so the index reconstructs itself faster than any persistence layer could be recovered. Recognising this converts a durability requirement into a non-requirement and removes most of the cost from the highest-volume path in the system.

**Search and atomic assignment**

```text
EXPANDING RING SEARCH
  radius = 1 km
  loop:
    fetch drivers in cells covering the radius
    filter: available, not claimed, vehicle type,
            rating
    if count >= 20 or radius >= max: break
    radius x= 2
  -> dense area: exits after the first ring
  -> sparse area: expands until it finds someone
  -> always include neighbouring cells; boundaries
     are the classic bug

RANK
  estimated time of arrival via routing
  -> batch the routing calls for the candidate set
  -> adjust for driver acceptance history, fairness
     rules, surge

ATOMIC CLAIM  (the correctness core)
  for the top candidate:
    UPDATE drivers
    SET state = 'offered', offer_id = ?,
        offer_expires = now() + 15 s
    WHERE driver_id = ?
      AND (state = 'available'
           OR (state = 'offered'
               AND offer_expires < now()))
    -> 1 row = claimed, send the offer
    -> 0 rows = someone else has them, try the next

OUTCOME
  accepted -> state = 'on_trip', rider assigned
  declined -> release, offer the next candidate
  expired  -> release, offer the next candidate

RIDER SIDE
  the request holds at most one active offer
  -> the rider cannot be assigned twice either
```

**The atomic claim is the only part of this system that must be strictly correct, and it is deliberately small.** Everything else — position freshness, candidate selection, ranking quality — is approximate and self-correcting, while assigning one driver to two riders produces two people waiting for the same car. Concentrating the correctness requirement into a single conditional update keeps it verifiable and keeps the rest of the system free to be fast and approximate.

**Offer expiry must be handled on read as well as by a sweeper.** A driver whose offer lapsed thirty seconds ago should be immediately available to the next search, and relying on a background job to release them leaves drivers idle during exactly the periods when demand is highest. Including the expiry condition in the claim itself makes lapsed offers self-releasing.

**Supply and demand imbalance is a product problem the system must expose.** When requests exceed available drivers in an area, the matching system cannot create supply — it can only decide who waits and how they are told. Surfacing the imbalance honestly through wait-time estimates, and using pricing or incentives to move supply, is the actual remedy; a matching algorithm tuned harder against an empty area produces nothing but latency.

**Fairness among drivers is an explicit policy, not an emergent property.** Always offering to the closest driver concentrates work on whoever positions themselves well, and drivers notice. Blending proximity with time since last trip, acceptance rate and earnings balance produces a distribution drivers perceive as fair, and the weighting is a business decision that belongs in configuration rather than buried in the ranking code.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Two riders offered the same driver | Double booking | Atomic claim with expiry condition |
| Driver claimed but never responds | Driver unavailable to everyone | Short offer expiry; release on read |
| Location index lost | No candidates found | Rebuilds in one reporting interval; degrade briefly |
| Driver phone offline | Offers sent to an unreachable driver | Mark stale after 30 s without an update |
| No drivers in a sparse area | Rider waits indefinitely | Expand radius, then decline honestly with an estimate |
| Cell boundary missed | Nearby drivers not found | Always search neighbouring cells |
| Routing service slow | Ranking delayed, matching latency rises | Fall back to straight-line distance with a note |
| Demand spike in one area | Matching latency and failures concentrated | Surge pricing; honest wait estimates; shed politely |

> **Double-booking a driver is a failure in the physical world, not in the database**  
> Two riders standing on different corners expecting the same car is not recoverable by retrying — someone is stranded, both are annoyed, and the driver is in the middle of it. This is why assignment uses a strict atomic claim rather than an optimistic approach with reconciliation, and why the claim is the one place in this system where correctness is not traded for speed.

The mirror failure — a rider holding two offers — is equally real and is prevented by the same technique applied to the request rather than the driver.

**The staff-level view**

The strong answer separates the high-volume approximate path from the low-volume exact one, and is specific about the atomic claim.

- **Split location tracking from matching** and justify keeping positions in memory by how quickly the index rebuilds.
- **Give the cell-change optimisation** with numbers, since it is the difference between 125,000 and 12,000 index writes per second.
- **Use an expanding ring search** so that density variation is handled by the same code path at both extremes.
- **Rank by arrival time, not distance**, and explain why straight-line proximity produces poor matches.
- **Write out the atomic claim** including the expiry condition, and say it is the only strictly correct part of the system.
- **Treat supply shortage as a product problem**, since no matching improvement creates drivers that are not there.

> **The question that raises the level**  
> What should a rider see when there is genuinely no driver available? Holding them in a searching state indefinitely is the worst option and the most common default. Telling them quickly, with an estimated wait or a suggestion to try again shortly, respects their time and reduces load on a system that is already short of supply — and deciding this in advance is what prevents the interface from implying a match is imminent when it is not.

---

### Notification Platform

*Turn application events into messages delivered across push, email and SMS, respecting preferences, deduplicating, and never spamming anyone.*

**Flow:** `Event intake` → `Preference and policy` → `Template rendering` → `Channel routing` → `Provider delivery` → `Feedback and suppression`

**The brief**

Every service in the company wants to notify users: an order shipped, someone commented, a payment failed, a password changed. Each notification may go to push, email or SMS depending on urgency, the user's settings, and which devices they have.

The platform's hardest requirement is restraint. A notification system that works perfectly and sends too much is worse than one that occasionally fails, because users disable notifications entirely and the channel is lost permanently.

- **Multi-channel**: push, email and SMS from one event, routed by policy.
- **Preferences**: per-user, per-category controls that are always honoured.
- **Deduplication**: one event produces one notification, however many times it is emitted.
- **Rate discipline**: bursts are batched; nobody receives twenty messages in a minute.
- **Scale**: 50 million users, 100 million notifications per day, 20,000 per second at peak.

> **The platform's job is deciding whether to send, not how to send**  
> Delivering a push message or an email is a solved problem handled by providers. What no provider does is decide whether this user wants this notification now, whether they already received it, whether it should be batched with three others, and whether they are in a quiet period. Those decisions are the platform, and treating it as a delivery pipeline with preferences bolted on produces exactly the over-sending that destroys the channel.

**Scale**

**Volume and channel arithmetic**

```text
INTAKE
  100,000,000 notification events/day
  ~1,150/s average, 20,000/s peak
  (peaks follow product events: a feature launch, a
  marketing send, a service incident)

AFTER POLICY FILTERING
  preferences remove 30-50%
  deduplication removes 5-10%
  batching collapses 20-30% into digests
  -> ~40-50 M actually delivered
  -> half the platform's work is deciding NOT to send

CHANNEL MIX AND COST
  push   ~35 M/day   near-zero marginal cost
  email  ~12 M/day   fractions of a cent each
  SMS    ~1 M/day    cents each
  -> SMS is 1% of volume and often the largest line
     on the bill
  -> routing policy has direct financial consequences

DELIVERY LATENCY
  urgent (security, payment failure): seconds
  transactional (order shipped): under a minute
  digest (activity summary): scheduled
  -> three classes, three paths, three queues

PROVIDER LIMITS
  each provider has rate limits and throttles
  -> the platform must shape outbound traffic per
     provider
  -> and handle throttling without losing messages

FEEDBACK
  bounces, complaints, unsubscribes, invalid tokens
  -> must feed back into suppression immediately
  -> sending to a bounced address repeatedly damages
     deliverability for everyone
```

| Metric | Value | Note |
|---|---|---|
| Events | 100 M/day | 20,000/s peak |
| Suppressed | ~50% | **half the work** |
| SMS | 1% of volume | largest cost |
| Feedback | immediate | or reputation falls |

The observation that half the platform's work is deciding not to send is the one that reframes the design. A pipeline built around delivery throughput with a preference check appended will send too much, because the checks are an afterthought in both code and mental model; a pipeline built around policy evaluation, with delivery as the final step for whatever survives, produces a system that protects users by default.

> **Email reputation is a shared resource that a single bad send can damage for months**  
> Providers judge senders on bounce and complaint rates, and a campaign sent to stale addresses or to people who did not want it depresses deliverability for every message the company sends afterwards — including password resets. Suppression lists, prompt handling of bounces, and separating transactional from marketing sending domains are protections for the whole platform, not hygiene for one campaign.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Intake model | Direct API calls, event stream | Event stream with an idempotency key | Producers retry; the platform must deduplicate |
| Preference model | Global on/off, per category per channel | Per category per channel, with defaults | Coarse controls lead users to disable everything |
| Batching | Send everything, digest bursts | Batch above a rate threshold | Twenty messages in a minute costs the channel permanently |
| Channel choice | Producer decides, platform decides | Platform decides from policy and urgency | Producers optimise for themselves, not for the user |
| Provider integration | Single provider, multiple with failover | Multiple per channel | Provider outages are routine and not your outage to fix |
| Delivery guarantee | Best effort, at-least-once with dedup | At-least-once plus deduplication at send | Losing a security alert is unacceptable; duplicates are visible |

1. **Accept events with a producer-supplied idempotency key** — Producers retry, and the same event must not notify twice.
2. **Evaluate preferences, quiet hours and suppression before anything else** — Policy is the expensive and important part; do it first.
3. **Batch bursts into digests above a per-user rate threshold** — This is what keeps the channel alive long term.
4. **Choose channels from urgency and user state, not from the producer's request** — Producers cannot see the user's other notifications.
5. **Render templates with locale and channel-specific variants** — One message shape does not fit push, email and SMS.
6. **Feed provider feedback into suppression immediately** — Bounces and complaints must stop the next send, not the next campaign.

Letting the platform rather than the producer choose the channel is the decision that makes restraint possible. A producing service knows only that its own event is important, so if each one requests push and SMS, the user receives everything from everyone — whereas the platform can see that this user already received four notifications in the last ten minutes and route the fifth into a digest.

> **Ask before building it**  
> Which notifications are transactional and which are promotional? The distinction is legal in many jurisdictions, affects which consent applies, and should separate the sending infrastructure — because a promotional send that damages deliverability must not be able to prevent a password reset from arriving.

**Deep dive**

Three subsystems define this platform: policy evaluation, the delivery pipeline with its provider handling, and the feedback loop.

**From event to decision**

```text
EVENT ARRIVES
  {user, category, payload, idempotency_key,
   urgency}

1  DEDUPLICATE
     seen this key before? -> drop
     -> producers retry; this is not optional

2  SUPPRESSION
     unsubscribed from this category? -> drop
     address bounced permanently? -> drop
     account deleted or dormant? -> drop

3  PREFERENCES
     which channels has the user enabled for this
     category?
     -> if none, drop

4  QUIET HOURS
     within the user's quiet window?
     urgent -> send anyway
     otherwise -> defer to the window's end

5  RATE AND BATCHING
     how many notifications has this user received
     recently?
     under threshold -> send now
     over threshold  -> add to a digest

6  CHANNEL SELECTION
     urgent + push token present -> push
     push unavailable or unacknowledged -> email
     critical and unacknowledged after N minutes
       -> SMS
     -> escalation, not simultaneous broadcast

7  RENDER AND ENQUEUE
     per-channel template, user locale
     -> the appropriate delivery queue

HALF OF THESE STEPS END IN 'DROP'
  which is the point
```

**Channel escalation rather than simultaneous multi-channel sending is what users experience as considerate.** Sending push, email and SMS for the same event means three interruptions for one piece of information; sending push first, falling back to email if it was not delivered, and reserving SMS for critical unacknowledged messages gives the same reliability with a fraction of the intrusion. It also controls the SMS spend, which is the platform's largest variable cost.

**Delivery, providers and feedback**

```text
PER-CHANNEL QUEUES  (by urgency)
  urgent        target < 10 s
  transactional target < 60 s
  digest        scheduled

PROVIDER HANDLING
  rate-shape outbound to each provider's limit
  on 429 or throttle: back off, do not drop
  on persistent failure: fail over to a secondary
    -> requires templates and tokens to work with
       both
  track per-provider delivery and latency

FEEDBACK LOOP
  push:  invalid token -> remove the token
  email: hard bounce   -> suppress the address
         complaint     -> suppress and record
  sms:   invalid number -> suppress
  -> applied IMMEDIATELY, before the next send
  -> a token that fails repeatedly and is not
     removed wastes quota and degrades metrics

WHAT MAKES DELIVERABILITY HOLD
  low bounce rate       (suppression works)
  low complaint rate    (preferences are honoured)
  separate domains      (marketing cannot damage
                         transactional)
  consistent volume     (sudden spikes look like
                         spam)

OBSERVABILITY
  per category: sent, delivered, opened, complained,
  unsubscribed
  -> the unsubscribe rate per category is the signal
     that a notification is unwanted
```

**The unsubscribe rate per category is the platform's most important quality metric.** Delivery rates measure whether the pipeline works; unsubscribe rates measure whether the notifications should have been sent at all. A category whose unsubscribe rate is rising is telling the organisation something that no amount of delivery optimisation addresses, and surfacing that per category — back to the team producing those events — is what keeps the channel healthy.

**Digests are more than a rate limit; they are a different product.** Collapsing twelve activity notifications into one summary requires templates that summarise rather than repeat, sensible ordering, and a cadence users can predict. Implemented as a simple hold-and-concatenate, digests read like a backlog and perform worse than the individual messages they replaced.

**Quiet hours require knowing the user's timezone, which is harder than it sounds.** Timezone from the account profile, from the device, and from recent activity can all disagree, and sending a notification at three in the morning is the kind of error users remember. Preferring the device's current timezone, falling back to the profile, and defaulting conservatively is the practical approach — and urgent messages should be explicitly exempt, with a narrow definition of urgent.

**Producers need a contract that makes the platform's decisions predictable.** A service emitting an event should know what urgency levels mean, which categories exist, what deduplication key to supply and what happens if the user has opted out — otherwise teams work around the platform by calling providers directly, which defeats every protection it provides. The contract, and its enforcement, is as much a part of the system as the pipeline.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Duplicate notifications | Users receive the same message twice | Deduplicate on the producer's idempotency key |
| Notification storm to one user | Twenty messages in a minute; user disables all | Per-user rate threshold with digest collapse |
| Preferences ignored | Trust broken; complaints rise | Evaluate preferences before anything else; audit regularly |
| Bounces not suppressed | Deliverability falls for all mail | Immediate feedback processing |
| Provider outage | Notifications undelivered | Secondary provider per channel; queue rather than drop |
| SMS cost spike | Large unexpected bill | Escalation rather than broadcast; per-category SMS budgets |
| Quiet hours violated | Messages at 3 a.m. | Timezone resolution with conservative defaults |
| Marketing send damages transactional delivery | Password resets landing in spam | Separate domains and sending infrastructure |

> **Over-notification is irreversible in a way that delivery failures are not**  
> A message that fails to arrive can be retried; a user who disables notifications after being flooded rarely re-enables them, and the channel to that person is gone for good. This asymmetry means the platform should be biased towards sending less — batching aggressively, defaulting categories off, and treating a rising unsubscribe rate as an incident rather than as a metric — because the cost of one message too many is permanent and the cost of one too few usually is not.

The corollary is that the rate limiter and the digest logic are more important than the delivery pipeline, and should be tested with the same seriousness.

**The staff-level view**

The strong answer treats restraint as the primary engineering requirement rather than as a product preference.

- **State that half the work is deciding not to send**, and structure the pipeline around policy evaluation rather than delivery.
- **Let the platform choose channels**, since producers cannot see what else the user has received.
- **Describe escalation rather than broadcast** — push, then email, then SMS — for both intrusion and cost.
- **Make feedback immediate**, and explain that bounce handling protects deliverability for the whole company.
- **Separate transactional from promotional infrastructure**, so one cannot damage the other.
- **Name the unsubscribe rate per category** as the quality metric that delivery statistics cannot replace.

> **The question that raises the level**  
> What is the default state for a new notification category — on or off? Defaulting on maximises reach and steadily erodes trust; defaulting off means categories must earn their audience and keeps engagement honest. Most organisations default on without ever deciding to, and the accumulated effect across dozens of categories is what turns a useful channel into one users mute.

---

### Webhook Delivery Platform

*Deliver events to customer-operated endpoints reliably, without letting one slow or broken subscriber affect anyone else.*

**Flow:** `Event source` → `Subscription registry` → `Per-subscriber queues` → `Signed delivery` → `Retry and backoff` → `Dead letter and replay`

**The brief**

Customers register URLs to receive events from your platform. Those endpoints are outside your control: they time out, return errors, disappear for days, or accept requests very slowly — and there are thousands of them, each with different reliability.

The platform must deliver events reliably despite this, prove that deliveries genuinely came from it, and ensure that one customer's broken endpoint cannot degrade delivery for anyone else.

- **Reliability**: every event is delivered at least once, or visibly abandoned after a defined effort.
- **Isolation**: a slow or failing subscriber affects only their own delivery.
- **Authenticity**: receivers can verify a request came from the platform and was not replayed.
- **Observability**: customers can see delivery attempts, failures and replay what they missed.
- **Scale**: 50,000 subscriptions, 200 million events per day.

> **Every subscriber is an independent, unreliable system, and must be queued independently**  
> A shared delivery queue means one endpoint that takes thirty seconds per request consumes workers that everyone else needs, and one that has been down for two days fills the queue with retries. Giving each subscription its own ordered queue with its own concurrency makes each customer's reliability their own concern — which is both fair and the only arrangement that survives thousands of independent failure modes.

**Scale**

**Delivery volume and isolation arithmetic**

```text
EVENTS
  200,000,000/day ≈ 2,300/s average
  fan-out: an event may match several subscriptions
  average 1.5 subscriptions per event
  -> ~3,500 deliveries/s

ENDPOINT LATENCY DISTRIBUTION
  median      120 ms
  p90         800 ms
  p99         5 s
  timeout     10 s
  -> concurrency needed = rate x latency
  -> 3,500/s x 0.5 s average = ~1,750 concurrent
  -> but during a slow-endpoint incident, that
     endpoint's share balloons

WHY SHARED QUEUES FAIL
  one subscriber at 10 s per request receiving
  100 events/s needs 1,000 concurrent slots
  -> in a shared pool, that is most of the capacity
  -> everyone else queues behind them

RETRY VOLUME
  typical failure rate 2-5% of deliveries
  with 6 retry attempts:
    3,500/s x 0.03 x 6 = ~630 extra attempts/s
  -> a large subscriber going down for a day adds
     millions of queued retries

BACKLOG PER SUBSCRIBER
  an endpoint down for 24 h at 50 events/s
  = 4,300,000 queued events
  -> retention policy required
  -> and a decision about what happens on recovery:
     replay everything, or skip to recent?
```

| Metric | Value | Note |
|---|---|---|
| Deliveries | ~3,500/s | after fan-out |
| Isolation | queue per subscriber | **mandatory** |
| Retry overhead | ~20% | of delivery volume |
| Backlog | millions per day | needs a policy |

The backlog question is one that platforms frequently leave undecided until it happens. A subscriber whose endpoint returns after two days of downtime faces millions of pending events, and delivering them all at full rate will knock the endpoint over again — while discarding them silently loses data the customer expected. Draining at a controlled rate, with the customer able to see the backlog and choose to skip it, is the arrangement that respects both.

> **A subscriber recovering from an outage is the moment they are most fragile**  
> An endpoint that has just come back is typically running on cold caches and reduced capacity, and receiving a two-day backlog at maximum concurrency will take it down again — producing a cycle where the platform repeatedly kills the endpoint it is trying to deliver to. Ramping delivery rate gradually after a recovery is what breaks that cycle, and it is the same reasoning that governs circuit breaker half-open states.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Queueing | Shared queue, queue per subscription | Per subscription | One bad endpoint must not consume shared capacity |
| Ordering | Unordered, ordered per subscription | Ordered, with a concurrency of one per subscription by default | Customers assume order; allow opting out for throughput |
| Authenticity | Shared secret, HMAC signature, mTLS | HMAC signature with a timestamp | Verifiable, replay-resistant, no client certificates to manage |
| Retry schedule | Fixed, exponential with jitter | Exponential with jitter, capped | Retrying a struggling endpoint immediately makes it worse |
| Failure handling | Retry forever, dead letter | Bounded retries then disable with notification | A permanently broken endpoint should not be retried indefinitely |
| Replay | Not supported, customer-initiated | Customer-initiated from a retention window | Customers need to recover from their own outages |

1. **Give each subscription its own ordered queue with its own concurrency limit** — This is the isolation mechanism; everything else depends on it.
2. **Sign every request with an HMAC over the body and a timestamp** — Receivers must be able to verify origin and reject replays.
3. **Retry with exponential backoff and jitter, to a bounded total duration** — Immediate retries deepen the failure they are responding to.
4. **Trip a circuit after sustained failures and notify the customer** — Continuing to hammer a dead endpoint helps nobody.
5. **Retain undelivered events for a defined window and expose replay** — Customers have outages too, and need a way to recover.
6. **Ramp delivery rate gradually after a recovery** — A recovering endpoint cannot absorb a full backlog at once.

Ordering per subscription with a concurrency of one is the safe default and should be explicitly overridable. Many receivers assume events arrive in order and behave incorrectly if they do not, so serialising per subscription prevents a class of customer bugs — while subscribers who can handle concurrency and need throughput should be able to opt in, since serial delivery caps their rate at one over the endpoint's latency.

> **Ask before building it**  
> Do customers depend on ordering, or on eventual completeness? If completeness is what matters, parallel delivery with sequence numbers lets receivers order events themselves and delivers far more throughput. If order is assumed, serial delivery is required and each subscription's rate is bounded by its own latency — which is a limit customers should be told about.

**Deep dive**

Three mechanisms carry this platform: per-subscription isolation, signed delivery, and the failure policy that decides when to stop.

**Delivery with signatures and retries**

```text
REQUEST
  POST https://customer.example/hooks
  X-Event-Id: evt_01H...
  X-Timestamp: 1757900000
  X-Signature: sha256=<hmac>
  body: {...}

SIGNATURE
  hmac = HMAC-SHA256(secret, timestamp + "." + body)
  -> receiver recomputes and compares
  -> receiver rejects if the timestamp is older than
     ~5 minutes, which prevents replay
  -> include the event id so receivers can
     deduplicate

RETRY SCHEDULE
  attempt 1  immediate
  attempt 2  +10 s
  attempt 3  +1 min
  attempt 4  +10 min
  attempt 5  +1 h
  attempt 6  +6 h
  -> total span ~7 h, jittered
  -> covers a deploy, a restart, a short outage

WHAT COUNTS AS SUCCESS
  2xx only
  -> 3xx is a misconfiguration, not a success
  -> 4xx (other than 429) means the endpoint rejects
     it; retrying rarely helps but is cheap
  -> 429 and 5xx: retry with backoff

AFTER THE LAST ATTEMPT
  move to the dead letter store
  -> retained for the replay window
  -> counted against the subscription's health

SUSTAINED FAILURE
  e.g. 100 consecutive failures or 24 h of total
  failure
  -> disable the subscription
  -> notify the customer through another channel
  -> stop wasting capacity on a dead endpoint
```

**Including a timestamp in the signature is what makes it replay-resistant.** A signature over the body alone can be captured and resent indefinitely by anyone who intercepts one request, whereas binding the timestamp into the signed material lets the receiver reject anything older than a few minutes. This is a small detail that distinguishes a signature scheme that authenticates from one that merely identifies.

**Isolation and health**

```text
PER-SUBSCRIPTION STATE
  queue        pending events, ordered
  concurrency  usually 1, configurable
  rate limit   optional, per customer request
  circuit      closed | open | half-open
  health       recent success rate, latency

WHY THE CIRCUIT MATTERS
  an endpoint failing every request
  -> without a circuit, every event burns a full
     timeout (10 s) before failing
  -> 50 events/s x 10 s = 500 concurrent slots
     doing nothing useful
  -> with a circuit: fail immediately, retry a probe
     periodically

RECOVERY RAMP
  circuit half-open -> send 1 event
  success -> send 5/s
  sustained success -> 20/s
  -> reach full rate over a few minutes
  -> prevents killing the endpoint that just
     recovered

CUSTOMER VISIBILITY
  delivery log per subscription
    attempted, response code, latency, attempt
    number
  -> customers debug their own endpoints
  -> and this is the single most requested feature
     of such platforms

REPLAY
  customer selects a time range or specific events
  -> re-enqueued, rate limited
  -> deduplication is the receiver's responsibility,
     which is why event ids matter
```

**The delivery log is a product feature, not an operational nicety.** Customers integrating webhooks spend most of their effort debugging why events did not arrive, and without visibility into attempts, response codes and payloads they raise support tickets for problems in their own systems. Exposing the log turns those into self-service investigations and is consistently the most valued part of such platforms.

**Disabling a subscription needs a notification path that does not use webhooks.** A customer whose endpoint has been failing for a day must be told, and telling them through the mechanism that is broken accomplishes nothing. Email to the account owner, a dashboard alert and an API status field are the usual combination, and platforms that disable silently generate the worst kind of support case — one where data has been discarded and nobody was informed.

**At-least-once delivery makes deduplication the receiver's responsibility, and the platform should make it easy.** Retries after ambiguous timeouts mean the same event can arrive twice, so every request carries a stable event identifier and the documentation should state plainly that receivers must be idempotent. Platforms that imply exactly-once delivery create integrations that break under retry, which happens on the first bad network day.

**Fan-out from event to subscriptions should be evaluated once and materialised per subscription.** Matching an event against fifty thousand subscription filters on every delivery attempt is expensive and repeated; evaluating the match once at intake and writing the event into each matching subscription's queue means retries cost nothing extra and subscription changes apply cleanly to future events only.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| One slow endpoint | Shared workers consumed | Per-subscription queues and concurrency |
| Endpoint down for days | Massive backlog accumulates | Retention window; controlled drain; customer-visible backlog |
| Recovery overwhelms the endpoint | It fails again immediately | Gradual rate ramp after the circuit closes |
| Duplicate deliveries | Receiver processes twice | Stable event ids; document idempotency as a requirement |
| Signature replay | Forged or repeated requests accepted | Timestamp inside the signature; receiver rejects old ones |
| Endpoint returns 200 but does not process | Silent data loss at the customer | Not detectable; document that 200 means accepted |
| Subscription disabled silently | Customer loses events without knowing | Notify through email and dashboard, not webhooks |
| Customer's TLS certificate expires | All deliveries fail | Surface the specific error; alert the customer |

> **An endpoint returning 200 without processing the event is invisible to the platform**  
> Delivery is confirmed, metrics look healthy, and the customer's data is missing — a failure entirely inside their system that the platform cannot see. The only defences are documentation making clear that a 2xx means accepted responsibility, and encouraging receivers to acknowledge only after durably persisting the event rather than after receiving it. Platforms that promise delivery without making this boundary explicit end up owning failures they cannot detect or fix.

This is the webhook equivalent of acknowledging a message before processing it, and the correct advice to receivers is identical: persist first, acknowledge second.

**The staff-level view**

The strong answer treats each subscriber as an independent failure domain and is specific about the failure policy.

- **Require per-subscription queues** and show the arithmetic of how one slow endpoint consumes a shared pool.
- **Sign with a timestamp inside the signature**, so captured requests cannot be replayed.
- **Define the full retry schedule and its total span**, then say explicitly what happens after the last attempt.
- **Add a circuit breaker per subscription** so that a dead endpoint stops consuming a timeout per event.
- **Ramp delivery after recovery**, because a backlog delivered at full rate re-breaks the endpoint.
- **Expose a delivery log**, which is both the most requested feature and the biggest reduction in support load.

> **The question that raises the level**  
> What happens to events generated while a subscription is disabled? Retaining them for the replay window means a customer can recover fully but may face millions of events on re-enabling; discarding them is simpler and loses data they expected to have. Stating the policy explicitly, and surfacing the pending count so customers can choose to skip, is what avoids the argument occurring during their incident rather than during onboarding.

---

### Multi-Tenant Rate Limiter

*Enforce per-tenant quotas across a distributed fleet without adding a round trip to every request or letting one customer exhaust shared capacity.*

**Flow:** `Request intake` → `Tenant resolution` → `Local allowance` → `Shared counter sync` → `Decision and headers` → `Quota reporting`

**The brief**

A platform serves thousands of tenants from shared infrastructure. Each has a contractual request quota, and the platform must enforce it — both to honour the commercial agreement and to stop any single tenant from consuming capacity everyone else needs.

The limiter sits in front of every request, so its own latency and availability directly become the platform's. A limiter that adds twenty milliseconds makes every API call slower, and one that fails closed takes the platform down with it.

- **Accuracy**: quotas enforced closely enough to be defensible commercially.
- **Latency**: under a millisecond added to the request path.
- **Isolation**: one tenant's burst cannot degrade another's service.
- **Transparency**: tenants can see their usage and understand rejections.
- **Scale**: 10,000 tenants, 500,000 requests per second, 200 service instances.

> **Perfect accuracy and low latency are incompatible, so the accuracy requirement must be stated**  
> Exact enforcement requires every instance to consult shared state on every request, which is a network round trip on the hot path. Local allowances synchronised periodically add nothing to the request and permit brief overshoot. Since the purpose of most limits is to bound load rather than to count precisely, the second is usually correct — but that is a decision to make explicitly, because for a metered billing quota it is the wrong one.

**Scale**

**Enforcement cost arithmetic**

```text
TRAFFIC
  500,000 requests/s across 200 instances
  = 2,500 requests/s per instance

OPTION A: SHARED COUNTER PER REQUEST
  500,000 round trips/s to the limiter store
  at 0.5 ms each -> 0.5 ms added to every request
  -> the store must sustain 500,000 ops/s
  -> and its availability is the platform's
     availability
  -> accurate, expensive, fragile

OPTION B: FULLY LOCAL
  each instance enforces the full quota locally
  tenant limit 1,000/s, 200 instances
  -> effective limit up to 200,000/s
  -> useless as enforcement

OPTION C: LOCAL ALLOWANCE WITH SYNC  (chosen)
  each instance holds a share of the quota
  syncs usage to a shared view every 1 s
  -> no per-request round trip
  -> overshoot bounded by one sync interval
  -> for a 1,000/s limit: worst case ~1,100/s
     observed briefly

ALLOCATION PROBLEM
  a tenant's traffic is not evenly distributed
  across 200 instances
  -> static shares waste quota on idle instances
  -> dynamic allocation: instances request more from
     the shared pool as they use their share
  -> the sync becomes a lease on quota, not a report

STATE SIZE
  10,000 tenants x ~100 B = 1 MB shared state
  per-instance working set: only active tenants
  -> trivial memory
```

| Metric | Value | Note |
|---|---|---|
| Added latency | < 1 ms | **no round trip** |
| Overshoot | one sync interval | bounded |
| Shared state | ~1 MB | 10,000 tenants |
| Failure mode | must fail open | or platform down |

Framing the periodic sync as a lease on quota rather than a usage report resolves the allocation problem neatly. An instance requests a block of allowance from the shared pool, spends it locally with no coordination, and requests more when it runs low — so quota flows to wherever the traffic actually is, and instances receiving no traffic for a tenant hold none of their quota.

> **A rate limiter that fails closed converts its own outage into a total platform outage**  
> If the shared state becomes unavailable and instances reject everything they cannot verify, then a failure in the component protecting the platform stops all traffic. Failing open risks a window of unenforced limits, which costs some capacity and no availability. For nearly every platform the second is obviously preferable, and the decision must be explicit rather than inherited from a client library's default behaviour.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Enforcement point | Application, gateway, mesh | Gateway | Before any backend work; state is already shared there |
| Algorithm | Fixed window, sliding window, token bucket | Token bucket | Separates sustained rate from burst allowance cleanly |
| Distribution | Shared counter, local with sync | Local with quota leases | Latency on the hot path is unacceptable |
| Dimensions | Per tenant only, per tenant per endpoint | Both, hierarchically | An expensive endpoint needs its own limit within the tenant's quota |
| Cost model | Per request, weighted | Weighted by operation cost | Counting requests limits the wrong quantity |
| Failure behaviour | Fail open, fail closed | Fail open with alerting | The limiter must not be a single point of failure |

1. **Resolve the tenant from the credential before anything else** — Every subsequent decision depends on knowing who is calling.
2. **Check a local token bucket funded by a quota lease** — No network call on the request path.
3. **Synchronise usage and request more allowance periodically** — The sync interval bounds the overshoot.
4. **Apply hierarchical limits: per tenant, then per endpoint** — A cheap endpoint should not be able to exhaust the quota an expensive one needs.
5. **Charge expensive operations more tokens** — The limit should bound work, not request count.
6. **Return quota headers and a retry hint on rejection** — Tenants cannot self-correct without visibility.

Weighting by operation cost is what makes the limit meaningful. A tenant within their thousand-requests-per-second quota can still saturate the platform by choosing only the most expensive endpoint, so the quota has to be denominated in something closer to work — cost units, where a simple lookup costs one and a complex aggregation costs fifty — which also gives the commercial team a more honest thing to sell.

> **Ask before building it**  
> Is the quota a commercial commitment or a protection mechanism? A contractual limit that appears on an invoice needs accurate accounting and a defensible audit trail, which means shared counters for the billing path even if enforcement remains approximate. A protection limit needs only to bound load, and approximate enforcement is entirely sufficient.

**Deep dive**

Three mechanisms define this system: the distributed enforcement model, hierarchical limits, and the behaviour tenants experience.

**Quota leases in practice**

```text
TENANT LIMIT: 1,000 requests/s

INSTANCE STARTUP
  holds no quota for any tenant

FIRST REQUEST FOR TENANT T
  local bucket empty
  -> lease request: "give me allowance for T"
  -> shared pool grants 100 units (0.1 s worth)
  -> serve from the local bucket

SUSTAINED TRAFFIC
  instance consumes its lease
  requests another before it runs out
  -> leases become larger with sustained use
  -> 200 instances share the 1,000/s naturally,
     proportional to where traffic lands

IDLE INSTANCE
  lease expires unused, returns to the pool
  -> no quota is stranded

POOL EXHAUSTED
  no allowance left this second
  -> lease request returns zero
  -> instance rejects locally with 429
  -> and knows when to ask again

OVERSHOOT
  worst case: every instance holds an unused lease
  when the tenant bursts
  -> bounded by (lease size x instances)
  -> keep leases small for tenants with small quotas

SHARED POOL UNAVAILABLE
  instances fall back to a static share
  = quota / instance count
  -> under-enforces for concentrated traffic,
     over-restricts for spread traffic
  -> alert loudly; this is a degraded mode
```

**The lease model makes quota follow traffic, which static sharding cannot do.** Dividing a tenant's limit by two hundred instances gives each five requests per second, so a tenant whose traffic lands on three instances is throttled at fifteen despite having bought a thousand. Leasing means those three instances acquire nearly all the quota and the rest hold none, which is both correct and self-adjusting as routing changes.

**Hierarchical limits and tenant experience**

```text
LIMIT HIERARCHY
  global platform capacity
    └─ per tenant quota           1,000 cost units/s
         ├─ per endpoint class
         │    reads      800 units/s
         │    writes     300 units/s
         │    exports     50 units/s
         └─ per API key within the tenant
              optional, tenant-configured

WHY ENDPOINT CLASSES MATTER
  a tenant running a bulk export can consume their
  whole quota
  -> their interactive traffic then fails
  -> sub-limits guarantee each class a share
  -> this is weighted fair queueing applied to
     quotas

WHAT THE TENANT SEES
  on every response:
    X-RateLimit-Limit, X-RateLimit-Remaining,
    X-RateLimit-Reset
  on rejection:
    429 with Retry-After
    a body explaining WHICH limit was hit
  -> "you exceeded your export limit, not your
     overall quota" is actionable
  -> "429" alone is not

BURST ALLOWANCE
  token bucket capacity above the sustained rate
  -> legitimate clients batch and page
  -> capacity 2-5x the per-second rate is usually
     right
  -> and it is a promise about what the platform
     absorbs
```

**Telling the tenant which limit they hit is the difference between an actionable error and a support ticket.** A client receiving a bare rejection cannot distinguish an overall quota exhaustion from an endpoint-specific limit or a global protection measure, so they retry the same way and fail again. Naming the limit, the current usage and the reset time lets a well-behaved client adjust automatically, which reduces both their failures and the platform's load.

**Sub-limits per endpoint class are what keep a tenant's interactive traffic working while they run something heavy.** Without them, a bulk export consumes the whole quota and the tenant's own users see failures — which they experience as the platform being unreliable rather than as their own batch job. Reserving shares by class turns that into a slower export and a working application, which is the outcome everyone wants.

**Global protection limits must exist above tenant quotas.** The sum of all contracted quotas typically exceeds the platform's actual capacity, on the assumption that not everyone peaks simultaneously — which is usually true and occasionally false. A platform-wide shedding layer that engages when aggregate load threatens stability, preferring to degrade the largest consumers first, is what prevents a correlated peak from taking everything down.

**Usage reporting is a separate concern from enforcement and should use a separate path.** Enforcement is approximate and optimised for latency; billing and reporting need accuracy and can tolerate seconds of delay. Emitting usage events to a stream for aggregation gives an accurate record without putting accounting on the request path, and it means a discrepancy between what was enforced and what was billed is visible rather than assumed away.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Shared state unavailable | Cannot lease quota | Fail open to a static share; alert loudly |
| Limiter adds latency | Every request slower | Local buckets; never a round trip per request |
| One tenant exceeds the platform | Others degraded | Global shedding above tenant quotas |
| Quota stranded on idle instances | Tenant throttled below their limit | Leases that expire and return to the pool |
| Bulk job starves interactive traffic | Tenant's own users see failures | Per-endpoint-class sub-limits |
| Clients retry immediately on 429 | Rejection load compounds | Retry-After header; document expected behaviour |
| Billing disagrees with enforcement | Commercial dispute | Separate accurate usage stream for billing |
| Cost-weighted limits misconfigured | Cheap endpoints exhaust quota | Calibrate weights against measured cost |

> **The limiter sits in front of everything, so its failure modes are the platform's failure modes**  
> Latency added here is added to every request; unavailability here stops every request if it fails closed; a bug here rejects traffic that should be served. This is why the design pushes everything expensive off the request path, treats failing open as the default, and keeps the hot-path logic simple enough to reason about — the component protecting the platform must not become the least reliable thing in it.

The same reasoning argues for deploying limiter changes with the caution normally reserved for the data layer, since a configuration error affects every tenant at once.

**The staff-level view**

The strong answer names the accuracy-versus-latency trade explicitly and chooses deliberately.

- **State the trade**: exact enforcement means a round trip per request, and explain why local allowances with sync are usually correct.
- **Describe quota leases** rather than static sharding, since traffic is never evenly distributed across instances.
- **Weight by operation cost**, because counting requests limits the wrong quantity.
- **Add hierarchical sub-limits** so a tenant's bulk work cannot starve their interactive traffic.
- **Choose fail-open explicitly** and justify it by what the limiter's own outage would otherwise cause.
- **Separate billing accuracy from enforcement**, using a usage stream rather than the hot path.

> **The question that raises the level**  
> Does the sum of contracted quotas exceed the platform's capacity? It almost always does, and that means the system is relying on tenants not peaking together. Knowing the ratio tells you how much global shedding capacity is needed and which tenants would be degraded first — and a platform that has never calculated it will discover the answer during a correlated peak, which is the worst time to be deciding who loses service.

---

### Durable Job Scheduler

*Run scheduled and deferred work exactly when it is due, exactly once per occurrence, surviving crashes, deploys and clock problems.*

**Flow:** `Schedule registry` → `Due-time index` → `Dispatcher` → `Execution workers` → `Completion and retry` → `History and observability`

**The brief**

Teams register work to run later: a report every morning, a reminder in three days, a retry in thirty seconds, a subscription renewal next month. The scheduler must fire each occurrence once, at approximately the right time, and keep doing so through restarts, deploys and partial failures.

The difficult requirements are not about throughput. They are about the guarantee — that a job registered today still fires in a month, that a crash does not skip an occurrence, and that a deploy does not run yesterday's work twice.

- **Durability**: a registered schedule survives everything short of losing the datastore.
- **Exactly-once per occurrence**: crashes and retries must not double-execute.
- **Punctuality**: jobs start within a few seconds of their due time under normal load.
- **Isolation**: a slow or failing job does not delay unrelated ones.
- **Scale**: 10 million active schedules, 50,000 executions per minute at peak.

> **Scheduling and execution are separate problems and should be separate systems**  
> Deciding what is due is a small, exact, stateful problem: an ordered set of times and a claim mechanism. Running the work is a large, messy, failure-prone problem involving other people's code. Combining them means a long-running job blocks the dispatcher, and a dispatcher restart affects in-flight work — whereas separating them lets the scheduler stay simple and the executors be as unreliable as their workloads require.

**Scale**

**Scheduling volume and clustering arithmetic**

```text
SCHEDULES
  10,000,000 active
  recurring (cron-style) + one-off (delays)
  each: {id, next_due, recurrence, payload_ref,
         owner}  ~200 B
  -> ~2 GB of schedule state

DUE-TIME QUERY
  the hot operation: "what is due now?"
  indexed by next_due
  -> a range scan over a narrow window
  -> the index is the entire performance story

EXECUTION RATE
  50,000/minute peak ≈ 830/s
  but NOT evenly distributed

THE CLUSTERING PROBLEM
  people schedule on round numbers
  00:00 UTC, on the hour, on the minute
  -> 40-60% of daily executions can land on a handful
     of timestamps
  -> midnight may see 500,000 due simultaneously
  -> at 830/s that is 10 minutes of backlog
  -> jitter at registration is the only real fix

DISPATCH CLAIM
  many dispatchers, one claim per occurrence
  -> conditional update or skip-locked selection
  -> the claim is the exactly-once mechanism

HISTORY
  50,000/min x 1,440 = 72 M executions/day
  ~300 B each -> ~22 GB/day
  -> retention policy required
  -> but history is what makes the system debuggable
```

| Metric | Value | Note |
|---|---|---|
| Schedules | 10 M active | ~2 GB |
| Peak clustering | midnight | **500k at once** |
| Mechanism | claim per occurrence | exactly-once |
| History | 22 GB/day | needs retention |

The clustering problem dominates capacity planning and is invisible in average figures. A scheduler sized for eight hundred executions per second copes perfectly with the daily average and falls an hour behind at midnight, because everyone who wrote a daily job chose midnight. Adding jitter at registration — spreading a nominal midnight job across several minutes — is far more effective than provisioning for the spike.

> **Punctuality degrades non-linearly once the due-time backlog forms**  
> When more jobs become due than can be dispatched, the backlog delays everything behind it, including jobs due later that would otherwise have been on time. A ten-minute midnight backlog therefore makes a job scheduled for 00:05 late as well, and the effect compounds across the cluster. Measuring lateness — due time to start time — rather than throughput is what makes this visible.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Due-time storage | Queue with delays, indexed table, sorted set | Indexed table or sorted set | Broker delays are capped; long schedules need arbitrary times |
| Claim mechanism | Leader dispatches, competitive claim | Competitive claim with skip-locked | No leader to fail; dispatchers scale horizontally |
| Execution | In the scheduler, separate workers | Separate workers via a queue | Job code must not be able to block dispatch |
| Exactly-once | Best effort, claim plus idempotency | Claim plus idempotent jobs | Claims can be lost; jobs must tolerate a repeat |
| Recurrence | Compute next after execution, compute at dispatch | At dispatch | Computing after execution loses occurrences when a run fails |
| Missed occurrences | Skip, run late, run all | Policy per schedule | A daily report should skip; a billing run must not |

1. **Store schedules with an indexed next-due timestamp** — The due-time query is the hot path and the index is what makes it cheap.
2. **Compute the next occurrence when dispatching, not after completion** — A failed execution must not stop future occurrences.
3. **Claim each occurrence atomically before enqueueing it** — This is the exactly-once boundary.
4. **Enqueue to a work queue; execute in separate workers** — Job duration and failure must not affect dispatch.
5. **Jitter registration times away from round numbers** — Clustering is the dominant capacity problem.
6. **Record every execution with its outcome and timing** — History is what makes schedules debuggable and lateness measurable.

Computing the next occurrence at dispatch rather than after completion is a small ordering choice with a large effect. If the next due time is only written after a successful run, a job that fails or crashes stops recurring entirely — silently, since nothing is scheduled to notice. Advancing the schedule when the occurrence is claimed means recurrence continues regardless of whether any particular run succeeds, and failures are handled by the retry policy rather than by stopping the schedule.

> **Ask before building it**  
> What should happen to occurrences missed while the system was down? A daily summary should probably skip to the current one; a billing cycle must run every missed occurrence; a reminder might be pointless late. This is a per-schedule policy, and a scheduler that hard-codes one answer will be wrong for most of its users.

**Deep dive**

Three mechanisms carry this system: the due-time index and claim, the separation of dispatch from execution, and the handling of time itself.

**Dispatch loop**

```text
EVERY TICK (e.g. every second)

  SELECT * FROM schedules
  WHERE next_due <= now()
    AND state = 'pending'
  ORDER BY next_due
  LIMIT 500
  FOR UPDATE SKIP LOCKED
  -> multiple dispatchers, no coordination, no
     double-claim

  FOR EACH CLAIMED SCHEDULE
    1  compute the next occurrence
         next_due = advance(recurrence, next_due)
         -> written NOW, so recurrence survives
            execution failure
    2  create an execution record
         {schedule_id, occurrence_time, attempt: 1}
         -> the unit of exactly-once
    3  enqueue the execution to the work queue
    4  commit

WORKER
  receives the execution
  runs the job
  records success or failure
  on failure: re-enqueue with backoff, up to the
  retry limit
  -> retries belong to the execution, not the
     schedule

WHY THE EXECUTION RECORD MATTERS
  it identifies the occurrence uniquely
  -> a worker that crashes and is redelivered
     recognises the same execution
  -> jobs can deduplicate on it
  -> and history is a natural by-product
```

**The execution record is what makes exactly-once meaningful at the job's own boundary.** The claim prevents two dispatchers creating two executions, but a worker can still crash after running a job and before recording completion, so the job will be redelivered. Giving each occurrence a stable identifier lets the job itself deduplicate, which converts the platform's at-least-once delivery into an effectively-once outcome for jobs that cooperate.

**Time is harder than it appears**

```text
TIMEZONES AND DAYLIGHT SAVING
  "every day at 02:30 Europe/London"
  spring forward: 02:30 does not exist
    -> skip, or run at 03:00?
  autumn back: 02:30 happens twice
    -> run once, or twice?
  -> the scheduler must define this, per schedule
     if necessary
  -> storing UTC only makes the question unanswerable

CLOCK SKEW
  dispatchers disagree about "now"
  -> a job may fire slightly early on one node
  -> claim-based dispatch makes this harmless: only
     one wins
  -> but a badly skewed node could claim jobs far in
     advance
  -> bound how far ahead a dispatcher may claim

LONG SCHEDULES
  "in 6 months"
  -> must survive deploys, migrations, schema
     changes
  -> the payload reference must still resolve
  -> storing a serialised closure is a trap; store a
     job type and parameters

CATCH-UP AFTER DOWNTIME
  4 hours of downtime, hourly job
  -> 4 missed occurrences
  skip-to-latest: run once
  run-all: run 4 times, possibly concurrently
  -> the policy must be explicit and per schedule
```

**Storing a schedule as a recurrence rule with a timezone, rather than as a series of UTC timestamps, is what makes daylight saving answerable.** Converting to UTC at registration means the job silently shifts by an hour twice a year relative to local time, which is wrong for anything users perceive as a local-time event. Keeping the rule and the zone lets the next occurrence be computed correctly, including the awkward cases, and lets the policy for those cases be stated.

**Storing job payloads as serialised code or closures makes long schedules unreliable.** A job registered six months ago must still be runnable after deploys that changed the code, and a serialised closure referencing classes that no longer exist cannot be. Storing a job type identifier and plain parameters keeps schedules decoupled from deployment, which is the property that makes long-horizon scheduling viable at all.

**Isolation between job types matters as much as in any queue.** A job type that runs for ten minutes and fails will occupy workers and its retries will compound, delaying short jobs that share the pool. Separate queues or concurrency limits per job type — the bulkhead pattern applied to scheduled work — keeps one team's slow report from delaying another team's time-sensitive job.

**Lateness, not throughput, is the metric that describes this system's health.** A scheduler processing fifty thousand jobs a minute while every one of them starts five minutes after its due time is failing at its actual purpose. Measuring the distribution of due-time-to-start-time, segmented by job type, reveals clustering, backlogs and starved queues in a way that execution counts never do.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Midnight clustering | Large backlog; everything late | Jitter at registration; capacity for the peak |
| Job execution fails | Occurrence not completed | Retry the execution, not the schedule |
| Recurrence advanced only on success | Schedule stops after one failure | Advance at dispatch |
| Worker crashes mid-job | Execution redelivered | Stable occurrence id; idempotent jobs |
| Two dispatchers claim one occurrence | Job runs twice | Skip-locked claim; unique execution record |
| Downtime spanning many occurrences | Ambiguous catch-up | Per-schedule missed-occurrence policy |
| Daylight saving transition | Job skipped or doubled | Store recurrence with timezone; define the policy |
| Slow job type saturating workers | Unrelated jobs delayed | Separate queues or concurrency limits per type |

> **A schedule that silently stops recurring is the worst failure this system has**  
> Nothing errors when a job simply never fires again — no alert, no failed execution, just an absence. Teams discover it weeks later when a report that should have arrived did not, and by then the missed work may be unrecoverable. The structural defence is advancing recurrence at dispatch so that failures cannot stop it; the operational defence is alerting on schedules that have not executed within their expected interval, which catches every cause including ones nobody anticipated.

That second defence is worth building even when the first is correct, because it detects the class of failure rather than a specific mechanism.

**The staff-level view**

The strong answer separates scheduling from execution and treats clustering and time as the real difficulties.

- **Split dispatch from execution** so that job code can never block the component deciding what is due.
- **Advance recurrence at dispatch**, and explain that advancing on success stops schedules silently when a run fails.
- **Raise the clustering problem** with numbers, and propose jitter at registration as the effective remedy.
- **Claim occurrences atomically** and give each an execution record that jobs can deduplicate on.
- **Address timezones and daylight saving** explicitly, including that storing UTC alone makes the question unanswerable.
- **Measure lateness rather than throughput**, since a punctual scheduler is the actual requirement.

> **The question that raises the level**  
> What is the policy for occurrences missed during an outage, and is it the same for every job? A billing run must execute every missed cycle; a dashboard refresh should skip to the latest; a notification may be worse than useless if delivered six hours late. Making this a per-schedule declaration rather than a platform-wide constant is the difference between a scheduler teams can rely on and one they each work around.

---

### Public API With Async Exports

*Serve synchronous API calls predictably while long-running exports proceed in the background, without either interfering with the other.*

**Flow:** `API gateway` → `Synchronous handlers` → `Export request` → `Job queue` → `Export workers` → `Download delivery`

**The brief**

A public API serves ordinary requests in milliseconds and also offers exports — a customer asking for every transaction from the last two years, which may be millions of rows and several gigabytes. The same platform, the same customers, the same underlying data.

Attempting to serve an export synchronously fails in every direction: the request times out, the connection is held for minutes, memory grows with the result set, and a handful of concurrent exports consume the capacity that ordinary requests need.

- **Predictability**: interactive API latency must be unaffected by export activity.
- **Completeness**: an export contains a consistent, complete view of the requested data.
- **Resumability**: a customer whose download fails can retrieve it again.
- **Fairness**: one customer's large export must not delay another's small one.
- **Scale**: 50,000 requests per second interactive; 5,000 exports per day up to 10 GB each.

> **An export is a job with a result, not a slow request**  
> Once the operation is reframed as work that is submitted, tracked and collected, every problem becomes tractable: it can be queued, prioritised, retried, resumed and rate-limited, and its result can be delivered from storage rather than through the API. The mistake that makes exports painful is treating them as requests that happen to take a while.

**Scale**

**Export cost and isolation arithmetic**

```text
INTERACTIVE
  50,000 requests/s, p99 under 100 ms
  -> the property that must be protected

EXPORTS
  5,000/day, average 500 MB, maximum 10 GB
  -> 2.5 TB/day generated
  -> each reads potentially millions of rows

WHY SYNCHRONOUS FAILS
  a 10 GB export at 50 MB/s of query throughput
  = 200 seconds of continuous reading
  -> exceeds every reasonable request timeout
  -> holds a connection, a thread and memory
  -> 20 concurrent exports = the whole read capacity

DATABASE ISOLATION
  export queries are long-running scans
  -> they compete with interactive queries for I/O
     and cache
  -> and in MVCC systems they hold snapshots open,
     preventing cleanup
  -> exports should read a replica, not the primary

GENERATION COST
  10 GB export, streamed and compressed
  -> memory must stay bounded: stream rows to the
     output, never materialise the result
  -> compressed output typically 10-20% of raw size

DELIVERY
  2.5 TB/day of downloads
  -> served from object storage with presigned URLs
  -> never through the API tier
```

| Metric | Value | Note |
|---|---|---|
| Interactive | protected | **unaffected** |
| Export read | replica only | not the primary |
| Memory | streamed | never materialised |
| Delivery | object storage | presigned |

Reading exports from a replica is the single most effective isolation measure. Long scans consume I/O and evict the cache entries interactive queries depend on, and in systems using multiversion concurrency control they hold a snapshot open for their whole duration, preventing cleanup across the database. Moving them to a replica removes all three effects from the primary at the cost of a small, well-defined staleness.

> **Materialising an export in memory before writing it will eventually exhaust a worker**  
> Building a ten-gigabyte result in memory and then writing it works during testing with small datasets and fails when a customer requests two years of data. Streaming rows from the query directly into a compressed output, flushing in bounded chunks to storage, keeps memory constant regardless of export size — and it is much harder to retrofit than to do initially.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Execution model | Synchronous, asynchronous job | Asynchronous with a status resource | Exports exceed any reasonable request lifetime |
| Data source | Primary, replica, warehouse | Replica, or a warehouse for large ranges | Long scans must not touch the interactive path |
| Consistency | Live read, snapshot at submission | Snapshot at submission | A multi-minute export must not mix old and new data |
| Result delivery | Through the API, presigned storage URL | Presigned URL with an expiry | Terabytes of downloads should not pass through the API tier |
| Fairness | First come first served, per-customer limits | Per-customer concurrency limits | One customer must not occupy every export worker |
| Retention | Indefinite, bounded window | Bounded, with the expiry communicated | Generated exports are large and mostly downloaded once |

1. **Accept the export request, validate it, return a job identifier immediately** — The customer gets an answer in milliseconds.
2. **Record the snapshot point at submission time** — The export reflects one consistent moment, not a smear.
3. **Queue the job with per-customer concurrency limits** — Fairness is enforced at admission, not by the workers.
4. **Stream rows from a replica into compressed output in storage** — Constant memory, no impact on the interactive path.
5. **Expose a status resource with progress and an eventual download link** — Customers need to know it is proceeding.
6. **Deliver via a presigned URL with an expiry, and state the retention** — Downloads bypass the API entirely.

Recording the snapshot point at submission is what makes an export coherent. Without it, a query running for four minutes sees data inserted during its own execution, so the result contains some of the changes that occurred while it ran and not others — producing exports that do not correspond to any actual state of the system and that cannot be reconciled against anything.

> **Ask before building it**  
> What does the customer do with the export? A one-off download and a nightly automated pull are different products: the second needs stable scheduling, predictable naming, incremental ranges and possibly delivery to the customer's own storage, and building it as a manual download feature means every customer writes the same fragile automation around it.

**Deep dive**

Three areas define this system: the job lifecycle, isolation of the heavy read path, and delivery of large results.

**The export lifecycle**

```text
SUBMIT
  POST /exports {range, filters, format}
  -> validate: is the range permitted? the filters
     supported? the estimated size acceptable?
  -> reject impossible requests immediately rather
     than failing an hour later
  -> record snapshot_at = now()
  -> return 202 with {export_id, status_url}

QUEUE
  enqueue with the customer's concurrency limit
  -> if they already have 3 running, it waits
  -> estimated size may route it to a different lane

EXECUTE
  stream from a replica as of snapshot_at
  write compressed chunks to object storage
  update progress periodically
  -> progress is why customers stop asking support
     whether it is working

COMPLETE
  finalise the object
  generate a presigned URL with an expiry
  update the status resource
  optionally notify by webhook or email

DOWNLOAD
  customer fetches directly from storage
  -> resumable by HTTP range requests
  -> never touches the API tier

EXPIRE
  after the retention window, delete the object
  -> status resource remains, marked expired
  -> customer can re-request
```

**Validating feasibility at submission prevents the worst customer experience this system can produce.** An export that is accepted, runs for ninety minutes and then fails because the range was too large wastes both the customer's time and the platform's capacity. Estimating the result size from the range and filters, and rejecting or splitting requests that exceed a threshold, moves that conversation to the first second rather than the last.

**Isolation in practice**

```text
WHY EXPORTS THREATEN INTERACTIVE TRAFFIC
  long scans evict hot cache entries
  sustained I/O competes with point queries
  open snapshots block vacuum and cleanup
  large result transfers saturate network paths

ISOLATION MEASURES  (in order of effect)
  1  read from a replica, never the primary
  2  a dedicated replica for exports if volume
     justifies it
  3  rate-limit rows read per second per export
       -> a slower export that does not disturb
          anything is better than a fast one that
          does
  4  separate worker pool from other background work
  5  schedule very large exports for off-peak where
     the customer accepts it

PER-CUSTOMER FAIRNESS
  concurrency limit per customer (e.g. 3)
  plus a global export worker pool
  -> one customer requesting 50 exports queues their
     own, not everyone else's
  -> and their 51st is rejected with a clear reason

SIZE-BASED LANES
  small exports (< 100 MB) in a fast lane
  large exports in a slow lane
  -> a customer's quick export is not stuck behind
     someone's 10 GB job
  -> this is the single most appreciated fairness
     feature
```

**Size-based lanes are what make the system feel responsive to most customers.** The majority of exports are small and would complete in seconds, and without lanes they queue behind multi-gigabyte jobs that take an hour. Routing by estimated size means the common case stays fast while large exports proceed at their own pace, which matches what customers expect far better than strict ordering does.

**Rate-limiting the export's own read throughput is counter-intuitive and correct.** Slowing an export deliberately, so that it reads a bounded number of rows per second, extends its duration and removes its impact on everything else — and customers care far more about their interactive API latency than about whether an export took twenty minutes or thirty. This is load shedding applied to the internal read path rather than to requests.

**Presigned delivery keeps terabytes off the API tier and makes downloads resumable.** A customer downloading a ten-gigabyte file over an unreliable connection needs range requests and retries, which object storage provides natively and an API endpoint would have to implement. It also means download bandwidth never sizes the application fleet, which for this volume is the difference between a modest API tier and one provisioned for bulk transfer.

**The status resource is the product surface for asynchronous work.** Customers cannot see into the queue, so progress, estimated completion, position and a clear terminal state — succeeded with a link, failed with a reason, expired with instructions — are what prevent support contacts. An asynchronous export with an opaque status is experienced as unreliable even when it always succeeds.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Export reads the primary | Interactive latency degrades | Read replicas only; dedicated replica if needed |
| Result materialised in memory | Worker exhausted on large exports | Stream to storage in bounded chunks |
| No snapshot point | Export mixes data from different moments | Record the snapshot at submission and read as of it |
| One customer floods the queue | Everyone else's exports delayed | Per-customer concurrency limits |
| Small exports behind large ones | Quick jobs take an hour | Size-based lanes |
| Export fails after an hour | Wasted time and capacity | Validate feasibility at submission; checkpoint progress |
| Download interrupted | Customer must start over | Presigned URL with HTTP range support |
| Storage costs growing | Generated exports retained indefinitely | Bounded retention, clearly communicated |

> **An export that degrades interactive API latency has failed even if it completes**  
> The platform's primary product is the synchronous API, and customers judge it on predictable latency. An export feature that occasionally makes every other request slower is worse than one that is slow itself, because the harm is spread across every customer rather than confined to the one who asked for the export. This is why isolation measures — replicas, rate limits, separate pools — take precedence over export completion time in every trade.

Stated as a rule: exports may be as slow as necessary, provided they are invisible to everyone not waiting for them.

**The staff-level view**

The strong answer reframes the export as a job immediately and spends its time on isolation and fairness.

- **Reframe exports as jobs**, with submission, status and collection as distinct steps rather than one long request.
- **Record a snapshot at submission** so the result corresponds to a single consistent moment.
- **Read from replicas and rate-limit the read itself**, accepting a slower export to protect interactive latency.
- **Stream to storage in bounded chunks**, never materialising the result in memory.
- **Add per-customer concurrency and size-based lanes**, since a small export queued behind a huge one is the common complaint.
- **Deliver by presigned URL** with resumable downloads and a stated retention window.

> **The question that raises the level**  
> Is this a human downloading a file, or a system pulling data on a schedule? The second case changes the design: it wants incremental ranges rather than full exports, stable scheduling, predictable naming and delivery to the customer's own storage. Building the human-facing version and letting customers automate around it produces a support burden that a purpose-built data-delivery feature would have avoided.

---

### Real-Time Analytics Dashboard

*Serve interactive aggregates over high-volume event data by precomputing what is asked repeatedly and querying raw data only when necessary.*

**Flow:** `Event ingest` → `Stream aggregation` → `Pre-aggregated store` → `Raw event store` → `Query router` → `Dashboard API`

**The brief**

A dashboard shows counts, rates and breakdowns over events arriving at high volume — page views, orders, errors, API calls. Users expect charts to load in under a second, to reflect the last few seconds of activity, and to support filtering and grouping they choose themselves.

Those requirements conflict. Arbitrary filtering over raw events is a scan; sub-second response requires precomputation; and precomputing every possible combination of dimensions is combinatorially impossible.

- **Latency**: dashboard panels load in under a second.
- **Freshness**: data no more than a few seconds old.
- **Flexibility**: users filter and group by several dimensions.
- **Retention**: recent data at full resolution, older data summarised.
- **Scale**: 500,000 events per second, 1,000 concurrent dashboard users.

> **Most queries are the same small set, and the rest can afford to be slower**  
> Dashboards are dominated by a handful of repeated queries — today's totals, the last hour by status, the standard breakdowns — while ad-hoc exploration is rare and its users expect to wait. Precomputing the common set and routing everything else to a columnar store over raw events serves both without attempting the impossible middle ground of precomputing everything.

**Scale**

**Ingest, aggregation and query arithmetic**

```text
INGEST
  500,000 events/s
  each ~300 B -> 150 MB/s, ~13 TB/day raw
  -> raw retention must be bounded or tiered

PRE-AGGREGATION
  dimensions: service (50), endpoint (500),
  status (10), region (20)
  full cross-product = 5,000,000 combinations
  x time buckets (1 min) x retention
  -> impossible to materialise completely

  instead materialise the combinations actually used
    service x status x minute       = 500 rows/min
    endpoint x status x minute      = 5,000 rows/min
    service x region x minute       = 1,000 rows/min
  -> a few thousand rows per minute, trivially
     queryable
  -> chosen from observed query patterns, not from
     the schema

QUERY MIX  (typical)
  85% hit a pre-aggregate     -> under 50 ms
  12% partial match, small scan -> 200-500 ms
  3% full ad-hoc scan          -> seconds
  -> the routing decision is what makes this work

RETENTION TIERS
  1-minute granularity   7 days
  1-hour granularity     90 days
  1-day granularity      2 years
  -> each rollup is ~1/60th the size of the previous
  -> total storage is dominated by the finest tier

CONCURRENCY
  1,000 users x ~8 panels, refreshing every 10 s
  = 800 queries/s against the aggregate store
  -> caching identical queries across users matters
```

| Metric | Value | Note |
|---|---|---|
| Ingest | 500,000/s | 13 TB/day raw |
| Pre-aggregates | thousands of rows/min | **not millions** |
| Query mix | 85% precomputed | under 50 ms |
| Rollups | 60× reduction | per tier |

Choosing which combinations to materialise from observed query patterns rather than from the dimension schema is what keeps this tractable. The full cross-product is millions of series; the set people actually look at is a few dozen. Instrumenting the query layer to record which groupings are requested, and materialising the top ones, converges quickly on a small set that covers the overwhelming majority of traffic.

> **Late-arriving events silently corrupt already-computed aggregates**  
> An event with a timestamp of 10:04 arriving at 10:09 belongs in a bucket that was closed and published five minutes ago. Ignoring it under-counts, and updating the bucket makes a value users already saw change retroactively. Both are defensible and the choice must be explicit — usually keeping buckets open for a grace period, then accepting the loss, with the grace period stated in the interface.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Query strategy | All precomputed, all on demand, hybrid routing | Hybrid with a router | Precomputing everything is combinatorial; scanning everything is slow |
| Aggregation | Batch, stream | Stream, with batch correction | Freshness in seconds rules out batch alone |
| Raw storage | Row store, columnar | Columnar | Analytical scans over few columns of many rows |
| Time buckets | Fixed 1 minute, multiple granularities | Multiple, rolled up progressively | Retention at full resolution is unaffordable |
| High-cardinality dimensions | Include, exclude from pre-aggregates | Exclude; serve from raw | User ids and request ids explode the combination count |
| Approximation | Exact counts, sketches for distinct | Sketches for cardinality | Exact distinct counts over billions of values are prohibitive |

1. **Ingest events into a partitioned log** — It feeds both the stream aggregator and the raw store.
2. **Aggregate in a stream processor into the chosen combinations** — Freshness comes from this path.
3. **Write raw events to a columnar store for ad-hoc queries** — The escape hatch for anything not precomputed.
4. **Route each query to the cheapest source that can answer it** — This routing is the core of the design.
5. **Roll up older buckets into coarser granularities** — Retention at minute resolution is not affordable for years.
6. **Use sketches for distinct counts** — Exact cardinality over high-volume dimensions is not worth its cost.

Excluding high-cardinality dimensions from pre-aggregation is a decision that must be made deliberately, because including one quietly destroys the whole approach. Adding user identifier as a dimension turns a few thousand rows per minute into millions, and the pre-aggregate store becomes as large as the raw data it was meant to summarise — at which point it provides no benefit and considerable cost.

> **Ask before building it**  
> Which queries must be fast, and which merely need to be possible? The answer determines what is materialised and what is routed to a scan, and it is a question about user behaviour rather than about data. Teams that cannot answer it typically build for the general case and end up with a system that is slow at everything.

**Deep dive**

Three mechanisms carry this system: streaming aggregation, query routing, and the handling of time.

**The dual-path architecture**

```text
EVENTS -> partitioned log
           │
           ├──> STREAM AGGREGATOR
           │      tumbling 1-minute windows
           │      keyed by each materialised
           │      combination
           │      -> pre-aggregate store
           │         (small, fast, indexed by time)
           │
           └──> RAW SINK
                  columnar files, partitioned by
                  time
                  -> ad-hoc query engine

QUERY ROUTER
  parse the query: dimensions, filters, time range
  1  exact match to a pre-aggregate?
       -> serve from it, under 50 ms
  2  does a materialised combination cover it?
       e.g. asked for service totals, have
       service x status
       -> aggregate over the finer one, still fast
  3  otherwise
       -> columnar scan over raw events
       -> slower, and the user is told

WHY THE ROUTER IS THE DESIGN
  without it, users either get a rigid dashboard
  (precomputed only) or a slow one (raw only)
  -> the router is what lets both properties coexist
  -> and its hit rate is the system's key metric

FEEDBACK LOOP
  record which queries fall through to raw scans
  -> the frequent ones become new pre-aggregates
  -> the materialised set tracks actual usage
```

**The feedback loop from missed queries to new pre-aggregates is what keeps the system aligned with usage.** Dashboards evolve, teams add panels, and a materialised set chosen at launch is wrong within months. Recording which queries required a raw scan, and promoting the frequent ones, turns the aggregation layer into something that adapts rather than something that ages.

**Time, lateness and correction**

```text
EVENT TIME vs ARRIVAL TIME
  events carry the time they occurred
  they arrive later, sometimes much later
  -> aggregate by EVENT time, or the numbers are
     wrong whenever ingestion lags

WATERMARK
  "no events older than T will be counted"
  typical lag: 30-60 s
  -> buckets close at watermark, then publish
  -> the dashboard's freshness is bounded by this

LATE EVENTS  (after the watermark)
  options:
    drop           simple, silently under-counts
    side output    collected, reported separately
    correct        update the bucket; values users
                   already saw change
  -> for operational dashboards, drop with a
     reported late-event rate is usually right
  -> for anything financial, correction is mandatory

BATCH CORRECTION
  a nightly batch job recomputes yesterday from raw
  events
  -> replaces the streamed aggregates
  -> catches late events, fixes streaming bugs
  -> this is the lambda arrangement, and it exists
     because streaming alone cannot be re-run
     against history

RETENTION ROLLUP
  after 7 days: collapse minute buckets to hourly
  after 90 days: collapse hourly to daily
  -> performed by batch, idempotently, by partition
```

**A nightly batch recomputation is worth its cost even when the streaming path is correct.** It catches late events that the watermark excluded, repairs the damage from any streaming bug deployed during the day, and provides a definitive version of history that the streaming path cannot produce because it cannot re-run against the past. The dashboard then shows streamed values for today and corrected values for previous days, which is honest and nearly always acceptable.

**Approximate distinct counts are the right default and should be labelled as such.** Exact cardinality over billions of values requires holding every value seen, whereas sketches provide an answer within a couple of per cent using a few kilobytes and merge across time buckets and dimensions cleanly. Users are almost always satisfied — provided the interface says approximately rather than implying precision the number does not have.

**Caching identical queries across users is effective because dashboards are shared.** A thousand people looking at the same operational dashboard issue the same queries within the same second, so a short-lived cache keyed on the query and time range collapses eight hundred queries per second into a few dozen. The cache lifetime is bounded by the freshness requirement, which at a few seconds is enough to capture the overlap.

**The raw store is what prevents the dashboard from becoming a cage.** Precomputed metrics answer known questions, and the valuable moments in operational analysis are the unknown ones — a specific customer's error pattern, a correlation nobody anticipated. Keeping raw events queryable, even slowly, is what allows those investigations, and systems that discard raw data after aggregation lose the ability to ask anything that was not foreseen.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Ingest lag | Dashboard shows stale or incomplete data | Surface the watermark; alert on consumer lag |
| Late events dropped | Silent under-counting | Report the late-event rate; batch correction overnight |
| High-cardinality dimension added | Pre-aggregate store explodes | Exclude from materialisation; serve from raw |
| Ad-hoc query scans everything | Slow, and competes with ingestion | Enforce time-range limits; separate query resources |
| Streaming aggregation bug | Wrong numbers, already displayed | Batch recomputation replaces the day |
| Rollup job fails | Storage grows; old data at full resolution | Idempotent rollups by partition; alert on lag |
| Dashboard refresh storm | Query load spikes on the minute | Jitter refresh intervals; cache identical queries |
| Raw retention exhausted | Cannot investigate past incidents | Tiered retention with a stated window |

> **A dashboard that is confidently wrong is worse than one that is visibly unavailable**  
> Operational decisions are made from these numbers, so an aggregation bug or silently dropped events lead people to act on figures that do not reflect reality — and because the dashboard renders normally, nothing indicates a problem. Surfacing the watermark, the late-event rate and the completeness of each time bucket lets viewers distinguish between the data being bad and the situation being bad, which is the difference between a tool and a hazard.

The corresponding operational practice is to alert on the pipeline's own health metrics — ingest lag, aggregation lag, rollup lag — with the same seriousness as on the business metrics the dashboard displays.

**The staff-level view**

The strong answer builds around query routing rather than trying to make one storage layer do everything.

- **Show the combinatorial problem** — millions of possible series against thousands actually viewed — and let it motivate selective materialisation.
- **Make the router the centrepiece**, routing each query to the cheapest source that can answer it.
- **Add a feedback loop** from missed queries to new pre-aggregates, so the materialised set tracks usage.
- **Aggregate by event time with an explicit watermark**, and state the late-event policy.
- **Keep a nightly batch correction**, because streaming cannot be re-run against history.
- **Surface data completeness in the interface**, since a confidently wrong dashboard drives wrong decisions.

> **The question that raises the level**  
> Are these numbers used for operational decisions or for financial reporting? Operational dashboards tolerate dropped late events and approximate distinct counts; financial figures tolerate neither, and require correction, auditability and exactness that change the design substantially. Systems built for the first and later used for the second produce disputes that nobody can resolve, because the pipeline was never designed to explain its own numbers.

---

### Data Warehouse Ingestion

*Move data from many operational sources into a warehouse reliably, with schema changes, late data and reprocessing treated as normal.*

**Flow:** `Source extraction` → `Landing zone` → `Validation and schema` → `Transformation` → `Warehouse tables` → `Lineage and quality`

**The brief**

Dozens of operational databases, event streams and third-party APIs must be consolidated into a warehouse where analysts and machine learning teams can query them together. Sources change their schemas without warning, deliver data late, and occasionally deliver it twice.

The warehouse's value depends on trust. A table that is sometimes incomplete, or whose numbers change retroactively without explanation, is worse than no table at all — because decisions will be made from it before anyone notices.

- **Completeness**: every source record eventually arrives, exactly once in the result.
- **Freshness**: operational data available for analysis within an hour, some sources within minutes.
- **Schema resilience**: a source adding or changing a column must not break the pipeline silently.
- **Reprocessability**: any day can be recomputed after a logic change or a bug.
- **Scale**: 200 sources, 50 TB of new data per day.

> **Keep the raw landing data forever and treat every transformation as derived**  
> Sources cannot be re-read at will — an operational database's history is truncated, an API's window has passed, a stream's retention expired. Landing raw extracts immutably and building everything downstream as recomputable transformations means any bug, schema change or new requirement can be applied retroactively, which is the property that makes a warehouse maintainable rather than a growing accumulation of irreversible decisions.

**Scale**

**Volume, partitioning and reprocessing arithmetic**

```text
DAILY VOLUME
  50 TB/day of new data across 200 sources
  heavily skewed: 5 sources produce 80%
  -> pipeline capacity is sized by the large sources
  -> but operational complexity is driven by the
     long tail

EXTRACTION MODES
  change data capture      continuous, low latency,
                           for transactional sources
  incremental batch        hourly or daily, by
                           updated-at watermark
  full snapshot            for small dimension
                           tables
  API pull                 rate-limited, paginated,
                           fragile
  -> most pipelines need all four

PARTITIONING
  land by source and ingestion date
    raw/source=orders/dt=2026-09-14/
  -> reprocessing a day means rewriting one
     partition
  -> without date partitioning, reprocessing means
     rewriting everything

REPROCESSING COST
  recompute 30 days of one large source
  = 30 partitions x ~2 TB = 60 TB read
  at 5 GB/s of cluster throughput -> ~3.3 hours
  -> acceptable if planned; impossible if the
     pipeline was not built for it

LATE DATA
  typical: 1-3% of records arrive after their
  partition was processed
  -> reprocess the affected partitions on a rolling
     basis
  -> "the last 3 days are provisional" is a common
     and honest policy

STORAGE GROWTH
  raw retained indefinitely: 50 TB/day = 18 PB/year
  -> tier aggressively; compress; columnar formats
```

| Metric | Value | Note |
|---|---|---|
| Daily | 50 TB | 200 sources |
| Partitioning | by source and date | **enables reprocessing** |
| Late data | 1-3% | rolling reprocess |
| Raw growth | 18 PB/year | tier it |

Partitioning by ingestion date is the decision that determines whether reprocessing is a routine operation or a project. With date partitions, fixing a transformation bug means rerunning the affected days and replacing those partitions atomically; without them, the same fix requires rewriting entire tables, which is expensive enough that teams patch data in place instead — and patched data cannot be reproduced, which is where warehouses start losing trust.

> **The long tail of small sources generates most of the operational burden**  
> Five sources produce eighty per cent of the volume and rarely fail; the other hundred and ninety-five produce little data and break constantly — expired credentials, changed schemas, rate limits, holiday outages. Capacity planning follows the volume, but staffing and alerting must follow the failure rate, which is concentrated in exactly the sources that look insignificant on a size chart.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Architecture | Transform then load, load then transform | Load raw, then transform | Raw data cannot be recovered later; transformations can be rerun |
| Extraction | Query the source, read its log | Change data capture where available | Queries load the source and miss deletes |
| Schema handling | Strict, permissive with detection | Permissive landing, strict transformation | A new column must not break ingestion, but must be noticed |
| Idempotency | Append, partition replacement | Replace the partition | Reruns must be safe; appending duplicates silently |
| Late data | Ignore, reprocess window | Rolling reprocess of recent partitions | Late records are routine, not exceptional |
| Quality | Check after loading, check as a gate | Gate publication on checks | Bad data that reaches analysts is worse than late data |

1. **Land raw extracts immutably, partitioned by source and date** — This is the recoverable base everything else derives from.
2. **Capture changes from the source log rather than querying it** — Queries add load and miss deleted rows entirely.
3. **Accept unexpected columns at landing, alert on them** — Ingestion must not break; humans must find out.
4. **Make every transformation an idempotent partition replacement** — Reruns are routine; appends corrupt silently.
5. **Reprocess a rolling window to absorb late data** — Recent partitions stay provisional until the window passes.
6. **Gate publication on data quality checks** — Row counts, null rates and referential checks before analysts see it.

Capturing changes from a source database's log rather than querying it repeatedly has a specific advantage beyond load: an incremental query by updated-at timestamp cannot see deleted rows, so deletions never propagate and the warehouse slowly accumulates records that no longer exist. Log-based capture emits the delete, which is the only way the warehouse stays faithful to the source.

> **Ask before building it**  
> Who is accountable when a source changes its schema? Ingestion pipelines break because upstream teams ship changes they have no reason to think about, and a pipeline with no contract has no recourse. Establishing schema contracts and change notification with the largest sources is organisational work that prevents more incidents than any technical measure.

**Deep dive**

Three areas define this platform: extraction, the transformation layer, and the quality controls that decide what analysts see.

**Landing and transformation layers**

```text
LAYER 1: RAW  (immutable)
  exactly as extracted, no transformation
  raw/source=orders/dt=2026-09-14/part-*.parquet
  -> never modified, retained long term
  -> the only thing that cannot be recomputed

LAYER 2: CLEANED  (derived, recomputable)
  types coerced, obvious corruption removed
  deduplicated by natural key and version
  schema enforced, unexpected columns preserved
  -> one transformation from raw, rerunnable

LAYER 3: MODELLED  (derived, recomputable)
  joined, conformed dimensions, business logic
  -> the tables analysts actually query

WHY THREE LAYERS
  a bug in business logic     -> rerun layer 3
  a bug in cleaning           -> rerun layers 2 and 3
  a source backfill           -> land it, rerun both
  -> the blast radius of any fix is bounded and
     obvious

PARTITION REPLACEMENT
  every transformation writes to a temporary
  location
  verifies row counts and checks
  then atomically swaps the partition
  -> readers never see a partial result
  -> and a failed run leaves the previous version
     intact

DEDUPLICATION
  change capture delivers at least once
  -> dedupe by (natural key, source version)
  -> keep the highest version per key per partition
```

**The layered structure bounds the cost of every kind of mistake, which is why it is worth the storage.** A business logic error affects only the final layer and is corrected by rerunning it; a parsing error affects two layers; a source problem requires re-landing. Without the separation, every fix is a full rebuild from whatever raw data still exists, and the temptation to patch tables in place becomes overwhelming.

**Quality gates and lineage**

```text
CHECKS BEFORE PUBLICATION
  volume      row count within expected bounds
              (a 90% drop is a broken extract, not a
               quiet day)
  nulls       null rate per column within tolerance
  uniqueness  primary keys actually unique
  referential foreign keys resolve
  freshness   the partition covers the expected
              period
  business    domain assertions (revenue is
              non-negative)

ON FAILURE
  do NOT publish
  keep the previous version visible
  alert the owning team
  -> analysts see yesterday's data, clearly marked,
     rather than today's wrong data

LINEAGE
  every table records which inputs and which code
  version produced it
  -> "why did this number change?" becomes
     answerable
  -> and the blast radius of a source problem is
     computable: which downstream tables are
     affected

FRESHNESS SIGNALLING
  every table exposes: last successful load, the
  period it covers, whether it is provisional
  -> analysts can distinguish "no orders today"
     from "the orders pipeline is broken"
  -> this distinction prevents most bad decisions
```

**Blocking publication on failed checks is the decision that preserves trust.** Publishing data that fails its own quality gates means analysts consume it, build reports on it and make decisions from it before anyone investigates — whereas holding the previous version and alerting means the worst case is staleness that is visible. Staleness is a known quantity; silently wrong data is not.

**Lineage turns a warehouse from a collection of tables into something explicable.** The most common question about any warehouse number is why it changed, and answering it requires knowing which sources, which transformations and which code version produced the value. Capturing that automatically, rather than reconstructing it from job logs, is what makes incidents resolvable in minutes rather than days.

**Schema evolution should be permissive at landing and strict downstream.** A source adding a column must not break ingestion — the pipeline should land it and carry on — while the transformation layer should fail loudly if a column it depends on disappears or changes type. This asymmetry means upstream changes cause a notification rather than an outage, and only changes that actually break something stop the pipeline.

**The provisional window should be stated in the interface, not buried in documentation.** If the last three days are subject to change as late data arrives, analysts need to see that when they query, because a report produced on Monday from provisional Sunday data will not match the same report run on Wednesday. Marking recent partitions as provisional makes the behaviour predictable instead of mysterious.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Source schema changed | Transformation fails or silently drops data | Permissive landing; strict transformation; alert on new columns |
| Extract runs twice | Duplicate rows | Idempotent partition replacement, never append |
| Deletes not propagated | Warehouse retains rows the source removed | Log-based capture rather than incremental queries |
| Late data arrives | Partitions under-count | Rolling reprocess window; mark recent data provisional |
| Quality check fails | Bad data would reach analysts | Block publication; keep the previous version |
| Small source silently stops | A table quietly stops updating | Freshness alerts per table, not just job success |
| Reprocessing needed but impossible | Cannot fix historical data | Retain raw landing data indefinitely |
| Job succeeds with zero rows | Empty table published | Volume checks with expected bounds |

> **A pipeline that succeeds while producing nothing is the failure that goes unnoticed longest**  
> An extract whose credentials expired may return an empty result with a success status, and a transformation over no rows completes perfectly, so every job in the chain reports green while a table quietly stops being updated. Analysts continue querying it and see a business that appears to have stopped. Volume checks against expected bounds and per-table freshness alerts catch this; job-level success monitoring never will.

This is the general lesson of warehouse operations: monitor the data, not the jobs, because a healthy job producing wrong or absent data is the normal shape of a data incident.

**The staff-level view**

The strong answer treats reprocessing as a first-class requirement and quality gates as a trust mechanism.

- **Land raw and transform downstream**, because raw data cannot be recovered later while transformations can always be rerun.
- **Partition by source and date** so that reprocessing is a partition replacement rather than a table rebuild.
- **Use log-based capture**, noting specifically that incremental queries never see deletions.
- **Be permissive at landing and strict in transformation**, so upstream changes notify rather than break.
- **Gate publication on quality checks**, keeping the previous version visible when they fail.
- **Monitor data freshness and volume per table**, because green jobs can produce empty tables.

> **The question that raises the level**  
> How long are the last few days considered provisional, and does the interface say so? Late data means recent partitions change after publication, so a report run today and rerun on Friday will differ — and analysts who do not know that will treat it as a bug, or worse, will not notice. Making the provisional window explicit converts a recurring source of confusion into a documented property of the platform.

---

### Metrics And Alerting

*Collect time-series data from thousands of services, evaluate alert rules continuously, and page a human only when someone needs to act.*

**Flow:** `Instrumentation` → `Collection agents` → `Time-series store` → `Rule evaluation` → `Alert routing` → `Dashboards and queries`

**The brief**

Thousands of service instances emit metrics continuously. The platform must store them, let engineers query them interactively during incidents, and evaluate alerting rules that decide when to wake someone up.

The difficulty is not volume, which is well understood, but signal. An alerting system that fires too often trains people to ignore it, and one that fires too rarely fails at its only job — and both failures are invisible from inside the system.

- **Ingest**: millions of samples per second from thousands of sources.
- **Query**: interactive exploration over recent data in under a second.
- **Alerting**: rules evaluated continuously, with alerts delivered within a minute.
- **Retention**: high resolution recently, summarised for longer.
- **Reliability**: the observability platform must survive the outages it is meant to observe.

> **Every alert must correspond to an action a human should take now**  
> Systems accumulate alerts for conditions that are interesting rather than actionable, and each one erodes the attention that genuine incidents need. The discipline is not technical — it is that a rule may exist only if someone can say what the person woken by it should do. Alerting on symptoms users feel, rather than on causes, is what keeps that list short.

**Scale**

**Cardinality and storage arithmetic**

```text
SERIES COUNT  (the thing that matters)
  5,000 instances x 200 metrics each
  = 1,000,000 base series
  each metric with labels multiplies this:
    status (5) x endpoint (50) x method (4)
    = 1,000 label combinations per metric
  -> a single badly labelled metric can create
     millions of series

SAMPLES
  1,000,000 series at 15 s resolution
  = ~67,000 samples/s
  with label expansion, realistically 2-5 M/s

STORAGE
  a compressed sample is ~1-2 bytes in a modern
  time-series format
  3,000,000 samples/s x 86,400 x 1.5 B
  = ~390 GB/day
  -> 15 days at full resolution: ~5.8 TB
  -> downsampled to 5-minute beyond that: 20x less

THE CARDINALITY TRAP
  adding user_id as a label to one metric
  100,000 users x 5 statuses = 500,000 new series
  -> memory in the ingest path, not just storage
  -> a single deploy can double the platform's
     footprint
  -> this is the most common way metrics systems fail

QUERY COST
  a dashboard panel over 7 days at 15 s resolution
  = 40,000 points per series
  x 50 series = 2 M points scanned
  -> pre-downsampled tiers make this tractable
```

| Metric | Value | Note |
|---|---|---|
| Series | millions | **cardinality is the limit** |
| Samples | 2-5 M/s | compressed to ~1.5 B |
| Retention | 15 d full | then downsampled |
| Failure mode | cardinality explosion | one label |

Cardinality, not sample volume, is what bounds a metrics platform. Samples compress extremely well and scale predictably; series count drives index size, memory in the ingest path and query planning cost, and a single label with unbounded values — a user identifier, a request identifier, a URL with parameters — can multiply the platform's footprint in one deploy. Enforcing cardinality limits per metric at ingestion is the protection that matters most.

> **The observability platform must not depend on the systems it observes**  
> An alerting pipeline that runs on the same cluster, uses the same database and routes through the same network as the services it monitors will be unavailable during exactly the incidents it exists to report. Isolation — separate infrastructure, separate failure domains, and an independent path for paging — is what makes it trustworthy, and it is the requirement most frequently compromised for convenience.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Collection | Push from services, pull from a scraper | Pull, with push for short-lived jobs | Pull gives discovery and detects absence; push suits ephemeral workloads |
| Storage | General database, time-series store | Purpose-built time-series store | Compression and time-range access patterns are specialised |
| Cardinality control | Unlimited, enforced limits | Enforced per metric at ingestion | A single bad label can take down the platform |
| Alert evaluation | In the query layer, dedicated evaluator | Dedicated, running independently | Alerting must work when the query layer is struggling |
| Alert basis | Cause-based, symptom-based | Symptom-based primarily | Users feel symptoms; causes are numerous and change |
| Retention | Uniform, tiered | Tiered with downsampling | Full resolution for years is unaffordable and unnecessary |

1. **Instrument with a small number of well-chosen metrics per service** — More metrics is not more insight; it is more cardinality.
2. **Enforce per-metric cardinality limits at ingestion** — Reject or truncate labels that explode rather than accepting them.
3. **Store at high resolution briefly and downsample for retention** — Investigations use recent data; trends use summarised data.
4. **Evaluate alert rules in an independent component** — Alerting must survive query-layer problems.
5. **Alert on symptoms with a defined action** — If nobody knows what to do, the rule should not page.
6. **Route by ownership with escalation and suppression** — An alert reaching nobody is the same as no alert.

Preferring pull-based collection gives one property that push cannot: the absence of a target is itself detectable. A scraper that fails to reach an instance knows it, whereas a push-based system cannot distinguish a service that has stopped reporting from one that has stopped existing — and instances disappearing silently is a common failure the platform should catch.

> **Ask before building it**  
> For each proposed alert, what should the person do when it fires? If the answer is look at a dashboard, it is not an alert but a dashboard. If it is nothing right now, it belongs in a report. Applying this test to an existing alert set typically removes more than half of it, and the remaining alerts get taken seriously.

**Deep dive**

Three areas define this platform: the ingest path and its cardinality controls, alert evaluation, and the routing that gets an alert to the right person.

**Alerting that people trust**

```text
SYMPTOM-BASED  (page on these)
  error rate above X% for 5 minutes
  p99 latency above the objective for 10 minutes
  request rate dropped to near zero
  queue age exceeding the processing deadline
  -> each corresponds to users being affected

CAUSE-BASED  (usually do not page)
  CPU above 80%
  memory above 70%
  a single instance unhealthy
  disk 60% full
  -> these may be normal; they may be harmless;
     they may not affect anyone
  -> useful on dashboards, poor as pages

THE EXCEPTION
  causes that reliably become symptoms with enough
  warning to act
    disk filling within 4 hours at the current rate
    certificate expiring in 7 days
  -> these page because there IS an action and a
     deadline

ALERT QUALITY METRICS
  pages per week per on-call person
    -> above ~2-3, fatigue sets in
  percentage of pages requiring action
    -> below ~80%, trust erodes
  percentage of incidents preceded by a page
    -> the coverage measure
  -> these three numbers describe whether alerting
     works; nothing else does

FOR EVERY RULE
  what does the responder do?
  what happens if nobody responds for an hour?
  -> if the answers are "look at it" and "nothing",
     delete the rule
```

**Measuring the fraction of pages that required action is the only honest assessment of an alerting system.** A platform can evaluate rules flawlessly and still be failing if most pages are noise, because the cost is paid in the attention available for real incidents. Reviewing that ratio regularly, and deleting or demoting rules that fail it, is maintenance work that alerting systems need continuously and rarely receive.

**Ingest, cardinality and reliability**

```text
INGEST PATH
  scrapers discover targets from service discovery
  pull metrics on an interval
  -> a target that disappears is immediately visible
  -> short-lived jobs push to a gateway instead

CARDINALITY ENFORCEMENT
  per metric: a maximum number of distinct series
  on exceeding it:
    drop the new series, record the violation,
    alert the owning team
  -> the alternative is the platform degrading for
     everyone because of one team's label
  -> this must be automatic; review processes do not
     catch it in time

HIGH-CARDINALITY DATA BELONGS ELSEWHERE
  per-request identifiers -> tracing
  per-user detail          -> logs or events
  -> metrics are for aggregates; pushing detail into
     labels is the fundamental misuse

RELIABILITY OF THE PLATFORM ITSELF
  separate infrastructure from the observed systems
  alert evaluation independent of the query layer
  an independent path to page (a second provider)
  a small external check that verifies the platform
  is alive
    -> "who watches the watcher" answered explicitly
  -> and alert rules that fire when ingestion stops,
     delivered by a different mechanism
```

**Automatic cardinality enforcement is not a courtesy to the platform team; it is what keeps one team's mistake from becoming everyone's outage.** A label added innocently in a deploy can multiply series count tenfold within minutes, exhausting memory in the ingest path and degrading queries for every service. Rejecting the excess automatically, with a clear alert to the owning team, contains the damage and provides the feedback that prevents a repeat.

**Routing and suppression determine whether an alert accomplishes anything.** An alert that reaches a rota nobody is on, or that fires a hundred times for one underlying cause, is functionally equivalent to no alert at all. Grouping related alerts, suppressing dependent ones when an upstream cause is already firing, and escalating when nobody acknowledges are what convert rule evaluation into a response.

**Dashboards and alerts serve different purposes and should be built differently.** A dashboard is for a human exploring a situation with context and judgement; an alert is a machine deciding that someone must act. Metrics that are valuable on a dashboard are frequently poor alert conditions, and treating every interesting chart as a candidate alert is how alert fatigue begins.

**Retention tiers should follow how data is actually used.** Incident investigation uses the last few hours at full resolution, capacity planning uses months at coarse resolution, and almost nothing uses week-old data at fifteen-second granularity. Downsampling aggressively past a short window reduces storage by more than an order of magnitude and affects essentially no real query.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Cardinality explosion | Ingest memory exhausted; queries slow for everyone | Per-metric limits enforced automatically |
| Alert fatigue | Real incidents ignored | Measure actionable-page ratio; delete noisy rules |
| Platform depends on the observed system | Blind during the incident | Separate infrastructure and failure domains |
| Alert storm from one cause | Hundreds of pages, one problem | Grouping and dependency suppression |
| Ingestion stops silently | Dashboards show flat lines that look normal | Dead-man alert on ingest, via an independent path |
| Rule evaluation lagging | Alerts delivered minutes late | Dedicated evaluator; monitor evaluation latency |
| Nobody acknowledges a page | Incident unattended | Escalation policy with a second and third tier |
| Query overload during an incident | Everyone queries at once; platform slows | Query resource limits; separate alerting path |

> **A metrics system that stops collecting looks exactly like a system with nothing to report**  
> Flat lines and absent alerts are indistinguishable from a quiet period, so an ingestion failure can persist for hours while everyone assumes things are healthy. The defence is a dead-man alert — a rule that fires when data stops arriving, delivered through a path independent of the platform itself — which is the only construct that turns absence into a signal. Without it, the most dangerous failure of an observability platform is also its quietest.

The same reasoning applies to alert delivery: a periodic synthetic alert that must arrive confirms the entire pipeline from rule evaluation to someone's phone.

**The staff-level view**

The strong answer treats alert quality as the primary engineering problem and cardinality as the primary scaling one.

- **Name cardinality as the binding constraint**, with the arithmetic showing how one label multiplies series count.
- **Enforce cardinality limits automatically**, because review processes do not act within the minutes required.
- **Alert on symptoms, not causes**, and apply the test that every rule must have a defined action.
- **Measure the actionable-page ratio**, since it is the only honest assessment of whether alerting works.
- **Isolate the platform from what it observes**, including an independent paging path.
- **Add a dead-man alert**, because silence and health are otherwise indistinguishable.

> **The question that raises the level**  
> How many pages does the on-call person receive in a typical week, and what fraction required action? These two numbers describe the alerting system better than any architectural detail, and teams that cannot answer them are almost certainly running an alert set that has accumulated rather than been designed. Improving them usually means deleting rules, which is the work nobody schedules.

---

### Transactional Ledger

*Record money movements as immutable double-entry postings that always balance, with balances derived rather than stored and corrections made by reversal.*

**Flow:** `Transfer request` → `Idempotency check` → `Balanced posting` → `Ledger append` → `Balance projection` → `Reconciliation`

**The brief**

A platform holds balances for users and merchants and moves money between them: payments, payouts, refunds, fees and adjustments. Every movement must be recorded exactly once, must be attributable, and must be reconstructible years later for audit.

Unlike most systems, being approximately right is worthless here. A balance that is occasionally off by a small amount is not a minor defect — it is an accounting failure that must be investigated, explained and corrected through a process, not a patch.

- **Correctness**: no money is created or destroyed; every entry balances.
- **Immutability**: recorded entries are never modified; corrections are new entries.
- **Auditability**: the full history of how any balance was reached is retrievable.
- **Idempotency**: a retried request produces one movement, never two.
- **Scale**: 5,000 transactions per second, tens of millions of accounts, years of history.

> **The ledger is an append-only log of balanced entries; a balance is a query over it**  
> Storing balances as mutable numbers makes every movement a read-modify-write that can be lost, doubled or reordered, and destroys the history that explains how a figure was reached. Recording immutable double-entry postings and deriving balances from them means the truth is the history, corrections are additions rather than edits, and any balance can be recomputed and explained at any point in time.

**Scale**

**Throughput, contention and storage arithmetic**

```text
TRANSACTIONS
  5,000/s peak
  each: 2+ entries (debit and credit)
  -> 10,000+ ledger rows/s
  -> ~900 M rows/day at peak rates

STORAGE
  entry ~150 B with indexes
  10,000/s x 86,400 x 150 B ≈ 130 GB/day at peak
  realistically with average load: ~20-40 GB/day
  -> years of retention: tens of TB
  -> partition by time; archive cold periods; never
     delete

CONTENTION
  most accounts are quiet
  a few are extremely hot:
    the platform fee account
    a payment processor's settlement account
  -> every transaction touches one of these
  -> 5,000/s against a single account row
  -> this is the real scaling problem

BALANCE COMPUTATION
  summing all history per query is untenable
  -> snapshot balances periodically
  -> balance = snapshot + entries since
  -> snapshot every N entries or every hour per
     account

RECONCILIATION
  daily comparison against external sources
    processor settlements, bank statements
  -> mismatches must be zero, and investigated when
     not
  -> this is the control that catches everything the
     live path missed
```

| Metric | Value | Note |
|---|---|---|
| Entries | 10,000/s | two per transaction |
| Hot accounts | 5,000/s on one row | **the real problem** |
| Balances | snapshot plus delta | not full history |
| Reconciliation | daily, must be zero | the control |

The hot account problem is what distinguishes ledger engineering from ordinary transactional systems. Every fee posting credits the same platform account, so that single row is touched by every transaction and becomes a serialisation point — which cannot be solved by sharding the ledger, because the contention is on one account by definition and its balance must remain correct.

> **Splitting a hot account's balance across sub-accounts changes what the balance means**  
> Dividing the platform fee account into a hundred sub-accounts removes the contention and means no single query returns the platform's fee balance atomically — it becomes a sum of values read at slightly different moments. For an internal account whose balance is reported daily this is entirely acceptable; for an account with a real-time constraint it is not, and the distinction has to be made per account rather than as a blanket technique.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Model | Mutable balances, double-entry postings | Double-entry, append-only | Balances lose history and cannot be audited |
| Balance storage | Derived on read, snapshot plus delta | Snapshot plus delta | Summing full history per query does not scale |
| Corrections | Update the entry, post a reversal | Reversal entries | Immutability is the audit property; edits destroy it |
| Idempotency | Client discipline, server-enforced key | Server-enforced key in the same transaction | Retries are certain and duplicates are money |
| Money representation | Floating point, integer minor units | Integer minor units with explicit currency | Floating point cannot represent decimal amounts exactly |
| Hot accounts | Single row, sub-accounts | Sub-accounts where the balance tolerates aggregation | Contention otherwise caps throughput at one row's write rate |

1. **Record every movement as a set of entries that sum to zero** — The invariant is checked at write time and is the core guarantee.
2. **Store amounts as integers in minor units with a currency code** — Floating point introduces rounding errors that accumulate.
3. **Claim an idempotency key in the same transaction as the entries** — The key's existence and the movement must be inseparable.
4. **Snapshot account balances periodically** — Balance queries read a snapshot plus recent entries.
5. **Correct by posting reversals, never by modifying entries** — The history must show what happened and what was corrected.
6. **Reconcile daily against external sources** — The live path will miss things; this is how they are found.

Using integers in minor units rather than floating point is not a stylistic preference. Binary floating point cannot represent most decimal fractions exactly, so amounts accumulate tiny errors that eventually produce balances that do not sum correctly — and in a system whose entire purpose is that things sum correctly, that is a defect with no acceptable magnitude.

> **Ask before building it**  
> Which balances must be accurate in real time, and which are reported periodically? A user's spendable balance must be exact at the moment of authorisation; the platform's aggregate fee revenue is reported daily. That distinction decides which accounts can be split for throughput and which must remain a single atomically readable row.

**Deep dive**

Three mechanisms carry this system: the balanced posting, balance derivation, and reconciliation with the outside world.

**A transfer as balanced entries**

```text
TRANSFER: user pays a merchant 100.00, platform fee
2.50

  entry 1  DEBIT  user:4471       -10000
  entry 2  CREDIT merchant:88     +9750
  entry 3  CREDIT platform:fees   +250
  -> sum = 0

THE INVARIANT
  every transaction's entries sum to zero
  -> checked at write time, in the same transaction
  -> a set that does not balance is rejected, not
     recorded
  -> money can therefore never be created or
     destroyed by a bug in one service

TRANSACTION RECORD
  {transaction_id, idempotency_key, type, timestamp,
   entries[], metadata}
  -> one atomic write
  -> the idempotency key claimed in the same
     transaction

IMMUTABILITY
  entries are never updated or deleted
  a mistake is corrected by a REVERSAL:
    DEBIT  merchant:88     -9750
    CREDIT user:4471       +9750
  -> the original stands; the correction is visible
  -> "why is my balance this?" is always answerable

WHY DOUBLE ENTRY
  it makes the invariant checkable locally
  -> any transaction can be validated on its own
  -> and the whole ledger's integrity is the sum of
     local checks
```

**The balancing invariant is what makes the system self-checking.** Because every transaction must sum to zero, a bug that credits without debiting is rejected at write time rather than discovered months later as an unexplained discrepancy. This is a much stronger guarantee than validating balances afterwards, since it turns a global property — money is conserved — into a local check on every write.

**Balances and hot accounts**

```text
DERIVING A BALANCE
  naive: SUM(amount) WHERE account = X
  -> years of entries; unusable

  snapshot approach:
    balance_snapshot {account, as_of_entry_id,
                      amount}
    balance = snapshot.amount
              + SUM(entries after
                    snapshot.as_of_entry_id)
  -> snapshot hourly, or every 1,000 entries
  -> the delta is small and fast to sum

SNAPSHOTS ARE DERIVED, NOT AUTHORITATIVE
  they can always be rebuilt from entries
  -> a corrupt snapshot is recoverable
  -> a corrupt ledger is not, which is why entries
     are immutable

HOT ACCOUNT MITIGATION
  platform:fees receives 5,000 credits/s
  -> single row: serialised, caps throughput

  SPLIT
    platform:fees:0 .. platform:fees:99
    each transaction picks one at random
    -> contention divided by 100
    -> balance = sum of 100 sub-accounts
    -> NOT atomic, but this account is reported
       daily

  DO NOT SPLIT
    a user's spendable balance
    -> the authorisation check must be atomic
    -> and a user account is not hot anyway

AUTHORISATION CHECK
  "does this user have sufficient funds?"
  -> must read the balance and post atomically
  -> a conditional insert that fails if the
     resulting balance would be negative
```

**The overdraft check must be part of the posting transaction, not a separate read.** Reading a balance, deciding it is sufficient and then posting leaves a window in which another transaction spends the same funds, so two payments can both pass a check that only one should have. Making the insufficiency condition part of the write — so the posting fails atomically if it would take the balance negative — is the only correct arrangement.

**Reconciliation against external systems is the control that catches everything else.** However careful the ledger, the authoritative record of money actually moved lives with banks and processors, and comparing their settlement files against the ledger daily is what surfaces charges with no entry, entries with no charge, and amounts that differ. A ledger without reconciliation is internally consistent and may still be wrong about reality.

**Immutability makes the ledger auditable and makes operations harder, deliberately.** Correcting a mistake requires understanding it well enough to post the right reversal, which is slower than editing a row and is the point: the history shows what happened and what was done about it. Systems that allow edits under operational pressure lose the property that made the ledger worth building.

**Multi-currency turns one invariant into several.** Entries must balance within each currency, and a conversion is two balanced transactions linked by an exchange record rather than one transaction spanning currencies. Attempting to balance across currencies requires a rate inside the invariant, which means the ledger's correctness depends on a number that changes — and that is not a property any ledger should have.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Duplicate transaction | Money moved twice | Idempotency key claimed in the same transaction as the entries |
| Unbalanced entries | Money created or destroyed | Reject at write time; the invariant is checked, not assumed |
| Race on an overdraft check | Balance goes negative | Make the check part of the atomic write |
| Hot account contention | Throughput capped at one row | Sub-accounts where the balance tolerates aggregation |
| Snapshot corrupted | Wrong balances reported | Rebuild from entries; snapshots are derived |
| Floating point rounding | Balances that do not sum | Integer minor units only |
| Entry edited to fix a mistake | Audit trail destroyed | Reversal entries; never modify |
| Reconciliation mismatch | Ledger disagrees with the bank | Investigate individually; never adjust to match |

> **Adjusting the ledger to make reconciliation balance destroys the thing reconciliation was measuring**  
> When a daily comparison shows a discrepancy, the tempting response is to post an adjustment that makes the numbers agree. That removes the signal without addressing the cause, and the next discrepancy is harder to interpret because the baseline is no longer trustworthy. Every mismatch must be traced to a specific cause and corrected by a reversal that describes what actually happened — which is slower, and is why the ledger remains believable.

A ledger's value is entirely in its credibility; any practice that trades credibility for operational convenience eventually costs more than it saved.

**The staff-level view**

The strong answer establishes immutability and the balancing invariant immediately, then addresses hot accounts and reconciliation.

- **Open with double-entry and the zero-sum invariant**, checked at write time rather than validated afterwards.
- **Derive balances from snapshots plus deltas**, and note that snapshots are rebuildable while entries are not.
- **Insist on integer minor units**, since floating point cannot represent decimal amounts exactly.
- **Make the overdraft check part of the write**, because a separate read-then-post races.
- **Address hot accounts** with sub-accounts, and be explicit that this changes what the balance means.
- **Require daily reconciliation** and state that mismatches are investigated, never adjusted away.

> **The question that raises the level**  
> Which accounts need an atomically readable balance, and which are aggregated for reporting? This single distinction decides where contention can be relieved by splitting and where it cannot, and it is the question that separates a ledger that scales from one that either caps out on a hot row or quietly gives up the atomicity a user's spendable balance requires.

---

### Multi-Region SaaS

*Serve tenants from the region nearest to them, keep each region authoritative for its own data, and survive losing any one of them.*

**Flow:** `Global routing` → `Regional stacks` → `Tenant home region` → `Cross-region replication` → `Global control plane` → `Failover orchestration`

**The brief**

A business application serves organisations worldwide. Users expect responsive interaction wherever they are, customers in some jurisdictions require their data to remain within specific borders, and the business requires that losing a region does not mean losing the service.

These requirements interact awkwardly: low latency wants data near users, residency wants data confined to regions, and availability wants data in more than one place — and a single global database satisfies none of them well.

- **Latency**: interactive requests under 100 milliseconds for users in served regions.
- **Residency**: a tenant's data can be constrained to a named region and provably stays there.
- **Availability**: a full region loss degrades but does not stop the service.
- **Consistency**: a tenant's own data must be strongly consistent to its users.
- **Scale**: 50,000 tenants, 4 regions, most tenants concentrated in one region each.

> **Partition by tenant, not by geography, and give each tenant a home**  
> Business applications have a property that consumer systems lack: almost all access to a tenant's data comes from that tenant's own users, who are usually in one place. Making each tenant authoritative in one region gives local reads and local writes with strong consistency, satisfies residency by construction, and reduces cross-region traffic to the rare case — which is a far better position than trying to replicate everything everywhere.

**Scale**

**Latency, replication and failover arithmetic**

```text
INTER-REGION LATENCY  (round trip)
  same continent      10-40 ms
  transatlantic       ~80 ms
  transpacific        ~150 ms
  Europe to Australia ~250 ms
  -> a synchronous cross-region write adds this to
     EVERY write, permanently

SINGLE-HOME TENANT
  user in the tenant's home region
    read  local, ~5 ms
    write local, ~10 ms
  user travelling, accessing from another region
    read  cross-region, +80-250 ms
    -> acceptable because it is rare
    -> or serve a read replica locally with
       staleness

REPLICATION FOR DISASTER RECOVERY
  asynchronous to a paired region
  typical lag 100 ms - 2 s
  -> recovery point objective = the lag
  -> "we may lose up to 2 seconds of writes" must be
     an accepted business position

FAILOVER TIME
  detect region failure      30-90 s
  promote the replica        10-60 s
  redirect traffic (DNS/anycast) 30-300 s
  -> realistic recovery time: 5-15 minutes
  -> claiming seconds is not credible

TENANT DISTRIBUTION
  50,000 tenants across 4 regions
  -> failover of one region moves ~12,500 tenants
  -> the paired region must have capacity for both
  -> which means running at under 50% utilisation,
     or accepting degradation
```

| Metric | Value | Note |
|---|---|---|
| Cross-region | 80-250 ms | **physics** |
| Async lag | 0.1-2 s | equals the RPO |
| Failover | 5-15 min | DNS dominates |
| Capacity | under 50% | to absorb a failover |

The capacity implication of failover is frequently overlooked. If a region's tenants must be absorbed by its pair, that pair needs enough headroom to run both workloads — which means either persistent over-provisioning or an explicit decision that service will be degraded after a failover. Both are defensible; discovering the question during an incident is not.

> **Failover capacity and failover correctness are separate problems and both must be tested**  
> A failover that promotes the replica correctly and then collapses under double load has failed just as completely as one that promoted the wrong thing. Regular exercises that actually move production tenants between regions are what reveal whether the capacity, the promotion path and the traffic redirection all work together — and they routinely reveal that one of the three does not.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Data partitioning | Global database, per-tenant home region | Per-tenant home region | Gives local reads and writes plus residency by construction |
| Routing | DNS by geography, routing by tenant | By tenant, resolved at the edge | A tenant's data is in one place; the user must reach it |
| Replication | Synchronous, asynchronous | Asynchronous to a paired region | Synchronous adds inter-region latency to every write |
| Control plane | Per region, single global | Single global, regionally replicated | Tenant placement and routing must have one answer |
| Cross-region access | Proxy to home, local replica | Proxy to home for writes, optional local read replica | Correctness first; staleness only where acceptable |
| Failover | Automatic, operator-initiated | Operator-initiated with automated execution | Automatic regional failover risks split brain for a transient problem |

1. **Assign every tenant a home region at creation, recorded centrally** — Everything downstream depends on knowing where a tenant lives.
2. **Route requests by tenant identity, resolved as early as possible** — Geographic routing sends users to regions that do not hold their data.
3. **Keep each tenant's writes within its home region** — Strong consistency for a tenant's own users, with no cross-region coordination.
4. **Replicate asynchronously to a paired region for recovery** — Accept and state the recovery point objective.
5. **Run the control plane globally with regional replicas** — Tenant placement must not be ambiguous during a partition.
6. **Make failover operator-initiated and fully automated once triggered** — Human judgement on whether, machine speed on how.

Making regional failover operator-initiated rather than automatic is a deliberate rejection of automation in one specific place. A partition between regions is indistinguishable from a region failing, and automatic promotion on both sides produces two authoritative copies of tenants' data — the one outcome that is genuinely unrecoverable. A person confirming that the region is actually gone costs minutes and removes that risk entirely.

> **Ask before building it**  
> How much data may a tenant lose if their region disappears right now? Asynchronous replication makes the answer equal to the replication lag, and it is a business decision rather than a technical one. Customers who cannot accept any loss need synchronous replication and will pay for it in write latency on every request — which is a product tier, not a default.

**Deep dive**

Three areas define this architecture: routing to the right region, the control plane that decides placement, and the failover path.

**Routing and the tenant home**

```text
REQUEST ARRIVES AT THE NEAREST EDGE
  1  identify the tenant
       from the subdomain, the token, or the path
  2  look up the tenant's home region
       cached at the edge, changes rarely
  3  route accordingly
       home region is here    -> serve locally
       home region elsewhere  -> proxy, or redirect

WHY NOT GEOGRAPHIC ROUTING
  a user in Singapore belonging to a European tenant
  -> geographic routing sends them to Asia
  -> the data is in Europe
  -> either a cross-region database call per query
     (worst case) or a redirect (correct)
  -> tenant-based routing gets it right the first
     time

TRAVELLING USERS
  rare, and their latency is unavoidable
  -> proxy at the edge keeps one cross-region hop
     rather than many
  -> an optional local read replica can serve
     read-only views with stated staleness

RESIDENCY ENFORCEMENT
  residency is not a routing preference; it is a
  constraint
  -> the tenant's data must not be replicated outside
     permitted regions
  -> backups, logs, caches and analytics exports all
     count
  -> enforcement belongs in the platform, not in each
     service's discipline
```

**Residency applies to everything derived from the data, not just the primary store.** Backups, search indexes, caches, log lines containing customer content and analytics exports all carry the same constraint, and a compliant database with logs shipped to a central region elsewhere is not compliant. Making region confinement a platform-level property — enforced in the deployment and data pipelines rather than remembered by each team — is the only approach that holds as the number of services grows.

**Control plane and failover**

```text
THE CONTROL PLANE HOLDS
  tenant -> home region mapping
  tenant -> paired region
  region health and status
  failover state
  -> small, critical, globally consistent
  -> backed by consensus, replicated across regions

WHY IT MUST BE CONSISTENT
  if two regions disagree about where a tenant lives
  -> writes go to both
  -> divergence that must be merged by hand
  -> so this small dataset gets strong consistency
     while the large datasets do not

FAILOVER SEQUENCE
  1  detect: region unreachable from multiple
     vantage points
  2  confirm: a human verifies it is not a partition
  3  promote: paired region's replicas become
     authoritative
       -> record the promotion in the control plane
          (consensus, so it cannot be ambiguous)
  4  fence: the old region, if it returns, must not
     accept writes
  5  redirect: routing updated; edge caches expire
  6  accept: whatever was not replicated is lost;
     reconcile what can be reconciled

FAILBACK
  the returning region must catch up before taking
  traffic
  -> it may have writes the pair never received
  -> those are reconciled manually; there is no
     automatic answer
  -> failback is harder than failover and is
     practised less
```

**Fencing the failed region is the step that prevents the worst outcome.** A region that returns after promotion still believes it holds those tenants, and without an explicit mechanism refusing its writes, it will accept them — producing two divergent histories for the same tenants. Recording the promotion in a consensus-backed control plane and having every component check it is what makes the returning region's writes rejected rather than accepted.

**Failback is harder than failover and receives far less attention.** The recovered region may hold writes that never replicated before it failed, so restoring it as authoritative means reconciling two partially divergent datasets with no automatic rule for which wins. Practising failover without practising failback means the second half of the recovery is improvised during an incident, which is when it is least likely to go well.

**A small globally consistent control plane and large regionally consistent data planes is the arrangement that works.** The control plane is tiny — tenant placement, region status — so consensus across regions is affordable, and it is exactly the data where ambiguity is catastrophic. The tenant data is large and does not need global consistency because it is accessed regionally, so it can be replicated asynchronously. Inverting this, by making tenant data globally consistent, pays inter-region latency on every write for a property almost nobody needs.

**Tenant migration between regions must exist and is rarely built early.** Customers relocate, acquire subsidiaries, change compliance requirements, or are simply placed wrongly at signup. Moving a tenant means copying data, switching the home region atomically and fencing the old one — essentially a planned failover for one tenant — and building it once as a supported operation is far better than performing it manually, differently, every time it is needed.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Region loss | Its tenants unavailable | Operator-confirmed failover to the paired region |
| Partition between regions | Each may think the other failed | Human confirmation; consensus-backed promotion |
| Old region returns after promotion | Two authoritative copies | Fencing enforced via the control plane |
| Paired region lacks capacity | Failover succeeds then collapses | Provision headroom or accept stated degradation |
| Routing sends users to the wrong region | Cross-region calls on every query | Route by tenant, not geography |
| Data replicated outside permitted regions | Compliance breach | Platform-level residency enforcement including logs and backups |
| Async lag larger than assumed | More data lost than the stated objective | Monitor lag continuously; alert on breaches |
| Failback improvised | Prolonged incident, possible data loss | Practise failback, not only failover |

> **Two regions both accepting writes for the same tenants is the failure with no clean recovery**  
> Every other failure here degrades service temporarily; split brain produces two divergent versions of a customer's data that must be merged by hand, with real loss and a conversation the customer will remember. This is why promotion requires human confirmation despite the delay, why the control plane uses consensus despite the cost, and why fencing is enforced rather than assumed — the asymmetry between a few extra minutes of downtime and unrecoverable divergence is large enough to decide the design.

Stated as a principle: in multi-region systems, prefer being unavailable to being inconsistent, unless someone has explicitly decided otherwise for a specific dataset.

**The staff-level view**

The strong answer partitions by tenant rather than replicating globally, and is honest about failover numbers.

- **Give each tenant a home region** and explain that it delivers latency, residency and consistency together.
- **Route by tenant identity, not geography**, since a user's location does not determine where their data lives.
- **State the recovery point objective explicitly** as the replication lag, and treat it as a business decision.
- **Give realistic failover timings** — minutes, dominated by detection and DNS — rather than implying seconds.
- **Require human confirmation before promotion** and automated execution afterwards, to avoid split brain on a partition.
- **Make residency a platform property** covering backups, logs and exports, not a per-service responsibility.

> **The question that raises the level**  
> Does the paired region have capacity to run both workloads, and when was that last tested? Failover plans are usually validated for correctness and rarely for capacity, so the promotion works and the surviving region then fails under double load. Knowing the utilisation headroom, and having exercised a real region evacuation, is what turns a documented recovery plan into one that will actually work.

---

### Fraud Decision Pipeline

*Decide within a payment's latency budget whether to approve, review or decline, using features computed from behaviour up to that moment.*

**Flow:** `Transaction request` → `Feature retrieval` → `Rules and model` → `Decision and reasons` → `Async enrichment` → `Feedback and retraining`

**The brief**

Every payment must be assessed before it is authorised. The decision has perhaps a hundred milliseconds, must use what is known about the card, the account, the device and recent behaviour, and must produce an outcome that can be explained to a regulator and to the customer.

Both errors cost real money in opposite directions. Approving fraud produces chargebacks and losses; declining a legitimate customer produces lost revenue, support contacts and sometimes a customer who never returns — and that second cost is usually larger and much harder to see.

- **Latency**: a decision within 100 milliseconds, always, including under load.
- **Accuracy**: catch material fraud while keeping false declines low.
- **Explainability**: every decision records the reasons it was made.
- **Adaptability**: rules and models can be changed quickly as fraud patterns shift.
- **Scale**: 10,000 decisions per second at peak.

> **The decision budget is spent on retrieving features, not on the model**  
> Scoring a model over a prepared feature vector takes single-digit milliseconds; assembling that vector — the card's spend in the last hour, the account's device history, the merchant's recent decline rate — is dozens of lookups against different stores. Every design choice that matters here is about having those features precomputed and locally available before the request arrives.

**Scale**

**Latency budget and feature arithmetic**

```text
BUDGET  (100 ms total)
  feature retrieval     40 ms
  rule evaluation        5 ms
  model scoring          10 ms
  decision assembly      5 ms
  logging (async)        0 ms
  network and overhead   20 ms
  -> ~80 ms, leaving headroom for the tail

FEATURES
  typically 100-300 features per decision
  sources:
    precomputed aggregates (card velocity, account
    history)          -> feature store, ~5 ms
    real-time counters (attempts in the last minute)
                      -> in-memory, ~1 ms
    entity attributes (account age, risk flags)
                      -> cache, ~2 ms
    third-party signals (device reputation)
                      -> 50-500 ms, TOO SLOW
  -> third-party calls cannot be inside the budget
  -> they are fetched asynchronously and used on the
     NEXT decision

FEATURE FRESHNESS
  velocity counters must include the transaction 3
  seconds ago
  -> streaming aggregation into the feature store
  -> lag budget under 1 s, or the most important
     signal is missing

THROUGHPUT
  10,000 decisions/s x 200 features
  = 2,000,000 feature reads/s
  -> batched multi-get; never one call per feature

DECISION MIX  (typical)
  approve      ~97%
  review        ~2%
  decline       ~1%
  -> the review queue's capacity bounds how many
     can be sent there
```

| Metric | Value | Note |
|---|---|---|
| Budget | 100 ms | **features dominate** |
| Features | 100-300 | batched retrieval |
| Freshness | under 1 s | velocity signals |
| Third party | too slow | async, next time |

Excluding third-party calls from the synchronous path is a constraint that shapes the whole design. Device reputation services, identity checks and consortium data are valuable and far too slow to consult inside a hundred milliseconds, so they are fetched asynchronously and their results attached to the entity for use in subsequent decisions — which means the first transaction from a new device is decided with less information than the second, and the rules must account for that.

> **Velocity features are the most valuable and the most sensitive to pipeline lag**  
> Rapid repeated attempts are the clearest signal of automated fraud, and detecting them requires counters that include transactions from seconds ago. If the streaming aggregation lags by thirty seconds, an attacker can complete an entire run before the counters reflect any of it — so the freshness of a handful of features matters more than the accuracy of the model consuming them.

**Key decisions**

**The decisions that shape this system**

| Decision | Options | Choose | Because |
|---|---|---|---|
| Feature computation | At decision time, precomputed | Precomputed in a feature store | Computing aggregates per request cannot meet the budget |
| Decision logic | Rules only, model only, both | Rules plus model | Rules give control and explainability; models generalise |
| Outcomes | Approve or decline, three-way | Approve, review, decline | A middle option converts hard calls into human review |
| Third-party signals | Synchronous, asynchronous | Asynchronous, used next time | They exceed the entire latency budget |
| Failure behaviour | Fail open, fail closed | Fail open with degraded rules | Declining everyone during an outage costs more than the fraud |
| Model updates | Deploy with code, independent | Independent, with shadow evaluation | Fraud patterns change faster than release cycles |

1. **Precompute aggregate features continuously from the transaction stream** — Retrieval must be a lookup, never a computation.
2. **Retrieve all features in batched calls** — Two hundred sequential lookups cannot fit in the budget.
3. **Evaluate deterministic rules first, then the model** — Rules catch known patterns cheaply and are trivially explainable.
4. **Produce a decision with reason codes** — Explainability is a requirement, not a debugging aid.
5. **Send uncertain cases to human review** — Binary decisions force errors that a review queue absorbs.
6. **Log every decision with its full feature vector** — Retraining and dispute resolution both depend on it.

Logging the complete feature vector alongside every decision is what makes the system improvable. Without it, a model retrained on later data cannot reconstruct what was known at decision time, so it learns from information the original decision did not have — producing offline accuracy that does not survive deployment. Storing the vector as it was makes evaluation honest.

> **Ask before building it**  
> What is the cost of a false decline relative to a fraud loss? Most organisations can state the fraud number precisely and have never measured the other, yet false declines are frequently the larger cost — lost revenue, support handling, and customers who do not come back. Without that ratio, the decision threshold is being set by instinct.

**Deep dive**

Three areas define this system: the feature pipeline, the decision layer, and the feedback loop that keeps it current.

**The feature pipeline**

```text
TRANSACTION STREAM
  every transaction, decision and outcome
  -> partitioned log

STREAMING AGGREGATION
  maintains, per entity, windowed counters:
    card:  attempts and amount in 1 m / 1 h / 24 h
    account: distinct devices in 7 d, average amount
    merchant: decline rate in 1 h
    device: accounts seen in 30 d
  -> written to the feature store as they update
  -> lag budget under 1 s

FEATURE STORE
  key: entity id
  value: the current feature values
  -> read path: batched multi-get, ~5 ms
  -> write path: streaming updates
  -> the same definitions serve training and serving

THE TRAINING / SERVING CONSISTENCY PROBLEM
  if training features are computed by a batch job
  and serving features by a stream, they will differ
  -> the model learns one distribution and sees
     another
  -> accuracy degrades in ways offline evaluation
     cannot detect
  -> using one definition for both is the single
     most valuable property of a feature store

COLD ENTITIES
  a first-time card has no history
  -> features are absent, not zero
  -> the model must handle missing values explicitly,
     and rules must not treat absence as safety
```

**Training and serving computing features differently is the most common cause of models that perform well in evaluation and poorly in production.** A batch job calculating a card's hourly velocity over a clean historical window produces subtly different values from a streaming aggregation under real-world lateness and lag, and the model learns the former while receiving the latter. Sharing one definition across both paths eliminates a whole class of unexplainable degradation.

**Decision, explanation and feedback**

```text
DECISION FLOW
  1  hard rules (deterministic, fast)
       card on a blocklist        -> decline
       amount over the account
       limit                      -> decline
       known-good recurring
       merchant                   -> approve
       -> these are policy, and must not be
          overridden by a score

  2  model score
       risk in [0, 1]

  3  thresholds
       score < 0.3            -> approve
       0.3 <= score < 0.8     -> review
       score >= 0.8           -> decline
       -> thresholds are business parameters, tuned
          against the false-decline cost

  4  REASONS
       record which rules fired and the top feature
       contributions
       -> required for disputes and regulation
       -> and indispensable when investigating a
          spike in declines

FEEDBACK
  chargebacks arrive 30-90 days later
  -> labels are delayed, which is intrinsic
  manual review outcomes arrive in hours
  -> a faster, smaller-volume label source
  -> both feed retraining

SHADOW EVALUATION
  run a candidate model alongside the live one
  record what it WOULD have decided
  -> compare before switching
  -> the only safe way to change a model that
     declines customers
```

**The review queue is what makes a three-way decision possible, and its capacity is a hard constraint.** Sending more cases to review than analysts can handle means they age past usefulness, so the review threshold is bounded by staffing rather than by where the model is uncertain. Treating queue capacity as an input to threshold tuning, rather than discovering the backlog afterwards, is what keeps the middle option functional.

**Label delay is intrinsic and shapes how quickly the system can adapt.** Chargebacks confirm fraud one to three months after the transaction, so a model retrained today learns from a fraud landscape that has already moved. Manual review outcomes and customer contacts provide faster if noisier signals, and rules — which can be changed within minutes — are what cover the gap when a new pattern appears, which is why rules remain essential alongside models rather than being a legacy layer.

**Failing open during an outage is almost always correct, and should be explicit.** If the feature store is unreachable, declining every transaction stops all revenue while preventing a small amount of fraud; approving with degraded rules accepts some additional loss and keeps the business running. The ratio between those two costs is usually overwhelming, but the behaviour must be a stated policy with a defined degraded ruleset rather than whatever the code happens to do on a timeout.

**Explainability is a first-class output, not instrumentation.** Regulators, disputes and customer support all require knowing why a specific transaction was declined, and reconstructing it later from model internals is not feasible. Emitting reason codes with every decision — which rules fired, which features contributed most — makes the system accountable and, incidentally, makes investigating a sudden change in decline rates a query rather than an inquiry.

**How it fails**

**What breaks, and what it does**

| Failure | Effect | Response |
|---|---|---|
| Feature store unavailable | No features; decisions blind | Fail open with a conservative degraded ruleset |
| Feature pipeline lagging | Velocity signals stale; fraud runs undetected | Alert on aggregation lag; treat as an incident |
| Training and serving features differ | Model underperforms unexplainably | Single feature definition for both paths |
| Model degrades as patterns shift | Fraud rate rises quietly | Monitor decision distribution; retrain on a schedule |
| False declines rise after a change | Revenue and customers lost | Shadow evaluation before switching; monitor decline rate by segment |
| Review queue overflows | Cases age past usefulness | Tune thresholds against queue capacity |
| Third-party signal times out | Latency budget blown | Never synchronous; asynchronous enrichment only |
| No reasons recorded | Disputes unanswerable | Emit reason codes with every decision |

> **False declines are the larger cost and the one nobody measures**  
> Fraud losses are counted precisely because they appear as chargebacks with amounts attached, while a legitimate customer who was declined simply leaves — taking their future revenue with them and generating no record beyond a support contact that may never happen. Systems tuned only against measured fraud drift steadily towards over-declining, because that direction has no visible cost, and correcting it requires deliberately measuring the decline rate by customer segment and treating a rise in it as an incident.

The practical defence is to monitor approval rates for known-good customers as closely as fraud rates, so that both errors have a number attached.

**The staff-level view**

The strong answer builds the latency budget around feature retrieval and treats false declines as a first-class cost.

- **Break down the latency budget** and show that feature retrieval, not model scoring, consumes most of it.
- **Precompute features from a stream** and state the freshness requirement for velocity signals.
- **Exclude third-party calls from the synchronous path**, using them asynchronously for subsequent decisions.
- **Combine rules with a model** — rules for policy and fast response to new patterns, the model for generalisation.
- **Add a review outcome** and tie the threshold to actual queue capacity.
- **Emit reason codes** and log the full feature vector, for disputes, regulation and honest retraining.

> **The question that raises the level**  
> What is the cost of a false decline compared with the cost of accepted fraud? Almost every organisation can quote the fraud number and almost none has measured the other, yet the ratio determines every threshold in the system. Asking it changes the conversation from catching more fraud to optimising a trade with two real costs — which is what the system is actually doing whether or not anyone has said so.

---
