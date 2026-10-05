# Database Gym — Let the workload decide

> 50 workloads. Note the access patterns, pick the primary store from four options, then open the reasoning and the staff-level view.

## Contents

- **Foundational** (5): [Storage for url shortener](#storage-for-url-shortener) · [Storage for distributed rate limiter](#storage-for-distributed-rate-limiter) · [Storage for distributed cache](#storage-for-distributed-cache) · [Storage for unique id service](#storage-for-unique-id-service) · [Storage for durable job scheduler](#storage-for-durable-job-scheduler)
- **Social** (5): [Storage for social news feed](#storage-for-social-news-feed) · [Storage for photo sharing](#storage-for-photo-sharing) · [Storage for follow graph](#storage-for-follow-graph) · [Storage for threaded comments](#storage-for-threaded-comments) · [Storage for ephemeral stories](#storage-for-ephemeral-stories)
- **Messaging** (5): [Storage for direct and group chat](#storage-for-direct-and-group-chat) · [Storage for notification platform](#storage-for-notification-platform) · [Storage for email service](#storage-for-email-service) · [Storage for webhook delivery](#storage-for-webhook-delivery) · [Storage for publish-subscribe event bus](#storage-for-publish-subscribe-event-bus)
- **Media** (5): [Storage for youtube-style video platform](#storage-for-youtube-style-video-platform) · [Storage for live streaming platform](#storage-for-live-streaming-platform) · [Storage for music streaming](#storage-for-music-streaming) · [Storage for video conferencing](#storage-for-video-conferencing) · [Storage for podcast platform](#storage-for-podcast-platform)
- **Storage** (5): [Storage for cloud file synchronization](#storage-for-cloud-file-synchronization) · [Storage for distributed object storage](#storage-for-distributed-object-storage) · [Storage for backup and point-in-time restore](#storage-for-backup-and-point-in-time-restore) · [Storage for content-addressed blob store](#storage-for-content-addressed-blob-store) · [Storage for distributed filesystem](#storage-for-distributed-filesystem)
- **Commerce** (5): [Storage for commerce checkout](#storage-for-commerce-checkout) · [Storage for inventory reservation](#storage-for-inventory-reservation) · [Storage for payment ledger](#storage-for-payment-ledger) · [Storage for ticket booking](#storage-for-ticket-booking) · [Storage for marketplace orders](#storage-for-marketplace-orders)
- **Real Time** (5): [Storage for ride dispatch](#storage-for-ride-dispatch) · [Storage for collaborative document editor](#storage-for-collaborative-document-editor) · [Storage for presence and typing indicators](#storage-for-presence-and-typing-indicators) · [Storage for game leaderboard](#storage-for-game-leaderboard) · [Storage for fleet tracking](#storage-for-fleet-tracking)
- **Search** (5): [Storage for web search engine](#storage-for-web-search-engine) · [Storage for search autocomplete](#storage-for-search-autocomplete) · [Storage for product search](#storage-for-product-search) · [Storage for log ingestion and search](#storage-for-log-ingestion-and-search) · [Storage for vector retrieval service](#storage-for-vector-retrieval-service)
- **Infrastructure** (5): [Storage for global load balancer](#storage-for-global-load-balancer) · [Storage for authoritative dns](#storage-for-authoritative-dns) · [Storage for metrics and alerting](#storage-for-metrics-and-alerting) · [Storage for feature flag platform](#storage-for-feature-flag-platform) · [Storage for service discovery and configuration](#storage-for-service-discovery-and-configuration)
- **Advanced** (5): [Storage for multi-region key-value store](#storage-for-multi-region-key-value-store) · [Storage for durable workflow engine](#storage-for-durable-workflow-engine) · [Storage for ad auction and budget pacing](#storage-for-ad-auction-and-budget-pacing) · [Storage for recommendation serving](#storage-for-recommendation-serving) · [Storage for streaming fraud detection](#storage-for-streaming-fraud-detection)

## Foundational

### Storage for url shortener

**Workload:** Create durable short links and serve redirects with a small latency budget. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 10M daily users; 20 redirects each; 100:1 read/write ratio; redirect p99 target 80 ms. Access model: links(code PK, target, owner, expires_at, version); click_events(event_id PK, code, timestamp).

**Pick the primary store:** DynamoDB · PostgreSQL · Elasticsearch · Redis

<details><summary>Answer</summary>

**DynamoDB** — Point lookups by code and conditional writes fit a key-value store; a relational table is also sufficient at smaller scale. Long edge TTLs improve redirect latency but delay target changes; analytics can be eventually consistent while alias ownership cannot.

**Staff-level view:** Model abuse takedown propagation separately from ordinary edits; establish a maximum revocation delay and test cache invalidation under partial regional failure.

</details>

---

### Storage for distributed rate limiter

**Workload:** Enforce burst and sustained request budgets across API workers. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 100M decisions/day; 1M tenants; 2 ms regional decision budget. Access model: bucket(tenant, tokens, last_refill, policy_version); policy(tenant PK, rate, burst).

**Pick the primary store:** Redis · PostgreSQL · ClickHouse · Amazon S3

<details><summary>Answer</summary>

**Redis** — Atomic scripts or commands suit regional bucket mutation; durable policy storage and audited usage should live outside the ephemeral bucket cache. A globally exact limit increases coordination latency; distributed leases buy availability with a calculable overshoot bound.

**Staff-level view:** Define fairness during regional evacuation and prevent one tenant from monopolizing the emergency budget. Roll out policy changes with explicit version semantics.

</details>

---

### Storage for distributed cache

**Workload:** Build a bounded cache in front of authoritative application data. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 2B GETs/day; 200GB hot data; 1KB median values. Access model: entry(key, value, version, expires_at); ring(epoch, virtual_nodes).

**Pick the primary store:** Redis · PostgreSQL · ClickHouse · Amazon S3

<details><summary>Answer</summary>

**Redis** — An in-memory key-value engine supplies TTLs and eviction; the cache is reconstructible and must not become the only durable copy. Aggressive eviction lowers memory cost but creates source traffic; serving stale data is acceptable only for explicitly chosen fields.

**Staff-level view:** Size for failover with one shard unavailable, including connection storms, allocator overhead, replication buffers, and the source load from a cold replacement.

</details>

---

### Storage for unique id service

**Workload:** Issue unique sortable identifiers without a global transaction per ID. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 500M IDs/day; bursts of 200K IDs/s; IDs fit in signed 64-bit storage. Access model: worker_lease(worker_id PK, epoch, expires_at); local_state(last_timestamp, sequence).

**Pick the primary store:** etcd · Redis · Elasticsearch · Cassandra

<details><summary>Answer</summary>

**etcd** — A small consensus store can assign worker identities; generating each ID locally avoids turning the registry into the throughput bottleneck. Sortable IDs expose approximate creation time; larger worker or sequence fields shorten the timestamp horizon.

**Staff-level view:** Prove uniqueness across clock rollback, restored VM snapshots, identity reuse, and disaster recovery. Include a documented migration before the timestamp field expires.

</details>

---

### Storage for durable job scheduler

**Workload:** Execute delayed and recurring jobs with retries and durable status. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 50M jobs/day; 30-day scheduling horizon; execution delay p99 under 10 seconds. Access model: jobs(job_id PK, run_at, state, attempt, lease_epoch); attempts(job_id, attempt, outcome).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — Transactional job state and indexed due-time scans suit moderate scale; partition the schedule and move ready work to a queue as volume grows. At-least-once execution is recoverable but requires idempotent effects; at-most-once execution avoids repeats by accepting lost work.

**Staff-level view:** Specify misfire behavior for missed recurring schedules, timezones, daylight-saving transitions, and prolonged outages; avoid replaying years of missed jobs on recovery.

</details>

---

## Social

### Storage for social news feed

**Workload:** Produce a personalized timeline from followed accounts. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 50M daily users; 20 feed reads/day; follower counts have a heavy tail. Access model: posts(post_id PK, author, created_at, visibility_version); inbox(user_id, rank_key, post_id); follows(user_id, author_id).

**Pick the primary store:** Cassandra · PostgreSQL · Neo4j · Elasticsearch

<details><summary>Answer</summary>

**Cassandra** — User-keyed timeline slices fit partitioned ordered wide rows; cap and bucket inbox partitions to keep hot accounts and history bounded. Fanout-on-write gives fast reads but amplifies celebrity writes; read-time merging saves work at the cost of more feed computation.

**Staff-level view:** Set separate SLOs for post acceptance, follower visibility, privacy revocation, and ranking freshness; a single availability number hides the important failures.

</details>

---

### Storage for photo sharing

**Workload:** Upload photos, generate variants, and display profile galleries. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 8M uploads/day; 4MB original photos; 10 thumbnail views per upload. Access model: photos(photo_id PK, owner, object_key, state, visibility); variants(photo_id, transform_version, object_key).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — Relational metadata and album membership need constrained updates; binary images belong in object storage with explicit publication state. Precomputing variants increases storage but reduces user-visible transform latency; signed URL lifetime sets part of the access-revocation delay.

**Staff-level view:** Coordinate deletion of originals, variants, caches, and backup-retention records; expose deletion progress without promising that a metadata delete erases every byte instantly.

</details>

---

### Storage for follow graph

**Workload:** Maintain directed relationships, counts, and mutual connections. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 200M accounts; 20B edges; most users have fewer than 1,000 edges. Access model: out_edges(source, target, version); in_edges(target, bucket, source); blocks(blocker, blocked).

**Pick the primary store:** DynamoDB · PostgreSQL · Elasticsearch · Redis

<details><summary>Answer</summary>

**DynamoDB** — Edge keys and adjacency-list access fit a partitioned key-value model; a graph engine becomes attractive only when complex traversals dominate. Duplicating inbound and outbound edges accelerates both directions but requires repairable projections and explicit count staleness.

**Staff-level view:** Design repair jobs that compare projection checkpoints and sample edge symmetry without loading the complete graph into memory.

</details>

---

### Storage for threaded comments

**Workload:** Support nested discussion with moderation and stable pagination. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 30M comments/day; one live event may receive 20K replies/s. Access model: comments(comment_id PK, root_id, parent_id, body, version, state); votes(user_id, comment_id, value).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — Parent/root indexes and transactional edits fit relational storage; partition large roots or archive history before a single root overwhelms an index. Materialized thread paths speed subtree reads but make moves expensive; restricting reparenting simplifies storage and moderation semantics.

**Staff-level view:** Distinguish author deletion, moderator hiding, legal removal, and thread locking, since each has different visibility and audit requirements.

</details>

---

### Storage for ephemeral stories

**Workload:** Publish short-lived media and track viewers without exposing expired stories. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 15M stories/day; 150M daily story views; media expiry is strict at serving time. Access model: stories(story_id PK, owner, expires_at, audience_version); views(story_id, viewer_id, first_seen_at).

**Pick the primary store:** DynamoDB · PostgreSQL · Elasticsearch · Redis

<details><summary>Answer</summary>

**DynamoDB** — Story ownership and expiration-index access are key-oriented; TTL cleanup is housekeeping and must not enforce the user-visible expiry invariant. Exact receipts improve creator analytics but increase privacy and storage cost; delayed aggregates are cheaper for huge audiences.

**Staff-level view:** Separate the promised visibility deadline from physical deletion and backup expiration, and measure each boundary independently.

</details>

---

## Messaging

### Storage for direct and group chat

**Workload:** Deliver durable messages across online and offline devices. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 50M daily users; 40 messages/user/day; groups up to 1,000 members. Access model: messages(conversation_id, sequence, message_id, body); memberships(conversation_id,user_id,version); device_cursor(device_id,conversation_id,sequence).

**Pick the primary store:** Cassandra · PostgreSQL · Neo4j · Elasticsearch

<details><summary>Answer</summary>

**Cassandra** — Conversation-plus-time buckets support ordered history and high write volume; a separate transactional authority manages membership and sequence allocation. Per-conversation order is useful and affordable; a global order across all chats introduces coordination with little user benefit.

**Staff-level view:** Specify device synchronization, end-to-end encryption metadata boundaries, abuse reporting, and multi-region ownership transfer without claiming that a WebSocket alone guarantees delivery.

</details>

---

### Storage for notification platform

**Workload:** Route product events to email, push, SMS, and in-app inboxes. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 500M notification intents/day; provider quotas differ by region and channel. Access model: intents(event_id,user_id,template_version); deliveries(intent_id,channel,state,provider_key); preferences(user_id,version).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — Preference and intent uniqueness need transactions; channel queues absorb bursts while an append-only delivery history supports audit and reconciliation. Provider failover improves availability but can duplicate ambiguous deliveries; status should preserve uncertainty instead of inventing certainty.

**Staff-level view:** Set channel-specific SLOs and per-tenant cost budgets, and rehearse a provider brownout with callbacks arriving after failover.

</details>

---

### Storage for email service

**Workload:** Store mailboxes and reliably accept, route, and search messages. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 100M inbound messages/day; 80KB average body plus attachments; 1M active mailboxes. Access model: messages(message_id PK, envelope, blob_key); mailbox_entries(mailbox_id, uid, message_id, flags); delivery_attempts(message_id,destination).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — Mailbox metadata benefits from indexed transactional updates; full message bodies and attachments can be stored as immutable objects. Scanning before acceptance reduces accepted malicious mail but lengthens SMTP latency; scanning after durable spooling requires quarantine and status handling.

**Staff-level view:** Plan mailbox export, delivery-loop protection, per-recipient bounce state, and restore tests that verify both bodies and mailbox indexes.

</details>

---

### Storage for webhook delivery

**Workload:** Deliver signed events to customer endpoints with retry visibility. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 200M deliveries/day; 100K endpoints; endpoints can be slow or unavailable. Access model: events(event_id PK, tenant_id, payload_version); subscriptions(id,url,secret_version); attempts(event_id,subscription_id,number,status).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — Subscription state, durable attempts, and unique event-endpoint keys are relational; queues and logs handle asynchronous delivery volume. Strict delivery order simplifies receivers but lets one poison event block a stream; versioned events permit parallel recovery.

**Staff-level view:** Make replay preserve original event identity while recording a distinct replay request, and bound customer endpoint access to prevent internal-network targeting.

</details>

---

### Storage for publish-subscribe event bus

**Workload:** Distribute durable events to independently checkpointed consumers. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 1B events/day; 1KB mean event; 20 consumer groups. Access model: log(topic,partition,offset,key,payload); group_offsets(group,topic,partition,offset); schema(subject,version).

**Pick the primary store:** Apache Kafka · PostgreSQL · Redis · Elasticsearch

<details><summary>Answer</summary>

**Apache Kafka** — A partitioned durable log supports replay and consumer offsets; it complements rather than replaces queryable application databases. Long retention enables recovery but increases storage and replay load; ordering by one hot key limits available parallelism.

**Staff-level view:** Budget replay traffic separately, govern schema compatibility, and provide tenant-level kill switches that preserve other streams during poison-event incidents.

</details>

---

## Media

### Storage for youtube-style video platform

**Workload:** Accept large video uploads, process adaptive renditions, and serve globally cached playback. This is a proposed interview design, not a claim about YouTube internals. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 20M daily viewers watch 30 minutes each at an assumed 3 Mb/s average: 13.5 PB/day of delivered video; 100K uploads/day averaging 600MB = 60TB/day of originals before renditions. Access model: videos(video_id PK, owner, state, source_version, visibility_version); uploads(upload_id PK, video_id, parts, checksum, expires_at); jobs(video_id, source_version, recipe_version, rendition, state, lease_epoch); assets(video_id, recipe_version, rendition, segment, object_key, checksum); publications(video_id PK, manifest_version, published_at).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — Use transactional metadata and publication pointers, object storage for originals and segments, a durable log for workflow events, and a search index as a rebuildable discovery projection. More codecs and renditions improve device coverage and bandwidth efficiency but increase encoding time and storage; eager encoding helps popular uploads while demand-driven renditions reduce waste.

**Staff-level view:** Separate upload, processing, playback-start, rebuffering, and deletion SLOs. Compare regional encoding cost, CDN egress, and origin shielding; plan takedown propagation through authorization, manifests, caches, search, and derived clips.

</details>

---

### Storage for live streaming platform

**Workload:** Ingest live broadcasts and deliver adaptive playback with a bounded live delay. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 10K concurrent broadcasters; 5M peak viewers; target glass-to-glass latency 5 seconds. Access model: streams(stream_id PK, ingest_region, state, epoch); segments(stream_id, epoch, sequence, duration, key); manifests(stream_id,version).

**Pick the primary store:** DynamoDB · PostgreSQL · Elasticsearch · Redis

<details><summary>Answer</summary>

**DynamoDB** — Stream metadata and segment-index lookups are key-based; media bytes belong in object storage, with a separate coordination authority for stream ownership. Smaller segments reduce live delay but increase request overhead and sensitivity to jitter; longer buffers improve resilience while moving viewers further behind live.

**Staff-level view:** Treat live latency, playback availability, recording completeness, and rights enforcement as independent objectives; test failover mid-segment and mid-ad-break.

</details>

---

### Storage for music streaming

**Workload:** Deliver licensed audio, maintain libraries, and synchronize playback state. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 20M daily listeners; 40 tracks/day; 4MB average delivered audio per track. Access model: tracks(track_id PK, asset_version, rights_version); playlist_entries(playlist_id,position_id,track_id); sessions(user_id,device_id,position,version).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — Playlists, catalog relationships, and subscription metadata fit relational constraints; media is immutable object data served by a CDN. Frequent entitlement checks improve revocation latency but add startup latency and dependency load; token lifetime sets a measurable compromise.

**Staff-level view:** Maintain an auditable rights-decision version and reconcile royalty events by stable playback IDs without counting network retries as extra listens.

</details>

---

### Storage for video conferencing

**Workload:** Route interactive audio and video between participants with adaptive quality. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 500K concurrent users; average room size 6; target interactive latency below 300 ms. Access model: rooms(room_id PK, region, owner_epoch); participants(room_id,user_id,session_id,role); recordings(recording_id,state,object_key).

**Pick the primary store:** Redis · PostgreSQL · ClickHouse · Amazon S3

<details><summary>Answer</summary>

**Redis** — Ephemeral room and presence routing can use a low-latency store; durable room policy and recordings require persistent metadata and object storage. An SFU saves endpoint uplink compared with full mesh but costs server egress; mixing simplifies clients at a substantial compute and latency cost.

**Staff-level view:** Partition failure domains by room, quantify regional evacuation capacity, and make recording consent and membership policy explicit without putting media through the signaling service.

</details>

---

### Storage for podcast platform

**Workload:** Publish episodes from creator feeds and deliver resumable audio playback. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 2M feeds; 200K updated episodes/day; 10M listeners. Access model: feeds(feed_id PK, url, etag, next_poll_at); episodes(show_id,guid,asset_version); progress(user_id,episode_id,position,session_version).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — Feed identity, episodes, subscriptions, and progress relationships fit relational indexes; media and download blobs belong in object storage. Aggressive polling improves freshness for active shows but wastes network and publisher capacity; adaptive schedules require starvation bounds for quiet feeds.

**Staff-level view:** Isolate hostile or malformed feeds with byte, recursion, redirect, and fetch-time limits, and build explainable ingestion status for creators.

</details>

---

## Storage

### Storage for cloud file synchronization

**Workload:** Synchronize file versions and folder changes across devices. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 10M active users; 100M file mutations/day; typical file 2MB. Access model: files(file_id PK,parent_id,name,current_version); versions(file_id,version,chunk_manifest); changes(account_id,sequence,operation).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — File trees, version pointers, and atomic rename semantics fit transactional metadata; chunk content is immutable object storage. Content-defined chunking reduces upload bytes for edits but adds client CPU, indexing complexity, and encryption/deduplication tradeoffs.

**Staff-level view:** Define rename atomicity across folders, deletion recovery, shared-folder permission revocation, and safe garbage collection in the presence of offline devices.

</details>

---

### Storage for distributed object storage

**Workload:** Store immutable or versioned blobs with replication and integrity checks. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 10PB logical data; 50M objects; 20GB/s peak read traffic. Access model: object_index(bucket,key,version,manifest); fragments(object_id,stripe,location,checksum); repairs(fragment_id,state).

**Pick the primary store:** FoundationDB · Redis · Elasticsearch · ClickHouse

<details><summary>Answer</summary>

**FoundationDB** — A transactional metadata layer can manage object pointers and manifests; large fragments live on storage nodes or a separate object backend, not in small database values. Erasure coding lowers storage overhead but increases repair and small-write complexity; replication is simpler and often better for hot small objects.

**Staff-level view:** State consistency separately for object writes, listings, multipart completion, and cross-region replication. Do not assume all object stores or regional copies share identical guarantees.

</details>

---

### Storage for backup and point-in-time restore

**Workload:** Create recoverable snapshots and continuous change archives. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 500TB source data; 2TB changes/day; target RPO 5 minutes and RTO 4 hours. Access model: snapshots(snapshot_id PK, base_checkpoint, manifest, state); log_segments(start,end,checksum); restore_jobs(id,target,verified_at).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — A relational catalog tracks snapshot dependencies and restore jobs; encrypted immutable backup objects provide the byte archive. Longer retention and immutable backups improve recovery options but increase cost; low RTO usually needs preprovisioned capacity rather than a cheaper archive alone.

**Staff-level view:** Test compromised credentials, key loss, and region loss as recovery scenarios. A backup success dashboard must show verified recovery points and measured restore time.

</details>

---

### Storage for content-addressed blob store

**Workload:** Deduplicate immutable content while preserving tenant isolation and safe collection. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 3PB logical content; expected 40% duplicate bytes; 100M references/day. Access model: blobs(digest PK,size,verified,state); references(tenant,reference_id,digest); gc_candidates(digest,generation).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — Unique references and deletion-generation checks fit transactions; content hashes point to immutable object bytes. Global deduplication saves more storage but complicates isolation, encryption, accounting, and deletion guarantees compared with tenant-scoped deduplication.

**Staff-level view:** Prove collection safety under concurrent references, delayed replicas, interrupted sweeps, and hash-verification failures; measure storage saved after accounting for metadata and replication.

</details>

---

### Storage for distributed filesystem

**Workload:** Expose a hierarchical namespace over replicated file chunks. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 1B files; 5PB data; 100K concurrent clients; most operations are metadata reads. Access model: inodes(inode_id PK,parent,name,version); chunk_map(inode_id,index,chunk_id); leases(inode_id,client,epoch).

**Pick the primary store:** FoundationDB · Redis · Elasticsearch · ClickHouse

<details><summary>Answer</summary>

**FoundationDB** — Transactional key ranges can represent a namespace and coordinate metadata changes; large file content belongs in separately replicated chunks. Fine-grained metadata sharding increases capacity but makes atomic rename and directory listings harder; central metadata is simpler until measured limits require partitioning.

**Staff-level view:** Write down the exact POSIX subset, append semantics, client-cache invalidation rules, and behavior during metadata quorum loss before drawing data-node boxes.

</details>

---

## Commerce

### Storage for commerce checkout

**Workload:** Convert a cart into an order with payment and inventory coordination. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 5M orders/day; flash-sale peak 20K checkout attempts/s. Access model: orders(order_id PK,state,amount,currency); reservations(order_id,sku,quantity,expires_at); payment_attempts(order_id,key,provider_state).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — Order state, reservation invariants, and payment-attempt uniqueness benefit from transactions; external payment effects still require a saga and reconciliation. Long inventory holds improve checkout completion but reduce availability to other buyers; short holds require well-defined late-payment compensation.

**Staff-level view:** Measure unknown payments, reservation age, refund backlog, and reconciliation lag alongside checkout success. Write operational playbooks that preserve evidence for ambiguous money movement.

</details>

---

### Storage for inventory reservation

**Workload:** Maintain available stock and bounded holds across warehouses. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 20M SKUs; 50M stock mutations/day; a hot SKU may receive 50K attempts/s. Access model: stock(warehouse,sku,on_hand,reserved,version); reservations(id,order_id,sku,qty,state,expires_at); stock_ledger(event_id,delta).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — Conditional updates, row locks, unique ledger events, and reservation transitions fit a transactional relational store. Regional quota allocation reduces contention and latency but can strand stock in idle regions; rebalancing must preserve ownership and prevent double allocation.

**Staff-level view:** Define invariants for damaged goods, returns, transfers in flight, backorders, and manual adjustments instead of treating stock as one mutable integer.

</details>

---

### Storage for payment ledger

**Workload:** Record balanced financial entries with immutable history and reconciled external transfers. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 10M ledger transactions/day; 7-year business retention assumption; exact integer minor units. Access model: transactions(tx_id PK,idempotency_key UNIQUE,state); entries(tx_id,account_id,currency,amount_minor); account_balances(account_id,currency,version).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — Transactions and constraints support atomic balanced postings and unique requests; append-only entries provide an audit trail independent of balance projections. Synchronous balance projections simplify immediate reads but add contention; asynchronous projections need freshness markers and cannot authorize overspend alone.

**Staff-level view:** Separate ledger truth, available balance, provider settlement, and bank reconciliation. Treat recovery, access control, and immutable audit evidence as design requirements rather than afterthoughts.

</details>

---

### Storage for ticket booking

**Workload:** Sell assigned seats with temporary holds and durable booking outcomes. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 100K seats for one event; 3M buyers arrive in a minute. Access model: seats(event_id,seat_id,state,hold_id,version); holds(hold_id,user_id,expires_at,state); bookings(booking_id,hold_id,payment_id).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — Assigned-seat uniqueness and atomic multi-seat holds fit relational transactions; read-heavy maps should be cached separately. Strict queue fairness can reduce throughput; opportunistic admission improves utilization but must not undermine the advertised purchase policy.

**Staff-level view:** Model bot resistance, waiting-room token replay, queue position continuity during failover, and the user-visible policy for ambiguous payment completion.

</details>

---

### Storage for marketplace orders

**Workload:** Coordinate multi-seller orders, shipments, returns, and seller payouts. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 2M orders/day; average 3 sellers/order; fulfillment can take weeks. Access model: orders(id,buyer,total); order_items(id,seller,state,amount); shipments(id,item_id,status_version); payout_items(item_id,state).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — Orders, items, returns, and payout eligibility form transactional relationships; external sellers and carriers still require asynchronous state machines. Independent item workflows improve resilience but make totals, customer communication, and compensation more complex than a single order status.

**Staff-level view:** Define operational ownership for stuck states and disputed deliveries, and make seller-specific failures visible without exposing unrelated customer data.

</details>

---

## Real Time

### Storage for ride dispatch

**Workload:** Match riders with nearby available drivers and track trip state. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 2M active drivers; location updates every 5 seconds; dispatch target under 2 seconds. Access model: driver_state(driver_id PK,availability,trip_id,version); locations(cell,driver_id,sequence,observed_at); trips(trip_id,state,rider,driver).

**Pick the primary store:** PostgreSQL with PostGIS · Cassandra · Redis · Elasticsearch

<details><summary>Answer</summary>

**PostgreSQL with PostGIS** — Spatial queries can start in PostGIS, with transactional driver/trip ownership; at high update volume a separate ephemeral location index reduces durable-write pressure. Frequent location updates improve matching but consume battery and ingest capacity; adaptive frequency can depend on trip state and movement.

**Staff-level view:** Prove assignment safety across retries and regional ownership changes, and distinguish dispatch latency from location freshness and driver acceptance rate.

</details>

---

### Storage for collaborative document editor

**Workload:** Merge concurrent document edits while supporting offline work and durable history. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 1M concurrent editors; 5 operations/s per active editor; documents up to 10MB. Access model: documents(id,permission_version,snapshot_version); operations(document_id,op_id,causal_metadata,payload); snapshots(document_id,version,object_key).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — Document ACLs and operation identity fit relational transactions; operation logs and immutable snapshots may use log and object storage as load grows. CRDTs help offline convergence but carry metadata and semantic complexity; OT can be compact but relies on carefully managed transformation context.

**Staff-level view:** Specify supported edit semantics, undo behavior across collaborators, offline horizon, and snapshot compatibility before choosing a merge algorithm.

</details>

---

### Storage for presence and typing indicators

**Workload:** Show approximate online and typing state with bounded freshness. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 30M connected devices; heartbeat every 30 seconds; typing expiry after 5 seconds. Access model: device_presence(user_id,device_id,session_epoch,last_seen); subscribers(user_id,watcher_id); typing(conversation_id,user_id,expires_at).

**Pick the primary store:** Redis · PostgreSQL · ClickHouse · Amazon S3

<details><summary>Answer</summary>

**Redis** — Short-lived per-device state and pub/sub-style fanout fit an ephemeral store; presence is reconstructible and should not gate durable message acceptance. Short heartbeat intervals improve freshness but increase battery and traffic; presence accuracy should be bounded and approximate by product contract.

**Staff-level view:** Budget fanout by watchers and privacy rules, not just connected users, and isolate presence outages from message delivery.

</details>

---

### Storage for game leaderboard

**Workload:** Rank players by validated scores with season boundaries and tie rules. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 50M players; 200M score events/day; top-100 reads dominate. Access model: score_events(event_id PK,player,season,delta); scores(season,player,total,version); season(id,state,final_checkpoint).

**Pick the primary store:** Redis · PostgreSQL · ClickHouse · Amazon S3

<details><summary>Answer</summary>

**Redis** — Ordered score sets are useful for fast ranking projections; durable validated score events belong in a persistent log or database for replay and correction. Exact live global rank requires expensive coordination or indexing; top-K and approximate personal rank can scale independently.

**Staff-level view:** Define tie-breaking, anti-cheat review, prize finality, and late correction policy as part of ranking correctness.

</details>

---

### Storage for fleet tracking

**Workload:** Ingest device positions, display moving fleets, and detect geofence transitions. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 5M devices; update every 10 seconds while moving; 90-day history. Access model: telemetry(device_id,time_bucket,sequence,observed_at,point); latest(device_id,sequence,point); geofence_state(device_id,fence_id,state,version).

**Pick the primary store:** TimescaleDB · Neo4j · Redis · PostgreSQL

<details><summary>Answer</summary>

**TimescaleDB** — Time-partitioned telemetry and SQL/geospatial queries fit a time-series relational extension; a separate latest-position cache serves dashboards. Keeping every raw point aids audits and analytics but increases storage; downsampling must preserve the fidelity required for billing or route evidence.

**Staff-level view:** Define clock-drift tolerance, device identity reset, tenant isolation, and the difference between real-time safety alerts and best-effort business notifications.

</details>

---

## Search

### Storage for web search engine

**Workload:** Crawl public pages, build an inverted index, and rank query results. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 10B pages; 100M queries/day; p95 search target 300 ms. Access model: urls(url_id PK,canonical_url,next_fetch_at,content_hash); documents(doc_id,version,terms,signals); postings(term,shard,doc_id,positions).

**Pick the primary store:** Elasticsearch · PostgreSQL · Redis · Amazon S3

<details><summary>Answer</summary>

**Elasticsearch** — An inverted-index engine supports lexical retrieval and ranking primitives; the crawl frontier and canonical document archive remain separate durable systems. Fresh indexing consumes resources that could improve query latency; tier pages by change rate and query value instead of recrawling everything uniformly.

**Staff-level view:** Set budgets for crawling, indexing freshness, query tail latency, and removal propagation, and plan index-generation rollback with durable tombstone preservation.

</details>

---

### Storage for search autocomplete

**Workload:** Return fast prefix suggestions with popularity, locale, and safety filtering. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 500M keystroke queries/day; p99 server budget 30 ms; 50 locales. Access model: suggestions(locale,prefix,rank,candidate_id); candidates(id,text,policy_version); query_counts(term,time_bucket,count).

**Pick the primary store:** Redis · PostgreSQL · ClickHouse · Amazon S3

<details><summary>Answer</summary>

**Redis** — Precomputed prefix-to-top-K lists fit a fast key-value serving layer; durable query aggregates and versioned snapshots can live elsewhere. Precomputed suggestions are fast but stale between builds; online ranking improves freshness while increasing latency and cache fragmentation.

**Staff-level view:** Specify minimum cohort thresholds, retention, locale normalization, and emergency removals before using raw search logs as a suggestion source.

</details>

---

### Storage for product search

**Workload:** Search a catalog with facets while respecting current price and availability. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 50M products; 80M queries/day; 10M product updates/day. Access model: products(product_id PK,price,stock_version,catalog_version); search_docs(product_id,index_version,terms,facets); price_history(product_id,version).

**Pick the primary store:** Elasticsearch · PostgreSQL · Redis · Amazon S3

<details><summary>Answer</summary>

**Elasticsearch** — Full-text retrieval, facets, and ranking fit a search index; authoritative stock, orders, and prices remain transactional records. Denormalizing catalog fields accelerates queries but adds update fanout and staleness; current checkout validation protects the purchase invariant.

**Staff-level view:** Define a freshness budget per field: descriptions can lag longer than price or availability. Measure zero-result rate and business correctness separately from query latency.

</details>

---

### Storage for log ingestion and search

**Workload:** Collect high-volume logs and support bounded operational queries. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 100TB raw logs/day; 30-day hot retention; 10K tenants. Access model: log_events(tenant,event_id,event_time,service,body); partitions(tenant,time_bucket,location); query_jobs(id,budget,state).

**Pick the primary store:** ClickHouse · PostgreSQL · Redis · Neo4j

<details><summary>Answer</summary>

**ClickHouse** — Columnar time-filtered scans and aggregation suit structured logs; full-text needs may justify a separate search engine rather than indexing every field blindly. Indexing every field improves arbitrary search but multiplies storage and ingest cost; selective indexing plus compressed scans is often more economical.

**Staff-level view:** Budget bytes, cardinality, query CPU, and result size per tenant, and provide operationally useful degradation during the exact incidents when logs surge.

</details>

---

### Storage for vector retrieval service

**Workload:** Retrieve semantically similar documents while enforcing metadata filters and permissions. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 500M chunks; 768-dimensional embeddings; 50M searches/day. Access model: chunks(chunk_id PK,document_id,text_version,embedding_version,acl_version); vectors(chunk_id,embedding); index_generations(id,model,checkpoint).

**Pick the primary store:** PostgreSQL with pgvector · Elasticsearch · Redis · ClickHouse

<details><summary>Answer</summary>

**PostgreSQL with pgvector** — A relational vector extension suits moderate vector sets with transactional metadata and filters; larger workloads may need a dedicated vector engine after measured recall and latency tests. Approximate indexes improve speed with a recall tradeoff; aggressive post-filtering and compression can further reduce useful recall.

**Staff-level view:** Evaluate semantic quality, filtered recall, freshness, deletion propagation, and per-query cost independently; never treat similarity score as proof of factual correctness.

</details>

---

## Infrastructure

### Storage for global load balancer

**Workload:** Route traffic to healthy capacity while controlling regional failover. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 10M requests/s global peak; three active regions; long-lived connections included. Access model: routes(service,version,policy); backends(id,region,capacity,health_epoch); health_samples(backend,time,result).

**Pick the primary store:** etcd · Redis · Elasticsearch · Cassandra

<details><summary>Answer</summary>

**etcd** — A consensus-backed configuration store can version routing policies; proxy data planes should retain a last-known-good configuration and not query consensus for every request. Fast health reaction shortens outages but risks flapping; slower convergence is steadier but routes more requests to failing capacity.

**Staff-level view:** Define overload policy before failover, test correlated health-check failures, and include long-lived connection migration in regional evacuation drills.

</details>

---

### Storage for authoritative dns

**Workload:** Serve authoritative records globally with controlled update propagation. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 5M queries/s; 10M zones; records range from 30-second to 1-day TTLs. Access model: zones(zone_id PK,owner,active_version); records(zone_id,version,name,type,value,ttl); publications(zone_id,version,regions).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — Zone ownership and versioned record changes fit relational transactions; serving nodes should use compiled in-memory snapshots. Low TTL speeds planned changes but increases query load and dependency on authoritative availability; high TTL improves resilience while slowing propagation.

**Staff-level view:** Test DNSSEC signing rollover where used, negative-cache behavior, delegation errors, and recovery from a control-plane compromise without disrupting stable serving.

</details>

---

### Storage for metrics and alerting

**Workload:** Collect time-series measurements and evaluate actionable alerts. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 50M active series; scrape every 15 seconds; 30-day raw retention. Access model: samples(series_id,timestamp,value); series(label_set_hash,labels); rules(rule_id,expression,for_duration); alert_state(rule_id,labels,state,since).

**Pick the primary store:** Prometheus-compatible TSDB · PostgreSQL · Redis · Neo4j

<details><summary>Answer</summary>

**Prometheus-compatible TSDB** — Time-series storage and label-based retrieval fit metrics; choose a distributed implementation when a single server cannot meet the stated active-series and retention workload. Fine scrape intervals improve resolution but multiply samples; downsampling reduces cost while losing short-lived spikes and some aggregation fidelity.

**Staff-level view:** Design alert ownership, deduplication, inhibition, and evaluation continuity during regional failover; monitor the monitor through an independent failure domain.

</details>

---

### Storage for feature flag platform

**Workload:** Distribute versioned configuration for safe progressive product rollouts. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 100K applications; 10B evaluations/day; configuration updates are comparatively rare. Access model: flags(flag_id PK,version,rules,default); segments(segment_id,version,members); audit(change_id,actor,before,after).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — Versioned rules, audit records, and administrative workflows fit a relational source of truth; evaluation happens from local immutable snapshots. Local evaluation improves availability and latency but makes immediate revocation depend on SDK refresh and connectivity; critical safety controls may need a separate path.

**Staff-level view:** Define stale-config limits, defaults during startup, audit and approval ownership, and the maximum supported offline period for each class of flag.

</details>

---

### Storage for service discovery and configuration

**Workload:** Maintain service endpoints and deliver coherent configuration revisions. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 1M service instances; 50K watch clients; endpoint churn during deployments. Access model: instances(service,id,address,lease_id); config(key,value,revision); watch_checkpoints(client,revision).

**Pick the primary store:** etcd · Redis · Elasticsearch · Cassandra

<details><summary>Answer</summary>

**etcd** — Small strongly ordered configuration and leases fit a consensus key-value store; large blobs and high-volume telemetry should be kept elsewhere. Strict configuration consistency can reduce control-plane availability during partitions; cached data-plane operation limits the user impact.

**Staff-level view:** Test quorum loss, watch compaction, mass reconnects, and certificate rotation together; control-plane recovery must not trigger a larger data-plane outage.

</details>

---

## Advanced

### Storage for multi-region key-value store

**Workload:** Serve partitioned keys across regions with an explicit conflict and consistency model. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 20TB hot data; 200K writes/s; three regions with 80-180 ms inter-region RTT. Access model: records(key,version,value,tombstone); ownership(range,epoch,region); replication_checkpoints(region,range,position).

**Pick the primary store:** DynamoDB · PostgreSQL · Elasticsearch · Redis

<details><summary>Answer</summary>

**DynamoDB** — A managed partitioned key-value store is one option, but choose and verify the exact regional consistency mode and supported configuration; do not infer global guarantees from local reads. Low-latency independent regional writes require conflict handling or weaker invariants; synchronous global coordination trades latency and partition availability for stronger ordering.

**Staff-level view:** State the permitted anomaly for each operation, prove fencing during failover, and test restoration of an old region without resurrecting deleted data.

</details>

---

### Storage for durable workflow engine

**Workload:** Run long-lived business processes with durable timers, activities, and compensation. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 100M active workflows; 20M activity completions/day; workflows may last a year. Access model: histories(workflow_id,event_sequence,type,payload); tasks(queue,task_id,lease_epoch); timers(bucket,workflow_id,fire_at).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — Append-only histories, timers, and task leases can be stored transactionally; larger deployments partition by workflow ID and separate task delivery from history authority. Durable orchestration centralizes recoverability but introduces history storage and versioning complexity; choreography distributes ownership while making global progress harder to inspect.

**Staff-level view:** Design version retirement, operator intervention, workflow cancellation, and data retention for processes that outlive several application releases.

</details>

---

### Storage for ad auction and budget pacing

**Workload:** Select eligible ads under tight latency while respecting advertiser budgets. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 5B auctions/day; p99 decision budget 100 ms; 1M campaigns. Access model: campaigns(id,budget,targeting,version); budget_allocations(campaign,region,epoch,remaining); auction_events(auction_id,winner,price); billable_events(event_id,amount).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — Campaign policy and billing invariants fit transactional records; serving uses cached candidates and bounded local budget allocations, with a durable event log for reconciliation. Tight pacing reduces overspend but can underspend during partitions; larger local allocations improve serving availability while increasing in-flight exposure.

**Staff-level view:** Separate auction latency, relevance, billable-event accuracy, advertiser budget safety, and fraud detection. Explain which accounting decisions are provisional.

</details>

---

### Storage for recommendation serving

**Workload:** Retrieve and rank personalized candidates with fresh features and safe fallbacks. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 100M daily users; 30 recommendations requests/day; 100ms p95 serving budget. Access model: features(entity_id,feature_version,event_time,values); embeddings(entity_id,model_version,vector); models(id,schema,artifact,active_epoch); exposures(request_id,model,items).

**Pick the primary store:** Redis · PostgreSQL · ClickHouse · Amazon S3

<details><summary>Answer</summary>

**Redis** — Online feature vectors and cached candidate lists fit low-latency key-value access; durable training data, event history, and model artifacts belong in separate stores. Fresh online features improve responsiveness but add dependency and consistency costs; precomputed candidates are resilient but may miss immediate intent.

**Staff-level view:** Evaluate quality, latency, feature freshness, policy compliance, experiment validity, and cost together; maintain a useful fallback that can run when the feature platform is unavailable.

</details>

---

### Storage for streaming fraud detection

**Workload:** Evaluate transaction risk using fresh event features and reviewable decisions. Choose the primary serving or authoritative store for the access pattern described; explain the supporting stores.

**Constraints:** 100M payment attempts/day; 50ms feature budget; events arrive up to 10 minutes late. Access model: events(event_id PK,entity,event_time,type); windows(entity,window_start,feature_version,state); decisions(transaction_id,rule_version,model_version,outcome).

**Pick the primary store:** PostgreSQL · Redis · Elasticsearch · Amazon S3

<details><summary>Answer</summary>

**PostgreSQL** — Auditable decisions and review state need durable transactional records; stream state and online features can use specialized stores with checkpointed recovery. Waiting for more events can improve signal completeness but violates real-time latency; the design needs an explicit policy for uncertain or stale features.

**Staff-level view:** Separate detection quality from financial loss and customer friction; document model versions, fallback decisions, appeals, and the operational response to a stale feature pipeline.

</details>

---
