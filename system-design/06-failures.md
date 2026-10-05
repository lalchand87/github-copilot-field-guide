# Failure Gym — Break my system

[← System Design index](README.md)

> 50 incidents. For each: the architecture, then impact, detection, mitigation, prevention and staff-level recovery.

## Contents

- **Foundational** (5): [Negative cache hides a new alias](#negative-cache-hides-a-new-alias) · [Limiter store is unreachable](#limiter-store-is-unreachable) · [Mass expiry stampede](#mass-expiry-stampede) · [Clock moves backwards](#clock-moves-backwards) · [Worker dies after sending a report](#worker-dies-after-sending-a-report)
- **Social** (5): [Feed contains a deleted post](#feed-contains-a-deleted-post) · [Incomplete image marked ready](#incomplete-image-marked-ready) · [Follower counts become negative](#follower-counts-become-negative) · [Moderation disappears after an edit retry](#moderation-disappears-after-an-edit-retry) · [Expired story remains playable](#expired-story-remains-playable)
- **Messaging** (5): [Messages vanish after reconnect](#messages-vanish-after-reconnect) · [SMS provider accepted an ambiguous request](#sms-provider-accepted-an-ambiguous-request) · [Accepted mail disappears during a crash](#accepted-mail-disappears-during-a-crash) · [Receiver processed the event but returned 500](#receiver-processed-the-event-but-returned-500) · [Consumer is behind retention](#consumer-is-behind-retention)
- **Media** (5): [A manifest advertises missing segments](#a-manifest-advertises-missing-segments) · [Encoder restart resets timestamps](#encoder-restart-resets-timestamps) · [Rights change but cached playback remains authorized](#rights-change-but-cached-playback-remains-authorized) · [UDP is blocked on a corporate network](#udp-is-blocked-on-a-corporate-network) · [A publisher reuses an episode GUID](#a-publisher-reuses-an-episode-guid)
- **Storage** (5): [A partial upload becomes the current version](#a-partial-upload-becomes-the-current-version) · [Silent corruption passes through replication](#silent-corruption-passes-through-replication) · [Backup is green but missing a log segment](#backup-is-green-but-missing-a-log-segment) · [Garbage collector races with a new reference](#garbage-collector-races-with-a-new-reference) · [A stale client writes after lease transfer](#a-stale-client-writes-after-lease-transfer)
- **Commerce** (5): [Payment succeeded but the order timed out](#payment-succeeded-but-the-order-timed-out) · [Expiry and checkout both decrement reservations](#expiry-and-checkout-both-decrement-reservations) · [Retry posts the same transfer twice](#retry-posts-the-same-transfer-twice) · [A timer releases a newly confirmed seat](#a-timer-releases-a-newly-confirmed-seat) · [Shipment callback moves an item backwards](#shipment-callback-moves-an-item-backwards)
- **Real Time** (5): [Stale locations produce impossible pickups](#stale-locations-produce-impossible-pickups) · [Concurrent insertions diverge across clients](#concurrent-insertions-diverge-across-clients) · [User remains online after a silent disconnect](#user-remains-online-after-a-silent-disconnect) · [Replayed score event doubles a win](#replayed-score-event-doubles-a-win) · [GPS jitter creates repeated enter-exit alerts](#gps-jitter-creates-repeated-enter-exit-alerts)
- **Search** (5): [Crawler falls into an infinite calendar](#crawler-falls-into-an-infinite-calendar) · [A blocked suggestion persists in cached lists](#a-blocked-suggestion-persists-in-cached-lists) · [Search advertises yesterday’s price](#search-advertises-yesterdays-price) · [Mapping explosion takes down indexing](#mapping-explosion-takes-down-indexing) · [Embedding upgrade mixes incompatible spaces](#embedding-upgrade-mixes-incompatible-spaces)
- **Infrastructure** (5): [Health checks remove every backend](#health-checks-remove-every-backend) · [Bad zone version reaches every region](#bad-zone-version-reaches-every-region) · [Alert evaluator fails silently](#alert-evaluator-fails-silently) · [Bad rollout config reaches all clients](#bad-rollout-config-reaches-all-clients) · [Watcher resumes from a compacted revision](#watcher-resumes-from-a-compacted-revision)
- **Advanced** (5): [Async failover loses acknowledged recent writes](#async-failover-loses-acknowledged-recent-writes) · [Deployment makes replay nondeterministic](#deployment-makes-replay-nondeterministic) · [Delayed impressions overshoot the budget](#delayed-impressions-overshoot-the-budget) · [Training and serving use different feature definitions](#training-and-serving-use-different-feature-definitions) · [Processor restarts and forgets window state](#processor-restarts-and-forgets-window-state)

## Foundational

### Negative cache hides a new alias

**Incident:** URL shortener: An unavailable alias was cached as missing before its owner created it.

**Architecture:** `Browser` → `Edge cache` → `Redirect API` → `Redis hot links` → `Durable link store` → `Async click log`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Custom aliases and expiry; Redirect without synchronous analytics; Prevent alias collisions. An unavailable alias was cached as missing before its owner created it. |
| Detection | Measure creation-to-redirect visibility and negative-cache hits for aliases that now exist in the link store; compare cache age with link creation time. |
| Mitigation | Invalidate the negative entry after the durable insert, use a short negative TTL, and optionally bypass cache for the owner after creation. Track negative-hit rate and creation-to-visibility delay. |
| Prevention | Separate positive and negative TTL policies, invalidate after a successful create, and exercise a create-versus-negative-refill race. |

**Staff-level recovery:** Model abuse takedown propagation separately from ordinary edits; establish a maximum revocation delay and test cache invalidation under partial regional failure.

---

### Limiter store is unreachable

**Incident:** Distributed rate limiter: The bucket datastore times out while the payment API remains healthy.

**Architecture:** `API gateway` → `Policy cache` → `Regional limiter` → `Atomic bucket store` → `Usage metrics`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Per-tenant limits; Weighted expensive operations; Return retry timing. The bucket datastore times out while the payment API remains healthy. |
| Detection | Alert on limiter timeout rate, fallback decisions by route, emergency-budget exhaustion, and downstream load while limiter availability drops. |
| Mitigation | Use conservative local emergency budgets for essential reads; reject expensive writes if their budget cannot be established. Avoid unlimited fail-open traffic and record every fallback decision. |
| Prevention | Load-test route-specific fallback budgets, isolate policy storage from hot counters, and rehearse loss of the limiter store without allowing unlimited traffic. |

**Staff-level recovery:** Define fairness during regional evacuation and prevent one tenant from monopolizing the emergency budget. Roll out policy changes with explicit version semantics.

---

### Mass expiry stampede

**Incident:** Distributed cache: A batch import gives ten million popular keys the same expiry.

**Architecture:** `Application` → `Client routing map` → `Cache shards` → `Read-through loader` → `Source database`

| Step | What to do |
|---|---|
| Impact | The violated contract is: TTL and eviction; Horizontal redistribution; Bound stale reads. A batch import gives ten million popular keys the same expiry. |
| Detection | Correlate expiry histograms, miss QPS, concurrent loaders per key, source connection-pool waits, and database p99 latency. |
| Mitigation | Add TTL jitter, single-flight refresh, and bounded stale-while-revalidate for permitted data. Gate refill concurrency against source capacity and shed cacheable traffic before overwhelming the database. |
| Prevention | Jitter expirations at write time, enable per-key refresh coalescing, reserve source headroom, and test a cold-cache restart at peak traffic. |

**Staff-level recovery:** Size for failover with one shard unavailable, including connection storms, allocator overhead, replication buffers, and the source load from a cold replacement.

---

### Clock moves backwards

**Incident:** Unique ID service: A host clock steps backwards by three seconds after synchronization.

**Architecture:** `Client library` → `Worker identity registry` → `Local timestamp and sequence generator` → `Lease monitor`

| Step | What to do |
|---|---|
| Impact | The violated contract is: No duplicate IDs; Predictable bit budget; Tolerate worker restarts. A host clock steps backwards by three seconds after synchronization. |
| Detection | Record negative clock deltas and generation stalls; sample timestamp fields and watch duplicate-key constraint violations as a last-resort signal. |
| Mitigation | Refuse or pause generation until time catches up, or use a persisted logical timestamp strategy with a documented skew budget. Alert on clock regressions; never reuse a timestamp-sequence pair. |
| Prevention | Use conservative clock-regression handling, monitor synchronization, persist safety state where required, and test VM snapshot restoration and worker-identity reuse. |

**Staff-level recovery:** Prove uniqueness across clock rollback, restored VM snapshots, identity reuse, and disaster recovery. Include a documented migration before the timestamp field expires.

---

### Worker dies after sending a report

**Incident:** Durable job scheduler: The email side effect succeeded but the attempt acknowledgement was lost.

**Architecture:** `Schedule API` → `Time-partitioned job store` → `Due-job scanners` → `Ready queue` → `Leased workers` → `Result store`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Schedule and cancel jobs; Retry transient failures; Isolate tenants. The email side effect succeeded but the attempt acknowledgement was lost. |
| Detection | Compare durable attempt status with provider delivery IDs; alert on ambiguous outcomes, duplicate delivery keys, and jobs stuck beyond the provider deadline. |
| Mitigation | Record a stable delivery key per job occurrence and pass it to an idempotent destination when available. Reconcile ambiguous outcomes before resending; represent unknown status if the provider cannot deduplicate. |
| Prevention | Persist delivery identity before the side effect, require destination deduplication where possible, and inject a crash immediately after provider success. |

**Staff-level recovery:** Specify misfire behavior for missed recurring schedules, timezones, daylight-saving transitions, and prolonged outages; avoid replaying years of missed jobs on recovery.

---

## Social

### Feed contains a deleted post

**Incident:** Social news feed: A cached inbox still references content removed by moderation.

**Architecture:** `Post API` → `Post store and outbox` → `Fanout workers` → `Feed inbox store` → `Ranking service` → `Feed API`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Publish posts; Cursor pagination; Enforce privacy at read time. A cached inbox still references content removed by moderation. |
| Detection | Probe removed post IDs through feed endpoints and CDN variants; measure removal-to-last-visible latency and filtered-candidate rates. |
| Mitigation | Filter candidates against current deletion and access state before rendering, propagate tombstones to caches, and backfill if filtering leaves a short page. Purge materialized entries asynchronously. |
| Prevention | Keep durable tombstones and a final visibility filter, prioritize removal propagation, and test feeds assembled from deliberately stale inboxes. |

**Staff-level recovery:** Set separate SLOs for post acceptance, follower visibility, privacy revocation, and ranking freshness; a single availability number hides the important failures.

---

### Incomplete image marked ready

**Incident:** Photo sharing: An object event arrives before image validation and the gallery publishes it.

**Architecture:** `Mobile client` → `Upload signer` → `Object storage` → `Transform queue` → `Image workers` → `CDN` → `Gallery API`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Resumable upload; Thumbnail generation; Private albums and deletion. An object event arrives before image validation and the gallery publishes it. |
| Detection | Track published photos with missing required variants, decode failures, checksum mismatches, and time spent in each upload state. |
| Mitigation | Validate format, dimensions, checksums, and required variants before atomically publishing ready metadata. Keep uploads staged and sweep abandoned objects; the gallery reads only ready records. |
| Prevention | Separate uploaded, validated, transformed, and ready states; validate before publication and test delayed, duplicate, and reordered object events. |

**Staff-level recovery:** Coordinate deletion of originals, variants, caches, and backup-retention records; expose deletion progress without promising that a metadata delete erases every byte instantly.

---

### Follower counts become negative

**Incident:** Follow graph: Unfollow retries decrement counters even when no edge was removed.

**Architecture:** `Relationship API` → `Authoritative outgoing edges` → `Change log` → `Incoming-edge projection` → `Count cache`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Follow and unfollow; List followers; Block accounts. Unfollow retries decrement counters even when no edge was removed. |
| Detection | Alert on impossible negative counts and drift between sampled authoritative edges and count projections; inspect duplicate transition IDs. |
| Mitigation | Make edge mutations conditional and emit a transition event only when state changes. Deduplicate projection events, reconcile counts from edge partitions, and clamp display only as a temporary presentation safeguard. |
| Prevention | Emit counter changes only from committed edge transitions, deduplicate projections, and schedule bounded reconciliation against authoritative edge slices. |

**Staff-level recovery:** Design repair jobs that compare projection checkpoints and sample edge symmetry without loading the complete graph into memory.

---

### Moderation disappears after an edit retry

**Incident:** Threaded comments: An old body-edit request overwrites a newer hidden state.

**Architecture:** `Comment API` → `Comment store` → `Moderation log` → `Thread index` → `Score projection` → `Thread cache`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Post and edit replies; Sort by score or time; Hide deleted content. An old body-edit request overwrites a newer hidden state. |
| Detection | Audit moderation-state version regressions and probe hidden comment IDs after edit retries; correlate writes by actor and expected version. |
| Mitigation | Use optimistic concurrency and field-specific updates; keep moderation state independently versioned and enforce it in rendering. Reject outdated edits and retain an audit trail of state transitions. |
| Prevention | Separate moderation ownership from body edits, reject stale versions, and run a concurrent edit-and-hide test against the exact write API. |

**Staff-level recovery:** Distinguish author deletion, moderator hiding, legal removal, and thread locking, since each has different visibility and audit requirements.

---

### Expired story remains playable

**Incident:** Ephemeral stories: A cleanup worker is late, leaving an expired object in storage and cache.

**Architecture:** `Story API` → `Story metadata` → `Media object store` → `Audience filter` → `CDN` → `View-event pipeline`

| Step | What to do |
|---|---|
| Impact | The violated contract is: 24-hour visibility; Audience rules; View receipts. A cleanup worker is late, leaving an expired object in storage and cache. |
| Detection | Continuously request expired story metadata and media access; measure successful post-expiry authorizations and stale CDN responses separately. |
| Mitigation | Enforce expires_at when issuing playback access and use access lifetimes no longer than remaining story life. Purge asynchronously, but deny new serving authorization after expiry even when bytes still exist. |
| Prevention | Enforce expiry at authorization time and cap access-token lifetime by remaining story life; make cleanup delays irrelevant to new playback eligibility. |

**Staff-level recovery:** Separate the promised visibility deadline from physical deletion and backup expiration, and measure each boundary independently.

---

## Messaging

### Messages vanish after reconnect

**Incident:** Direct and group chat: A gateway acknowledged receipt before durable storage and crashed.

**Architecture:** `Client` → `WebSocket gateway` → `Conversation owner` → `Durable message log` → `Message store` → `Recipient fanout` → `Push fallback`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Conversation ordering; Offline sync; Delivery and read receipts. A gateway acknowledged receipt before durable storage and crashed. |
| Detection | Compare client-acknowledged message IDs with durable conversation sequences and monitor reconnect gap reports and acceptance-to-persistence timing. |
| Mitigation | Acknowledge acceptance only after durable append; use clientMessageId for retry deduplication. On reconnect, fetch from the last durable sequence and deduplicate live deliveries against replay. |
| Prevention | Tie acceptance acknowledgements to durable append, retain stable client message IDs, and kill gateways between receive, append, and acknowledgement. |

**Staff-level recovery:** Specify device synchronization, end-to-end encryption metadata boundaries, abuse reporting, and multi-region ownership transfer without claiming that a WebSocket alone guarantees delivery.

---

### SMS provider accepted an ambiguous request

**Incident:** Notification platform: A request timed out after the provider may have sent the SMS.

**Architecture:** `Event producers` → `Intent log` → `Preference evaluator` → `Channel queues` → `Provider adapters` → `Callback processor`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Preferences and quiet hours; Per-channel retries; Durable delivery status. A request timed out after the provider may have sent the SMS. |
| Detection | Track provider timeouts by attempt key, callback lag, ambiguous delivery count, and duplicate provider IDs per notification intent. |
| Mitigation | Retry with the same provider idempotency key where supported and reconcile using delivery lookup or callbacks. Otherwise mark the attempt uncertain and apply a product-specific duplicate-versus-loss policy. |
| Prevention | Persist provider request identity and a reconciliation state before submission; rehearse late callbacks after retry and provider failover. |

**Staff-level recovery:** Set channel-specific SLOs and per-tenant cost budgets, and rehearse a provider brownout with callbacks arriving after failover.

---

### Accepted mail disappears during a crash

**Incident:** Email service: The SMTP edge returns success before its local spool is replicated.

**Architecture:** `SMTP ingress` → `Durable spool` → `Spam scanning` → `Blob storage` → `Mailbox index` → `Search projection` → `Delivery workers`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Durable acceptance; Attachments; Mailbox folders and search. The SMTP edge returns success before its local spool is replicated. |
| Detection | Compare accepted SMTP transaction IDs against replicated spool records and end-to-end synthetic inbox deliveries after node loss. |
| Mitigation | Acknowledge only after the configured durable replicated write, then perform routing asynchronously. Recover from spool checkpoints and deduplicate delivery by message and recipient identifiers. |
| Prevention | Document the exact acceptance durability boundary, fail admission when it cannot be met, and test power loss immediately after an accepted response. |

**Staff-level recovery:** Plan mailbox export, delivery-loop protection, per-recipient bounce state, and restore tests that verify both bodies and mailbox indexes.

---

### Receiver processed the event but returned 500

**Incident:** Webhook delivery: A receiver commits a database change and then its response path fails.

**Architecture:** `Application outbox` → `Event log` → `Subscription router` → `Per-endpoint queues` → `HTTP workers` → `Retry scheduler`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Durable events; Endpoint-specific retries; Replay and signing rotation. A receiver commits a database change and then its response path fails. |
| Detection | Correlate repeated stable event IDs with receiver status and delivery-attempt history; monitor retry age and endpoint-specific error patterns. |
| Mitigation | Retry the same event ID with a new attempt ID, document at-least-once delivery, and provide signing over immutable event bytes plus timestamp. Receivers must deduplicate durable effects by event ID. |
| Prevention | Document at-least-once delivery, keep event identity stable, provide a receiver dedupe example, and test success followed by an intentionally failed response. |

**Staff-level recovery:** Make replay preserve original event identity while recording a distinct replay request, and bound customer endpoint access to prevent internal-network targeting.

---

### Consumer is behind retention

**Incident:** Publish-subscribe event bus: A broken analytics consumer resumes after its oldest required offset was deleted.

**Architecture:** `Producers` → `Partition routers` → `Replicated log brokers` → `Consumer groups` → `Sink adapters` → `Checkpoint store`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Partition ordering; Replay retention; Backpressure and tenant quotas. A broken analytics consumer resumes after its oldest required offset was deleted. |
| Detection | Alert before consumer lag age approaches retention; compare requested offsets with each partition earliest available offset. |
| Mitigation | Detect offset-out-of-range explicitly, restore a consistent snapshot, and replay from its associated checkpoint. Do not silently skip to newest; publish data-loss or rebuild status to downstream owners. |
| Prevention | Set lag-age budgets, retain recoverable source snapshots with checkpoints, and rehearse a full consumer rebuild instead of only broker failover. |

**Staff-level recovery:** Budget replay traffic separately, govern schema compatibility, and provide tenant-level kill switches that preserve other streams during poison-event incidents.

---

## Media

### A manifest advertises missing segments

**Incident:** YouTube-style video platform: A packager updates the published manifest before all rendition segments are durable and validated.

**Architecture:** `Client multipart uploader` → `Signed upload API` → `Staging object storage` → `Completion validator` → `Durable workflow and queues` → `Probe and moderation stages` → `Transcode workers by codec and resolution` → `HLS/DASH packager` → `Asset validator` → `Atomic publication metadata` → `Origin shield and CDN` → `Adaptive player` → `QoE telemetry and watch-event pipeline`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Resumable multipart uploads; HLS and DASH adaptive playback; Search, subscriptions, and watch history; Publish only validated playable assets. A packager updates the published manifest before all rendition segments are durable and validated. |
| Detection | Probe every manifest rendition in a canary player; alert on segment 404s, timeline gaps, decode errors, startup failures, and publication-to-asset-validation ordering. |
| Mitigation | Write outputs under immutable video/source/recipe/rendition keys. Validate checksums, codec metadata, aligned segment timelines, and required assets, then conditionally promote a publication pointer. Keep the previous complete version available. For live playback, publish only segments that meet the availability contract and distinguish manifest TTL from immutable segment TTL. |
| Prevention | Promote only a fully validated immutable asset generation, fence late workers, and test packaging crashes immediately before and after publication. |

**Staff-level recovery:** Separate upload, processing, playback-start, rebuffering, and deletion SLOs. Compare regional encoding cost, CDN egress, and origin shielding; plan takedown propagation through authorization, manifests, caches, search, and derived clips.

---

### Encoder restart resets timestamps

**Incident:** Live streaming platform: An encoder reconnects with a new timeline and existing players stall.

**Architecture:** `Broadcaster` → `Regional ingest` → `Transcoder` → `Live packager` → `Segment origin` → `CDN` → `Player` → `Recording assembler`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Authenticated ingest; Live manifest updates; DVR window and recording. An encoder reconnects with a new timeline and existing players stall. |
| Detection | Monitor timestamp discontinuities, segment sequence regressions, live-edge drift, rebuffer ratio, and player errors by ingest epoch. |
| Mitigation | Start a new ingest epoch, emit protocol-appropriate discontinuity or period boundaries, and preserve coherent sequence and timing metadata. Validate with players across codecs; keep DVR references tied to immutable segment identities. |
| Prevention | Validate timeline continuity on reconnect, encode epoch changes into packaging boundaries, and test broadcaster restarts with real player implementations. |

**Staff-level recovery:** Treat live latency, playback availability, recording completeness, and rights enforcement as independent objectives; test failover mid-segment and mid-ad-break.

---

### Rights change but cached playback remains authorized

**Incident:** Music streaming: A track loses rights in a territory while long-lived playback tokens remain valid.

**Architecture:** `Client` → `Entitlement API` → `Catalog metadata` → `Audio CDN` → `Playlist service` → `Playback-event log`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Low-latency track start; Playlist edits; Territory and subscription rules. A track loses rights in a territory while long-lived playback tokens remain valid. |
| Detection | Run territory-specific authorization probes after rights changes and measure tokens accepted beyond their rights-version or expiry policy. |
| Mitigation | Issue short-lived playback authorization tied to rights version and region, recheck on renewal, and invalidate urgent removals. Keep immutable audio caching efficient while applying access controls at the serving boundary. |
| Prevention | Decouple audio cache TTL from entitlement validity, version rights decisions, and rehearse urgent removal with old tokens and warm CDN caches. |

**Staff-level recovery:** Maintain an auditable rights-decision version and reconcile royalty events by stable playback IDs without counting network retries as extra listens.

---

### UDP is blocked on a corporate network

**Incident:** Video conferencing: Signaling succeeds but participants see no media.

**Architecture:** `Browser` → `Signaling service` → `Room allocator` → `Regional SFU` → `STUN/TURN support` → `Optional recording workers`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Join and leave rooms; Screen share; Recover from packet loss and reconnects. Signaling succeeds but participants see no media. |
| Detection | Break connection setup into signaling, ICE gathering, candidate selection, DTLS, and first-media timing; monitor TURN allocation and relay bandwidth. |
| Mitigation | Use ICE connectivity checks with provisioned TURN fallback, track relay selection and connection setup stages, and reduce bitrate on constrained paths. Show connection recovery status rather than retrying room creation. |
| Prevention | Provision and load-test relay fallback, test restrictive corporate-network profiles, and keep connectivity recovery independent of room recreation. |

**Staff-level recovery:** Partition failure domains by room, quantify regional evacuation capacity, and make recording consent and membership policy explicit without putting media through the signaling service.

---

### A publisher reuses an episode GUID

**Incident:** Podcast platform: A feed replaces audio under an existing identifier and users download mixed versions.

**Architecture:** `Feed scheduler` → `Conditional HTTP fetchers` → `Parser and validator` → `Episode catalog` → `Audio origin/CDN` → `Progress service`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Feed ingestion; Episode deduplication; Progress sync and downloads. A feed replaces audio under an existing identifier and users download mixed versions. |
| Detection | Detect changes in enclosure size, validators, or content checksum for a stable GUID; monitor resumed downloads that fail validation. |
| Mitigation | Track GUID plus observed enclosure metadata and content version, validate changed media, and publish a new asset version deliberately. Keep existing range downloads pinned to their original immutable object. |
| Prevention | Version observed media assets immutably, validate publisher replacements, and test GUID reuse during an in-progress range download. |

**Staff-level recovery:** Isolate hostile or malformed feeds with byte, recursion, redirect, and fetch-time limits, and build explainable ingestion status for creators.

---

## Storage

### A partial upload becomes the current version

**Incident:** Cloud file synchronization: The metadata pointer changes while some chunks are still missing.

**Architecture:** `Desktop client` → `Metadata API` → `Chunk object store` → `Version transaction` → `Change journal` → `Sync notifications`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Resumable uploads; Offline edits; Recoverable version history. The metadata pointer changes while some chunks are still missing. |
| Detection | Probe current manifests for absent or corrupt chunks and compare publication timestamps with chunk-validation records. |
| Mitigation | Stage chunks and verify their identities, then atomically commit a version manifest and current pointer. Readers see only complete versions; a garbage collector removes unreferenced staged chunks after a grace period. |
| Prevention | Require verified complete manifests for atomic version promotion, retain previous versions, and inject missing chunks before completion. |

**Staff-level recovery:** Define rename atomicity across folders, deletion recovery, shared-folder permission revocation, and safe garbage collection in the presence of offline devices.

---

### Silent corruption passes through replication

**Incident:** Distributed object storage: One disk returns wrong bytes without a read error.

**Architecture:** `Object gateway` → `Metadata quorum` → `Placement service` → `Storage nodes` → `Integrity scrubbers` → `Repair workers`

| Step | What to do |
|---|---|
| Impact | The violated contract is: PUT/GET/range reads; Checksums and versions; Repair after disk loss. One disk returns wrong bytes without a read error. |
| Detection | Track checksum failures by disk and stripe, repair success, unreadable-object count, and age since last scrub for cold data. |
| Mitigation | Verify end-to-end object or fragment checksums on write and read, retrieve a healthy replica or reconstruct from parity, and quarantine the bad copy. Schedule scrubbing so cold corruption is found before another failure. |
| Prevention | Verify checksums end to end, scrub on a bounded schedule, separate corruption from transport errors, and test recovery with one corrupt fragment plus one missing fragment. |

**Staff-level recovery:** State consistency separately for object writes, listings, multipart completion, and cross-region replication. Do not assume all object stores or regional copies share identical guarantees.

---

### Backup is green but missing a log segment

**Incident:** Backup and point-in-time restore: Snapshots completed successfully, yet a gap prevents replay to the requested time.

**Architecture:** `Database snapshotter` → `Change-log archiver` → `Immutable backup storage` → `Catalog` → `Restore coordinator` → `Verification runner`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Point-in-time restore; Encryption and retention; Verified recovery. Snapshots completed successfully, yet a gap prevents replay to the requested time. |
| Detection | Continuously verify log-sequence continuity and report the newest verified recoverable timestamp; run application checks on sampled restored snapshots. |
| Mitigation | Validate checkpoint-to-log continuity continuously, mark the recoverable time range precisely, and restore to the last verified point if a gap cannot be repaired. Run automated restore drills with application-level checks. |
| Prevention | Gate backup health on a complete recoverable chain, preserve checkpoint metadata, and perform timed restores with injected archive gaps. |

**Staff-level recovery:** Test compromised credentials, key loss, and region loss as recovery scenarios. A backup success dashboard must show verified recovery points and measured restore time.

---

### Garbage collector races with a new reference

**Incident:** Content-addressed blob store: An unreferenced blob is selected for deletion just before a new tenant links it.

**Architecture:** `Upload API` → `Tenant authorization` → `Digest verifier` → `Blob object storage` → `Reference database` → `Mark-and-sweep collector`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Hash-verified uploads; Reference tracking; Safe garbage collection. An unreferenced blob is selected for deletion just before a new tenant links it. |
| Detection | Audit deleted digests against live reference generations and watch missing-blob errors after new references or collector runs. |
| Mitigation | Mark candidates, delay physical removal, and atomically recheck the reference generation before final deletion. New references either cancel deletion or require a verified re-upload; never trust a stale zero count. |
| Prevention | Use generation-checked two-phase collection with a grace period and test a new reference inserted between marking and physical deletion. |

**Staff-level recovery:** Prove collection safety under concurrent references, delayed replicas, interrupted sweeps, and hash-verification failures; measure storage saved after accounting for metadata and replication.

---

### A stale client writes after lease transfer

**Incident:** Distributed filesystem: A paused writer resumes after a new client became the file owner.

**Architecture:** `Filesystem client` → `Namespace service` → `Metadata consensus group` → `Chunk placement` → `Data nodes` → `Lease and repair services`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Atomic namespace operations; Chunk replication; Client cache coherence. A paused writer resumes after a new client became the file owner. |
| Detection | Log rejected stale write epochs at data nodes and compare file tail checksums across lease transfers. |
| Mitigation | Attach a monotonic fencing epoch to writes and have data nodes reject epochs older than the current file lease. Recover or truncate uncommitted tails according to the documented append contract. |
| Prevention | Enforce fencing in the storage write path and test a paused writer that resumes after another client acquires ownership. |

**Staff-level recovery:** Write down the exact POSIX subset, append semantics, client-cache invalidation rules, and behavior during metadata quorum loss before drawing data-node boxes.

---

## Commerce

### Payment succeeded but the order timed out

**Incident:** Commerce checkout: The provider captured funds while the checkout process crashed before recording success.

**Architecture:** `Checkout API` → `Order transaction and outbox` → `Inventory reservation` → `Payment adapter` → `Fulfillment queue` → `Reconciliation worker`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Stable checkout retries; Inventory reservation; Payment reconciliation. The provider captured funds while the checkout process crashed before recording success. |
| Detection | Track orders with provider-success evidence but unresolved order state, aged payment attempts, duplicate charge keys, and reconciliation lag. |
| Mitigation | Persist a payment attempt with a stable provider idempotency key before calling the provider. Reconcile callbacks and provider lookup into a monotonic order state machine; never create a fresh charge merely because the client retried. |
| Prevention | Persist payment attempts before provider calls, use stable idempotency keys, and inject failures between charge success, callback, and order update. |

**Staff-level recovery:** Measure unknown payments, reservation age, refund backlog, and reconciliation lag alongside checkout success. Write operational playbooks that preserve evidence for ambiguous money movement.

---

### Expiry and checkout both decrement reservations

**Incident:** Inventory reservation: A hold is committed while an expiry worker also releases it.

**Architecture:** `Reservation API` → `SKU owner` → `Stock database` → `Reservation expiry queue` → `Warehouse events` → `Reconciliation`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Prevent overselling; Expiring reservations; Warehouse allocation. A hold is committed while an expiry worker also releases it. |
| Detection | Audit each hold for exactly one terminal transition and compare reserved counters with the sum of active holds. |
| Mitigation | Use a transactional compare-and-set from held to committed or expired, with stock counters updated in that same transaction. Exactly one transition wins; retries return the recorded result. |
| Prevention | Update hold state and inventory counters in one conditional transaction; stress-test expiry, commit, and release against the same hold. |

**Staff-level recovery:** Define invariants for damaged goods, returns, transfers in flight, backorders, and manual adjustments instead of treating stock as one mutable integer.

---

### Retry posts the same transfer twice

**Incident:** Payment ledger: A client retries after a lost response and creates duplicate debit-credit pairs.

**Architecture:** `Ledger API` → `Validation` → `Transactional ledger store` → `Outbox` → `Balance projections` → `Provider reconciliation`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Balanced double-entry transactions; Idempotent posting; Auditable reversals. A client retries after a lost response and creates duplicate debit-credit pairs. |
| Detection | Find repeated business request keys, compare canonical payload hashes, and reconcile duplicate provider references even when ledger entries balance. |
| Mitigation | Use a durable unique idempotency key and compare a canonical payload hash on reuse. Return the original transaction for identical retries and reject mismatched payloads; keep the dedupe window aligned with actual retry and replay behavior. |
| Prevention | Keep durable request uniqueness inside the posting transaction and test retries after a committed transaction whose response was dropped. |

**Staff-level recovery:** Separate ledger truth, available balance, provider settlement, and bank reconciliation. Treat recovery, access control, and immutable audit evidence as design requirements rather than afterthoughts.

---

### A timer releases a newly confirmed seat

**Incident:** Ticket booking: An old hold-expiry task runs after payment confirmation.

**Architecture:** `Waiting room` → `Seat map cache` → `Hold service` → `Transactional seat store` → `Payment workflow` → `Booking confirmation`

| Step | What to do |
|---|---|
| Impact | The violated contract is: One owner per seat; Hold expiry; Fair admission under bursts. An old hold-expiry task runs after payment confirmation. |
| Detection | Alert on sold-to-available transitions, booking-seat ownership mismatches, and expiry tasks whose hold ID differs from current seat ownership. |
| Mitigation | Condition expiry on matching hold ID, held state, and expected version. Confirm and expire compete atomically; late expiry becomes a no-op and cannot free a sold seat. |
| Prevention | Guard expiry by hold identity and version, preserve a seat transition audit, and race confirmation against delayed expiry in testing. |

**Staff-level recovery:** Model bot resistance, waiting-room token replay, queue position continuity during failover, and the user-visible policy for ambiguous payment completion.

---

### Shipment callback moves an item backwards

**Incident:** Marketplace orders: A delayed in-transit callback arrives after delivered and triggers another payout hold.

**Architecture:** `Order API` → `Order database` → `Saga coordinator` → `Seller fulfillment adapters` → `Shipment events` → `Returns service` → `Payout ledger`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Split fulfillment; Partial cancellation; Auditable payout eligibility. A delayed in-transit callback arrives after delivered and triggers another payout hold. |
| Detection | Monitor invalid shipment transitions, decreasing lifecycle versions, duplicate callback IDs, and payout eligibility changes caused by old events. |
| Mitigation | Deduplicate carrier events, validate allowed state transitions, and preserve raw evidence. Ignore superseded updates while escalating genuine corrections through a separate correction workflow. |
| Prevention | Retain raw callbacks, enforce version-aware transitions, and test out-of-order carrier events including explicit corrections. |

**Staff-level recovery:** Define operational ownership for stuck states and disputed deliveries, and make seller-specific failures visible without exposing unrelated customer data.

---

## Real Time

### Stale locations produce impossible pickups

**Incident:** Ride dispatch: A driver went offline but remains in the nearby-driver index.

**Architecture:** `Driver app` → `Location ingest` → `Geospatial index` → `Dispatch matcher` → `Assignment authority` → `Trip store` → `Rider updates`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Fresh driver locations; Exclusive driver assignment; Recoverable trip state. A driver went offline but remains in the nearby-driver index. |
| Detection | Measure candidate location age at matching, driver confirmation failures, sequence regressions, and pickup ETA error by freshness cohort. |
| Mitigation | Require monotonic per-device sequences and server-observed freshness bounds; filter stale candidates and expire presence. Request confirmation before assignment and monitor stale-candidate rejection and pickup ETA error. |
| Prevention | Expire stale presence, reject old location sequences, and test delayed mobile packets and driver network loss during dispatch. |

**Staff-level recovery:** Prove assignment safety across retries and regional ownership changes, and distinguish dispatch latency from location freshness and driver acceptance rate.

---

### Concurrent insertions diverge across clients

**Incident:** Collaborative document editor: Two edits at the same position produce different text depending on arrival order.

**Architecture:** `Editor client` → `Authenticated room gateway` → `Document operation authority` → `Durable operation log` → `Snapshot compactor` → `Collaboration fanout`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Convergent edits; Reconnect and replay; Access revocation. Two edits at the same position produce different text depending on arrival order. |
| Detection | Compare converged document hashes after applying identical operation sets in different orders; retain sanitized causal traces for diverging sessions. |
| Mitigation | Use a specified OT or CRDT algorithm with stable operation IDs and causal/version metadata, rather than last-write-wins on whole text. Reproduce the operation schedule and test that all valid delivery orders converge. |
| Prevention | Use a specified and tested merge algorithm, stable operation IDs, and deterministic simulations of duplication, reordering, disconnect, and concurrent undo. |

**Staff-level recovery:** Specify supported edit semantics, undo behavior across collaborators, offline horizon, and snapshot compatibility before choosing a merge algorithm.

---

### User remains online after a silent disconnect

**Incident:** Presence and typing indicators: A mobile device loses network without closing its socket.

**Architecture:** `Client` → `Connection gateway` → `Presence shards` → `Subscription fanout` → `Recipient gateways`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Per-device heartbeats; Multi-device aggregation; Automatic expiry. A mobile device loses network without closing its socket. |
| Detection | Track heartbeat age and active sessions separately from socket-open count; use synthetic abrupt disconnects to measure offline-detection delay. |
| Mitigation | Derive online status from expiring heartbeats and aggregate across active device sessions. Use server time for expiry, expire silent devices, and display approximate last-seen state rather than claiming exact liveness. |
| Prevention | Derive presence from expiring per-session heartbeats, aggregate devices correctly, and make graceful disconnect events an optimization rather than the only cleanup mechanism. |

**Staff-level recovery:** Budget fanout by watchers and privacy rules, not just connected users, and isolate presence outages from message delivery.

---

### Replayed score event doubles a win

**Incident:** Game leaderboard: A consumer replay applies the same match result twice.

**Architecture:** `Game server` → `Score validation` → `Event log` → `Score aggregator` → `Ordered ranking index` → `Leaderboard API`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Top K and user rank; Idempotent score events; Season rollover. A consumer replay applies the same match result twice. |
| Detection | Compare unique validated match results with applied score-event IDs; alert on projection totals that diverge from replayed samples. |
| Mitigation | Deduplicate the event in the same durable update as the score mutation, or rebuild scores from unique validated results. Keep an auditable event log so incorrect projections can be discarded and recomputed. |
| Prevention | Deduplicate score mutations transactionally and regularly rebuild sample leaderboards from unique source results to detect drift. |

**Staff-level recovery:** Define tie-breaking, anti-cheat review, prize finality, and late correction policy as part of ranking correctness.

---

### GPS jitter creates repeated enter-exit alerts

**Incident:** Fleet tracking: A parked truck near a boundary generates dozens of geofence transitions.

**Architecture:** `Devices` → `Regional ingest` → `Partitioned telemetry log` → `Latest-position processor` → `Time-series archive` → `Geofence processor` → `Fleet dashboard`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Out-of-order GPS handling; Current location and history; Geofence alerts. A parked truck near a boundary generates dozens of geofence transitions. |
| Detection | Measure geofence transition rate per stationary device, GPS accuracy, dwell time, and repeated enter-exit pairs near boundaries. |
| Mitigation | Use entry/exit buffers, minimum dwell time, and accuracy-aware filtering. Deduplicate alert transitions by fence state version and preserve raw points for diagnosis instead of treating every crossing as certain. |
| Prevention | Use accuracy-aware hysteresis and dwell thresholds, version transitions, and replay noisy stationary traces before changing alert policy. |

**Staff-level recovery:** Define clock-drift tolerance, device identity reset, tenant isolation, and the difference between real-time safety alerts and best-effort business notifications.

---

## Search

### Crawler falls into an infinite calendar

**Incident:** Web search engine: A site generates unlimited distinct URLs with equivalent low-value pages.

**Architecture:** `URL frontier` → `Host-aware crawlers` → `Parser and canonicalizer` → `Document store` → `Inverted-index builders` → `Query fanout` → `Ranker`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Polite crawling; Freshness and deduplication; Low-latency ranked retrieval. A site generates unlimited distinct URLs with equivalent low-value pages. |
| Detection | Track unique URLs per host/pattern, duplicate-content ratio, frontier growth, and useful indexed pages per fetched byte. |
| Mitigation | Cap host and pattern crawl budgets, canonicalize known query parameters, detect content duplicates, and stop low-yield URL families. Respect crawl policies and avoid letting one host consume the frontier. |
| Prevention | Apply host and pattern budgets, normalize URLs carefully, and test crawl traps with infinite calendars, redirect loops, and session parameters. |

**Staff-level recovery:** Set budgets for crawling, indexing freshness, query tail latency, and removal propagation, and plan index-generation rollback with durable tombstone preservation.

---

### A blocked suggestion persists in cached lists

**Incident:** Search autocomplete: A moderation update removes a candidate but old prefix snapshots still include it.

**Architecture:** `Client debounce` → `Edge suggestion cache` → `Prefix index` → `Policy filter` → `Async query aggregation` → `Snapshot builder`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Prefix lookup; Fresh trending suggestions; Remove unsafe suggestions. A moderation update removes a candidate but old prefix snapshots still include it. |
| Detection | Probe blocked candidate IDs across locales and cache generations, measuring policy-change-to-last-visible latency. |
| Mitigation | Maintain a small versioned deny filter in serving, invalidate affected hot prefixes, and rebuild snapshots asynchronously. Check policy after retrieval so old index generations cannot resurrect the candidate. |
| Prevention | Keep a fast independent serving deny filter and durable policy versions; test removal while old snapshots and edge caches remain warm. |

**Staff-level recovery:** Specify minimum cohort thresholds, retention, locale normalization, and emergency removals before using raw search logs as a suggestion source.

---

### Search advertises yesterday’s price

**Incident:** Product search: Index lag leaves an old sale price visible after the catalog changed.

**Architecture:** `Catalog database` → `Transactional outbox` → `Search indexer` → `Search cluster` → `Availability and price enrichment` → `Results API`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Text relevance and filters; Facets and sorting; Search freshness. Index lag leaves an old sale price visible after the catalog changed. |
| Detection | Compare indexed catalog versions with authoritative versions for sampled products and monitor stale-price impressions and checkout repricing rates. |
| Mitigation | Display enrichment from current price for top results where feasible, show bounded freshness, and always reprice at checkout with user-visible confirmation. Monitor source-to-index lag and replay missing catalog events. |
| Prevention | Use versioned catalog events, lag alerts, current-price validation, and a replay path; distinguish search visibility from authoritative pricing. |

**Staff-level recovery:** Define a freshness budget per field: descriptions can lag longer than price or availability. Measure zero-result rate and business correctness separately from query latency.

---

### Mapping explosion takes down indexing

**Incident:** Log ingestion and search: User-supplied JSON creates a new field name for every request ID.

**Architecture:** `Agents` → `Ingest gateway` → `Durable buffer` → `Parsing workers` → `Time-partitioned search store` → `Query planner` → `Archive storage`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Durable ingestion; Tenant isolation; Time-range search and retention. User-supplied JSON creates a new field name for every request ID. |
| Detection | Track new indexed fields per tenant, rejected documents, mapping memory, and ingest backlog after schema changes. |
| Mitigation | Limit indexed field counts, normalize dynamic keys, and store arbitrary attributes in a controlled representation. Quarantine offending streams and reprocess from the durable buffer after fixing the schema. |
| Prevention | Enforce field-count limits and controlled dynamic attributes at admission, validate producers, and test unbounded user-generated keys. |

**Staff-level recovery:** Budget bytes, cardinality, query CPU, and result size per tenant, and provide operationally useful degradation during the exact incidents when logs surge.

---

### Embedding upgrade mixes incompatible spaces

**Incident:** Vector retrieval service: New documents use a different model while queries still use the old model.

**Architecture:** `Document ingest` → `Chunker` → `Embedding workers` → `Vector index` → `Lexical index` → `Hybrid candidate merge` → `ACL filter` → `Reranker`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Embedding versioning; Hybrid retrieval; Access-aware results. New documents use a different model while queries still use the old model. |
| Detection | Monitor query/index embedding model IDs and evaluate recall on fixed labeled queries; a sudden score-distribution shift is a clue, not proof. |
| Mitigation | Build a separate model-versioned index, generate matching query embeddings, backfill and validate retrieval quality, then switch routing atomically. Dual-serve during evaluation and retain a rollback generation. |
| Prevention | Version model, chunker, and index together; shadow-evaluate a separate generation before routing queries to it and preserve rollback artifacts. |

**Staff-level recovery:** Evaluate semantic quality, filtered recall, freshness, deletion propagation, and per-query cost independently; never treat similarity score as proof of factual correctness.

---

## Infrastructure

### Health checks remove every backend

**Incident:** Global load balancer: A shared dependency makes a deep health endpoint fail on all otherwise useful servers.

**Architecture:** `DNS or anycast entry` → `Regional edge proxies` → `Service routing table` → `Backend pools` → `Active/passive health evaluators`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Health-aware routing; Connection draining; Tenant and regional policy. A shared dependency makes a deep health endpoint fail on all otherwise useful servers. |
| Detection | Correlate backend ejections with health-check reasons and real request success; alert on simultaneous pool shrinkage across failure domains. |
| Mitigation | Separate process health from dependency readiness, require evidence across samples, and limit simultaneous ejection. Maintain a carefully defined last-resort pool for safe operations and expose dependency-specific degradation. |
| Prevention | Separate liveness from dependency readiness, cap correlated ejections, and test a shared dependency outage without removing safe surviving operations. |

**Staff-level recovery:** Define overload policy before failover, test correlated health-check failures, and include long-lived connection migration in regional evacuation drills.

---

### Bad zone version reaches every region

**Incident:** Authoritative DNS: A configuration update accidentally removes an essential record.

**Architecture:** `Zone API` → `Validation and signing` → `Versioned zone store` → `Regional publication pipeline` → `Authoritative anycast servers`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Fast lookup; Zone versioning; Signed and audited configuration changes. A configuration update accidentally removes an essential record. |
| Detection | Compare critical authoritative answers by zone version and region; monitor publication convergence and external resolution canaries. |
| Mitigation | Validate zone invariants, canary the publication, compare critical answers, and roll back the active version quickly. Preserve the prior signed generation and audit who changed which records. |
| Prevention | Validate zone invariants, canary version publication, retain signed rollback generations, and test deletion of a critical record in staging. |

**Staff-level recovery:** Test DNSSEC signing rollover where used, negative-cache behavior, delegation errors, and recovery from a control-plane compromise without disrupting stable serving.

---

### Alert evaluator fails silently

**Incident:** Metrics and alerting: Ingestion works but no rules are evaluated for twenty minutes.

**Architecture:** `Collectors` → `Ingest distributor` → `Time-series shards` → `Compactor` → `Query frontend` → `Rule evaluators` → `Notification router`

| Step | What to do |
|---|---|
| Impact | The violated contract is: High-volume ingestion; Range queries; Reliable alert state. Ingestion works but no rules are evaluated for twenty minutes. |
| Detection | Use an independent heartbeat for rule evaluations, track last-successful evaluation age and source data age, and send synthetic alerts end to end. |
| Mitigation | Alert on evaluation heartbeat and staleness from an independent path, fail over rule ownership with deduplication, and expose gaps. Evaluate missed windows according to a defined catch-up policy rather than fabricating continuous health. |
| Prevention | Place monitor-of-monitor checks in another failure domain, persist alert state, and test evaluator loss while ingestion remains healthy. |

**Staff-level recovery:** Design alert ownership, deduplication, inhibition, and evaluation continuity during regional failover; monitor the monitor through an independent failure domain.

---

### Bad rollout config reaches all clients

**Incident:** Feature flag platform: A malformed rule crashes an old SDK version.

**Architecture:** `Admin API` → `Validated config store` → `Publication log` → `SDK streaming/polling` → `Local evaluator` → `Exposure analytics`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Stable targeting; Fast local evaluation; Emergency rollback. A malformed rule crashes an old SDK version. |
| Detection | Break evaluation errors down by SDK and config version, compare fallback rates after publication, and watch canary application health. |
| Mitigation | Validate configurations against supported SDK schemas, canary by client version, and retain a fallback snapshot. Reject unsupported operators or safely use the flag default; provide a minimal emergency-disable path. |
| Prevention | Validate against supported client schemas, publish gradually, retain a last-known-good snapshot, and test unknown rule operators on old SDKs. |

**Staff-level recovery:** Define stale-config limits, defaults during startup, audit and approval ownership, and the maximum supported offline period for each class of flag.

---

### Watcher resumes from a compacted revision

**Incident:** Service discovery and configuration: A disconnected client requests events older than retained history.

**Architecture:** `Service agents` → `Registration API` → `Consensus store` → `Watch distribution` → `Client caches` → `Health-aware callers`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Lease-based registration; Watch and resync; Quorum-safe updates. A disconnected client requests events older than retained history. |
| Detection | Track watch-resume errors, requested versus retained revisions, resync counts, and client view divergence from committed service snapshots. |
| Mitigation | Return an explicit resync-required result, fetch a consistent snapshot with its revision, then watch subsequent changes. Do not silently continue from current time and leave missing endpoint updates undetected. |
| Prevention | Implement snapshot-plus-revision resynchronization explicitly and test disconnection longer than the compaction window with endpoint churn. |

**Staff-level recovery:** Test quorum loss, watch compaction, mass reconnects, and certificate rotation together; control-plane recovery must not trigger a larger data-plane outage.

---

## Advanced

### Async failover loses acknowledged recent writes

**Incident:** Multi-region key-value store: The primary region fails before its replication backlog reaches the standby.

**Architecture:** `Client router` → `Regional replicas` → `Per-range write authority` → `Replication log` → `Anti-entropy repair` → `Conflict resolver`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Regional reads; Defined write ownership; Recovery and anti-entropy. The primary region fails before its replication backlog reaches the standby. |
| Detection | Measure acknowledged commit positions against standby replay positions; identify the exact potentially missing range before failover. |
| Mitigation | Quantify the missing log range, fence the old primary, and promote only under the accepted RPO policy. If zero acknowledged-write loss is required, use synchronous cross-region quorum before acknowledgement and accept its latency. |
| Prevention | Match acknowledgement policy to the RPO promise, retain replication evidence, fence the old writer, and test failover with deliberate replica lag. |

**Staff-level recovery:** State the permitted anomaly for each operation, prove fencing during failover, and test restoration of an old region without resurrecting deleted data.

---

### Deployment makes replay nondeterministic

**Incident:** Durable workflow engine: New workflow code takes a different branch while replaying an old history.

**Architecture:** `Workflow API` → `History store` → `Deterministic workflow executor` → `Activity queues` → `External services` → `Durable timer service`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Crash-resilient progress; Versioned workflow logic; Human approval and timers. New workflow code takes a different branch while replaying an old history. |
| Detection | Replay sampled histories against candidate code and alert on nondeterminism errors, stuck decision tasks, and workflow-version cohorts. |
| Mitigation | Use workflow version markers or compatible code paths for existing histories; record external activity results and avoid uncontrolled wall-clock or random calls in deterministic execution. Canary replay against real sanitized histories. |
| Prevention | Version workflow branches, record activity results, and make replay compatibility a deployment gate for code used by long-lived executions. |

**Staff-level recovery:** Design version retirement, operator intervention, workflow cancellation, and data retention for processes that outlive several application releases.

---

### Delayed impressions overshoot the budget

**Incident:** Ad auction and budget pacing: The system allocates new ads before earlier impressions are reported as billable.

**Architecture:** `Ad request` → `Eligibility filter` → `Candidate retrieval` → `Bid and quality scoring` → `Budget lease check` → `Auction result` → `Impression log` → `Billing reconciliation`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Candidate eligibility; Auction scoring; Budget accounting and pacing. The system allocates new ads before earlier impressions are reported as billable. |
| Detection | Track reserved versus settled spend, outstanding impression age, event-reporting lag, and campaign exposure beyond remaining budget. |
| Mitigation | Reserve estimated spend at auction or delivery according to billing rules, reconcile actual billable events, and release unused reservations with a bounded timeout. Include outstanding reservations in pacing and monitor reporting lag. |
| Prevention | Include outstanding reservations in pacing, bound regional allocations, and replay delayed impression streams during budget safety testing. |

**Staff-level recovery:** Separate auction latency, relevance, billable-event accuracy, advertiser budget safety, and fraud detection. Explain which accounting decisions are provisional.

---

### Training and serving use different feature definitions

**Incident:** Recommendation serving: Offline evaluation improves while production recommendations regress.

**Architecture:** `Interaction log` → `Stream and batch features` → `Feature store` → `Candidate retrieval` → `Ranker` → `Policy and diversity filter` → `Response cache`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Low-latency recommendations; Feature freshness; Experiment attribution. Offline evaluation improves while production recommendations regress. |
| Detection | Compare sampled online features with point-in-time offline recomputation, segmented by model and transformation version. |
| Mitigation | Share or validate transformation definitions, version feature schemas with the model, and compare sampled online vectors to point-in-time-correct offline recomputation. Roll back the model-feature bundle rather than only model weights. |
| Prevention | Version model-feature bundles, validate transformation parity, and keep a rollback path for both weights and feature definitions. |

**Staff-level recovery:** Evaluate quality, latency, feature freshness, policy compliance, experiment validity, and cost together; maintain a useful fallback that can run when the feature platform is unavailable.

---

### Processor restarts and forgets window state

**Incident:** Streaming fraud detection: A deployment resets rolling velocity counts and risky transactions are approved.

**Architecture:** `Transaction API` → `Online feature lookup` → `Rules and model scorer` → `Decision store` → `Event log` → `Windowed stream processors` → `Review queue`

| Step | What to do |
|---|---|
| Impact | The violated contract is: Low-latency decisioning; Late-event handling; Auditable model and rule versions. A deployment resets rolling velocity counts and risky transactions are approved. |
| Detection | Track stream checkpoint age, event-time lag, restored-state version, and decision volume using missing or stale velocity features. |
| Mitigation | Restore a consistent state checkpoint with its source offsets, replay the gap, and mark features stale until caught up. Use a defined degraded decision policy and monitor event-time lag rather than CPU alone. |
| Prevention | Restore state and source offsets as a consistent checkpoint, block or degrade decisioning until freshness policy is met, and test restart with a known event sequence. |

**Staff-level recovery:** Separate detection quality from financial loss and customer friction; document model versions, fallback decisions, appeals, and the operational response to a stale feature pipeline.

---
