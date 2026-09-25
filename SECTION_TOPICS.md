# System Design Olympic Training Camp: section and topic map

Source snapshot: current local checkout, 2026-09-25. The sidebar has 24 sections. Nine fixed libraries contain 618 topic cards. The Question bank contains 929 prompts derived from Curriculum and System workouts.

Navigation and route behavior were checked in [data/navigation.ts](data/navigation.ts), [features/shell.tsx](features/shell.tsx), and the feature modules. Exact titles and categories below come from the JSON files under [public/content](public/content), which the app fetches at runtime. Authored content under [content](content) is compiled by [scripts/build-content.mjs](scripts/build-content.mjs). This is a snapshot of the current JSON, including local edits.

## Section index

| Section | Entries or scope |
| --- | ---: |
| [Overview](#dashboard) | Personal |
| [Today's training](#training) | Personal |
| [Recall room](#recall) | Personal |
| [Curriculum](#curriculum) | 120 |
| [Pattern gym](#patterns) | 70 |
| [Drill gym](#drills) | 150 |
| [System workouts](#systems) | 50 |
| [Trade-off gym](#tradeoffs) | 54 |
| [Failure gym](#failures) | 50 |
| [Capacity gym](#capacity) | 50 |
| [Database gym](#database) | 50 |
| [Pattern recognition](#recognition) | 210 |
| [Mock interviews](#mock) | 50 |
| [Interviewer sparring](#hostile) | 50 |
| [First principles](#principles) | 50 |
| [Whiteboard](#whiteboard) | Personal |
| [Notes & sketches](#notes) | Personal |
| [Training plan](#plan) | 6 |
| [Performance](#progress) | Personal |
| [Training journal](#journal) | Personal |
| [Design recipes](#recipes) | 24 |
| [Bookmarks](#favorites) | Personal |
| [Question bank](#questions) | 929 |
| [Pattern map](#map) | 8 |

## Your camp

<a id="dashboard"></a>

### Overview

Progress dashboard: practice readiness, streak, training minutes, completed drills and system workouts, today's circuit, recall, seven-day rhythm, and the next mock. Its subjects come from the training libraries and saved progress.

<a id="training"></a>

### Today's training

Choose a 30, 60, 90, or 120 minute circuit. Blocks link into recall, curriculum, patterns, drills, systems, capacity, sparring, and journal. Marking a block complete is separate from logging mastery.

| Circuit | Blocks in order |
| --- | --- |
| 30 min | Recall warm-up; database partitioning; hot-partition drill; reflection |
| 60 min | Recall warm-up; database partitioning; fan-out pattern; news-feed design; reflection |
| 90 min | Recall warm-up; database partitioning; hot-partition drill; YouTube design; reflection |
| 120 min | Capacity estimation; database partitioning; fan-out pattern; YouTube design; architecture defense; mistakes and reflection |

<a id="recall"></a>

### Recall room

Spaced retrieval draws from due reviews in any library and fresh Curriculum concepts. Explain the problem, mechanism, trade-offs, and failure modes, then rate the recall as Again, Hard, Good, or Easy.

## Training grounds

<a id="curriculum"></a>

### Curriculum (120 topics)

System design fundamentals. Each lesson has a mental model, mechanism, examples, use and avoid cases, trade-offs, failures, alternatives, real-system context, deeper reading, and interview questions.

Source: [curriculum.json](public/content/curriculum.json). Topics follow the app's category order.

#### foundations (8)

- Requirements and invariants — ID: requirements-and-invariants
- Capacity estimation — ID: capacity-estimation
- Latency versus throughput — ID: latency-versus-throughput
- Little's law — ID: little-s-law
- Tail latency — ID: tail-latency
- Horizontal and vertical scaling — ID: horizontal-and-vertical-scaling
- Stateless service design — ID: stateless-service-design
- Partitioning by ownership — ID: partitioning-by-ownership

#### networking (8)

- DNS resolution and caching — ID: dns-resolution-and-caching
- TCP reliability and congestion — ID: tcp-reliability-and-congestion
- TLS termination — ID: tls-termination
- Layer 4 and layer 7 load balancing — ID: layer-4-and-layer-7-load-balancing
- Connection pooling — ID: connection-pooling
- HTTP/2 multiplexing — ID: http-2-multiplexing
- QUIC and HTTP/3 — ID: quic-and-http-3
- Service discovery — ID: service-discovery

#### data storage (8)

- Relational data modeling — ID: relational-data-modeling
- Document storage — ID: document-storage
- Key-value storage — ID: key-value-storage
- Wide-column storage — ID: wide-column-storage
- Object storage — ID: object-storage
- Time-series data modeling — ID: time-series-data-modeling
- Graph data modeling — ID: graph-data-modeling
- Columnar analytical storage — ID: columnar-analytical-storage

#### database internals (8)

- B-tree indexes — ID: b-tree-indexes
- LSM trees and compaction — ID: lsm-trees-and-compaction
- Write-ahead logging — ID: write-ahead-logging
- MVCC snapshots — ID: mvcc-snapshots
- Transaction isolation levels — ID: transaction-isolation-levels
- Optimistic concurrency control — ID: optimistic-concurrency-control
- Deadlocks and lock ordering — ID: deadlocks-and-lock-ordering
- Query planning and statistics — ID: query-planning-and-statistics

#### caching (8)

- Cache-aside — ID: cache-aside
- Write-through caching — ID: write-through-caching
- Write-behind caching — ID: write-behind-caching
- TTL and invalidation — ID: ttl-and-invalidation
- Cache stampede protection — ID: cache-stampede-protection
- Cache eviction policies — ID: cache-eviction-policies
- Negative caching — ID: negative-caching
- Multi-level caching — ID: multi-level-caching

#### messaging (8)

- Work queues — ID: work-queues
- Publish-subscribe — ID: publish-subscribe
- Partitioned event logs — ID: partitioned-event-logs
- Consumer groups and rebalancing — ID: consumer-groups-and-rebalancing
- Delivery semantics — ID: delivery-semantics
- Dead-letter queues — ID: dead-letter-queues
- Transactional outbox — ID: transactional-outbox
- Schema evolution for events — ID: schema-evolution-for-events

#### distributed systems (8)

- CAP during partitions — ID: cap-during-partitions
- Consistency models — ID: consistency-models
- Leader-based replication — ID: leader-based-replication
- Quorum reads and writes — ID: quorum-reads-and-writes
- Consensus and Raft — ID: consensus-and-raft
- Leases and fencing tokens — ID: leases-and-fencing-tokens
- Logical clocks and causal order — ID: logical-clocks-and-causal-order
- CRDTs — ID: crdts

#### API design (8)

- Resource-oriented HTTP APIs — ID: resource-oriented-http-apis
- RPC and gRPC — ID: rpc-and-grpc
- GraphQL execution — ID: graphql-execution
- Cursor pagination — ID: cursor-pagination
- Idempotency keys — ID: idempotency-keys
- Rate limiting algorithms — ID: rate-limiting-algorithms
- API versioning and compatibility — ID: api-versioning-and-compatibility
- Authentication and authorization boundaries — ID: authentication-and-authorization-boundaries

#### async (8)

- Background job lifecycle — ID: background-job-lifecycle
- Retry, backoff and jitter — ID: retry-backoff-and-jitter
- Backpressure — ID: backpressure
- Batching and microbatching — ID: batching-and-microbatching
- Durable workflow orchestration — ID: durable-workflow-orchestration
- Sagas and compensation — ID: sagas-and-compensation
- Fanout and aggregation — ID: fanout-and-aggregation
- Distributed scheduling — ID: distributed-scheduling

#### search (8)

- Inverted indexes — ID: inverted-indexes
- Text analysis and normalization — ID: text-analysis-and-normalization
- BM25 relevance ranking — ID: bm25-relevance-ranking
- Autocomplete and prefix search — ID: autocomplete-and-prefix-search
- Faceted search — ID: faceted-search
- Vector similarity search — ID: vector-similarity-search
- Approximate nearest neighbors — ID: approximate-nearest-neighbors
- Hybrid retrieval and reranking — ID: hybrid-retrieval-and-reranking

#### media (8)

- Resumable multipart uploads — ID: resumable-multipart-uploads
- Media transcoding pipelines — ID: media-transcoding-pipelines
- Adaptive bitrate streaming — ID: adaptive-bitrate-streaming
- Content delivery networks — ID: content-delivery-networks
- Signed media access — ID: signed-media-access
- Image transformation services — ID: image-transformation-services
- Live streaming ingest — ID: live-streaming-ingest
- Content addressing and deduplication — ID: content-addressing-and-deduplication

#### realtime (8)

- WebSocket connections — ID: websocket-connections
- Server-sent events — ID: server-sent-events
- Long polling — ID: long-polling
- Presence and heartbeats — ID: presence-and-heartbeats
- Realtime fan-out routing — ID: realtime-fanout-routing
- Ordering, acknowledgments and resume — ID: ordering-acknowledgments-and-resume
- Offline-first synchronization — ID: offline-first-synchronization
- Operational transformation — ID: operational-transformation

#### reliability (8)

- SLOs and error budgets — ID: slos-and-error-budgets
- Circuit breakers — ID: circuit-breakers
- Bulkheads and isolation — ID: bulkheads-and-isolation
- Load shedding and admission control — ID: load-shedding-and-admission-control
- Timeouts, deadlines and cancellation — ID: timeouts-deadlines-and-cancellation
- Health, readiness and liveness — ID: health-readiness-and-liveness
- Disaster recovery and restore — ID: disaster-recovery-and-restore
- Graceful degradation — ID: graceful-degradation

#### observability (8)

- Metrics and golden signals — ID: metrics-and-golden-signals
- Latency histograms and percentiles — ID: latency-histograms-and-percentiles
- Structured logging — ID: structured-logging
- Distributed tracing — ID: distributed-tracing
- Telemetry sampling — ID: telemetry-sampling
- Cardinality and label design — ID: cardinality-and-label-design
- Actionable alerting and burn rates — ID: actionable-alerting-and-burn-rates
- Profiling and resource attribution — ID: profiling-and-resource-attribution

#### deployment (8)

- Container images and reproducible releases — ID: container-images-and-reproducible-releases
- Orchestration control loops — ID: orchestration-control-loops
- Rolling deployments — ID: rolling-deployments
- Blue-green deployment — ID: blue-green-deployment
- Canary releases — ID: canary-releases
- Feature flags — ID: feature-flags
- Expand-contract schema migrations — ID: expand-contract-schema-migrations
- Autoscaling control and hysteresis — ID: autoscaling-control-and-hysteresis

<a id="patterns"></a>

### Pattern gym (70 topics)

Architecture patterns organized by trigger and problem. Lessons cover implementation, technologies, trade-offs, failures, related patterns, and interview application.

Source: [patterns.json](public/content/patterns.json). Topics follow the app's category order.

#### Caching (7)

- Cache Aside — ID: cache-aside
- Read Through Cache — ID: read-through-cache
- Write Through Cache — ID: write-through-cache
- Write Behind Cache — ID: write-behind-cache
- Cache Invalidation — ID: cache-invalidation
- Stale While Revalidate — ID: stale-while-revalidate
- Single Flight — ID: single-flight

#### Data Architecture (6)

- CQRS — ID: cqrs
- Event Sourcing — ID: event-sourcing
- Change Data Capture — ID: change-data-capture
- Materialized View — ID: materialized-view
- Consistent Snapshot — ID: consistent-snapshot
- Schema Evolution — ID: schema-evolution

#### Transactions (2)

- Saga — ID: saga
- Two Phase Commit — ID: two-phase-commit

#### Messaging (4)

- Transactional Outbox — ID: transactional-outbox
- Inbox Deduplication — ID: inbox-deduplication
- Work Queue — ID: work-queue
- Partitioned Log — ID: partitioned-log

#### Reliability (5)

- Idempotency Key — ID: idempotency-key
- Retry With Jitter — ID: retry-with-jitter
- Circuit Breaker — ID: circuit-breaker
- Bulkhead — ID: bulkhead
- Hedged Requests — ID: hedged-requests

#### Coordination (5)

- Leader Election — ID: leader-election
- Distributed Lock — ID: distributed-lock
- Leases — ID: leases
- Heartbeat Failure Detection — ID: heartbeat-failure-detection
- Gossip Membership — ID: gossip-membership

#### Partitioning (3)

- Consistent Hashing — ID: consistent-hashing
- Sharding — ID: sharding
- Hot Key Mitigation — ID: hot-key-mitigation

#### Replication (4)

- Quorum Replication — ID: quorum-replication
- Geographic Replication — ID: geographic-replication
- Read Repair — ID: read-repair
- Anti Entropy — ID: anti-entropy

#### Feeds (3)

- Fanout On Write — ID: fanout-on-write
- Fanout On Read — ID: fanout-on-read
- Hybrid Fanout — ID: hybrid-fanout

#### Traffic Control (6)

- Token Bucket — ID: token-bucket
- Leaky Bucket — ID: leaky-bucket
- Fixed Window Rate Limit — ID: fixed-window-rate-limit
- Sliding Window Rate Limit — ID: sliding-window-rate-limit
- Backpressure — ID: backpressure
- Load Shedding — ID: load-shedding

#### Service Architecture (6)

- API Gateway — ID: api-gateway
- Backend For Frontend — ID: backend-for-frontend
- Service Discovery — ID: service-discovery
- Sidecar — ID: sidecar
- Aggregator — ID: aggregator
- Scatter Gather — ID: scatter-gather

#### Data Processing (2)

- Batch Processing — ID: batch-processing
- Stream Processing — ID: stream-processing

#### Realtime Delivery (3)

- WebSocket — ID: websocket
- Long Polling — ID: long-polling
- Server Sent Events — ID: server-sent-events

#### Media Delivery (2)

- CDN — ID: cdn
- Adaptive Bitrate Streaming — ID: adaptive-bitrate-streaming

#### Media Storage (3)

- Object Storage — ID: object-storage
- Multipart Upload — ID: multipart-upload
- Presigned URL — ID: presigned-url

#### Storage Internals (4)

- Write Ahead Log — ID: write-ahead-log
- MVCC — ID: mvcc
- Bloom Filter — ID: bloom-filter
- LSM Tree — ID: lsm-tree

#### Search (2)

- Inverted Index — ID: inverted-index
- Spatial Index — ID: spatial-index

#### Scheduling (3)

- Priority Queue — ID: priority-queue
- Delay Queue — ID: delay-queue
- Durable Workflow — ID: durable-workflow

<a id="drills"></a>

### Drill gym (150 topics)

Short scenarios with constraints, hints, reference reasoning, trade-offs, and follow-up questions.

Source: [drills.json](public/content/drills.json). Topics follow the app's category order.

#### Foundational (15)

- A single viral alias — ID: url-shortener-scaling
- Negative cache hides a new alias — ID: url-shortener-failure
- Two writers claim one custom alias — ID: url-shortener-correctness
- The enterprise tenant hot key — ID: rate-limiter-scaling
- Limiter store is unreachable — ID: rate-limiter-failure
- Double spending across regions — ID: rate-limiter-correctness
- Rebalancing a warm cache — ID: distributed-cache-scaling
- Mass expiry stampede — ID: distributed-cache-failure
- Stale writer resurrects a value — ID: distributed-cache-correctness
- Sequence space is exhausted — ID: id-generator-scaling
- Clock moves backwards — ID: id-generator-failure
- A worker ID is reused too soon — ID: id-generator-correctness
- Midnight job cliff — ID: job-scheduler-scaling
- Worker dies after sending a report — ID: job-scheduler-failure
- Cancel races with execution — ID: job-scheduler-correctness

#### Social (15)

- Celebrity fanout explosion — ID: social-feed-scaling
- Feed contains a deleted post — ID: social-feed-failure
- Pagination repeats ranked items — ID: social-feed-correctness
- A new thumbnail size goes viral — ID: photo-sharing-scaling
- Incomplete image marked ready — ID: photo-sharing-failure
- Private image survives album removal — ID: photo-sharing-correctness
- A celebrity has a huge follower partition — ID: follow-graph-scaling
- Follower counts become negative — ID: follow-graph-failure
- Blocked user remains in a stale projection — ID: follow-graph-correctness
- One event dominates replies — ID: threaded-comments-scaling
- Moderation disappears after an edit retry — ID: threaded-comments-failure
- Deleting a parent orphans discussion — ID: threaded-comments-correctness
- Viewer list overwhelms one story — ID: ephemeral-stories-scaling
- Expired story remains playable — ID: ephemeral-stories-failure
- Audience changes during viewing — ID: ephemeral-stories-correctness

#### Messaging (15)

- One busy group overloads its owner — ID: chat-scaling
- Messages vanish after reconnect — ID: chat-failure
- A removed member receives delayed messages — ID: chat-correctness
- Marketing consumes transactional capacity — ID: notification-platform-scaling
- SMS provider accepted an ambiguous request — ID: notification-platform-failure
- Opt-out races with a queued campaign — ID: notification-platform-correctness
- Large attachments dominate ingress — ID: email-service-scaling
- Accepted mail disappears during a crash — ID: email-service-failure
- Folder moves race with flag updates — ID: email-service-correctness
- One slow endpoint consumes workers — ID: webhooks-scaling
- Receiver processed the event but returned 500 — ID: webhooks-failure
- A newer update arrives before an older retry — ID: webhooks-correctness
- Consumer fanout multiplies bandwidth — ID: pubsub-scaling
- Consumer is behind retention — ID: pubsub-failure
- A database sink duplicates increments — ID: pubsub-correctness

#### Media (15)

- A viral upload is cold at every edge — ID: youtube-scaling
- A manifest advertises missing segments — ID: youtube-failure
- A transcode worker retries after lease loss — ID: youtube-correctness
- Final-minute sports traffic surge — ID: live-streaming-scaling
- Encoder restart resets timestamps — ID: live-streaming-failure
- Two ingests believe they own a stream — ID: live-streaming-correctness
- A new album creates a cache cliff — ID: music-streaming-scaling
- Rights change but cached playback remains authorized — ID: music-streaming-failure
- Two devices overwrite playlist edits — ID: music-streaming-correctness
- Large town hall exhausts SFU egress — ID: video-conference-scaling
- UDP is blocked on a corporate network — ID: video-conference-failure
- A removed participant reconnects with an old token — ID: video-conference-correctness
- Polling every feed wastes most requests — ID: podcast-platform-scaling
- A publisher reuses an episode GUID — ID: podcast-platform-failure
- Progress regresses after offline sync — ID: podcast-platform-correctness

#### Storage (15)

- A shared folder has a million small files — ID: file-sync-scaling
- A partial upload becomes the current version — ID: file-sync-failure
- Two offline devices modify the same file — ID: file-sync-correctness
- Repair competes with serving traffic — ID: object-store-scaling
- Silent corruption passes through replication — ID: object-store-failure
- Concurrent overwrites return mixed fragments — ID: object-store-correctness
- Restore bandwidth misses the promised RTO — ID: backup-restore-scaling
- Backup is green but missing a log segment — ID: backup-restore-failure
- Retention cleanup removes an active restore dependency — ID: backup-restore-correctness
- One shared blob has a hot reference counter — ID: blob-dedup-scaling
- Garbage collector races with a new reference — ID: blob-dedup-failure
- A guessed digest reveals private content — ID: blob-dedup-correctness
- Small files overwhelm namespace memory — ID: distributed-filesystem-scaling
- A stale client writes after lease transfer — ID: distributed-filesystem-failure
- Rename crosses metadata shards — ID: distributed-filesystem-correctness

#### Commerce (15)

- A sale overwhelms inventory reservation — ID: checkout-scaling
- Payment succeeded but the order timed out — ID: checkout-failure
- A reservation expires as payment completes — ID: checkout-correctness
- Hot SKU saturates one row lock — ID: inventory-scaling
- Expiry and checkout both decrement reservations — ID: inventory-failure
- Warehouse event is delivered twice — ID: inventory-correctness
- A settlement account becomes a hot row — ID: payment-ledger-scaling
- Retry posts the same transfer twice — ID: payment-ledger-failure
- Currency rounding breaks ledger balance — ID: payment-ledger-correctness
- Seat-map refresh melts the backend — ID: ticket-booking-scaling
- A timer releases a newly confirmed seat — ID: ticket-booking-failure
- Two groups partially acquire adjacent seats — ID: ticket-booking-correctness
- A slow seller blocks every order item — ID: marketplace-scaling
- Shipment callback moves an item backwards — ID: marketplace-failure
- Refund and payout race — ID: marketplace-correctness

#### Real time (15)

- Airport arrivals overload one map cell — ID: ride-hailing-scaling
- Stale locations produce impossible pickups — ID: ride-hailing-failure
- Two riders get the same driver — ID: ride-hailing-correctness
- A giant document has years of operations — ID: collaborative-editor-scaling
- Concurrent insertions diverge across clients — ID: collaborative-editor-failure
- Revoked editor uploads offline operations — ID: collaborative-editor-correctness
- Reconnect storm floods heartbeats — ID: presence-scaling
- User remains online after a silent disconnect — ID: presence-failure
- Old disconnect clears a new session — ID: presence-correctness
- One global sorted set runs out of headroom — ID: leaderboard-scaling
- Replayed score event doubles a win — ID: leaderboard-failure
- Late scores cross a season boundary — ID: leaderboard-correctness
- A map zoom returns every device — ID: fleet-tracking-scaling
- GPS jitter creates repeated enter-exit alerts — ID: fleet-tracking-failure
- Offline backlog overwrites the latest position — ID: fleet-tracking-correctness

#### Search (15)

- Common query fans out to every shard — ID: web-search-scaling
- Crawler falls into an infinite calendar — ID: web-search-failure
- Deleted pages remain in search results — ID: web-search-correctness
- Every keystroke becomes a server request — ID: autocomplete-scaling
- A blocked suggestion persists in cached lists — ID: autocomplete-failure
- Personalized cache leaks another user’s history — ID: autocomplete-correctness
- High-cardinality facets blow query memory — ID: product-search-scaling
- Search advertises yesterday’s price — ID: product-search-failure
- Backfill overwrites a newer product update — ID: product-search-correctness
- One incident causes a logging storm — ID: log-search-scaling
- Mapping explosion takes down indexing — ID: log-search-failure
- Late logs fall into an already-expired partition — ID: log-search-correctness
- Restrictive filters return too few neighbors — ID: vector-retrieval-scaling
- Embedding upgrade mixes incompatible spaces — ID: vector-retrieval-failure
- Private document appears in retrieved context — ID: vector-retrieval-correctness

#### Infrastructure (15)

- Failover overwhelms the healthy region — ID: load-balancer-scaling
- Health checks remove every backend — ID: load-balancer-failure
- Draining drops long-lived connections — ID: load-balancer-correctness
- A tiny zone receives a massive query flood — ID: dns-service-scaling
- Bad zone version reaches every region — ID: dns-service-failure
- DNS rollback appears ineffective — ID: dns-service-correctness
- Request IDs create millions of new series — ID: metrics-platform-scaling
- Alert evaluator fails silently — ID: metrics-platform-failure
- Missing samples are treated as zero — ID: metrics-platform-correctness
- Remote evaluation adds a dependency to every request — ID: feature-flags-scaling
- Bad rollout config reaches all clients — ID: feature-flags-failure
- Users jump between rollout cohorts — ID: feature-flags-correctness
- Deployment generates a watch storm — ID: service-discovery-scaling
- Watcher resumes from a compacted revision — ID: service-discovery-failure
- Two leaders push conflicting configuration — ID: service-discovery-correctness

#### Advanced (15)

- A hot tenant dominates one partition — ID: multi-region-kv-scaling
- Async failover loses acknowledged recent writes — ID: multi-region-kv-failure
- Concurrent regions overwrite each other — ID: multi-region-kv-correctness
- Long histories make every decision expensive — ID: workflow-engine-scaling
- Deployment makes replay nondeterministic — ID: workflow-engine-failure
- Compensation itself fails — ID: workflow-engine-correctness
- A global campaign hot-spots budget checks — ID: ad-auction-scaling
- Delayed impressions overshoot the budget — ID: ad-auction-failure
- Duplicate impression events bill twice — ID: ad-auction-correctness
- One slow feature source consumes the latency budget — ID: recommendation-platform-scaling
- Training and serving use different feature definitions — ID: recommendation-platform-failure
- Experiment exposure is counted before delivery — ID: recommendation-platform-correctness
- One shared IP becomes a hot aggregation key — ID: fraud-stream-scaling
- Processor restarts and forgets window state — ID: fraud-stream-failure
- Late events change a decision after funds moved — ID: fraud-stream-correctness

<a id="systems"></a>

### System workouts (50 topics)

Full design workouts: Requirements, Scale, API, Data Model, Architecture, Deep Dives, Scaling, Failures, Trade-offs, and Interview Questions. YouTube is the featured workout.

Source: [systems.json](public/content/systems.json). Topics follow the app's category order.

#### Foundational (5)

- URL shortener — ID: url-shortener
- Distributed rate limiter — ID: rate-limiter
- Distributed cache — ID: distributed-cache
- Unique ID service — ID: id-generator
- Durable job scheduler — ID: job-scheduler

#### Social (5)

- Social news feed — ID: social-feed
- Photo sharing — ID: photo-sharing
- Follow graph — ID: follow-graph
- Threaded comments — ID: threaded-comments
- Ephemeral stories — ID: ephemeral-stories

#### Messaging (5)

- Direct and group chat — ID: chat
- Notification platform — ID: notification-platform
- Email service — ID: email-service
- Webhook delivery — ID: webhooks
- Publish-subscribe event bus — ID: pubsub

#### Media (5)

- YouTube-style video platform — ID: youtube
- Live streaming platform — ID: live-streaming
- Music streaming — ID: music-streaming
- Video conferencing — ID: video-conference
- Podcast platform — ID: podcast-platform

#### Storage (5)

- Cloud file synchronization — ID: file-sync
- Distributed object storage — ID: object-store
- Backup and point-in-time restore — ID: backup-restore
- Content-addressed blob store — ID: blob-dedup
- Distributed filesystem — ID: distributed-filesystem

#### Commerce (5)

- Commerce checkout — ID: checkout
- Inventory reservation — ID: inventory
- Payment ledger — ID: payment-ledger
- Ticket booking — ID: ticket-booking
- Marketplace orders — ID: marketplace

#### Real time (5)

- Ride dispatch — ID: ride-hailing
- Collaborative document editor — ID: collaborative-editor
- Presence and typing indicators — ID: presence
- Game leaderboard — ID: leaderboard
- Fleet tracking — ID: fleet-tracking

#### Search (5)

- Web search engine — ID: web-search
- Search autocomplete — ID: autocomplete
- Product search — ID: product-search
- Log ingestion and search — ID: log-search
- Vector retrieval service — ID: vector-retrieval

#### Infrastructure (5)

- Global load balancer — ID: load-balancer
- Authoritative DNS — ID: dns-service
- Metrics and alerting — ID: metrics-platform
- Feature flag platform — ID: feature-flags
- Service discovery and configuration — ID: service-discovery

#### Advanced (5)

- Multi-region key-value store — ID: multi-region-kv
- Durable workflow engine — ID: workflow-engine
- Ad auction and budget pacing — ID: ad-auction
- Recommendation serving — ID: recommendation-platform
- Streaming fraud detection — ID: fraud-stream

<a id="tradeoffs"></a>

### Trade-off gym (54 topics)

Paired choices analyzed by latency, throughput, consistency, complexity, cost, and failure behavior.

Source: [tradeoffs.json](public/content/tradeoffs.json). Topics follow the app's category order.

#### Storage (2)

- Relational Store vs Key-Value Store — ID: relational-store-vs-key-value-store
- Row Store vs Column Store — ID: row-store-vs-column-store

#### Capacity (1)

- Scale Up vs Scale Out — ID: scale-up-vs-scale-out

#### Service Architecture (2)

- Modular Monolith vs Microservices — ID: modular-monolith-vs-microservices
- Shared Database vs Database Per Service — ID: shared-database-vs-database-per-service

#### Communication (2)

- Synchronous RPC vs Asynchronous Messaging — ID: synchronous-rpc-vs-asynchronous-messaging
- REST JSON vs gRPC Protobuf — ID: rest-json-vs-grpc-protobuf

#### API Design (2)

- REST Resources vs GraphQL — ID: rest-resources-vs-graphql
- Offset vs Cursor Pagination — ID: offset-vs-cursor-pagination

#### Realtime Delivery (2)

- Server Sent Events vs WebSocket — ID: server-sent-events-vs-websocket
- Short Polling vs Push Updates — ID: short-polling-vs-push-updates

#### Caching (5)

- Cache Aside vs Read Through — ID: cache-aside-vs-read-through
- Write Through vs Write Behind — ID: write-through-vs-write-behind
- TTL Expiry vs Event Invalidation — ID: ttl-expiry-vs-event-invalidation
- Local Cache vs Shared Remote Cache — ID: local-cache-vs-shared-remote-cache
- LRU vs LFU Eviction — ID: lru-vs-lfu-eviction

#### Consistency (1)

- Strong vs Eventual Read Consistency — ID: strong-vs-eventual-read-consistency

#### Replication (2)

- Synchronous vs Asynchronous Replication — ID: synchronous-vs-asynchronous-replication
- Single Writer vs Multiple Writers — ID: single-writer-vs-multiple-writers

#### Partitioning (3)

- Hash Sharding vs Range Sharding — ID: hash-sharding-vs-range-sharding
- Tenant Shard Key vs Entity Shard Key — ID: tenant-shard-key-vs-entity-shard-key
- Consistent Hash Ring vs Directory Placement — ID: consistent-hash-ring-vs-directory-placement

#### Feeds (1)

- Fanout On Write vs Fanout On Read — ID: fanout-on-write-vs-fanout-on-read

#### Messaging (2)

- Work Queue vs Replayable Log — ID: work-queue-vs-replayable-log
- At-Most-Once vs At-Least-Once Processing — ID: at-most-once-vs-at-least-once-processing

#### Data Integration (2)

- CDC vs Table Polling — ID: cdc-vs-table-polling
- ETL vs ELT — ID: etl-vs-elt

#### Transactions (4)

- Transactional Outbox vs Direct Dual Write — ID: transactional-outbox-vs-direct-dual-write
- Saga vs Two Phase Commit — ID: saga-vs-two-phase-commit
- Saga Orchestration vs Choreography — ID: saga-orchestration-vs-choreography
- Optimistic vs Pessimistic Concurrency — ID: optimistic-vs-pessimistic-concurrency

#### Data Architecture (2)

- CQRS vs One Read-Write Model — ID: cqrs-vs-one-read-write-model
- Event Sourcing vs Current-State Storage — ID: event-sourcing-vs-current-state-storage

#### Coordination (1)

- Distributed Lock vs Conditional Update — ID: distributed-lock-vs-conditional-update

#### Data Processing (1)

- Batch vs Stream Processing — ID: batch-vs-stream-processing

#### Storage Internals (1)

- B-Tree vs LSM Storage — ID: b-tree-vs-lsm-storage

#### Data Modeling (2)

- Normalize vs Denormalize — ID: normalize-vs-denormalize
- Document Model vs Relational Model — ID: document-model-vs-relational-model

#### Traffic Control (4)

- Token Bucket vs Leaky Bucket — ID: token-bucket-vs-leaky-bucket
- Fixed vs Sliding Rate Window — ID: fixed-vs-sliding-rate-window
- Queue Excess vs Reject Excess — ID: queue-excess-vs-reject-excess
- Local vs Global Rate Limit — ID: local-vs-global-rate-limit

#### Reliability (1)

- Retry vs Hedge — ID: retry-vs-hedge

#### Media Storage (2)

- Object Storage vs Database Blobs — ID: object-storage-vs-database-blobs
- Direct Signed Upload vs API Proxy Upload — ID: direct-signed-upload-vs-api-proxy-upload

#### Media Delivery (1)

- Short vs Long Video Segments — ID: short-vs-long-video-segments

#### Media Processing (1)

- Precompute Renditions vs Just-In-Time Encoding — ID: precompute-renditions-vs-just-in-time-encoding

#### Global Architecture (1)

- Active-Passive vs Active-Active Regions — ID: active-passive-vs-active-active-regions

#### Operations (1)

- Managed Service vs Self-Hosted Infrastructure — ID: managed-service-vs-self-hosted-infrastructure

#### Compute (1)

- Serverless Functions vs Long-Running Containers — ID: serverless-functions-vs-long-running-containers

#### Analytics (1)

- Exact Count vs Approximate Sketch — ID: exact-count-vs-approximate-sketch

#### Scheduling (2)

- FIFO vs Priority Scheduling — ID: fifo-vs-priority-scheduling
- Process Timer vs Durable Timer — ID: process-timer-vs-durable-timer

#### Networking (1)

- Compress Payloads vs Send Uncompressed — ID: compress-payloads-vs-send-uncompressed

<a id="failures"></a>

### Failure gym (50 topics)

Incidents analyzed through impact, detection, immediate mitigation, prevention, and staff-level recovery.

Source: [failures.json](public/content/failures.json). Topics follow the app's category order.

#### Foundational (5)

- Negative cache hides a new alias — ID: url-shortener-incident
- Limiter store is unreachable — ID: rate-limiter-incident
- Mass expiry stampede — ID: distributed-cache-incident
- Clock moves backwards — ID: id-generator-incident
- Worker dies after sending a report — ID: job-scheduler-incident

#### Social (5)

- Feed contains a deleted post — ID: social-feed-incident
- Incomplete image marked ready — ID: photo-sharing-incident
- Follower counts become negative — ID: follow-graph-incident
- Moderation disappears after an edit retry — ID: threaded-comments-incident
- Expired story remains playable — ID: ephemeral-stories-incident

#### Messaging (5)

- Messages vanish after reconnect — ID: chat-incident
- SMS provider accepted an ambiguous request — ID: notification-platform-incident
- Accepted mail disappears during a crash — ID: email-service-incident
- Receiver processed the event but returned 500 — ID: webhooks-incident
- Consumer is behind retention — ID: pubsub-incident

#### Media (5)

- A manifest advertises missing segments — ID: youtube-incident
- Encoder restart resets timestamps — ID: live-streaming-incident
- Rights change but cached playback remains authorized — ID: music-streaming-incident
- UDP is blocked on a corporate network — ID: video-conference-incident
- A publisher reuses an episode GUID — ID: podcast-platform-incident

#### Storage (5)

- A partial upload becomes the current version — ID: file-sync-incident
- Silent corruption passes through replication — ID: object-store-incident
- Backup is green but missing a log segment — ID: backup-restore-incident
- Garbage collector races with a new reference — ID: blob-dedup-incident
- A stale client writes after lease transfer — ID: distributed-filesystem-incident

#### Commerce (5)

- Payment succeeded but the order timed out — ID: checkout-incident
- Expiry and checkout both decrement reservations — ID: inventory-incident
- Retry posts the same transfer twice — ID: payment-ledger-incident
- A timer releases a newly confirmed seat — ID: ticket-booking-incident
- Shipment callback moves an item backwards — ID: marketplace-incident

#### Real time (5)

- Stale locations produce impossible pickups — ID: ride-hailing-incident
- Concurrent insertions diverge across clients — ID: collaborative-editor-incident
- User remains online after a silent disconnect — ID: presence-incident
- Replayed score event doubles a win — ID: leaderboard-incident
- GPS jitter creates repeated enter-exit alerts — ID: fleet-tracking-incident

#### Search (5)

- Crawler falls into an infinite calendar — ID: web-search-incident
- A blocked suggestion persists in cached lists — ID: autocomplete-incident
- Search advertises yesterday’s price — ID: product-search-incident
- Mapping explosion takes down indexing — ID: log-search-incident
- Embedding upgrade mixes incompatible spaces — ID: vector-retrieval-incident

#### Infrastructure (5)

- Health checks remove every backend — ID: load-balancer-incident
- Bad zone version reaches every region — ID: dns-service-incident
- Alert evaluator fails silently — ID: metrics-platform-incident
- Bad rollout config reaches all clients — ID: feature-flags-incident
- Watcher resumes from a compacted revision — ID: service-discovery-incident

#### Advanced (5)

- Async failover loses acknowledged recent writes — ID: multi-region-kv-incident
- Deployment makes replay nondeterministic — ID: workflow-engine-incident
- Delayed impressions overshoot the budget — ID: ad-auction-incident
- Training and serving use different feature definitions — ID: recommendation-platform-incident
- Processor restarts and forgets window state — ID: fraud-stream-incident

<a id="capacity"></a>

### Capacity gym (50 topics)

Numeric sizing scenarios: requests, average and peak QPS, bandwidth, retained storage, cache effects, partitions, and server lower bounds.

Source: [capacity.json](public/content/capacity.json). Topics follow the app's category order.

#### Foundational (5)

- URL shortener workload — ID: url-shortener-capacity
- Distributed rate limiter workload — ID: rate-limiter-capacity
- Distributed cache workload — ID: distributed-cache-capacity
- Unique ID service workload — ID: id-generator-capacity
- Durable job scheduler workload — ID: job-scheduler-capacity

#### Social (5)

- Social news feed workload — ID: social-feed-capacity
- Photo sharing workload — ID: photo-sharing-capacity
- Follow graph workload — ID: follow-graph-capacity
- Threaded comments workload — ID: threaded-comments-capacity
- Ephemeral stories workload — ID: ephemeral-stories-capacity

#### Messaging (5)

- Direct and group chat workload — ID: chat-capacity
- Notification platform workload — ID: notification-platform-capacity
- Email service workload — ID: email-service-capacity
- Webhook delivery workload — ID: webhooks-capacity
- Publish-subscribe event bus workload — ID: pubsub-capacity

#### Media (5)

- YouTube-style video platform workload — ID: youtube-capacity
- Live streaming platform workload — ID: live-streaming-capacity
- Music streaming workload — ID: music-streaming-capacity
- Video conferencing workload — ID: video-conference-capacity
- Podcast platform workload — ID: podcast-platform-capacity

#### Storage (5)

- Cloud file synchronization workload — ID: file-sync-capacity
- Distributed object storage workload — ID: object-store-capacity
- Backup and point-in-time restore workload — ID: backup-restore-capacity
- Content-addressed blob store workload — ID: blob-dedup-capacity
- Distributed filesystem workload — ID: distributed-filesystem-capacity

#### Commerce (5)

- Commerce checkout workload — ID: checkout-capacity
- Inventory reservation workload — ID: inventory-capacity
- Payment ledger workload — ID: payment-ledger-capacity
- Ticket booking workload — ID: ticket-booking-capacity
- Marketplace orders workload — ID: marketplace-capacity

#### Real time (5)

- Ride dispatch workload — ID: ride-hailing-capacity
- Collaborative document editor workload — ID: collaborative-editor-capacity
- Presence and typing indicators workload — ID: presence-capacity
- Game leaderboard workload — ID: leaderboard-capacity
- Fleet tracking workload — ID: fleet-tracking-capacity

#### Search (5)

- Web search engine workload — ID: web-search-capacity
- Search autocomplete workload — ID: autocomplete-capacity
- Product search workload — ID: product-search-capacity
- Log ingestion and search workload — ID: log-search-capacity
- Vector retrieval service workload — ID: vector-retrieval-capacity

#### Infrastructure (5)

- Global load balancer workload — ID: load-balancer-capacity
- Authoritative DNS workload — ID: dns-service-capacity
- Metrics and alerting workload — ID: metrics-platform-capacity
- Feature flag platform workload — ID: feature-flags-capacity
- Service discovery and configuration workload — ID: service-discovery-capacity

#### Advanced (5)

- Multi-region key-value store workload — ID: multi-region-kv-capacity
- Durable workflow engine workload — ID: workflow-engine-capacity
- Ad auction and budget pacing workload — ID: ad-auction-capacity
- Recommendation serving workload — ID: recommendation-platform-capacity
- Streaming fraud detection workload — ID: fraud-stream-capacity

<a id="database"></a>

### Database gym (50 topics)

Storage choices based on workload, access patterns, consistency, indexes, partitioning, and operational constraints.

Source: [database.json](public/content/database.json). Topics follow the app's category order.

#### Foundational (5)

- Storage for url shortener — ID: url-shortener-storage
- Storage for distributed rate limiter — ID: rate-limiter-storage
- Storage for distributed cache — ID: distributed-cache-storage
- Storage for unique id service — ID: id-generator-storage
- Storage for durable job scheduler — ID: job-scheduler-storage

#### Social (5)

- Storage for social news feed — ID: social-feed-storage
- Storage for photo sharing — ID: photo-sharing-storage
- Storage for follow graph — ID: follow-graph-storage
- Storage for threaded comments — ID: threaded-comments-storage
- Storage for ephemeral stories — ID: ephemeral-stories-storage

#### Messaging (5)

- Storage for direct and group chat — ID: chat-storage
- Storage for notification platform — ID: notification-platform-storage
- Storage for email service — ID: email-service-storage
- Storage for webhook delivery — ID: webhooks-storage
- Storage for publish-subscribe event bus — ID: pubsub-storage

#### Media (5)

- Storage for youtube-style video platform — ID: youtube-storage
- Storage for live streaming platform — ID: live-streaming-storage
- Storage for music streaming — ID: music-streaming-storage
- Storage for video conferencing — ID: video-conference-storage
- Storage for podcast platform — ID: podcast-platform-storage

#### Storage (5)

- Storage for cloud file synchronization — ID: file-sync-storage
- Storage for distributed object storage — ID: object-store-storage
- Storage for backup and point-in-time restore — ID: backup-restore-storage
- Storage for content-addressed blob store — ID: blob-dedup-storage
- Storage for distributed filesystem — ID: distributed-filesystem-storage

#### Commerce (5)

- Storage for commerce checkout — ID: checkout-storage
- Storage for inventory reservation — ID: inventory-storage
- Storage for payment ledger — ID: payment-ledger-storage
- Storage for ticket booking — ID: ticket-booking-storage
- Storage for marketplace orders — ID: marketplace-storage

#### Real time (5)

- Storage for ride dispatch — ID: ride-hailing-storage
- Storage for collaborative document editor — ID: collaborative-editor-storage
- Storage for presence and typing indicators — ID: presence-storage
- Storage for game leaderboard — ID: leaderboard-storage
- Storage for fleet tracking — ID: fleet-tracking-storage

#### Search (5)

- Storage for web search engine — ID: web-search-storage
- Storage for search autocomplete — ID: autocomplete-storage
- Storage for product search — ID: product-search-storage
- Storage for log ingestion and search — ID: log-search-storage
- Storage for vector retrieval service — ID: vector-retrieval-storage

#### Infrastructure (5)

- Storage for global load balancer — ID: load-balancer-storage
- Storage for authoritative dns — ID: dns-service-storage
- Storage for metrics and alerting — ID: metrics-platform-storage
- Storage for feature flag platform — ID: feature-flags-storage
- Storage for service discovery and configuration — ID: service-discovery-storage

#### Advanced (5)

- Storage for multi-region key-value store — ID: multi-region-kv-storage
- Storage for durable workflow engine — ID: workflow-engine-storage
- Storage for ad auction and budget pacing — ID: ad-auction-storage
- Storage for recommendation serving — ID: recommendation-platform-storage
- Storage for streaming fraud detection — ID: fraud-stream-storage

<a id="recognition"></a>

### Pattern recognition (210 scenarios)

Three rounds over all 70 Pattern gym triggers: direct requirement, 10× growth, and degraded regional dependency. The learner selects a pattern and defends it. The covered patterns are listed under Pattern gym.

<a id="mock"></a>

### Mock interviews (50 selectable systems)

Any of the 50 System workouts may be selected for a strict 45-minute mock. Phases: Requirements, Scale, API and data, Architecture, Deep dive, Scaling and failures, Trade-offs and summary. Debrief dimensions: Requirements, Capacity estimation, Architecture, Data modeling, Scalability, Reliability, Trade-offs, Communication, Depth, and Staff-level thinking.

<a id="hostile"></a>

### Interviewer sparring (50 selectable systems)

Interviewer sparring uses all 50 System workouts. Follow-ups challenge Kafka versus SQS, cache outage, database failover, hot partition, WebSocket reconnects, duplicate queue delivery, region isolation, idempotency, 10× scaling, cost, ownership, migration, and observability.

<a id="principles"></a>

### First principles (50 selectable systems)

First-principles practice starts with telemetry from 20 million devices every five seconds. Derive ingestion, partitioning, streaming, storage, aggregation, queries, retention, and monitoring. The same 50 systems are selectable for further rounds.

<a id="whiteboard"></a>

### Whiteboard

Free-form Excalidraw architecture canvas. Presets: Service, Users, Load balancer, Database, Queue, Cache, CDN, Region, Annotation. Its shapes, arrows, drawing, labels, and saved diagrams can be used for any subject; no fixed lesson list.

## Your playbook

<a id="notes"></a>

### Notes & sketches

Personal handwritten architecture sketches and image notes. Images may be attached to a topic or kept as general notes. The content depends on what the learner saves.

<a id="plan"></a>

### Training plan (6 phases)

Four-, eight-, twelve-, or twenty-four-week plans share six training phases shown below. Weekly links lead into concepts, patterns, micro drills, systems, competition, and recovery.
- **Build your foundation:** Foundations and networking; latency, throughput, consistency, CAP, load balancing; URL shortener and rate limiter.
- **Own the data path:** Indexes, replication, sharding, caching, Kafka delivery; chat and notifications.
- **Recognize the architecture:** Consensus, asynchronous workflows, CQRS, sagas, fan-out, scatter-gather, materialized views; news feed and payments.
- **Design complete systems:** Media delivery, search, geospatial indexing, multipart upload, CDN, adaptive bitrate; YouTube, Uber, web search.
- **Find the breaking point:** Multi-region consistency, recovery, SLOs, observability, bulkheads, circuit breakers, backpressure.
- **Perform under pressure:** Migration, ownership, cost, governance, architecture alternatives, Staff+ mock interviews.

<a id="progress"></a>

### Performance

Practice analytics: readiness, completed drills, applicable patterns, scored mocks, fourteen-day activity, recent mistakes, weak areas, and suggested retraining. Topics depend on the learner's history.

<a id="journal"></a>

### Training journal

Six reflection prompts shown below. Entries are the learner's own, with no fixed topic list.
- What did I learn?
- What did I forget?
- Where did I struggle?
- What surprised me?
- What should I repeat?
- What is tomorrow's priority?

<a id="recipes"></a>

### Design recipes (24 topics)

Reusable end-to-end design flows with brief, scale, decisions, deep dives, failures, and staff-level discussion.

Source: [recipes.json](public/content/recipes.json). Topics follow the app's category order.

#### Consumer Systems (1)

- Read-Heavy URL Shortener — ID: read-heavy-url-shortener

#### Commerce (4)

- Global Product Catalog — ID: global-product-catalog
- Reliable Checkout — ID: reliable-checkout
- Flash Sale Inventory — ID: flash-sale-inventory
- Ticket Reservation — ID: ticket-reservation

#### Social Systems (1)

- Hybrid Social Feed — ID: hybrid-social-feed

#### Realtime Systems (2)

- Durable Chat — ID: durable-chat
- Live Collaboration — ID: live-collaboration

#### Media Systems (2)

- Video On Demand — ID: video-on-demand
- Live Video Broadcast — ID: live-video-broadcast

#### Storage Systems (1)

- Resumable File Drive — ID: resumable-file-drive

#### Search Systems (1)

- Searchable Knowledge Base — ID: searchable-knowledge-base

#### Geospatial Systems (1)

- Location-Based Ride Matching — ID: location-based-ride-matching

#### Messaging Systems (2)

- Notification Platform — ID: notification-platform
- Webhook Delivery Platform — ID: webhook-delivery-platform

#### Platform Systems (3)

- Multi-Tenant Rate Limiter — ID: multi-tenant-rate-limiter
- Durable Job Scheduler — ID: durable-job-scheduler
- Public API With Async Exports — ID: public-api-with-async-exports

#### Data Systems (2)

- Real-Time Analytics Dashboard — ID: real-time-analytics-dashboard
- Data Warehouse Ingestion — ID: data-warehouse-ingestion

#### Observability Systems (1)

- Metrics And Alerting — ID: metrics-and-alerting

#### Financial Systems (1)

- Transactional Ledger — ID: transactional-ledger

#### Global Systems (1)

- Multi-Region SaaS — ID: multi-region-saas

#### Decision Systems (1)

- Fraud Decision Pipeline — ID: fraud-decision-pipeline

<a id="favorites"></a>

### Bookmarks

The learner's saved items from the libraries and Question bank. This has no predefined topics.

<a id="questions"></a>

### Question bank (929 prompts)

Generated from Curriculum lesson questions and System workout questions. The question set is listed below and does not add separate concepts beyond those source sections.

The 729 Curriculum questions and 200 System workout questions are grouped by their source topic below. Each bullet is a prompt shown in the Question bank.

#### Curriculum (729 prompts)

##### foundations

- **Requirements and invariants** (7 prompts)
  - What is an invariant, and how is it different from a requirement?
  - Give an SLO for a checkout endpoint and explain each part.
  - Which requirements force synchronous coordination, and which do not?
  - A product manager says the system must be “highly available and strongly consistent.” How do you respond?
  - Who owns exceptions during a regional outage, and how do you decide in advance?
  - Your invariant now spans two services after a team split. What do you do?
  - How would you negotiate incompatible availability and allocation guarantees?
- **Capacity estimation** (7 prompts)
  - Estimate the storage for 1M new posts per day at 2 KB each, kept for 3 years with 3× replication.
  - Convert 1 billion requests per day into QPS, and give the peak.
  - Which single assumption usually dominates a capacity estimate, and how do you defend it?
  - Your estimate says one database node is enough, but production falls over at half the predicted load. What do you check?
  - What changes when you add 3× replication and a 10:1 read/write ratio?
  - How would you size a system for a flash sale with a 1000× peak lasting 90 seconds?
  - How do you make capacity estimation an organisational practice rather than a one-off exercise?
- **Latency versus throughput** (7 prompts)
  - A service handles 1,000 requests per second with 50 ms latency. How many requests are in flight?
  - Why does adding a second server sometimes not improve latency at all?
  - Your p50 is 20 ms and p99 is 2 seconds. Where do you look?
  - How do you choose a batch size for an async write pipeline?
  - At what utilisation would you run a user-facing service, and why?
  - Throughput is at an all-time high and users are complaining. Walk me through the diagnosis.
  - How do you get an organisation to stop optimising throughput at the expense of latency?
- **Little's law** (7 prompts)
  - State Little's law and size a connection pool with it.
  - A queue has 10,000 messages and consumers process 500 per second. How long to drain?
  - A dependency's latency goes from 20 ms to 200 ms at constant traffic. What happens?
  - How would you use Little's law to choose a concurrency limit rather than a rate limit?
  - Where does Little's law stop being useful, and what do you use instead?
  - Design the overload behaviour of a service using this law.
  - How do you make in-flight concurrency a first-class concept across many teams?
- **Tail latency** (7 prompts)
  - A service has a 1% chance of being slow. A page calls it 50 times in parallel. How often is the page slow?
  - What is a hedged request and when is it safe?
  - Your p50 is 15 ms and p99 is 900 ms. How do you find the cause?
  - How do you set a timeout, and how does it interact with retries?
  - Design tail-latency control for a page that fans out to 40 services.
  - When would you deliberately not fix the tail?
  - How do you make tail latency a fleet-wide property rather than a per-team crusade?
- **Horizontal and vertical scaling** (7 prompts)
  - What is the difference between scaling up and scaling out?
  - Why can't you always scale out?
  - When would you choose vertical scaling for a database?
  - A team added ten application servers and latency got worse. Why?
  - How do you decide when it is finally time to shard?
  - Your CFO asks why you run three instances when one is enough. What do you say?
  - How would you set organisational policy on scaling decisions?
- **Stateless service design** (7 prompts)
  - What makes a service stateless, and why does it matter?
  - Is an in-memory cache allowed in a stateless service?
  - Compare JWTs and server-side sessions for a consumer web app.
  - How would you migrate a sticky-session service to stateless with no downtime?
  - Which components in a typical architecture should not be stateless, and how do you handle them?
  - You externalised sessions and now Redis is a single point of failure. What now?
  - How do you prevent hidden state from creeping back in across many teams?
- **Partitioning by ownership** (7 prompts)
  - What does “ownership” mean in a partitioned system?
  - Why is consistent hashing preferred over hashing modulo the node count?
  - How do you choose the partition key?
  - What is a fencing token and why is it required?
  - Your partitioning scheme has a hot key taking 40% of one partition's capacity. Options?
  - Design ownership for a multi-tenant SaaS with tenants varying 1000× in size.
  - When is partitioning by ownership the wrong architecture entirely?

##### networking

- **DNS resolution and caching** (6 prompts)
  - Walk through what happens when a client resolves `api.example.com`.
  - What does TTL actually guarantee?
  - Why is DNS a poor mechanism for fast failover, and what do you use instead?
  - What does a low TTL actually cost, and when is it worth paying?
  - Design regional failover for a service where DNS is the only cross-region mechanism available.
  - How do you reduce DNS as a company-wide single point of failure?
- **TCP reliability and congestion** (6 prompts)
  - How does TCP provide reliability?
  - Why is a new TCP connection slow for small responses?
  - A transfer between regions gets 6 Mbps on a 10 Gbps link. Diagnose it.
  - Why does packet loss hurt throughput so disproportionately?
  - When would you choose QUIC over TCP, and what do you give up?
  - How do you decide whether to invest in transport tuning at all?
- **TLS termination** (6 prompts)
  - What is TLS termination?
  - When would you choose passthrough over termination at the load balancer?
  - How do you preserve client identity after terminating TLS?
  - What does terminating TLS at the edge cost you?
  - Design the TLS topology for a system handling payment data.
  - How do you move an organisation from perimeter trust to zero trust without an outage?
- **Layer 4 and layer 7 load balancing** (6 prompts)
  - What is the difference between L4 and L7 load balancing?
  - Why does L4 balancing behave badly in front of gRPC?
  - How should a health check be designed?
  - Why does layer 7 balancing cost more than layer 4, and when is it worth it?
  - Design the load balancing layers for a large public API.
  - Where do you draw the line on what logic belongs in the load balancer?
- **Connection pooling** (7 prompts)
  - Why use a connection pool?
  - How do you calculate the right pool size?
  - Your pool is exhausted during an incident. Should you increase it?
  - Why bound connection lifetime?
  - Design connection management for 200 application instances against one PostgreSQL primary.
  - You move to an async driver that multiplexes many queries per connection. What changes?
  - How do you prevent connection-limit incidents across a large organisation?
- **HTTP/2 multiplexing** (6 prompts)
  - What does HTTP/2 multiplexing do?
  - Does HTTP/2 eliminate head-of-line blocking?
  - Why is L4 load balancing a problem for HTTP/2?
  - A gRPC client shows high latency but the server looks healthy. What do you check?
  - When would you move a service from HTTP/2 to HTTP/3?
  - How do you roll HTTP/2 out across an organisation without operational surprises?
- **QUIC and HTTP/3** (6 prompts)
  - What problem does QUIC solve that HTTP/2 does not?
  - Why is QUIC built on UDP rather than being a new transport protocol?
  - What is connection migration and why does it matter?
  - What are the risks of 0-RTT, and how do you handle them?
  - How would you decide whether to adopt HTTP/3 for a given service?
  - What is the longer-term significance of moving the transport into userspace?
- **Service discovery** (6 prompts)
  - What does a service registry do?
  - Why must you deregister before shutting down?
  - What should a client do when the registry is unreachable?
  - Compare client-side and server-side discovery.
  - Quantify the staleness window for an unplanned instance failure and design around it.
  - How do you prevent the discovery control plane from becoming a company-wide single point of failure?

##### data storage

- **Relational data modeling** (6 prompts)
  - Why put constraints in the database rather than the application?
  - When would you denormalise?
  - How do you prevent overselling inventory?
  - Surrogate or natural primary keys?
  - A single table is now written by three services. How do you fix it?
  - How do you make schema change safe at organisational scale?
- **Document storage** (6 prompts)
  - When should you embed versus reference?
  - What atomicity does a document store give you?
  - How do you handle concurrent updates to the same document?
  - How do you manage schema evolution without migrations?
  - How do you decide between a document store and a relational database with JSON columns?
  - A team wants to move a transactional system from relational to document storage for scaling reasons. How do you evaluate it?
- **Key-value storage** (6 prompts)
  - Why do key-value stores scale so well?
  - How do you implement a counter safely?
  - How do you support “find the user by email” in a key-value store?
  - What do you do about a hot key?
  - Design an idempotency mechanism for a payments API using a key-value store.
  - What organisational practices keep a large shared key-value store healthy?
- **Wide-column storage** (6 prompts)
  - What do the partition key and clustering key each do?
  - Why would you store the same data in two tables?
  - How do you prevent hot and unbounded partitions?
  - Why do tombstones matter?
  - When would you choose wide-column over a relational database with read replicas?
  - A team's wide-column cluster is degrading and they want to add nodes. How do you approach it?
- **Object storage** (6 prompts)
  - Why is object storage cheaper than block storage?
  - Why do you need a separate metadata store?
  - Describe a reliable large-file upload flow.
  - How would you reduce the cost of a petabyte-scale store?
  - A compliance requirement says user data must be deleted within 30 days. What do you actually have to do?
  - How do you keep storage cost from growing unbounded across many teams?
- **Time-series data modeling** (6 prompts)
  - What is cardinality and why does it matter?
  - Why roll up old data?
  - Why can't you average p99 values when rolling up?
  - How do you handle late-arriving data?
  - Design the metrics model for a platform with 5,000 containers.
  - How do you prevent cardinality incidents organisationally?
- **Graph data modeling** (6 prompts)
  - When is a graph database actually the right choice?
  - What is a supernode and why does it matter?
  - How do you bound a traversal safely?
  - Why is horizontal scaling hard for graphs?
  - A team wants to migrate an authorisation system to a graph database. How do you evaluate it?
  - What organisational risks come with adopting a graph database?
- **Columnar analytical storage** (6 prompts)
  - Why is columnar storage faster for analytics?
  - Why is it bad for point lookups?
  - What matters more, the file format or the data layout?
  - What is the small-file problem and how do you avoid it?
  - Design the storage layer for a product analytics system with 5 billion events per year.
  - How do you control cost in a per-byte-scanned analytics platform?

##### database internals

- **B-tree indexes** (6 prompts)
  - Why is a B-tree lookup fast even on a huge table?
  - What is the leftmost prefix rule?
  - Why might the planner ignore an index you created?
  - What is a covering index and when is it worth it?
  - A write-heavy table has twelve indexes and insert throughput has degraded. How do you approach it?
  - How do you keep index design healthy across many teams and databases?
- **LSM trees and compaction** (6 prompts)
  - Why are LSM writes faster than B-tree writes?
  - What is a tombstone and why does it matter?
  - Explain the three amplification factors and how compaction strategy trades between them.
  - How do Bloom filters make LSM reads practical?
  - An LSM-backed service is hitting write stalls. Walk through your diagnosis.
  - How would you decide between B-tree and LSM storage engines for a new platform?
- **Write-ahead logging** (6 prompts)
  - What is the write-ahead rule?
  - Why is committing faster with a WAL than without one?
  - Walk through crash recovery.
  - What does “committed” actually guarantee?
  - How do you configure durability for a payments system, and what must you decide in advance?
  - How do you make durability guarantees trustworthy across an organisation?
- **MVCC snapshots** (6 prompts)
  - How does MVCC let readers avoid blocking writers?
  - What is a snapshot?
  - Why does a long-running read-only query cause bloat?
  - What is write skew and why does snapshot isolation permit it?
  - Compare heap-versioned and undo-log MVCC implementations operationally.
  - What platform defaults would you set to prevent MVCC incidents?
- **Transaction isolation levels** (6 prompts)
  - What does Read Committed guarantee?
  - What is a lost update and how do you prevent it?
  - Explain write skew and why Repeatable Read does not prevent it.
  - When would you use `SELECT ... FOR UPDATE` instead of raising the isolation level?
  - A team reports rare double bookings that they cannot reproduce. How do you approach it?
  - How do you prevent concurrency anomalies across an organisation rather than fixing them one at a time?
- **Optimistic concurrency control** (6 prompts)
  - How does optimistic concurrency control work?
  - What does `If-Match` do in HTTP?
  - When does optimistic concurrency stop being the right choice?
  - How do you choose version granularity?
  - Design conflict handling for a system where users edit shared records.
  - Where does this pattern appear beyond databases, and why does that matter?
- **Deadlocks and lock ordering** (6 prompts)
  - What causes a deadlock?
  - How do you prevent deadlocks?
  - How do you diagnose a deadlock in production?
  - Why can an unindexed foreign key cause deadlocks?
  - Deadlock rate jumped after a release. Walk through your response.
  - How do you keep deadlocks from recurring across many teams and services?
- **Query planning and statistics** (6 prompts)
  - What does a query planner do?
  - How do you read a query plan?
  - Why would the same query be fast for one parameter and slow for another?
  - Why are correlated predicates a problem?
  - A query that has been fine for a year suddenly got slow, with no deployment. What happened?
  - How do you make query performance debuggable across an organisation?

##### caching

- **Cache-aside** (6 prompts)
  - Describe the cache-aside read and write paths.
  - Why does every cached entry need a TTL?
  - How do you prevent a cache stampede?
  - What happens when the cache tier goes down?
  - How would you eliminate the stale-set race rather than merely bounding it?
  - How do you decide what should never be cached?
- **Write-through caching** (6 prompts)
  - What is write-through caching?
  - Why must you write to the source before the cache?
  - What should happen if the cache update fails after a successful commit?
  - When is write-through a poor choice?
  - Design caching for a feature flag service serving 500,000 reads per second.
  - How would you get cache freshness guarantees that cover every writer, not just your own?
- **Write-behind caching** (6 prompts)
  - What is write-behind caching and what does it trade?
  - Where does the big performance win come from?
  - How do you make write-behind safe for data that matters?
  - Why is an unbounded buffer dangerous?
  - Design counting for two billion page views a day.
  - How do you govern write-behind so its risks stay visible?
- **TTL and invalidation** (6 prompts)
  - Why is TTL necessary even with invalidation?
  - How should you choose a TTL?
  - What is stale-while-revalidate and when would you use it?
  - Why is application-level invalidation insufficient in many systems?
  - A requirement says price changes must be live within 30 seconds. How do you design for it?
  - How do you manage staleness across multiple cache tiers and many teams?
- **Cache stampede protection** (6 prompts)
  - What is a cache stampede?
  - What is single-flight?
  - Why is stale-while-revalidate often better than coalescing?
  - What happens if the refill fails while requests are coalesced?
  - Design stampede protection for a homepage query costing 400 ms at 50,000 requests per second.
  - Why should stampede protection live in shared infrastructure rather than in each service?
- **Cache eviction policies** (6 prompts)
  - Why does LRU perform badly with scans?
  - What is an admission policy?
  - How do you size a cache?
  - Your hit rate drops from 99% to 97%. How bad is that?
  - Nightly batch jobs are destroying your cache hit rate. What do you do?
  - How do you manage cache behaviour in a shared, multi-tenant cache?
- **Negative caching** (6 prompts)
  - What is negative caching and why is it needed?
  - Why should negative TTLs be shorter than positive ones?
  - Why must you never cache a source error as absence?
  - When does negative caching make things worse?
  - Design protection for a public lookup API facing enumeration.
  - How do you decide between a Bloom filter and negative caching in general?
- **Multi-level caching** (6 prompts)
  - Why add an in-process cache in front of a shared one?
  - What is the main cost of a local cache layer?
  - How do you keep a local cache coherent?
  - Won't a short local TTL destroy the hit rate?
  - Design caching for a permissions service where revocation must take effect within 5 seconds.
  - What organisational practice prevents multi-level caching from eroding guarantees?

##### messaging

- **Work queues** (6 prompts)
  - Why do work queues deliver at least once rather than exactly once?
  - What is a visibility timeout or lease?
  - How do you make a handler idempotent?
  - Why alert on queue age rather than queue depth?
  - Design the async pipeline for order processing with four downstream effects.
  - What makes asynchronous systems harder to operate, and how do you compensate?
- **Publish-subscribe** (6 prompts)
  - What is the difference between a queue and a topic?
  - Why publish events rather than commands?
  - How much data should an event carry?
  - What is the dual-write problem in event publishing?
  - A new team wants to consume your events for fraud detection. What determines whether that is easy?
  - How do you govern event schemas across an organisation?
- **Partitioned event logs** (6 prompts)
  - What ordering guarantee does a partitioned log provide?
  - Why is partition count a scaling ceiling?
  - How do offsets differ from acknowledgements?
  - What do you do about a poison message in a log?
  - Design the partitioning for a payment event stream requiring per-account ordering.
  - What makes a partitioned log a different architectural commitment from a queue?
- **Consumer groups and rebalancing** (6 prompts)
  - What does a consumer group do?
  - Why can't you have more consumers than partitions?
  - What is the difference between session timeout and max poll interval?
  - Why is raising the timeout the wrong response to frequent rebalances?
  - How do you make rolling deploys of a large stateful stream processor non-disruptive?
  - Why do consumer groups cause so much operational pain relative to their conceptual simplicity?
- **Delivery semantics** (6 prompts)
  - Why is exactly-once delivery impossible?
  - How do you make a handler idempotent?
  - Where should the idempotency key come from?
  - Why store the response rather than just a flag?
  - What does a system mean when it claims exactly-once semantics?
  - How do you make correct delivery handling the default across an organisation?
- **Dead-letter queues** (6 prompts)
  - What is a dead-letter queue for?
  - Why classify failures before retrying?
  - What must a dead-lettered message carry?
  - How should redrive work?
  - Your dead-letter rate jumps from zero to 340 per minute. Walk through your response.
  - How do you ensure dead-letter queues do not become silent data loss?
- **Transactional outbox** (6 prompts)
  - What is the dual-write problem?
  - How does the outbox solve it?
  - Why do consumers still need idempotency with an outbox?
  - Compare a polling relay with a log-based one.
  - Why use an outbox rather than capturing changes directly from your business tables?
  - How would you eliminate dual writes across an organisation?
- **Schema evolution for events** (6 prompts)
  - What is the difference between backward and forward compatibility?
  - Why can't you just rename a field?
  - What is the most dangerous kind of schema change?
  - How do you choose a default value for a new field?
  - You need to make a genuinely breaking change to a widely consumed event. How?
  - How do you govern event schema evolution across many teams?

##### distributed systems

- **CAP during partitions** (6 prompts)
  - What does CAP actually say?
  - Why can't both sides of a partition just keep accepting writes?
  - How do quorums implement the CP choice?
  - Why is fencing necessary if you already detect failures?
  - How would you apply CAP to a multi-region e-commerce system?
  - What is wrong with how CAP is usually taught, and what would you teach instead?
- **Consistency models** (6 prompts)
  - What does eventual consistency actually guarantee?
  - What is read-your-writes and why does it matter?
  - How does linearizability differ from sequential consistency?
  - Why do users notice some inconsistencies and not others?
  - How would you assign consistency models across a social application?
  - How do you reason about end-to-end consistency across many services?
- **Leader-based replication** (6 prompts)
  - Why does single-leader replication avoid write conflicts?
  - What is replication lag and why does it matter?
  - What are the risks of failover?
  - Why is silent fallback from synchronous to asynchronous replication dangerous?
  - Design the replication topology and policy for a service that cannot lose acknowledged writes.
  - When would you move from leader-based replication to consensus-based?
- **Quorum reads and writes** (6 prompts)
  - What does W + R > N guarantee?
  - Why is W=1, R=1 not safe for read-your-writes?
  - Why are version vectors better than timestamps for conflict resolution?
  - Are quorum reads linearizable?
  - Design quorum settings and conflict handling for a shopping cart.
  - When is a quorum system the right choice over consensus or a single leader?
- **Consensus and Raft** (6 prompts)
  - Why does consensus require a majority?
  - How many nodes should a consensus group have?
  - Why aren't reads from a Raft leader automatically linearizable?
  - Why do most systems keep bulk data out of the consensus group?
  - What determines the actual fault tolerance of a consensus group?
  - When would you implement consensus yourself versus delegating to a coordination service?
- **Leases and fencing tokens** (6 prompts)
  - Why isn't a distributed lock enough?
  - What is a fencing token?
  - Why does lease duration not affect safety?
  - Where must the fencing check happen, and why?
  - How would you fence shard ownership in a distributed storage system?
  - What do you do when the resource cannot be fenced?
- **Logical clocks and causal order** (6 prompts)
  - Why can't you order distributed events by timestamp?
  - What does happens-before mean?
  - What is the difference between Lamport clocks and vector clocks?
  - What is the cost of vector clocks and how do you bound it?
  - When is last-writer-wins acceptable?
  - How would you approach ordering across an organisation's distributed systems?
- **CRDTs** (6 prompts)
  - What makes a data type conflict-free?
  - Why can't you merge a counter by taking the maximum?
  - Why does an OR-Set need tags?
  - Why can't CRDTs enforce invariants?
  - How would you design synchronisation for an offline-first task list?
  - When do CRDTs earn their complexity, and when do they not?

##### API design

- **Resource-oriented HTTP APIs** (6 prompts)
  - Why does it matter which HTTP method you use?
  - What is the most important distinction in status codes?
  - How do you handle concurrency in a REST API?
  - When is a sub-resource action better than a PATCH?
  - When would you choose gRPC over a resource-oriented HTTP API?
  - What API conventions would you standardise across an organisation, and why those?
- **RPC and gRPC** (6 prompts)
  - What does gRPC give you over JSON on HTTP?
  - Why does a deadline matter more than a timeout?
  - Why does gRPC break L4 load balancing?
  - How do you evolve a gRPC schema safely?
  - You are migrating internal services to gRPC. What operational work does it require?
  - Where would you draw the boundary between gRPC and HTTP in an architecture?
- **GraphQL execution** (6 prompts)
  - What problem does GraphQL solve?
  - What is the N+1 problem in GraphQL?
  - Why is caching harder with GraphQL?
  - How do you prevent a single query from overwhelming the service?
  - How would you harden GraphQL for production?
  - When would you advise against GraphQL?
- **Cursor pagination** (6 prompts)
  - Why is offset pagination incorrect under concurrent writes?
  - Why is offset slow at depth?
  - Why does a cursor need a unique tiebreaker?
  - Why should cursors be opaque?
  - What do you lose with cursor pagination, and does it matter?
  - What pagination standard would you set for an organisation's APIs?
- **Idempotency keys** (6 prompts)
  - What problem do idempotency keys solve?
  - Why must the key come from the client?
  - Why store the response rather than just recording that the operation happened?
  - How do you handle a retry arriving while the original is still in flight?
  - Where can idempotency still fail even with a correct implementation?
  - How would you make idempotency reliable across an organisation?
- **Rate limiting algorithms** (6 prompts)
  - What is the boundary problem with fixed-window rate limiting?
  - Why is token bucket a good default?
  - Why can't rate limits protect against overload?
  - How do you enforce a global limit across many instances?
  - Design rate limiting for a public API with tiers.
  - What is the most common structural mistake organisations make with rate limiting?
- **API versioning and compatibility** (6 prompts)
  - What makes a change to an API backward compatible?
  - Why is versioning something to avoid rather than adopt?
  - What is expand-contract and why does it usually avoid a version?
  - Why are semantic changes more dangerous than structural ones?
  - What actually prevents teams from removing deprecated fields?
  - How would you set API evolution policy across an organisation?
- **Authentication and authorization boundaries** (6 prompts)
  - What is the difference between authentication and authorisation?
  - Why can't the gateway do all the authorisation?
  - Why should you not propagate the user's credential to downstream services?
  - Why is a missing authorisation check particularly dangerous?
  - How would you structure authorisation across a microservice system?
  - What makes authorisation hard to get right at organisational scale?

##### async

- **Background job lifecycle** (6 prompts)
  - Why model a background job as a durable record rather than just a queue message?
  - Why does every non-terminal state need a timeout?
  - How do you safely claim a job so two workers cannot both run it?
  - Why is idempotency still necessary if the state machine is correct?
  - What operational signals would you expose for a background job system?
  - What makes background job systems hard to operate well?
- **Retry, backoff and jitter** (6 prompts)
  - Why is jitter necessary if you already have exponential backoff?
  - What should you not retry?
  - Why are nested retries dangerous?
  - Why does a total deadline matter more than a per-attempt timeout?
  - Design a retry policy for a checkout calling a payment provider.
  - Why should retry policy be a platform concern rather than left to teams?
- **Backpressure** (6 prompts)
  - What is backpressure?
  - Why doesn't a bigger buffer solve overload?
  - What should happen when a bounded buffer is full?
  - Why must the pressure signal reach the original producer?
  - Walk through the consequences of running an ingestion pipeline without backpressure.
  - Why do teams resist backpressure, and how do you address that?
- **Batching and microbatching** (6 prompts)
  - Where does the gain from batching come from?
  - Why do you need both a size and a time threshold?
  - Why is failure granularity the detail that matters most?
  - How do you choose the linger time?
  - How would you configure batching for a 50,000 events per second pipeline?
  - When is batching the wrong answer to a throughput problem?
- **Durable workflow orchestration** (6 prompts)
  - How does a durable workflow survive a crash?
  - Why must workflow code be deterministic?
  - Why do activities still need to be idempotent?
  - What does a durable timer give you that a scheduled job does not?
  - When is a workflow engine over-engineering, and when is it under-engineering not to have one?
  - What is the real cost of adopting a workflow engine?
- **Sagas and compensation** (6 prompts)
  - What is a saga?
  - How is compensation different from rollback?
  - What is the pivot step and why does its placement matter?
  - Why must compensations be able to succeed unconditionally?
  - What do sagas not give you, and what fills the gap?
  - When should a saga make you question the architecture rather than build one?
- **Fanout and aggregation** (6 prompts)
  - What does fan-out change about latency?
  - Why does a shared deadline matter more than per-call timeouts?
  - How does treating every dependency as required affect availability?
  - Why is per-item fan-out dangerous?
  - Design a product page that assembles data from six services.
  - When should a fan-out be replaced rather than optimised?
- **Distributed scheduling** (6 prompts)
  - Why doesn't cron work on a fleet?
  - How do you guarantee a scheduled job runs exactly once?
  - What should happen to windows missed during an outage?
  - Why is alerting on absence harder than alerting on failure?
  - Design scheduling for subscription billing across 200,000 subscriptions.
  - When should you not build a scheduler at all?

##### search

- **Inverted indexes** (6 prompts)
  - What does an inverted index invert?
  - Why do posting lists compress so well?
  - Why does intersection order matter?
  - Why must index-time and query-time analysis match?
  - What are the space implications of supporting phrase queries?
  - How should a search index relate to the system of record?
- **Text analysis and normalization** (6 prompts)
  - What does an analyser do?
  - Why does analysis have to match between indexing and querying?
  - How do you balance recall and precision through analysis?
  - Why prefer query-time synonym expansion?
  - How would you handle product model numbers in search?
  - Why is analysis often the limiting factor on search quality?
- **BM25 relevance ranking** (6 prompts)
  - What are the three ideas in BM25?
  - Why does term frequency saturate?
  - Why is BM25 rarely the final ranking in production?
  - Why can't you compare BM25 scores across queries?
  - A team reports poor search relevance. How do you approach it?
  - When does it make sense to move beyond BM25?
- **Autocomplete and prefix search** (6 prompts)
  - Why is autocomplete a different problem from search?
  - Why suggest queries rather than documents?
  - How do you build the suggestion corpus?
  - What client-side work matters for autocomplete?
  - How would you handle typos in autocomplete?
  - Why should autocomplete be treated as a separate system from search?
- **Faceted search** (6 prompts)
  - What makes faceting different from filtering?
  - Why must facet counts exclude their own filter?
  - Why are high-cardinality facets dangerous?
  - How should multi-select behave across facets?
  - A team wants to facet on user-generated tags with millions of distinct values. How do you respond?
  - How do you keep faceting from degrading search performance over time?
- **Vector similarity search** (6 prompts)
  - How does vector search find semantically similar documents?
  - Why does chunking matter?
  - What can vector search not do?
  - Why is hybrid retrieval the practical default?
  - What is the operational cost of adopting vector search?
  - How would you approach retrieval quality for a documentation or knowledge system?
- **Approximate nearest neighbors** (6 prompts)
  - Why is exact nearest-neighbour search impractical in high dimensions?
  - What does approximate mean in this context?
  - How do graph-based and cluster-based indexes differ?
  - Why is filtering hard with approximate indexes?
  - How would you size and choose an index for ten million 768-dimensional vectors?
  - What is the most important operational discipline with approximate search?
- **Hybrid retrieval and reranking** (6 prompts)
  - Why use more than one retriever?
  - Why fuse by rank rather than by score?
  - Why is reranking done over candidates rather than the corpus?
  - Why measure retrieval recall separately from ranking quality?
  - Which tuning lever would you reach for first to improve a hybrid pipeline?
  - How would you structure ownership and evaluation of a search pipeline?

##### media

- **Resumable multipart uploads** (6 prompts)
  - What does multipart upload actually buy you?
  - How do you choose part size?
  - What happens to parts from an abandoned upload?
  - Why must the metadata record remain authoritative?
  - Design the upload flow for a two-gigabyte video from a mobile client.
  - What makes upload systems fail in production rather than in testing?
- **Media transcoding pipelines** (6 prompts)
  - Why transcode at all rather than serving the original?
  - Why validate before encoding?
  - Why should each rendition be a separate job?
  - Why publish the manifest last?
  - How would you reduce transcoding cost on a large media platform?
  - What makes transcoding pipelines architecturally distinctive?
- **Adaptive bitrate streaming** (6 prompts)
  - What problem does adaptive bitrate streaming solve?
  - Why must keyframes align across variants?
  - Why is buffer level a better signal than throughput?
  - Why step down fast but climb up slowly?
  - How would you configure adaptive streaming for a live event versus on-demand video?
  - What makes adaptive streaming scale to millions of concurrent viewers?
- **Content delivery networks** (6 prompts)
  - What does a CDN actually give you?
  - Why do immutable URLs make caching easy?
  - Why does hit ratio matter so much more than it looks?
  - What is stale-while-revalidate and why prefer it?
  - How would you configure a CDN for an application with static assets, media, public APIs and personalised pages?
  - What are the failure modes that make CDNs risky rather than just beneficial?
- **Signed media access** (6 prompts)
  - What is a signed URL?
  - Why not just use unguessable URLs?
  - How do you revoke a signed URL?
  - When would you sign a prefix rather than a single path?
  - Design signed access for a subscription video service that must stop playback within minutes of a cancellation.
  - What are the systemic risks of signed access, as opposed to implementation bugs?
- **Image transformation services** (6 prompts)
  - Why transform on demand rather than pre-generating?
  - Why does the transform go in the URL?
  - Why must transform parameters be bounded?
  - What are the security concerns in image transformation?
  - Design image delivery for a product catalogue with heavy mobile traffic.
  - Why is image delivery so often a major performance problem despite being conceptually simple?
- **Live streaming ingest** (6 prompts)
  - How does live ingest differ from uploading a file?
  - Why is segment length the main latency lever?
  - Why can a live pipeline never catch up after falling behind?
  - How should a broadcaster disconnection be handled?
  - Design the live pipeline for a platform hosting thousands of concurrent broadcasters.
  - What makes live streaming operationally harder than on-demand, given the delivery path is identical?
- **Content addressing and deduplication** (6 prompts)
  - What does content addressing mean?
  - Why is deletion hard in a deduplicated store?
  - Why does content-defined chunking matter?
  - What is the privacy concern with cross-user deduplication?
  - Design a backup system using content addressing.
  - Content addressing is often adopted for deduplication. What else does it give you?

##### realtime

- **WebSocket connections** (6 prompts)
  - What does a WebSocket give you that HTTP does not?
  - When should you use server-sent events instead?
  - What does persistent connection state do to scaling?
  - Why do registry entries need TTLs?
  - Design the connection layer for a collaborative editor with a million concurrent users.
  - What makes realtime connection layers disproportionately expensive to own?
- **Server-sent events** (6 prompts)
  - What is a server-sent event stream?
  - When would you choose SSE over WebSockets?
  - How does resume work, and what does it require of the server?
  - What are the two environment problems that catch people out?
  - Design a live notification system for a web application.
  - Why is SSE underused relative to how well it fits?
- **Long polling** (6 prompts)
  - How does long polling differ from ordinary polling?
  - When is long polling the right choice?
  - Why is a cursor essential?
  - Why does the threading model matter so much?
  - Design a notification channel that must work behind restrictive corporate proxies.
  - Long polling is widely considered obsolete. Is it?
- **Presence and heartbeats** (6 prompts)
  - Why can't presence just be tracked by connection state?
  - How does the heartbeat lease work?
  - How do you choose the heartbeat interval and TTL?
  - Why is fan-out often the dominant cost?
  - Design presence for a messaging application with a million concurrent users.
  - What do people consistently underestimate about presence?
- **Realtime fan-out routing** (6 prompts)
  - What makes fan-out hard?
  - Why not broadcast every message to every process?
  - When should you switch from push to read-time fan-out?
  - How do you handle a slow consumer?
  - Design fan-out for a messaging product with channels from three to a hundred thousand members.
  - What do teams get wrong about realtime fan-out at scale?
- **Ordering, acknowledgments and resume** (6 prompts)
  - Why do realtime messages need sequence numbers?
  - What is the resume flow on reconnection?
  - Why does the order of applying and advancing the cursor matter?
  - Why is exactly-once delivery not achievable?
  - Design the reliability layer for a collaborative editing channel.
  - Why is this layer so often missing, and what does its absence cost?
- **Offline-first synchronization** (6 prompts)
  - What makes an application offline-first?
  - Why are conflicts unavoidable?
  - Why is last-write-wins dangerous?
  - How does modelling changes as operations help?
  - Design offline support for a field service app used in warehouses with no signal.
  - When would you argue against offline-first?
- **Operational transformation** (6 prompts)
  - What problem does operational transformation solve?
  - What does transformation actually do?
  - Why do production OT systems almost always have a central server?
  - Why is convergence insufficient as a correctness criterion?
  - How would you approach building a collaborative editor today?
  - How do you weigh OT against CRDTs?

##### reliability

- **SLOs and error budgets** (6 prompts)
  - What is an error budget?
  - Why not target 100 per cent reliability?
  - What makes a good service level indicator?
  - Why alert on burn rate rather than on the objective itself?
  - How would you set an SLO for a checkout service?
  - Why do SLO programmes fail in practice?
- **Circuit breakers** (6 prompts)
  - What does a circuit breaker do?
  - Why is a slow dependency more dangerous than a failing one?
  - Why do timeouts matter more than the breaker itself?
  - Why trip on failure rate rather than failure count?
  - How would you protect a product page that calls recommendations, reviews and inventory?
  - What do teams misunderstand about circuit breakers?
- **Bulkheads and isolation** (6 prompts)
  - What is a bulkhead in software?
  - Why does sharing a pool transmit failure?
  - What is the utilisation cost of partitioning?
  - Why is isolation only as strong as the least-isolated component?
  - Design isolation for a multi-tenant analytics platform where customers write their own queries.
  - When is cellular architecture worth its cost, and when is it premature?
- **Load shedding and admission control** (6 prompts)
  - Why reject requests instead of trying to serve them all?
  - What makes a good saturation signal?
  - Why does shedding require client cooperation?
  - Why is priority classification essential?
  - Design admission control for an API gateway facing unpredictable traffic surges.
  - Why is load shedding so often missing, and what is the consequence?
- **Timeouts, deadlines and cancellation** (6 prompts)
  - What is the difference between a timeout and a deadline?
  - Why does a timeout alone not free resources?
  - How do you choose a timeout value?
  - Why is a timeout an ambiguous outcome?
  - Design timeout handling for a checkout flow with a three-second user tolerance.
  - Why are timeouts so often wrong in production systems?
- **Health, readiness and liveness** (6 prompts)
  - What is the difference between liveness and readiness?
  - Why does a startup probe exist?
  - Why must liveness never check dependencies?
  - Why fail readiness before closing connections during shutdown?
  - Design health checks for a service with a database, a cache, and an optional recommendations dependency.
  - Why is health check configuration disproportionately dangerous?
- **Disaster recovery and restore** (6 prompts)
  - What are RPO and RTO?
  - Why is replication not a backup?
  - Why does an untested backup not count?
  - Why must backups live in a separate failure domain?
  - Design disaster recovery for an e-commerce platform.
  - Why do organisations consistently overestimate their recovery capability?
- **Graceful degradation** (6 prompts)
  - What is graceful degradation?
  - Why does treating everything as required hurt availability?
  - Why do timeouts matter as much as fallbacks?
  - Why must degraded states be alerted?
  - Design degradation for a product page with seven backing services.
  - What makes graceful degradation hard in practice, given the idea is simple?

##### observability

- **Metrics and golden signals** (6 prompts)
  - What are the golden signals?
  - Why is average latency misleading?
  - Why separate latency by success and failure?
  - Why is CPU usually the wrong saturation metric?
  - How would you instrument a new service from scratch?
  - Why do organisations end up with hundreds of dashboards and poor visibility?
- **Latency histograms and percentiles** (6 prompts)
  - Why store histograms rather than percentiles?
  - How is a percentile estimated from a histogram?
  - Why can percentiles not be averaged?
  - How should bucket boundaries be chosen?
  - Design latency instrumentation for an API with a 300 millisecond objective.
  - What does a histogram show that percentiles do not?
- **Structured logging** (6 prompts)
  - What makes a log structured?
  - Why does severity matter?
  - Why is a correlation identifier essential?
  - Why is logging a security concern?
  - Design a logging standard for a payment service.
  - How do you think about the boundary between logs, metrics and traces?
- **Distributed tracing** (6 prompts)
  - What does a trace show that metrics do not?
  - Why does context propagation matter so much?
  - Why is head-based sampling problematic?
  - Where do traces typically break?
  - How would you introduce tracing to an existing microservice system?
  - When does tracing not pay for itself?
- **Telemetry sampling** (6 prompts)
  - Why sample telemetry at all?
  - What is wrong with keeping a uniform one per cent?
  - What is the difference between head-based and tail-based sampling?
  - Why can you not compute rates from sampled data?
  - Design a sampling policy for a service handling ten thousand requests per second.
  - What makes sampling decisions consequential beyond cost?
- **Cardinality and label design** (6 prompts)
  - What is cardinality in metrics?
  - Why are user identifiers bad labels?
  - Where should high-cardinality questions go?
  - Why are histograms especially cardinality-sensitive?
  - How would you prevent cardinality incidents in an organisation?
  - Why does cardinality remain a recurring problem despite being well understood?
- **Actionable alerting and burn rates** (6 prompts)
  - Why alert on symptoms rather than causes?
  - What is a burn rate?
  - Why use multiple windows?
  - Why is alert fatigue a safety issue?
  - Design alerting for a checkout service with a 99.9 per cent objective.
  - How would you improve alerting in an organisation with severe fatigue?
- **Profiling and resource attribution** (6 prompts)
  - What does a profiler do?
  - Why profile before optimising?
  - What is the difference between self time and total time?
  - Why is a CPU profile useless for an I/O-bound service?
  - How would you investigate a service using more CPU than expected?
  - Why does intuition about performance fail so consistently?

##### deployment

- **Container images and reproducible releases** (6 prompts)
  - Why are container images immutable?
  - Why does layer ordering matter?
  - Why build once and promote rather than rebuilding per environment?
  - Why can't you remove a secret by deleting it in a later layer?
  - Design a build and release pipeline for a compiled service.
  - What does reproducibility actually buy, given the effort?
- **Orchestration control loops** (6 prompts)
  - What is a reconciliation loop?
  - Why is declarative better than imperative here?
  - What is the difference between level-triggered and edge-triggered?
  - Why do controllers need rate limits and rollout constraints?
  - Design the deployment behaviour you would want from an orchestrator.
  - What is the conceptual shift that makes this model work?
- **Rolling deployments** (6 prompts)
  - How does a rolling deployment work?
  - What is the main constraint rolling deployment imposes?
  - Why does the health gate matter so much?
  - How do you change a database schema safely under rolling deployment?
  - Design a rolling deployment for a twenty-instance service including a schema change.
  - Why do rolling deployments fail in practice, given how well understood they are?
- **Blue-green deployment** (6 prompts)
  - How does blue-green deployment work?
  - What is the main advantage over rolling deployment?
  - Why does blue-green not remove schema compatibility requirements?
  - Why is rollback not as complete as it appears?
  - When would you choose blue-green over rolling or canary?
  - What does blue-green actually guarantee, and what do teams assume it guarantees?
- **Canary releases** (6 prompts)
  - What is a canary release?
  - Why compare against the baseline rather than a fixed threshold?
  - Why does a canary need automated evaluation?
  - What statistical problem do low-traffic services have with canaries?
  - Design an automated canary pipeline for a checkout service.
  - How do canary releases fit alongside other deployment strategies?
- **Feature flags** (6 prompts)
  - What do feature flags actually decouple?
  - Why must percentage rollouts be deterministic?
  - Why is flag removal the critical discipline?
  - Why should flag evaluation be local?
  - Design the rollout of a redesigned checkout using flags.
  - What is the long-term cost of feature flags, and how do you manage it?
- **Expand-contract schema migrations** (6 prompts)
  - What is expand-contract?
  - Why can't you just rename the column?
  - Why continue dual writing after reads have switched?
  - Why are backfills dangerous?
  - Walk through splitting a name column into first and last names on a fifty-million-row table.
  - Why is this discipline so often skipped, and what is the real cost?
- **Autoscaling control and hysteresis** (6 prompts)
  - Why does autoscaling need hysteresis?
  - Why should scale-up and scale-down behave differently?
  - Why is CPU often the wrong scaling signal?
  - Why can't autoscaling handle traffic spikes?
  - Configure autoscaling for an API service with a daily traffic pattern.
  - What is the most common misconception about autoscaling?

#### System workouts (200 prompts)

##### Foundational

- **URL shortener** (4 prompts)
  - One alias suddenly receives 200,000 redirects per second.
  - An unavailable alias was cached as missing before its owner created it.
  - Two create requests both observe that /launch is available.
  - Model abuse takedown propagation separately from ordinary edits; establish a maximum revocation delay and test cache invalidation under partial regional failure.
- **Distributed rate limiter** (4 prompts)
  - A single tenant sends half of all requests to one token bucket.
  - The bucket datastore times out while the payment API remains healthy.
  - A tenant uses the full global allowance simultaneously in three regions.
  - Define fairness during regional evacuation and prevent one tenant from monopolizing the emergency budget. Roll out policy changes with explicit version semantics.
- **Distributed cache** (4 prompts)
  - Adding four nodes remaps keys and doubles database reads.
  - A batch import gives ten million popular keys the same expiry.
  - A delayed loader writes version 7 after an update invalidated version 8.
  - Size for failover with one shard unavailable, including connection storms, allocator overhead, replication buffers, and the source load from a cold replacement.
- **Unique ID service** (4 prompts)
  - A worker needs more IDs in one clock tick than its sequence bits allow.
  - A host clock steps backwards by three seconds after synchronization.
  - A partitioned old worker continues issuing IDs after its lease was reassigned.
  - Prove uniqueness across clock rollback, restored VM snapshots, identity reuse, and disaster recovery. Include a documented migration before the timestamp field expires.
- **Durable job scheduler** (4 prompts)
  - Three million reports become due at midnight for the same timezone.
  - The email side effect succeeded but the attempt acknowledgement was lost.
  - A user cancels a job while a worker has acquired its lease.
  - Specify misfire behavior for missed recurring schedules, timezones, daylight-saving transitions, and prolonged outages; avoid replaying years of missed jobs on recovery.

##### Social

- **Social news feed** (4 prompts)
  - An account with 80M followers posts during peak traffic.
  - A cached inbox still references content removed by moderation.
  - Scores change between page one and page two, causing duplicates and skips.
  - Set separate SLOs for post acceptance, follower visibility, privacy revocation, and ranking freshness; a single availability number hides the important failures.
- **Photo sharing** (4 prompts)
  - A layout release asks for a previously uncached transform for every photo.
  - An object event arrives before image validation and the gallery publishes it.
  - A long-lived CDN URL continues to expose a photo after access is revoked.
  - Coordinate deletion of originals, variants, caches, and backup-retention records; expose deletion progress without promising that a metadata delete erases every byte instantly.
- **Follow graph** (4 prompts)
  - One account has 60M followers and its inbound row becomes unmanageable.
  - Unfollow retries decrement counters even when no edge was removed.
  - A blocked account appears in a mutual-friends result produced from lagging indexes.
  - Design repair jobs that compare projection checkpoints and sample edge symmetry without loading the complete graph into memory.
- **Threaded comments** (4 prompts)
  - A match-final thread receives 20,000 comments per second.
  - An old body-edit request overwrites a newer hidden state.
  - A parent comment is deleted but hundreds of valid replies remain.
  - Distinguish author deletion, moderator hiding, legal removal, and thread locking, since each has different visibility and audit requirements.
- **Ephemeral stories** (4 prompts)
  - A public story accumulates 30M unique viewers in an hour.
  - A cleanup worker is late, leaving an expired object in storage and cache.
  - A creator switches from public to close friends while existing sessions are active.
  - Separate the promised visibility deadline from physical deletion and backup expiration, and measure each boundary independently.

##### Messaging

- **Direct and group chat** (4 prompts)
  - A 1,000-member trading room generates 10,000 messages per second.
  - A gateway acknowledged receipt before durable storage and crashed.
  - Queued fanout includes a user whose membership was revoked.
  - Specify device synchronization, end-to-end encryption metadata boundaries, abuse reporting, and multi-region ownership transfer without claiming that a WebSocket alone guarantees delivery.
- **Notification platform** (4 prompts)
  - A campaign backlog delays password reset messages by twenty minutes.
  - A request timed out after the provider may have sent the SMS.
  - The user disables promotional email after an intent was queued.
  - Set channel-specific SLOs and per-tenant cost budgets, and rehearse a provider brownout with callbacks arriving after failover.
- **Email service** (4 prompts)
  - A few senders upload 100MB attachments and exhaust spool disk.
  - The SMTP edge returns success before its local spool is replicated.
  - Two devices move and mark a message read using stale mailbox state.
  - Plan mailbox export, delivery-loop protection, per-recipient bounce state, and restore tests that verify both bodies and mailbox indexes.
- **Webhook delivery** (4 prompts)
  - An enterprise endpoint takes 25 seconds per request and blocks unrelated tenants.
  - A receiver commits a database change and then its response path fails.
  - The delete event is delivered before a delayed create event.
  - Make replay preserve original event identity while recording a distinct replay request, and bound customer endpoint access to prevent internal-network targeting.
- **Publish-subscribe event bus** (4 prompts)
  - Twenty consumer groups each read the full billion-event stream.
  - A broken analytics consumer resumes after its oldest required offset was deleted.
  - The consumer writes to a database then crashes before committing its offset.
  - Budget replay traffic separately, govern schema compatibility, and provide tenant-level kill switches that preserve other streams during poison-event incidents.

##### Media

- **YouTube-style video platform** (4 prompts)
  - A worker finishes a 4K encode after its lease was reassigned. Which exact keys and conditional transition prevent stale publication?
  - Show the upload-to-publish state machine and identify when you acknowledge upload completion versus playback readiness.
  - Estimate CDN and origin bandwidth for 20M viewers, 30 minutes/day, 3Mb/s, a 4x peak, and a 97% byte-hit ratio. Which assumptions break for a viral cold video?
  - Explain how HLS/DASH manifests, aligned rendition segments, adaptive bitrate selection, and CDN cache keys interact during version rollout and takedown.
- **Live streaming platform** (4 prompts)
  - A match audience grows from 200K to 5M viewers in ninety seconds.
  - An encoder reconnects with a new timeline and existing players stall.
  - A broadcaster reconnects to another region while the old ingest keeps publishing.
  - Treat live latency, playback availability, recording completeness, and rights enforcement as independent objectives; test failover mid-segment and mid-ad-break.
- **Music streaming** (4 prompts)
  - Ten million listeners start the same album immediately at release.
  - A track loses rights in a territory while long-lived playback tokens remain valid.
  - A phone adds tracks while a laptop reorders an old playlist snapshot.
  - Maintain an auditable rights-decision version and reconcile royalty events by stable playback IDs without counting network retries as extra listens.
- **Video conferencing** (4 prompts)
  - A 2,000-person meeting sends every camera to every participant.
  - Signaling succeeds but participants see no media.
  - A previously valid join credential is reused after host removal.
  - Partition failure domains by room, quantify regional evacuation capacity, and make recording consent and membership policy explicit without putting media through the signaling service.
- **Podcast platform** (4 prompts)
  - Two million feeds are polled every minute although most publish weekly.
  - A feed replaces audio under an existing identifier and users download mixed versions.
  - An old offline device overwrites a newer position with a smaller value.
  - Isolate hostile or malformed feeds with byte, recursion, redirect, and fetch-time limits, and build explainable ingestion status for creators.

##### Storage

- **Cloud file synchronization** (4 prompts)
  - Initial synchronization performs one request per file and takes days.
  - The metadata pointer changes while some chunks are still missing.
  - Both devices upload valid content based on version 12.
  - Define rename atomicity across folders, deletion recovery, shared-folder permission revocation, and safe garbage collection in the presence of offline devices.
- **Distributed object storage** (4 prompts)
  - Losing a rack requires reconstructing petabytes while clients continue reading.
  - One disk returns wrong bytes without a read error.
  - Two uploads target the same key and a reader assembles pieces from both.
  - State consistency separately for object writes, listings, multipart completion, and cross-region replication. Do not assume all object stores or regional copies share identical guarantees.
- **Backup and point-in-time restore** (4 prompts)
  - A 500TB snapshot must be restored over a 10Gb/s link.
  - Snapshots completed successfully, yet a gap prevents replay to the requested time.
  - A cleanup task deletes an incremental base while a restore is reading it.
  - Test compromised credentials, key loss, and region loss as recovery scenarios. A backup success dashboard must show verified recovery points and measured restore time.
- **Content-addressed blob store** (4 prompts)
  - A common dependency receives millions of reference additions per minute.
  - An unreferenced blob is selected for deletion just before a new tenant links it.
  - A tenant can probe whether another tenant uploaded a known file.
  - Prove collection safety under concurrent references, delayed replicas, interrupted sweeps, and hash-verification failures; measure storage saved after accounting for metadata and replication.
- **Distributed filesystem** (4 prompts)
  - A billion tiny files fit on disks but exhaust metadata servers.
  - A paused writer resumes after a new client became the file owner.
  - Moving a directory between shards must not expose it twice or lose it.
  - Write down the exact POSIX subset, append semantics, client-cache invalidation rules, and behavior during metadata quorum loss before drawing data-node boxes.

##### Commerce

- **Commerce checkout** (4 prompts)
  - Most checkouts target one scarce SKU while the rest of the store is healthy.
  - The provider captured funds while the checkout process crashed before recording success.
  - The inventory hold expires while a delayed payment success arrives.
  - Measure unknown payments, reservation age, refund backlog, and reconciliation lag alongside checkout success. Write operational playbooks that preserve evidence for ambiguous money movement.
- **Inventory reservation** (4 prompts)
  - Thousands of buyers contend for the last thousand units.
  - A hold is committed while an expiry worker also releases it.
  - A duplicated shipment event deducts stock twice.
  - Define invariants for damaged goods, returns, transfers in flight, backorders, and manual adjustments instead of treating stock as one mutable integer.
- **Payment ledger** (4 prompts)
  - Every merchant payment updates the same platform settlement balance.
  - A client retries after a lost response and creates duplicate debit-credit pairs.
  - A multi-currency operation sums decimal approximations across entries.
  - Separate ledger truth, available balance, provider settlement, and bank reconciliation. Treat recovery, access control, and immutable audit evidence as design requirements rather than afterthoughts.
- **Ticket booking** (4 prompts)
  - Three million waiting users poll the full seat map every second.
  - An old hold-expiry task runs after payment confirmation.
  - Each group obtains some seats and neither can complete the requested block.
  - Model bot resistance, waiting-room token replay, queue position continuity during failover, and the user-visible policy for ambiguous payment completion.
- **Marketplace orders** (4 prompts)
  - One seller integration is unavailable while other items are ready to ship.
  - A delayed in-transit callback arrives after delivered and triggers another payout hold.
  - A returned item becomes refundable just as its seller payout is released.
  - Define operational ownership for stuck states and disputed deliveries, and make seller-specific failures visible without exposing unrelated customer data.

##### Real time

- **Ride dispatch** (4 prompts)
  - Thousands of riders and drivers concentrate in one geospatial bucket.
  - A driver went offline but remains in the nearby-driver index.
  - Two matchers choose the same available driver from a stale index.
  - Prove assignment safety across retries and regional ownership changes, and distinguish dispatch latency from location freshness and driver acceptance rate.
- **Collaborative document editor** (4 prompts)
  - Opening a document replays five million edits and freezes the client.
  - Two edits at the same position produce different text depending on arrival order.
  - A former collaborator reconnects with edits made while disconnected.
  - Specify supported edit semantics, undo behavior across collaborators, offline horizon, and snapshot compatibility before choosing a merge algorithm.
- **Presence and typing indicators** (4 prompts)
  - A regional network outage ends and 10M devices reconnect at once.
  - A mobile device loses network without closing its socket.
  - A delayed close event for session A marks session B offline.
  - Budget fanout by watchers and privacy rules, not just connected users, and isolate presence outages from message delivery.
- **Game leaderboard** (4 prompts)
  - Fifty million players and constant writes overload one ranking shard.
  - A consumer replay applies the same match result twice.
  - A valid match ends before midnight but arrives after the season is finalized.
  - Define tie-breaking, anti-cheat review, prize finality, and late correction policy as part of ranking correctness.
- **Fleet tracking** (4 prompts)
  - A large customer opens a world map containing one million vehicles.
  - A parked truck near a boundary generates dozens of geofence transitions.
  - A device reconnects and uploads yesterday’s points after a fresh live update.
  - Define clock-drift tolerance, device identity reset, tenant isolation, and the difference between real-time safety alerts and best-effort business notifications.

##### Search

- **Web search engine** (4 prompts)
  - A short query matches billions of postings and exhausts tail latency.
  - A site generates unlimited distinct URLs with equivalent low-value pages.
  - The source is gone but an old index generation still serves it.
  - Set budgets for crawling, indexing freshness, query tail latency, and removal propagation, and plan index-generation rollback with durable tombstone preservation.
- **Search autocomplete** (4 prompts)
  - Fast typists create six requests for one intended search.
  - A moderation update removes a candidate but old prefix snapshots still include it.
  - Suggestions cached only by prefix include a previous user’s private searches.
  - Specify minimum cohort thresholds, retention, locale normalization, and emergency removals before using raw search logs as a suggestion source.
- **Product search** (4 prompts)
  - A request aggregates millions of distinct seller and attribute combinations.
  - Index lag leaves an old sale price visible after the catalog changed.
  - A slow full reindex writes version 40 after live ingestion wrote version 44.
  - Define a freshness budget per field: descriptions can lag longer than price or availability. Measure zero-result rate and business correctness separately from query latency.
- **Log ingestion and search** (4 prompts)
  - A failing service emits the same stack trace millions of times per second.
  - User-supplied JSON creates a new field name for every request ID.
  - An offline agent uploads events older than the hot-retention boundary.
  - Budget bytes, cardinality, query CPU, and result size per tenant, and provide operationally useful degradation during the exact incidents when logs surge.
- **Vector retrieval service** (4 prompts)
  - Approximate search retrieves nearby vectors that are mostly outside the requested tenant or category.
  - New documents use a different model while queries still use the old model.
  - The vector index contains an old ACL and passes private text to a downstream model.
  - Evaluate semantic quality, filtered recall, freshness, deletion propagation, and per-query cost independently; never treat similarity score as proof of factual correctness.

##### Infrastructure

- **Global load balancer** (4 prompts)
  - A failed region’s entire traffic is redirected to a region with only 30% spare capacity.
  - A shared dependency makes a deep health endpoint fail on all otherwise useful servers.
  - A deployment removes a backend while WebSocket sessions still rely on it.
  - Define overload policy before failover, test correlated health-check failures, and include long-lived connection migration in regional evacuation drills.
- **Authoritative DNS** (4 prompts)
  - One popular or attacked name attracts millions of queries per second.
  - A configuration update accidentally removes an essential record.
  - A record is corrected but recursive resolvers still return the previous answer.
  - Test DNSSEC signing rollover where used, negative-cache behavior, delegation errors, and recovery from a control-plane compromise without disrupting stable serving.
- **Metrics and alerting** (4 prompts)
  - An instrumentation change adds request_id as a metric label.
  - Ingestion works but no rules are evaluated for twenty minutes.
  - A network partition makes an error-rate alert falsely report healthy traffic.
  - Design alert ownership, deduplication, inhibition, and evaluation continuity during regional failover; monitor the monitor through an independent failure domain.
- **Feature flag platform** (4 prompts)
  - Ten billion evaluations per day call the flag server synchronously.
  - A malformed rule crashes an old SDK version.
  - A 10% rollout uses a random number on every evaluation.
  - Define stale-config limits, defaults during startup, audit and approval ownership, and the maximum supported offline period for each class of flag.
- **Service discovery and configuration** (4 prompts)
  - Restarting 100K instances sends endpoint updates to every client.
  - A disconnected client requests events older than retained history.
  - A partitioned former leader continues sending updates after a new leader is elected.
  - Test quorum loss, watch compaction, mass reconnects, and certificate rotation together; control-plane recovery must not trigger a larger data-plane outage.

##### Advanced

- **Multi-region key-value store** (4 prompts)
  - One tenant key prefix generates 40% of global write traffic.
  - The primary region fails before its replication backlog reaches the standby.
  - Two regions update the same shopping preference while disconnected.
  - State the permitted anomaly for each operation, prove fencing during failover, and test restoration of an old region without resurrecting deleted data.
- **Durable workflow engine** (4 prompts)
  - A workflow has accumulated a million events and replay takes minutes.
  - New workflow code takes a different branch while replaying an old history.
  - A booking workflow needs to refund payment after hotel reservation fails, but refunds are unavailable.
  - Design version retirement, operator intervention, workflow cancellation, and data retention for processes that outlive several application releases.
- **Ad auction and budget pacing** (4 prompts)
  - One campaign participates in millions of auctions across regions.
  - The system allocates new ads before earlier impressions are reported as billable.
  - A client retries impression tracking after a lost response.
  - Separate auction latency, relevance, billable-event accuracy, advertiser budget safety, and fraud detection. Explain which accounting decisions are provisional.
- **Recommendation serving** (4 prompts)
  - The ranker waits 120ms for a rarely useful feature on every request.
  - Offline evaluation improves while production recommendations regress.
  - A request times out but its assigned treatment is logged as if the user saw the recommendations.
  - Evaluate quality, latency, feature freshness, policy compliance, experiment validity, and cost together; maintain a useful fallback that can run when the feature platform is unavailable.
- **Streaming fraud detection** (4 prompts)
  - A mobile carrier NAT causes millions of users to share one IP feature bucket.
  - A deployment resets rolling velocity counts and risky transactions are approved.
  - An old device event arrives after the original transaction was approved.
  - Separate detection quality from financial loss and customer friction; document model versions, fallback decisions, appeals, and the operational response to a stale feature pipeline.

<a id="map"></a>

### Pattern map (8 mappings)

Eight requirement-to-pattern-to-technology mappings shown below.

| Requirement | Pattern | Example technology |
| --- | --- | --- |
| Real-time, bidirectional updates | WebSocket | Connection gateway + Pub/Sub |
| High read throughput | Cache Aside | Redis + durable database |
| Large media objects | Object Storage | S3-compatible object storage |
| Global static and media delivery | CDN | Edge cache + origin |
| Work can happen later | Work Queue | Queue + asynchronous workers |
| Search across documents | Search Inverted Index | Elasticsearch / Lucene |
| An event may arrive twice | Idempotent Consumer | Deduplication record + transaction |
| A transaction spans services | Saga | Workflow + compensation |

## Shared tools

Global search spans library titles, categories, and saved notes. Workspace settings cover the interview track, session length, appearance, and backup or restore. Topic pages also provide scratchpads, bookmarks, and handwritten notes. These tools support the sections above and have no separate fixed topic catalogs.
