# Workouts — Think in systems (50 designs)

[← System Design index](README.md)

> 50 full system designs, each worked through in 9 stages: requirements, scale, API, data model, architecture, deep dives, failures, trade-offs and interview questions, plus the staff-level view.

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

## Foundational

### URL shortener

*Create durable short links and serve redirects with a small latency budget.*

**1. Requirements**

- Custom aliases and expiry
- Redirect without synchronous analytics
- Prevent alias collisions

**2. Scale:** 10M daily users; 20 redirects each; 100:1 read/write ratio; redirect p99 target 80 ms. — practise the numbers: [Capacity Gym](07-capacity.md#url-shortener-workload)

**3. API**

- `POST /links {url,alias?,expiresAt?}`
- `GET /{code} -> 302`
- `DELETE /links/{code}`

**4. Data model**

- `links(code PK, target, owner, expires_at, version)`
- `click_events(event_id PK, code, timestamp)` 

Choose the store: [Database Gym](08-database.md#storage-for-url-shortener)

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

**2. Scale:** 100M decisions/day; 1M tenants; 2 ms regional decision budget. — practise the numbers: [Capacity Gym](07-capacity.md#distributed-rate-limiter-workload)

**3. API**

- `POST /decisions {tenant,cost,requestId}`
- `PUT /policies/{tenant}`

**4. Data model**

- `bucket(tenant, tokens, last_refill, policy_version)`
- `policy(tenant PK, rate, burst)` 

Choose the store: [Database Gym](08-database.md#storage-for-distributed-rate-limiter)

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

**2. Scale:** 2B GETs/day; 200GB hot data; 1KB median values. — practise the numbers: [Capacity Gym](07-capacity.md#distributed-cache-workload)

**3. API**

- `GET /cache/{key}`
- `PUT /cache/{key} {value,ttl,version}`
- `DELETE /cache/{key}`

**4. Data model**

- `entry(key, value, version, expires_at)`
- `ring(epoch, virtual_nodes)` 

Choose the store: [Database Gym](08-database.md#storage-for-distributed-cache)

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

**2. Scale:** 500M IDs/day; bursts of 200K IDs/s; IDs fit in signed 64-bit storage. — practise the numbers: [Capacity Gym](07-capacity.md#unique-id-service-workload)

**3. API**

- `POST /ids:allocate {count}`
- `GET /workers/{id}/lease`

**4. Data model**

- `worker_lease(worker_id PK, epoch, expires_at)`
- `local_state(last_timestamp, sequence)` 

Choose the store: [Database Gym](08-database.md#storage-for-unique-id-service)

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

**2. Scale:** 50M jobs/day; 30-day scheduling horizon; execution delay p99 under 10 seconds. — practise the numbers: [Capacity Gym](07-capacity.md#durable-job-scheduler-workload)

**3. API**

- `POST /jobs {runAt,payload,idempotencyKey}`
- `POST /jobs/{id}:cancel`
- `GET /jobs/{id}`

**4. Data model**

- `jobs(job_id PK, run_at, state, attempt, lease_epoch)`
- `attempts(job_id, attempt, outcome)` 

Choose the store: [Database Gym](08-database.md#storage-for-durable-job-scheduler)

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

**2. Scale:** 50M daily users; 20 feed reads/day; follower counts have a heavy tail. — practise the numbers: [Capacity Gym](07-capacity.md#social-news-feed-workload)

**3. API**

- `POST /posts`
- `GET /feed?cursor=`
- `PUT /follows/{author}`

**4. Data model**

- `posts(post_id PK, author, created_at, visibility_version)`
- `inbox(user_id, rank_key, post_id)`
- `follows(user_id, author_id)` 

Choose the store: [Database Gym](08-database.md#storage-for-social-news-feed)

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

**2. Scale:** 8M uploads/day; 4MB original photos; 10 thumbnail views per upload. — practise the numbers: [Capacity Gym](07-capacity.md#photo-sharing-workload)

**3. API**

- `POST /uploads`
- `POST /uploads/{id}:complete`
- `GET /users/{id}/photos?cursor=`

**4. Data model**

- `photos(photo_id PK, owner, object_key, state, visibility)`
- `variants(photo_id, transform_version, object_key)` 

Choose the store: [Database Gym](08-database.md#storage-for-photo-sharing)

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

**2. Scale:** 200M accounts; 20B edges; most users have fewer than 1,000 edges. — practise the numbers: [Capacity Gym](07-capacity.md#follow-graph-workload)

**3. API**

- `PUT /users/{id}/following/{target}`
- `DELETE /users/{id}/following/{target}`
- `GET /users/{id}/followers`

**4. Data model**

- `out_edges(source, target, version)`
- `in_edges(target, bucket, source)`
- `blocks(blocker, blocked)` 

Choose the store: [Database Gym](08-database.md#storage-for-follow-graph)

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

**2. Scale:** 30M comments/day; one live event may receive 20K replies/s. — practise the numbers: [Capacity Gym](07-capacity.md#threaded-comments-workload)

**3. API**

- `POST /posts/{id}/comments`
- `PATCH /comments/{id}`
- `GET /comments?parent=&cursor=`

**4. Data model**

- `comments(comment_id PK, root_id, parent_id, body, version, state)`
- `votes(user_id, comment_id, value)` 

Choose the store: [Database Gym](08-database.md#storage-for-threaded-comments)

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

**2. Scale:** 15M stories/day; 150M daily story views; media expiry is strict at serving time. — practise the numbers: [Capacity Gym](07-capacity.md#ephemeral-stories-workload)

**3. API**

- `POST /stories`
- `GET /stories?owner=`
- `PUT /stories/{id}/views/{viewer}`

**4. Data model**

- `stories(story_id PK, owner, expires_at, audience_version)`
- `views(story_id, viewer_id, first_seen_at)` 

Choose the store: [Database Gym](08-database.md#storage-for-ephemeral-stories)

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

**2. Scale:** 50M daily users; 40 messages/user/day; groups up to 1,000 members. — practise the numbers: [Capacity Gym](07-capacity.md#direct-and-group-chat-workload)

**3. API**

- `POST /conversations/{id}/messages {clientMessageId,body}`
- `GET /sync?cursor=`
- `PUT /receipts`

**4. Data model**

- `messages(conversation_id, sequence, message_id, body)`
- `memberships(conversation_id,user_id,version)`
- `device_cursor(device_id,conversation_id,sequence)` 

Choose the store: [Database Gym](08-database.md#storage-for-direct-and-group-chat)

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

**2. Scale:** 500M notification intents/day; provider quotas differ by region and channel. — practise the numbers: [Capacity Gym](07-capacity.md#notification-platform-workload)

**3. API**

- `POST /notifications {eventId,userId,templateId}`
- `PUT /preferences`
- `GET /notifications/{id}`

**4. Data model**

- `intents(event_id,user_id,template_version)`
- `deliveries(intent_id,channel,state,provider_key)`
- `preferences(user_id,version)` 

Choose the store: [Database Gym](08-database.md#storage-for-notification-platform)

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

**2. Scale:** 100M inbound messages/day; 80KB average body plus attachments; 1M active mailboxes. — practise the numbers: [Capacity Gym](07-capacity.md#email-service-workload)

**3. API**

- `POST /messages:send`
- `GET /mailboxes/{id}/messages?cursor=`
- `PATCH /messages/{id}/labels`

**4. Data model**

- `messages(message_id PK, envelope, blob_key)`
- `mailbox_entries(mailbox_id, uid, message_id, flags)`
- `delivery_attempts(message_id,destination)` 

Choose the store: [Database Gym](08-database.md#storage-for-email-service)

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

**2. Scale:** 200M deliveries/day; 100K endpoints; endpoints can be slow or unavailable. — practise the numbers: [Capacity Gym](07-capacity.md#webhook-delivery-workload)

**3. API**

- `POST /subscriptions`
- `GET /deliveries/{id}`
- `POST /events/{id}:replay`

**4. Data model**

- `events(event_id PK, tenant_id, payload_version)`
- `subscriptions(id,url,secret_version)`
- `attempts(event_id,subscription_id,number,status)` 

Choose the store: [Database Gym](08-database.md#storage-for-webhook-delivery)

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

**2. Scale:** 1B events/day; 1KB mean event; 20 consumer groups. — practise the numbers: [Capacity Gym](07-capacity.md#publish-subscribe-event-bus-workload)

**3. API**

- `POST /topics/{id}/events`
- `GET /topics/{id}/partitions/{p}?offset=`
- `PUT /groups/{id}/checkpoints`

**4. Data model**

- `log(topic,partition,offset,key,payload)`
- `group_offsets(group,topic,partition,offset)`
- `schema(subject,version)` 

Choose the store: [Database Gym](08-database.md#storage-for-publish-subscribe-event-bus)

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

**2. Scale:** 20M daily viewers watch 30 minutes each at an assumed 3 Mb/s average: 13.5 PB/day of delivered video; 100K uploads/day averaging 600MB = 60TB/day of originals before renditions. — practise the numbers: [Capacity Gym](07-capacity.md#youtube-style-video-platform-workload)

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

Choose the store: [Database Gym](08-database.md#storage-for-youtube-style-video-platform)

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

**2. Scale:** 10K concurrent broadcasters; 5M peak viewers; target glass-to-glass latency 5 seconds. — practise the numbers: [Capacity Gym](07-capacity.md#live-streaming-platform-workload)

**3. API**

- `POST /streams`
- `POST /streams/{id}:start`
- `GET /streams/{id}/playback`
- `GET /streams/{id}/health`

**4. Data model**

- `streams(stream_id PK, ingest_region, state, epoch)`
- `segments(stream_id, epoch, sequence, duration, key)`
- `manifests(stream_id,version)` 

Choose the store: [Database Gym](08-database.md#storage-for-live-streaming-platform)

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

**2. Scale:** 20M daily listeners; 40 tracks/day; 4MB average delivered audio per track. — practise the numbers: [Capacity Gym](07-capacity.md#music-streaming-workload)

**3. API**

- `GET /tracks/{id}/playback`
- `PATCH /playlists/{id} {baseVersion,operations}`
- `PUT /sessions/{id}/position`

**4. Data model**

- `tracks(track_id PK, asset_version, rights_version)`
- `playlist_entries(playlist_id,position_id,track_id)`
- `sessions(user_id,device_id,position,version)` 

Choose the store: [Database Gym](08-database.md#storage-for-music-streaming)

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

**2. Scale:** 500K concurrent users; average room size 6; target interactive latency below 300 ms. — practise the numbers: [Capacity Gym](07-capacity.md#video-conferencing-workload)

**3. API**

- `POST /rooms`
- `POST /rooms/{id}/tokens`
- `WebRTC signaling offer/answer`
- `POST /rooms/{id}:record`

**4. Data model**

- `rooms(room_id PK, region, owner_epoch)`
- `participants(room_id,user_id,session_id,role)`
- `recordings(recording_id,state,object_key)` 

Choose the store: [Database Gym](08-database.md#storage-for-video-conferencing)

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

**2. Scale:** 2M feeds; 200K updated episodes/day; 10M listeners. — practise the numbers: [Capacity Gym](07-capacity.md#podcast-platform-workload)

**3. API**

- `POST /feeds`
- `GET /shows/{id}/episodes`
- `PUT /progress/{episodeId} {position,sessionVersion}`

**4. Data model**

- `feeds(feed_id PK, url, etag, next_poll_at)`
- `episodes(show_id,guid,asset_version)`
- `progress(user_id,episode_id,position,session_version)` 

Choose the store: [Database Gym](08-database.md#storage-for-podcast-platform)

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

**2. Scale:** 10M active users; 100M file mutations/day; typical file 2MB. — practise the numbers: [Capacity Gym](07-capacity.md#cloud-file-synchronization-workload)

**3. API**

- `POST /files/{id}/uploads`
- `PATCH /files/{id} {baseVersion,manifest}`
- `GET /changes?cursor=`

**4. Data model**

- `files(file_id PK,parent_id,name,current_version)`
- `versions(file_id,version,chunk_manifest)`
- `changes(account_id,sequence,operation)` 

Choose the store: [Database Gym](08-database.md#storage-for-cloud-file-synchronization)

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

**2. Scale:** 10PB logical data; 50M objects; 20GB/s peak read traffic. — practise the numbers: [Capacity Gym](07-capacity.md#distributed-object-storage-workload)

**3. API**

- `PUT /buckets/{b}/objects/{key}`
- `GET /objects/{key}?version=`
- `POST /multipart:complete`

**4. Data model**

- `object_index(bucket,key,version,manifest)`
- `fragments(object_id,stripe,location,checksum)`
- `repairs(fragment_id,state)` 

Choose the store: [Database Gym](08-database.md#storage-for-distributed-object-storage)

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

**2. Scale:** 500TB source data; 2TB changes/day; target RPO 5 minutes and RTO 4 hours. — practise the numbers: [Capacity Gym](07-capacity.md#backup-and-point-in-time-restore-workload)

**3. API**

- `POST /backups`
- `POST /restores {snapshotId,targetTime}`
- `GET /restores/{id}`

**4. Data model**

- `snapshots(snapshot_id PK, base_checkpoint, manifest, state)`
- `log_segments(start,end,checksum)`
- `restore_jobs(id,target,verified_at)` 

Choose the store: [Database Gym](08-database.md#storage-for-backup-and-point-in-time-restore)

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

**2. Scale:** 3PB logical content; expected 40% duplicate bytes; 100M references/day. — practise the numbers: [Capacity Gym](07-capacity.md#content-addressed-blob-store-workload)

**3. API**

- `POST /blobs {digest,size}`
- `PUT /blobs/{digest}/content`
- `POST /references`
- `DELETE /references/{id}`

**4. Data model**

- `blobs(digest PK,size,verified,state)`
- `references(tenant,reference_id,digest)`
- `gc_candidates(digest,generation)` 

Choose the store: [Database Gym](08-database.md#storage-for-content-addressed-blob-store)

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

**2. Scale:** 1B files; 5PB data; 100K concurrent clients; most operations are metadata reads. — practise the numbers: [Capacity Gym](07-capacity.md#distributed-filesystem-workload)

**3. API**

- `OPEN /path`
- `READ /file/{id}?offset=&length=`
- `RENAME /path {destination}`
- `APPEND /file/{id}`

**4. Data model**

- `inodes(inode_id PK,parent,name,version)`
- `chunk_map(inode_id,index,chunk_id)`
- `leases(inode_id,client,epoch)` 

Choose the store: [Database Gym](08-database.md#storage-for-distributed-filesystem)

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

**2. Scale:** 5M orders/day; flash-sale peak 20K checkout attempts/s. — practise the numbers: [Capacity Gym](07-capacity.md#commerce-checkout-workload)

**3. API**

- `POST /checkouts {cartVersion,idempotencyKey}`
- `GET /orders/{id}`
- `POST /orders/{id}:cancel`

**4. Data model**

- `orders(order_id PK,state,amount,currency)`
- `reservations(order_id,sku,quantity,expires_at)`
- `payment_attempts(order_id,key,provider_state)` 

Choose the store: [Database Gym](08-database.md#storage-for-commerce-checkout)

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

**2. Scale:** 20M SKUs; 50M stock mutations/day; a hot SKU may receive 50K attempts/s. — practise the numbers: [Capacity Gym](07-capacity.md#inventory-reservation-workload)

**3. API**

- `POST /reservations {sku,quantity,orderId}`
- `POST /reservations/{id}:commit`
- `POST /reservations/{id}:release`

**4. Data model**

- `stock(warehouse,sku,on_hand,reserved,version)`
- `reservations(id,order_id,sku,qty,state,expires_at)`
- `stock_ledger(event_id,delta)` 

Choose the store: [Database Gym](08-database.md#storage-for-inventory-reservation)

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

**2. Scale:** 10M ledger transactions/day; 7-year business retention assumption; exact integer minor units. — practise the numbers: [Capacity Gym](07-capacity.md#payment-ledger-workload)

**3. API**

- `POST /ledger/transactions {key,entries}`
- `GET /accounts/{id}/balance`
- `POST /ledger/transactions/{id}:reverse`

**4. Data model**

- `transactions(tx_id PK,idempotency_key UNIQUE,state)`
- `entries(tx_id,account_id,currency,amount_minor)`
- `account_balances(account_id,currency,version)` 

Choose the store: [Database Gym](08-database.md#storage-for-payment-ledger)

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

**2. Scale:** 100K seats for one event; 3M buyers arrive in a minute. — practise the numbers: [Capacity Gym](07-capacity.md#ticket-booking-workload)

**3. API**

- `POST /events/{id}/holds {seats,key}`
- `POST /holds/{id}:confirm`
- `GET /events/{id}/seats`

**4. Data model**

- `seats(event_id,seat_id,state,hold_id,version)`
- `holds(hold_id,user_id,expires_at,state)`
- `bookings(booking_id,hold_id,payment_id)` 

Choose the store: [Database Gym](08-database.md#storage-for-ticket-booking)

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

**2. Scale:** 2M orders/day; average 3 sellers/order; fulfillment can take weeks. — practise the numbers: [Capacity Gym](07-capacity.md#marketplace-orders-workload)

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

Choose the store: [Database Gym](08-database.md#storage-for-marketplace-orders)

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

**2. Scale:** 2M active drivers; location updates every 5 seconds; dispatch target under 2 seconds. — practise the numbers: [Capacity Gym](07-capacity.md#ride-dispatch-workload)

**3. API**

- `PUT /drivers/{id}/location {sequence,lat,lon}`
- `POST /rides`
- `POST /offers/{id}:accept`

**4. Data model**

- `driver_state(driver_id PK,availability,trip_id,version)`
- `locations(cell,driver_id,sequence,observed_at)`
- `trips(trip_id,state,rider,driver)` 

Choose the store: [Database Gym](08-database.md#storage-for-ride-dispatch)

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

**2. Scale:** 1M concurrent editors; 5 operations/s per active editor; documents up to 10MB. — practise the numbers: [Capacity Gym](07-capacity.md#collaborative-document-editor-workload)

**3. API**

- `POST /documents`
- `WebSocket /documents/{id}/operations`
- `GET /documents/{id}/snapshot?version=`

**4. Data model**

- `documents(id,permission_version,snapshot_version)`
- `operations(document_id,op_id,causal_metadata,payload)`
- `snapshots(document_id,version,object_key)` 

Choose the store: [Database Gym](08-database.md#storage-for-collaborative-document-editor)

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

**2. Scale:** 30M connected devices; heartbeat every 30 seconds; typing expiry after 5 seconds. — practise the numbers: [Capacity Gym](07-capacity.md#presence-and-typing-indicators-workload)

**3. API**

- `PUT /presence/{deviceId} {sequence,state}`
- `POST /typing {conversationId,expiresAt}`
- `SUBSCRIBE /presence`

**4. Data model**

- `device_presence(user_id,device_id,session_epoch,last_seen)`
- `subscribers(user_id,watcher_id)`
- `typing(conversation_id,user_id,expires_at)` 

Choose the store: [Database Gym](08-database.md#storage-for-presence-and-typing-indicators)

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

**2. Scale:** 50M players; 200M score events/day; top-100 reads dominate. — practise the numbers: [Capacity Gym](07-capacity.md#game-leaderboard-workload)

**3. API**

- `POST /scores {eventId,playerId,delta,season}`
- `GET /leaderboards/{id}?top=100`
- `GET /players/{id}/rank`

**4. Data model**

- `score_events(event_id PK,player,season,delta)`
- `scores(season,player,total,version)`
- `season(id,state,final_checkpoint)` 

Choose the store: [Database Gym](08-database.md#storage-for-game-leaderboard)

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

**2. Scale:** 5M devices; update every 10 seconds while moving; 90-day history. — practise the numbers: [Capacity Gym](07-capacity.md#fleet-tracking-workload)

**3. API**

- `POST /telemetry {deviceId,sequence,observedAt,lat,lon}`
- `GET /fleets/{id}/positions`
- `GET /devices/{id}/history`

**4. Data model**

- `telemetry(device_id,time_bucket,sequence,observed_at,point)`
- `latest(device_id,sequence,point)`
- `geofence_state(device_id,fence_id,state,version)` 

Choose the store: [Database Gym](08-database.md#storage-for-fleet-tracking)

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

**2. Scale:** 10B pages; 100M queries/day; p95 search target 300 ms. — practise the numbers: [Capacity Gym](07-capacity.md#web-search-engine-workload)

**3. API**

- `GET /search?q=&cursor=`
- `POST /crawl-seeds`
- `GET /crawl-status/{url}`

**4. Data model**

- `urls(url_id PK,canonical_url,next_fetch_at,content_hash)`
- `documents(doc_id,version,terms,signals)`
- `postings(term,shard,doc_id,positions)` 

Choose the store: [Database Gym](08-database.md#storage-for-web-search-engine)

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

**2. Scale:** 500M keystroke queries/day; p99 server budget 30 ms; 50 locales. — practise the numbers: [Capacity Gym](07-capacity.md#search-autocomplete-workload)

**3. API**

- `GET /suggest?q=&locale=&context=`
- `POST /suggestion-events`

**4. Data model**

- `suggestions(locale,prefix,rank,candidate_id)`
- `candidates(id,text,policy_version)`
- `query_counts(term,time_bucket,count)` 

Choose the store: [Database Gym](08-database.md#storage-for-search-autocomplete)

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

**2. Scale:** 50M products; 80M queries/day; 10M product updates/day. — practise the numbers: [Capacity Gym](07-capacity.md#product-search-workload)

**3. API**

- `GET /products/search?q=&filters=&sort=`
- `GET /products/{id}`

**4. Data model**

- `products(product_id PK,price,stock_version,catalog_version)`
- `search_docs(product_id,index_version,terms,facets)`
- `price_history(product_id,version)` 

Choose the store: [Database Gym](08-database.md#storage-for-product-search)

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

**2. Scale:** 100TB raw logs/day; 30-day hot retention; 10K tenants. — practise the numbers: [Capacity Gym](07-capacity.md#log-ingestion-and-search-workload)

**3. API**

- `POST /logs:batch`
- `POST /queries {timeRange,filter}`
- `GET /queries/{id}`

**4. Data model**

- `log_events(tenant,event_id,event_time,service,body)`
- `partitions(tenant,time_bucket,location)`
- `query_jobs(id,budget,state)` 

Choose the store: [Database Gym](08-database.md#storage-for-log-ingestion-and-search)

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

**2. Scale:** 500M chunks; 768-dimensional embeddings; 50M searches/day. — practise the numbers: [Capacity Gym](07-capacity.md#vector-retrieval-service-workload)

**3. API**

- `POST /documents`
- `POST /search {query,filters,topK}`
- `DELETE /documents/{id}`

**4. Data model**

- `chunks(chunk_id PK,document_id,text_version,embedding_version,acl_version)`
- `vectors(chunk_id,embedding)`
- `index_generations(id,model,checkpoint)` 

Choose the store: [Database Gym](08-database.md#storage-for-vector-retrieval-service)

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

**2. Scale:** 10M requests/s global peak; three active regions; long-lived connections included. — practise the numbers: [Capacity Gym](07-capacity.md#global-load-balancer-workload)

**3. API**

- `PUT /routes/{service}`
- `GET /backends/health`
- `POST /backends/{id}:drain`

**4. Data model**

- `routes(service,version,policy)`
- `backends(id,region,capacity,health_epoch)`
- `health_samples(backend,time,result)` 

Choose the store: [Database Gym](08-database.md#storage-for-global-load-balancer)

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

**2. Scale:** 5M queries/s; 10M zones; records range from 30-second to 1-day TTLs. — practise the numbers: [Capacity Gym](07-capacity.md#authoritative-dns-workload)

**3. API**

- `PUT /zones/{id}/records`
- `POST /zones/{id}:publish`
- `GET /zones/{id}/versions`

**4. Data model**

- `zones(zone_id PK,owner,active_version)`
- `records(zone_id,version,name,type,value,ttl)`
- `publications(zone_id,version,regions)` 

Choose the store: [Database Gym](08-database.md#storage-for-authoritative-dns)

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

**2. Scale:** 50M active series; scrape every 15 seconds; 30-day raw retention. — practise the numbers: [Capacity Gym](07-capacity.md#metrics-and-alerting-workload)

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

Choose the store: [Database Gym](08-database.md#storage-for-metrics-and-alerting)

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

**2. Scale:** 100K applications; 10B evaluations/day; configuration updates are comparatively rare. — practise the numbers: [Capacity Gym](07-capacity.md#feature-flag-platform-workload)

**3. API**

- `PUT /flags/{id}`
- `POST /flags/{id}:rollout`
- `GET /sdk-config?version=`

**4. Data model**

- `flags(flag_id PK,version,rules,default)`
- `segments(segment_id,version,members)`
- `audit(change_id,actor,before,after)` 

Choose the store: [Database Gym](08-database.md#storage-for-feature-flag-platform)

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

**2. Scale:** 1M service instances; 50K watch clients; endpoint churn during deployments. — practise the numbers: [Capacity Gym](07-capacity.md#service-discovery-and-configuration-workload)

**3. API**

- `PUT /services/{name}/instances/{id} {lease}`
- `GET /services/{name}?revision=`
- `WATCH /config?fromRevision=`

**4. Data model**

- `instances(service,id,address,lease_id)`
- `config(key,value,revision)`
- `watch_checkpoints(client,revision)` 

Choose the store: [Database Gym](08-database.md#storage-for-service-discovery-and-configuration)

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

**2. Scale:** 20TB hot data; 200K writes/s; three regions with 80-180 ms inter-region RTT. — practise the numbers: [Capacity Gym](07-capacity.md#multi-region-key-value-store-workload)

**3. API**

- `GET /keys/{key}?consistency=`
- `PUT /keys/{key} {value,expectedVersion}`
- `GET /conflicts/{key}`

**4. Data model**

- `records(key,version,value,tombstone)`
- `ownership(range,epoch,region)`
- `replication_checkpoints(region,range,position)` 

Choose the store: [Database Gym](08-database.md#storage-for-multi-region-key-value-store)

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

**2. Scale:** 100M active workflows; 20M activity completions/day; workflows may last a year. — practise the numbers: [Capacity Gym](07-capacity.md#durable-workflow-engine-workload)

**3. API**

- `POST /workflows {type,key,input}`
- `POST /workflows/{id}/signals`
- `GET /workflows/{id}/history`

**4. Data model**

- `histories(workflow_id,event_sequence,type,payload)`
- `tasks(queue,task_id,lease_epoch)`
- `timers(bucket,workflow_id,fire_at)` 

Choose the store: [Database Gym](08-database.md#storage-for-durable-workflow-engine)

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

**2. Scale:** 5B auctions/day; p99 decision budget 100 ms; 1M campaigns. — practise the numbers: [Capacity Gym](07-capacity.md#ad-auction-and-budget-pacing-workload)

**3. API**

- `POST /auctions {requestId,context}`
- `POST /impressions {eventId,auctionId}`
- `PUT /campaigns/{id}/budget`

**4. Data model**

- `campaigns(id,budget,targeting,version)`
- `budget_allocations(campaign,region,epoch,remaining)`
- `auction_events(auction_id,winner,price)`
- `billable_events(event_id,amount)` 

Choose the store: [Database Gym](08-database.md#storage-for-ad-auction-and-budget-pacing)

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

**2. Scale:** 100M daily users; 30 recommendations requests/day; 100ms p95 serving budget. — practise the numbers: [Capacity Gym](07-capacity.md#recommendation-serving-workload)

**3. API**

- `GET /recommendations?surface=&cursor=`
- `POST /interactions {eventId,itemId,type}`
- `POST /models/{id}:activate`

**4. Data model**

- `features(entity_id,feature_version,event_time,values)`
- `embeddings(entity_id,model_version,vector)`
- `models(id,schema,artifact,active_epoch)`
- `exposures(request_id,model,items)` 

Choose the store: [Database Gym](08-database.md#storage-for-recommendation-serving)

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

**2. Scale:** 100M payment attempts/day; 50ms feature budget; events arrive up to 10 minutes late. — practise the numbers: [Capacity Gym](07-capacity.md#streaming-fraud-detection-workload)

**3. API**

- `POST /risk-decisions {transactionId,features}`
- `POST /risk-events`
- `GET /decisions/{id}/explanation`

**4. Data model**

- `events(event_id PK,entity,event_time,type)`
- `windows(entity,window_start,feature_version,state)`
- `decisions(transaction_id,rule_version,model_version,outcome)` 

Choose the store: [Database Gym](08-database.md#storage-for-streaming-fraud-detection)

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
