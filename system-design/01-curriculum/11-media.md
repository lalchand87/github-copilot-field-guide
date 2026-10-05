# Curriculum · Media

[← System Design index](../README.md)

> 8 lessons in **Media**. Part of the Curriculum — Build the fundamentals.

## Contents

- **Media** (8): [Resumable multipart uploads](#resumable-multipart-uploads) · [Media transcoding pipelines](#media-transcoding-pipelines) · [Adaptive bitrate streaming](#adaptive-bitrate-streaming) · [Content delivery networks](#content-delivery-networks) · [Signed media access](#signed-media-access) · [Image transformation services](#image-transformation-services) · [Live streaming ingest](#live-streaming-ingest) · [Content addressing and deduplication](#content-addressing-and-deduplication)

## Media

### Resumable multipart uploads

*Split a large upload into independently retryable parts so a failure costs one part rather than the whole transfer.*

**Flow:** `Upload session` → `Numbered parts` → `Checksums` → `Completion manifest` → `Published object`

> **The 30-second version**  
> Split a large upload into numbered parts that retry independently, upload them in parallel, and complete atomically — with a lifecycle rule to abort the ones nobody finishes.

**The problem**

A user uploads a two-gigabyte video over a mobile connection. Twenty minutes in, at ninety per cent, the connection drops. With a single HTTP request, everything is lost and the upload starts again — and on a connection unreliable enough to fail once, it will probably fail again.

The problem compounds at the server too: a single request means buffering or streaming two gigabytes through an application process, holding a connection for twenty minutes, and having no way to parallelise the transfer across the available bandwidth.

> **Multipart makes failure granular and progress durable**  
> Splitting the object into independently uploaded parts changes the unit of loss from the whole file to a single part. It also allows parallelism, since parts can upload concurrently, and it makes progress durable, since completed parts survive a client restart. Those three benefits all follow from the same decomposition.

**Mental model**

An upload becomes a session: initiate to get an identifier, upload numbered parts in any order and with any concurrency, then complete by listing the parts. The object does not exist until completion, so partial uploads are never visible.

1. **Initiate** — Create an upload session and receive an identifier. Nothing is visible to readers yet.
2. **Part** — A numbered chunk uploaded independently, each returning an entity tag that identifies its content.
3. **Retry** — A failed part is re-uploaded alone. Parts already stored are unaffected.
4. **Complete** — Submit the ordered list of part numbers and tags. The service assembles them and the object becomes visible atomically.
5. **Abort** — Discard the session and its parts. Necessary because abandoned sessions consume storage indefinitely.

> **Abandoned uploads are billed storage nobody can see**  
> Parts from incomplete uploads occupy storage but do not appear in ordinary object listings, so they accumulate invisibly — a client that abandons uploads leaves gigabytes behind with no indication. A lifecycle rule aborting incomplete multipart uploads after a few days is not an optimisation but a requirement, and its absence is one of the most common sources of unexplained storage cost.

**How it works**

**The session lifecycle**

```text
1  INITIATE
   POST /object?uploads
   -> uploadId = "abc123"
   nothing is visible to readers

2  UPLOAD PARTS  (parallel, any order)
   PUT /object?partNumber=1&uploadId=abc123   -> ETag e1
   PUT /object?partNumber=2&uploadId=abc123   -> ETag e2
   PUT /object?partNumber=3&uploadId=abc123   -> FAILS
   PUT /object?partNumber=3&uploadId=abc123   -> ETag e3
   -> only part 3 was retried; parts 1 and 2 were untouched

3  COMPLETE
   POST /object?uploadId=abc123
   body: [{1, e1}, {2, e2}, {3, e3}]
   -> the service assembles the parts
   -> the object becomes visible ATOMICALLY
   -> readers never see a partial object

4  ABORT (if abandoned)
   DELETE /object?uploadId=abc123
   -> parts are discarded
   -> WITHOUT this, or a lifecycle rule, they persist
      and are billed forever

RESUMPTION AFTER A CLIENT RESTART
   LIST parts for uploadId -> discover what already exists
   -> upload only the missing ones
```

1. **Choose part size from network reliability, not from file size** — Smaller parts mean less lost per failure and more overhead; larger parts mean fewer requests and more lost per retry. On unreliable mobile networks, smaller is better even for large files.
2. **Upload parts in parallel** — Sequential uploads leave bandwidth unused, because a single TCP connection is often window-limited. Four to eight concurrent parts typically saturates a link.
3. **Verify with checksums per part and for the whole object** — A part that transfers corrupted will otherwise be assembled silently. Per-part checksums catch it at the point of failure rather than after completion.
4. **Let clients upload directly to storage** — Presigned URLs mean application servers never handle the bytes, which removes bandwidth, memory and connection-duration pressure entirely.
5. **Keep the metadata record authoritative** — The object appearing in storage does not mean the upload is valid. A database row tracks the session's state, and the object is only usable once that row says so.
6. **Always configure the abort lifecycle rule** — Abandoned parts are invisible in listings and billed indefinitely; this is the single most common operational mistake with multipart uploads.

**Choosing the part size**

```text
TRADE-OFFS
  small parts (5 MB)
    + little lost per failed part
    + more parallelism opportunities
    - more requests -> more per-request cost and overhead
    - part limits: services typically cap parts per upload
  large parts (100 MB)
    + fewer requests
    - a failure wastes up to 100 MB of transfer
    - less granular progress reporting

WORKED EXAMPLE  2 GB file, mobile connection with a
5% chance of failure per minute of transfer
  at 5 MB parts over a 5 Mbps link: 8s per part
    -> a failure costs ~8s of work
    -> 400 parts (check the service's part limit)
  at 100 MB parts: 160s per part
    -> a failure costs ~160s
    -> at 5%/min, a 2.7-minute part fails often

RULE OF THUMB
  part size such that one part takes 10-30 seconds on the
  expected connection
  -> failures cost seconds, not minutes
  -> and check the service's minimum part size and
     maximum part count
```

> **Completion is the only atomic moment**  
> Until the completion call, the object does not exist for readers, and after it, the whole object exists. That atomicity is valuable — no consumer ever sees a half-written file — but it means the client must reach completion. A client that uploads every part and then crashes before completing has produced nothing visible and everything billable, which is why resumption requires storing the upload identifier durably on the client side.

**Worked example**

A video upload flow, showing where each mechanism earns its place.

**End-to-end design**

```text
CLIENT                    API                     STORAGE

1  request upload  ---->  create job row
                          (status: uploading)
                   <----  uploadId + presigned
                          part URLs

2  upload parts 1..N  ------------------------->  parts stored
   (4 concurrent, 8 MB each, retry per part)
   client persists uploadId locally so a restart
   can resume rather than restart

3  complete       ---->  verify part count and
                         checksums
                         call storage complete  ->  object
                         update job row
                         (status: processing)
                   <----  job id for polling

4  transcode, thumbnail, publish...
                         (status: ready)

WHAT EACH PIECE PROVIDES
  presigned URLs: application servers never touch 2 GB
  parts:          a dropped connection costs 8 MB
  parallelism:    saturates the uplink rather than one stream
  job row:        the object existing is not the same as
                  the video being ready
  lifecycle rule: abandoned uploads aborted after 7 days

FAILURE CASES
  client vanishes mid-upload
    -> parts persist; lifecycle rule aborts after 7 days
    -> job row stays "uploading"; a reaper marks it failed
  client completes but crashes before calling the API
    -> object exists in storage, job row says "uploading"
    -> reconciliation detects and completes it, or aborts
```

| Metric | Value | Note |
|---|---|---|
| Part size | 8 MB | ~10 s on target link |
| Concurrency | 4 parts | saturates uplink |
| Failure cost | 8 MB | **not 2 GB** |
| Abandoned | aborted at 7 days | lifecycle rule |

> **The object existing is not the same as the upload succeeding**  
> Storage completing an assembly means the bytes are there; it does not mean the application knows, that validation passed, or that processing has run. The metadata record must remain authoritative, and there must be reconciliation for the case where storage and the database disagree — a client that completes the upload and then fails to notify the API leaves an orphaned object that nothing will ever use.

**When to use it**

- **Files large enough that a single request is risky** — generally anything above a few tens of megabytes.
- **Unreliable networks**, especially mobile, where failure mid-transfer is likely rather than exceptional.
- **When parallelism helps**, since a single stream is often window-limited well below available bandwidth.
- **Where resumption matters**, so a user can close the app and continue later.
- **Any upload that should not pass through application servers**, using presigned part URLs.

**When to avoid it**

- **Do not use it for small files**, where session overhead exceeds the transfer and a simple PUT is better.
- **Do not omit the abort lifecycle rule**, which leaves invisible billed storage accumulating.
- **Do not proxy parts through application servers**, which reintroduces the bandwidth and memory problem.
- **Do not treat storage completion as application completion**; the metadata record remains authoritative.
- **Do not choose part size from file size alone** — network reliability is the more relevant input.

**Advantages**

- **A failure costs one part**, not the whole transfer.
- **Parallel part upload** uses available bandwidth that a single stream would leave idle.
- **Resumable across client restarts**, since completed parts persist and can be listed.
- **Atomic visibility** — readers never observe a partially written object.
- **Per-part checksums** catch corruption at the point it occurred.
- **No bytes through application servers** when combined with presigned URLs.

**Disadvantages**

- **More complex client logic**: session management, part tracking, retry, resumption.
- **Abandoned parts consume storage invisibly** unless lifecycle rules are configured.
- **Per-request overhead** makes it wasteful for small files.
- **Service limits** on minimum part size and maximum part count constrain the design.
- **Two sources of truth** — storage and the metadata record — that can disagree and need reconciliation.

**Trade-offs**

**Part size trade-offs**

| Part size | Loss per failure | Request overhead | Best for |
|---|---|---|---|
| 5 MB | Seconds of transfer | High part count | Unreliable mobile networks |
| 8–16 MB | ~10–30 s on typical links | Balanced | General purpose |
| 100 MB+ | Minutes of transfer | Minimal | Reliable data-centre links |
| Single request | Everything | None | Files under ~50 MB |

The guiding heuristic is to choose a part size such that one part takes ten to thirty seconds on the expected connection: failures then cost seconds rather than minutes, while request overhead stays negligible.

**How it fails**

**Upload failure modes**

| Failure | Cause | Fix |
|---|---|---|
| Storage bill grows unexplained | Abandoned multipart parts | Lifecycle rule to abort incomplete uploads |
| Object exists but is unusable | Client completed storage but never notified the API | Reconciliation between storage and the metadata record |
| Upload restarts from zero | Client did not persist the upload identifier | Store it durably on the client; list parts to resume |
| Corrupted file after assembly | No per-part checksums | Checksum each part and the assembled object |
| Slow uploads despite bandwidth | Sequential parts on a window-limited connection | Upload parts in parallel |
| Application servers memory-exhausted | Proxying bytes through the API | Presigned part URLs direct to storage |
| Part count limit exceeded | Part size too small for a very large file | Scale part size with file size within service limits |

**Limits**

> **Practical parameters**
>
> - **Part size**: aim for 10–30 seconds of transfer on the expected connection; typically 5–16 MB for consumer uploads.
> - **Concurrency**: 4–8 parallel parts usually saturates a consumer uplink.
> - **Service limits**: minimum part size and maximum part count per upload constrain very large objects.
> - **Abort lifecycle**: a few days; abandoned parts are invisible in listings and billed until removed.
> - **Single-request threshold**: below roughly 50 MB, a simple upload is usually better.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Single PUT | Small files on reliable networks | All-or-nothing |
| Multipart upload | Large files; unreliable networks | Client complexity; abandoned parts |
| Resumable protocol (tus-style) | Byte-offset resumption without part management | Server support required |
| Chunked streaming upload | Unknown final size | No resumption |
| Client-side splitting into separate objects | Full control over assembly | You implement completion atomicity |
| Torrent or peer-assisted | Very large distributions | Complexity; not for user uploads |

Byte-offset resumable protocols are worth knowing as an alternative shape: rather than managing numbered parts, the client asks how many bytes the server already has and continues from there. It is simpler for clients and requires explicit server support, whereas multipart is available from object storage without building anything.

**In real systems**

- **S3 multipart upload** is the reference implementation, and its abort-incomplete lifecycle rule exists precisely because abandoned parts are otherwise invisible and billed.
- **Consumer file sync and video platforms** universally use resumable uploads, since mobile connections fail often enough that all-or-nothing transfer is unusable.
- **Presigned part URLs** let browsers and mobile clients upload directly to storage, which is why large-file uploads do not scale application server capacity.
- **The tus protocol** standardises byte-offset resumable uploads as an alternative to part-based schemes.
- **Parallel part upload** is what allows cloud transfer tools to saturate high-bandwidth links that a single TCP stream cannot fill.

**Common mistakes**

- **No lifecycle rule to abort incomplete uploads**, accumulating invisible billed storage.
- **Proxying parts through application servers**, exhausting memory and bandwidth.
- **Treating storage completion as application completion**, leaving orphaned objects.
- **Sequential part upload**, leaving bandwidth unused.
- **No per-part checksums**, so corruption is discovered after assembly or never.
- **Client not persisting the upload id**, so a restart begins from zero.
- **Part size chosen from file size**, producing minute-long parts on unreliable links.

**The staff-level view**

Upload design is usually correct on the happy path and wrong on the two failure cases that actually occur: abandonment and partial completion.

- **Require the abort lifecycle rule as part of any multipart implementation.** Abandoned parts are billed, invisible in listings, and accumulate indefinitely — this is the most common and most expensive omission.
- **Keep the metadata record authoritative and reconcile it with storage.** An object existing does not mean the application knows about it, and a client that completes storage then crashes leaves an orphan nothing will use.
- **Mandate presigned direct upload.** Proxying large files through application servers is a capacity ceiling teams hit repeatedly and a bandwidth cost they pay twice.
- **Derive part size from network conditions**, not from file size, so failures cost seconds rather than minutes on the connections users actually have.
- **Ensure clients persist the upload identifier**, since resumption after an app restart is the feature users notice and it is trivially easy to omit.

**Go deeper**

Multipart upload decomposes a large transfer into independently retryable parts. A dropped connection costs one part rather than the whole file, parts upload concurrently to use bandwidth a single stream would leave idle, and completed parts persist so a client restart can resume by listing what already exists. The object becomes visible atomically at completion, so readers never observe a partial file.

Part size should be derived from network conditions rather than file size: a part that takes ten to thirty seconds on the expected connection means failures cost seconds, while request overhead stays negligible. Combine with presigned part URLs so clients upload directly to storage and application servers never handle the bytes, which removes a capacity ceiling teams otherwise hit repeatedly.

Two failure cases dominate production and neither appears in testing. Abandoned uploads leave parts that are billed but invisible in object listings, accumulating indefinitely unless a lifecycle rule aborts them — the most common and most expensive omission. And a client that completes storage but crashes before notifying the API leaves an orphaned object with a job stuck in an uploading state, which requires reconciliation between storage and the authoritative metadata record.

Resumable multipart upload solves a failure-granularity problem: for large transfers over unreliable networks, the unit of loss must not be the whole file.

**One decomposition, three benefits.** Splitting an object into numbered parts means a failure costs only that part; parts can be uploaded concurrently, which matters because a single TCP connection is frequently window-limited well below available bandwidth; and completed parts are durable, so a client that restarts can enumerate what exists and upload only the remainder. Completion is atomic — the object does not exist for readers until the parts are assembled — so no consumer ever observes a partially written file.

**Part size is a network decision.** The useful heuristic is that one part should take ten to thirty seconds on the expected connection. On an unreliable mobile link that argues for small parts even for very large files, since a failure then costs seconds rather than minutes; on a fast reliable link larger parts reduce per-request overhead. The constraints are service limits on minimum part size and maximum part count, which for multi-gigabyte objects force part size upward regardless of network quality.

**Direct upload is what makes it scale.** Presigned part URLs let clients write straight to storage, so application servers never buffer or stream the bytes. Proxying uploads instead ties process memory, bandwidth and connection duration to file size, which becomes a capacity ceiling quickly and means paying for the same bytes twice. This single decision usually matters more than any other in the design.

**Abandonment is invisible and billed.** Parts from incomplete uploads occupy storage but do not appear in normal object listings, so they accumulate with no visible trace. A client population that abandons uploads regularly leaves substantial storage behind, and the cost surfaces months later with no obvious attribution. A lifecycle rule aborting incomplete multipart uploads after a few days belongs in the implementation, not in a later cleanup effort.

**Storage and application state can disagree.** An assembled object means the bytes exist; it does not mean validation ran, processing started, or the application knows about it. A client that calls storage completion and then fails before notifying the API leaves an orphaned object and a job record stuck in an uploading state, which nothing will surface unless something is actively reconciling. Keeping the metadata record authoritative, running a scheduled reconciliation against storage, and alerting on jobs stuck in non-terminal states are what convert this from silent inconsistency into an ordinary recoverable case.

**Test the failure cases that actually occur.** The happy path works in every environment. What breaks in production is abandonment, mid-completion crashes, and resumption after an app restart — the last being the feature users notice most and the one most easily omitted, since it requires the client to persist the upload identifier durably. Explicitly exercising those three scenarios catches nearly everything that goes wrong with upload systems.

**Prove it — interview questions**

1. **[Basic] What does multipart upload actually buy you?**

   <details><summary>Model answer</summary>

   Three things from one decomposition. Failure granularity: a dropped connection costs one part rather than the entire transfer. Parallelism: parts can upload concurrently, which matters because a single TCP stream is often window-limited well below the available bandwidth. And resumability: completed parts persist, so a client that restarts can list what already exists and upload only the remainder. The object itself becomes visible atomically at completion, so readers never see a partial file.

   </details>

2. **[Basic] How do you choose part size?**

   <details><summary>Model answer</summary>

   From the network rather than the file. The useful heuristic is a part size such that one part takes roughly ten to thirty seconds on the expected connection, so a failure costs seconds of retransmission rather than minutes. On a slow or unreliable mobile link that means small parts even for a large file; on a fast data-centre connection larger parts reduce request overhead. The constraints are the service's minimum part size and maximum part count, which for very large objects force the part size upward.

   </details>

3. **[Senior] What happens to parts from an abandoned upload?**

   <details><summary>Model answer</summary>

   They stay in storage, consume space, and are billed — but they do not appear in ordinary object listings, so they accumulate entirely invisibly. A client that abandons uploads regularly can leave many gigabytes behind that nobody can see or account for, and this is one of the most common causes of unexplained storage cost. The fix is a lifecycle rule that aborts incomplete multipart uploads after a few days, which should be considered part of implementing multipart rather than an optional optimisation.

   </details>

4. **[Senior] Why must the metadata record remain authoritative?**

   <details><summary>Model answer</summary>

   Because the object existing in storage does not mean the upload succeeded from the application's point of view. Validation may not have run, processing may not have started, and the application may not even know the object is there — a client that calls storage completion and then crashes before notifying the API leaves an object nothing will ever reference. Keeping a database record of the upload's state, and reconciling it periodically against storage, is what makes the difference between “bytes exist” and “this asset is usable” explicit and recoverable.

   </details>

5. **[Staff] Design the upload flow for a two-gigabyte video from a mobile client.**

   <details><summary>Model answer</summary>

   The client requests an upload, the API creates a job record in an uploading state and returns presigned part URLs so no bytes ever pass through application servers — that removes the bandwidth, memory and connection-duration pressure entirely and is what makes the design scale. Parts sized around eight megabytes, so each takes roughly ten seconds on a typical mobile uplink and a failure costs seconds, uploaded four at a time to saturate the link, with per-part checksums so corruption is caught where it occurred. The client persists the upload identifier locally so an app restart resumes by listing existing parts rather than starting over. On completion the API verifies and moves the job to processing, and a lifecycle rule aborts incomplete uploads after seven days. Finally a reconciliation job handles the case where storage and the job record disagree, which is the failure that otherwise leaves orphaned objects and stuck jobs.

   </details>

6. **[Principal] What makes upload systems fail in production rather than in testing?**

   <details><summary>Model answer</summary>

   The failure cases are all about abandonment and partial completion, and neither occurs in a test environment where clients behave. Abandoned uploads leave parts that are billed and invisible, so the cost appears months later with no obvious cause and no easy way to attribute it. Clients that complete storage but fail to notify the API leave orphaned objects and jobs stuck in an uploading state forever, which nothing surfaces unless something is actively looking. And resumption is the feature users notice most and the one most likely to be omitted, because the happy path works without it. So the disciplines I would enforce are: a lifecycle rule as a required part of any multipart implementation, reconciliation between storage and metadata as a scheduled job rather than a hope, an alert on jobs stuck in non-terminal states, and explicit testing of abandonment and mid-completion crashes — because those three cases are where the real failures live and none of them produces an error anyone will see.

   </details>

---

### Media transcoding pipelines

*Convert an uploaded master into the ladder of renditions needed for playback, as a fan-out of independent, idempotent, expensive jobs.*

**Flow:** `Source media` → `Validation` → `Encoding workers` → `Verified renditions` → `Manifest`

> **The 30-second version**  
> Validate cheaply, fan out into independent idempotent encoding jobs, verify each output, and publish the manifest last — with cost as the primary design constraint.

**The problem**

A user uploads a video recorded on an unknown device, in an unpredictable codec, resolution and container. Playback must work on a phone over cellular, a laptop on broadband and a television on a fast connection — which means several renditions at different resolutions and bitrates, plus thumbnails, plus a manifest describing them.

Transcoding is also unusually expensive: encoding an hour of video can take an hour of CPU per rendition, so a single upload may consume several CPU-hours. That cost profile makes the pipeline's efficiency and failure behaviour matter far more than in ordinary job processing.

> **Transcoding is a fan-out of expensive, independent, idempotent jobs**  
> Each rendition is independent of the others, so they parallelise perfectly. Each is expensive enough that wasted work is costly, so idempotency matters more than usual. And each can fail differently, so partial success must be a first-class outcome rather than a reason to restart everything.

**Mental model**

The uploaded file is a master that is validated once, then fanned out into independent encoding jobs — one per rendition — whose outputs are verified and finally described by a manifest that makes them playable.

1. **Validation** — Probe the file to determine codec, duration, resolution and integrity before spending any encoding budget on it.
2. **Ladder** — The set of renditions to produce — resolutions and bitrates — chosen from the source's properties and the audience's devices.
3. **Encoding jobs** — One per rendition, independent and parallel. The expensive part.
4. **Verification** — Confirm each output is playable and matches expectations, since a corrupt rendition fails only at playback time.
5. **Manifest** — The playlist describing available renditions, written last so a player never sees a rendition that does not exist.

> **Encoding failures are expensive and often late**  
> A codec the encoder cannot handle, a truncated file, an unusual pixel format or an audio track with no samples may only manifest after an hour of processing. Validating aggressively before encoding — probing the container, checking duration and streams — converts an expensive late failure into a cheap immediate one, and is the single highest-return step in the pipeline.

**How it works**

**Pipeline stages and their cost profile**

```text
1  VALIDATE  (seconds, cheap)
   probe container, codecs, duration, resolution, streams
   reject unsupported or corrupt input NOW
   -> avoids spending CPU-hours on a file that cannot work

2  CHOOSE THE LADDER  (instant)
   from source resolution and duration
   never encode ABOVE the source resolution - upscaling
     costs money and adds no quality
   a 480p source gets a 480p/360p/240p ladder, not 1080p

3  FAN OUT  (expensive, parallel)
   one job per rendition + thumbnails + audio extraction
   each independent, each idempotent, each retryable
   -> a failed 720p job does not affect the 1080p output

4  VERIFY  (cheap)
   probe each output: duration matches, streams present,
   playable
   -> a silently corrupt rendition otherwise fails at
      playback, for users, days later

5  PUBLISH MANIFEST  (instant, atomic)
   write the playlist listing the completed renditions
   -> written LAST, so a player never references a
      rendition that does not exist

THE ORDERING MATTERS: cheap validation before expensive
work, and the manifest after everything it describes.
```

1. **Validate before spending encoding budget** — A probe costs seconds and rejects files that would otherwise fail after an hour of CPU. It is the highest-return step in the pipeline.
2. **Never encode above the source resolution** — Upscaling consumes CPU and storage while adding no information. The ladder must be derived from the source, not from a fixed list.
3. **Make each rendition an independent job** — Independent failure, independent retry, independent scaling. A single job producing all renditions cannot be retried safely.
4. **Key idempotency on source and rendition** — A retry must not re-encode work already completed. The output object's existence, keyed by source hash plus rendition profile, is the natural check.
5. **Publish the manifest last and atomically** — A manifest referencing a rendition that is still encoding produces playback errors, and players cache manifests aggressively.
6. **Allow partial publication deliberately** — Publishing the lower renditions while higher ones still encode lets playback start sooner — but only if the manifest is updated atomically and players handle it.

**Cost, and why the ladder choice matters**

```text
ONE HOUR OF VIDEO, typical encoder

LADDER                    CPU-HOURS   STORAGE
  1080p @ 5 Mbps            ~1.0       2.2 GB
  720p  @ 2.8 Mbps          ~0.6       1.3 GB
  480p  @ 1.4 Mbps          ~0.3       0.6 GB
  360p  @ 0.8 Mbps          ~0.2       0.4 GB
  240p  @ 0.4 Mbps          ~0.1       0.2 GB
  total                     ~2.2       4.7 GB
  + the original master                 3.0 GB

IMPLICATIONS
  1,000 hours uploaded per day = ~2,200 CPU-hours/day
  -> ~92 dedicated cores running continuously
  -> encoding cost frequently exceeds storage cost

LEVERS
  drop renditions nobody watches
    -> measure playback by rendition; the bottom of the
       ladder is often unused on modern networks
  encode on demand for rarely watched content
    -> popular videos get the full ladder eagerly;
       the long tail gets encoded on first request
  hardware acceleration
    -> much faster, slightly worse quality per bitrate
  newer codecs
    -> better compression, much higher encoding cost
```

> **The long tail is where transcoding money is wasted**  
> In most catalogues a small fraction of content receives the overwhelming majority of views, yet every upload is typically encoded into every rendition eagerly. Encoding the full ladder for content nobody watches is pure cost. Encoding popular content eagerly and the long tail lazily on first request — accepting a delay for the first viewer — often reduces transcoding spend dramatically.

**Worked example**

A user-generated video platform, sized and with its failure handling described.

**Pipeline design with costs and failure paths**

```text
VOLUME  10,000 videos/day, average 6 minutes = 1,000 hours

VALIDATION
  ffprobe each upload: ~2 seconds
  reject: unsupported codec, zero-duration, corrupt
  -> rejects ~2% of uploads for the cost of 5 CPU-hours/day
     rather than ~44 CPU-hours of wasted encoding

LADDER (derived from source)
  4K source   -> 1080p, 720p, 480p, 360p
  1080p       -> 1080p, 720p, 480p, 360p
  720p        -> 720p, 480p, 360p
  480p        -> 480p, 360p
  -> never upscale

FAN-OUT
  one message per (video, rendition)
  idempotency key: hash(source) + profile
  -> a retry checks whether the output already exists
  workers on spot/preemptible capacity, since jobs are
  independent and retryable

VERIFY
  probe each output: duration within tolerance, streams
  present, no decode errors
  -> catches silent corruption before users do

PUBLISH
  write the HLS manifest listing completed renditions
  update the video record to "ready"

FAILURE HANDLING
  one rendition fails permanently
    -> publish without it if enough renditions exist
    -> alert; do not block the whole video
  all renditions fail
    -> video marked failed with the reason surfaced to
       the uploader
  worker preempted mid-encode
    -> job returns to the queue; idempotency prevents
       duplicate output
```

| Metric | Value | Note |
|---|---|---|
| Volume | 1,000 h/day | ~2,200 CPU-hours |
| Validation | 5 CPU-hours | **saves 44** |
| Fan-out | per rendition | independent retry |
| Partial | publish anyway | if enough renditions |

> **Partial success should be the default outcome, not a failure**  
> If the 1080p rendition fails but 720p and below succeeded, the video is perfectly watchable for almost everyone. Blocking publication until every rendition completes converts a minor quality reduction into a total failure for the uploader. Publishing what succeeded, alerting on the gap, and retrying the missing rendition separately is almost always the better product outcome.

**When to use it**

- **User-generated video and audio**, where source formats are unpredictable and playback targets are diverse.
- **Adaptive bitrate streaming**, which requires a ladder of renditions by construction.
- **Image processing at scale**, which has the same fan-out shape at much lower cost per job.
- **Any expensive derived-asset generation**, where the pattern of validate, fan out, verify and publish applies.
- **Archival normalisation**, converting heterogeneous sources into a consistent internal format.

**When to avoid it**

- **Do not encode eagerly for content with an unknown audience**; lazy encoding of the long tail is usually far cheaper.
- **Do not upscale beyond the source resolution**, which costs money and adds nothing.
- **Do not produce all renditions in one job**, which cannot be retried or scaled independently.
- **Do not publish the manifest before the renditions exist**, since players cache aggressively and will fail.
- **Do not skip output verification**, which turns silent corruption into a user-facing playback failure days later.

**Advantages**

- **Perfectly parallel**, since renditions are independent — throughput scales with workers.
- **Failure is granular**, so one rendition failing does not lose the others.
- **Suited to preemptible capacity**, because jobs are independent, retryable and idempotent.
- **Partial success is usable**, letting playback proceed with fewer renditions.
- **Validation is cheap relative to encoding**, so most failures can be made early and inexpensive.

**Disadvantages**

- **Extremely expensive in CPU**, frequently exceeding storage cost for a media platform.
- **Long-running jobs** complicate leases, retries and deploys.
- **Format diversity is endless**, so validation and encoder configuration are a continuing maintenance burden.
- **Quality is subjective**, making regressions hard to detect automatically.
- **Storage multiplies** — every rendition plus the master is retained.

**Trade-offs**

**Encoding strategy trade-offs**

| Strategy | Cost | First-view latency | Best for |
|---|---|---|---|
| Eager full ladder | Highest | None | Content with known demand |
| Eager low renditions, lazy high | Moderate | None for most viewers | General user-generated content |
| Fully lazy on first request | Lowest | Minutes for the first viewer | Long-tail archives |
| Hardware-accelerated | Much lower CPU | None | High volume, quality tolerance |
| Newer codec (AV1 etc.) | Much higher encode cost | None | High-view content where bandwidth savings repay it |

The last row is worth understanding: a more efficient codec reduces delivery bandwidth substantially but costs far more to encode, so it only pays for content with enough views that the bandwidth saving exceeds the additional encoding cost — which is a per-asset decision, not a global one.

**How it fails**

**Transcoding pipeline failures**

| Failure | Cause | Fix |
|---|---|---|
| CPU-hours wasted on unusable files | No validation before encoding | Probe and reject upfront |
| Playback fails for some users | Manifest published before renditions completed | Publish the manifest last, atomically |
| Silent corruption discovered by users | Outputs not verified | Probe every output before publishing |
| Duplicate encoding after a retry | No idempotency on the rendition job | Key on source hash plus profile; check for existing output |
| Whole video fails because one rendition did | All-or-nothing publication | Publish partial ladders; retry the missing rendition |
| Encoding cost far exceeds expectations | Eager full ladder for long-tail content | Lazy encoding for the tail; drop unwatched renditions |
| Jobs lost on worker preemption | No lease or reaper for long jobs | Lease with heartbeat; idempotent re-execution |

**Limits**

> **Cost reference**
>
> - **Encoding time**: roughly real-time per rendition on a CPU for standard codecs, so an hour of video is several CPU-hours across a ladder.
> - **Storage**: the full ladder is typically 1.5–2× the master, and the master is usually retained.
> - **Validation** costs seconds and prevents hours of wasted encoding — the best ratio in the pipeline.
> - **Hardware encoding** is substantially faster with a modest quality cost per bitrate.
> - **Long-tail skew**: a small fraction of content typically receives most views, which is what makes lazy encoding pay.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Self-hosted encoding fleet | High volume; cost control | Operating a fleet; encoder maintenance |
| Managed transcoding service | Low to moderate volume | Per-minute cost; less control |
| Lazy / just-in-time encoding | Long-tail catalogues | First-view latency |
| Hardware-accelerated encoding | Very high volume | Slight quality cost |
| Client-side pre-encoding | Reduces upload size and server cost | Device variability; trust issues |
| Single rendition only | Internal or controlled-device use | No adaptive streaming |

Client-side pre-encoding deserves consideration for mobile apps: encoding on the device before upload reduces upload size, upload time and server cost simultaneously. The costs are device battery, inconsistent results across hardware, and the fact that you cannot trust client output without validating it anyway.

**In real systems**

- **Video platforms** encode a ladder per upload and increasingly encode the long tail lazily, since a small fraction of content receives most views.
- **Per-title encoding** adjusts the bitrate ladder to the complexity of each video rather than using fixed settings, saving substantial bandwidth.
- **Preemptible and spot compute** is heavily used for transcoding, since jobs are independent, retryable and idempotent — an ideal fit.
- **ffmpeg and ffprobe** underpin most pipelines, with probing before encoding being the standard validation step.
- **Newer codecs** are typically applied selectively to high-view content, because their encoding cost only repays through bandwidth savings at scale.

**Common mistakes**

- **Encoding before validating**, spending CPU-hours on files that cannot work.
- **Upscaling beyond the source**, adding cost and no quality.
- **One job producing all renditions**, preventing independent retry and scaling.
- **Publishing the manifest before renditions exist**, causing playback failures.
- **Skipping output verification**, so corruption is found by users.
- **Eager full-ladder encoding** for long-tail content nobody watches.
- **No idempotency**, so retries duplicate the most expensive work in the system.

**The staff-level view**

Transcoding is one of the few pipelines where compute cost is a primary architectural constraint rather than an afterthought.

- **Validate before encoding, always.** Seconds of probing prevents hours of wasted CPU, and it is the highest-leverage step by a wide margin.
- **Derive the ladder from the source and from measured playback.** Upscaling is pure waste, and the bottom renditions are frequently unused on modern networks — measure which are actually played before producing them.
- **Treat the long tail as a cost decision.** Eager full-ladder encoding for content nobody watches is often the largest single line of avoidable spend in a media platform.
- **Make partial success publishable.** A missing top rendition should reduce quality for some viewers, not block the video entirely.
- **Design for preemption.** Independent, idempotent, retryable jobs let the whole pipeline run on cheap interruptible capacity, which is a substantial cost lever.

**Go deeper**

Transcoding converts an unpredictable uploaded master into the ladder of renditions playback requires. The shape is a fan-out of independent jobs: one per rendition, perfectly parallel, each retryable alone. What distinguishes it from ordinary job processing is cost — encoding an hour of video takes roughly an hour of CPU per rendition, so a single upload can consume several CPU-hours and wasted work is directly expensive.

Two orderings matter. Validation comes first, because probing a file costs seconds and rejects corrupt or unsupported uploads that would otherwise fail after hours of encoding — the highest-return step in the pipeline. And the manifest is published last, because a playlist referencing a rendition that does not yet exist causes playback failures that persist through aggressive client caching.

The dominant cost lever is the view distribution. Most catalogues are heavily skewed, so eagerly encoding the full ladder for every upload spends most of the budget on content nobody watches; encoding popular content eagerly and the long tail lazily on first request often reduces spend dramatically. Deriving the ladder from the source so nothing is upscaled, dropping renditions that measurement shows are unplayed, and running on preemptible capacity — which the independent, idempotent job shape suits perfectly — are the other main levers.

Media transcoding is a fan-out pipeline whose defining characteristic is that each unit of work is expensive enough that compute cost becomes a primary architectural constraint.

**Stage ordering is driven by cost asymmetry.** Validation — probing the container, codecs, duration and stream integrity — costs seconds and rejects files that would otherwise consume CPU-hours before failing. Given that encoding dominates the budget, this is the single highest-leverage decision in the pipeline and it is frequently omitted because the happy path does not need it. At the other end, the manifest must be written after every rendition it references has been produced and verified, because a playlist pointing at a missing rendition produces playback errors that persist through client caching.

**Independent jobs per rendition.** A single job producing the whole ladder cannot be retried without redoing completed work, cannot be scaled per rendition, and turns one failure into total loss. Separate jobs make failure granular, allow different renditions to run on different capacity, and — importantly — make partial success publishable: a video with 720p and below available is perfectly watchable, and blocking publication because 1080p failed converts a minor quality reduction into a complete failure for the uploader.

**Idempotency matters more here than usual.** Every distributed job system needs idempotent handlers, but when a duplicate execution costs an hour of CPU rather than a database write, the incentive is financial as well as correctness-driven. Keying on source hash plus rendition profile, and checking for the existing output object before starting, makes retries and preemption free rather than expensive — which in turn is what makes running the whole pipeline on interruptible spot capacity viable, a substantial cost lever that the job shape suits perfectly.

**The ladder is a cost decision, not a fixed list.** Encoding above the source resolution consumes CPU and storage while adding no information, so the ladder must be derived from the source's properties. Beyond that, measurement usually shows the lowest rungs are rarely played on modern networks, so producing them is waste. And per-title approaches that adjust bitrates to the complexity of each video rather than using fixed settings save meaningful bandwidth without quality loss.

**The view distribution is the largest lever.** Catalogues are typically heavily skewed, with a small fraction of content receiving most views, yet the default is to eagerly encode every upload into every rendition. That spends most of the transcoding budget on content nobody watches. Encoding popular content eagerly and the long tail lazily on first request — accepting minutes of delay for the first viewer of an obscure asset — frequently reduces spend by a large multiple, and is invisible to almost all users.

**Verification prevents late, user-visible failures.** A rendition can encode without erroring and still be unplayable, truncated, or missing audio. Probing every output before publishing catches this at the point it occurred rather than when a viewer encounters it days later, at which point diagnosis requires reconstructing which worker produced it and why. Given how cheap the check is relative to the encode, skipping it is never justified.

**Prove it — interview questions**

1. **[Basic] Why transcode at all rather than serving the original?**

   <details><summary>Model answer</summary>

   Because the original is in whatever format the uploader's device produced, which may not be playable on the viewer's device, and because playback needs multiple quality levels. A phone on a cellular connection needs a low bitrate, a television on fibre wants a high one, and adaptive streaming requires several renditions so the player can switch as conditions change. Serving one original file means some viewers cannot play it and others get quality unsuited to their connection.

   </details>

2. **[Basic] Why validate before encoding?**

   <details><summary>Model answer</summary>

   Because validation costs seconds and encoding costs hours. Probing the file to check the container, codecs, duration and stream integrity rejects unsupported or corrupt uploads immediately, rather than discovering the problem after a worker has spent an hour of CPU on it. Given that encoding is usually the dominant cost in a media pipeline, this is the highest-return step available — a couple of seconds of probing regularly saves several CPU-hours per rejected file.

   </details>

3. **[Senior] Why should each rendition be a separate job?**

   <details><summary>Model answer</summary>

   For independent failure, retry and scaling. If one job produces the whole ladder, a failure encoding the 1080p rendition wastes the 720p and 480p work already done and cannot be retried without redoing everything — which is very expensive when each rendition is CPU-hours. Separate jobs mean a failed rendition is retried alone, renditions can run on different capacity, and partial success is possible: publishing with 720p and below when 1080p failed gives a perfectly watchable video instead of a total failure.

   </details>

4. **[Senior] Why publish the manifest last?**

   <details><summary>Model answer</summary>

   Because a manifest listing a rendition that does not yet exist causes playback errors, and players cache manifests aggressively enough that the error persists after the rendition appears. Writing it after every rendition it references has been produced and verified means the player never encounters a broken reference. If renditions are published progressively to start playback sooner, the manifest must be updated atomically at each step rather than written once optimistically.

   </details>

5. **[Staff] How would you reduce transcoding cost on a large media platform?**

   <details><summary>Model answer</summary>

   By attacking the ladder and the tail. First, measure which renditions are actually played — on modern networks the lowest rungs are frequently unused, and producing them is pure waste. Second, derive the ladder from the source so nothing is upscaled, since encoding above the source resolution costs CPU and storage while adding no information. Third, and usually the largest lever, exploit the view distribution: most catalogues are heavily skewed, so encoding the full ladder eagerly for every upload spends most of the budget on content nobody watches. Encoding popular content eagerly and the long tail lazily on first request — accepting a delay for the first viewer — often reduces spend substantially. Fourth, run on preemptible capacity, which the pipeline suits perfectly because jobs are independent, idempotent and retryable. Hardware acceleration is a further lever where a small quality cost per bitrate is acceptable.

   </details>

6. **[Principal] What makes transcoding pipelines architecturally distinctive?**

   <details><summary>Model answer</summary>

   That compute cost is a primary constraint rather than a secondary concern. In most pipelines the per-item cost is small enough that the architecture is driven by latency, correctness and operability; here a single item can consume several CPU-hours, so wasted work is directly and visibly expensive, and decisions that would be micro-optimisations elsewhere — validating before processing, not upscaling, encoding the tail lazily — are the dominant design considerations. That cost profile also shapes the failure handling: idempotency matters more because a duplicate execution wastes real money, partial success matters more because restarting is expensive, and preemptible capacity becomes attractive in a way it would not be for cheap jobs. The practical consequence is that a media pipeline should be designed with a cost model alongside the architecture diagram, because the difference between a reasonable design and a careless one is not milliseconds but a multiple of the compute bill.

   </details>

---

### Adaptive bitrate streaming

*Cut media into short segments at several qualities and let the player choose per segment, so playback degrades in quality rather than stalling.*

**Flow:** `Rendition ladder` → `Segmenter` → `Manifest` → `Player buffer` → `Switch decision`

> **The 30-second version**  
> Encode a ladder of bitrates as keyframe-aligned segments and let the player pick per segment from buffer level and throughput — dropping fast, climbing slowly.

**The problem**

A viewer on a train watches a video. Bandwidth swings from twenty megabits to two and back within a minute. A single-quality stream has only two options: pick a high bitrate and stall repeatedly when bandwidth drops, or pick a low bitrate and look poor for the entire session even when the connection is excellent.

Neither is acceptable, and the problem is not solvable by choosing better once at the start, because the network condition that matters is the one during playback and it changes continuously.

> **The core trade is quality for continuity, decided repeatedly**  
> Adaptive streaming makes the quality choice per segment rather than per session. When bandwidth falls, the player selects a lower-bitrate segment and keeps playing; when it recovers, quality rises again. Viewers tolerate quality changes far better than they tolerate stalls, so trading resolution for continuity is almost always the right exchange — and making that trade requires the decision to be frequent and local to the player.

**Mental model**

The media is encoded at several bitrates, each cut into short segments at identical boundaries, and described by a manifest. The player downloads segments into a buffer and, before each one, decides which quality to request based on measured throughput and how much buffer it holds.

1. **Ladder** — The set of bitrate and resolution variants — the options the player can choose between.
2. **Segments** — Each variant cut into two-to-ten-second chunks, aligned across variants so a switch is seamless.
3. **Manifest** — A playlist listing the variants and their segments, fetched first and periodically refreshed for live.
4. **Buffer** — Seconds of downloaded media ahead of the playhead — the shock absorber that converts bandwidth variance into quality variance instead of stalls.
5. **Decision** — Before each segment, choose a variant from measured throughput and current buffer level.

> **Segment boundaries must align across variants**  
> A player switching quality mid-stream downloads segment N from a different variant than segment N−1. If the variants were encoded with independent keyframe placement, the segments do not start at the same media timestamp and the switch produces a visible glitch, an audio gap, or a decode error. Aligned keyframes across the entire ladder is a hard requirement of the encoding stage, not a refinement — and it is the most common reason adaptive playback misbehaves.

**How it works**

**What the player actually does**

```text
STARTUP
  fetch manifest
  choose a conservative starting variant
    -> no throughput measurement exists yet
    -> starting too high risks an immediate stall,
       which is the worst moment for one
  download segments to fill the buffer

STEADY STATE  (before each segment)
  estimate throughput from recent segment downloads
  read buffer level (seconds ahead of the playhead)

  if buffer is healthy and throughput > next-up bitrate
      for a sustained period
    -> step UP one rung (never jump several)
  if buffer is draining or throughput < current bitrate
    -> step DOWN, possibly several rungs at once
  otherwise
    -> stay

THE ASYMMETRY IS DELIBERATE
  step down fast   - a stall is far worse than low quality
  step up slowly   - an over-eager switch up causes the
                     stall it was trying to avoid

BUFFER IS THE REAL SIGNAL
  throughput estimates are noisy and lag reality
  buffer level is a direct measure of whether the player
  is winning or losing the race
  -> modern algorithms weight buffer heavily, using
     throughput mainly at startup when no buffer exists
```

1. **Align keyframes across every variant** — Segments must start at identical media timestamps, or switching produces glitches. This constrains the encoder, not the player.
2. **Choose segment duration deliberately** — Short segments adapt faster and add request overhead; long segments are efficient but slow to react and raise live latency.
3. **Start conservatively, then climb** — No throughput history exists at startup, and a stall in the first seconds is the one viewers abandon over.
4. **Step down aggressively, up cautiously** — The asymmetry reflects that a stall costs far more than a quality reduction.
5. **Weight buffer level over throughput estimates** — Throughput is noisy and backward-looking; buffer occupancy directly reflects whether the player is keeping up.
6. **Keep the ladder rungs close enough to step between** — Large bitrate gaps force the player to choose between too high and much too low, which causes oscillation.

**Segment duration trade-offs**

```text
SHORT SEGMENTS  (2 s)
  + adapts within ~2 s of a bandwidth change
  + lower live latency (latency >= a few segments)
  - more requests: 1,800 per hour per stream
  - more manifest entries, more per-request overhead
  - keyframe every 2 s costs compression efficiency

LONG SEGMENTS  (10 s)
  + fewer requests, better compression
  - up to 10 s to react to a bandwidth drop
  - a partially downloaded segment is wasted on a switch
  - live latency of 30 s or more

TYPICAL CHOICES
  live, low latency        2 s or chunked delivery
  live, normal             4-6 s
  on demand                6-10 s

BUFFER TARGET
  on demand: 30 s or more - large buffer absorbs
    almost all variance, quality rarely drops
  live: a few segments only - buffer IS latency, so
    the shock absorber must stay small, which is why
    live playback degrades more visibly
```

> **Live streaming and buffering are in direct conflict**  
> For on-demand playback, a large buffer is free insurance: thirty seconds of buffer absorbs nearly all bandwidth variance. For live, buffer is latency — every second buffered is a second behind the event. That means live players run with a few seconds of buffer and therefore have almost no shock absorber, which is exactly why live streams visibly drop quality and stall where on-demand playback of the same content would not.

**Worked example**

A video platform's adaptive configuration, with the reasoning for each number.

**Configuration and behaviour**

```text
LADDER  (rungs close enough to step between)
  1080p  5.0 Mbps
  720p   2.8 Mbps
  480p   1.4 Mbps
  360p   0.8 Mbps
  240p   0.4 Mbps
ratio between rungs ~1.8x
  -> a step down roughly halves the requirement,
     which is usually enough to recover
  -> gaps much larger than 2x cause oscillation:
     the next rung down is far more than needed,
     so the player immediately wants to climb back

SEGMENTS  6 s, keyframe-aligned across all variants
BUFFER TARGET  30 s on demand

SCENARIO: TRAIN JOURNEY
  t=0    20 Mbps  -> 1080p, buffer fills to 30 s
  t=60   drops to 1.5 Mbps
         buffer drains, throughput estimate falls
         player steps down 1080p -> 480p (several rungs,
           immediately - it does not walk down)
         buffer stops draining; playback continues
  t=90   recovers to 20 Mbps
         player waits for sustained evidence, then
           climbs 480p -> 720p -> 1080p over ~30 s
  RESULT: quality varied; playback never stalled

WHAT WOULD HAVE HAPPENED OTHERWISE
  fixed 1080p: stall at t=65, repeated rebuffering
  fixed 480p:  poor quality for the whole session
```

| Metric | Value | Note |
|---|---|---|
| Rung ratio | ~1.8× | avoids oscillation |
| Segment | 6 s | on-demand balance |
| Buffer | 30 s | absorbs variance |
| Down/up | fast / slow | **stalls cost more** |

> **Startup is the highest-stakes decision in the whole algorithm**  
> Viewers abandon during the first few seconds far more readily than at any later point, and at startup the player has no throughput history and no buffer — the two signals it relies on. That is why players start conservatively and climb, why low-latency startup often fetches the first segment at a low rung regardless, and why measuring time-to-first-frame separately from rebuffer rate matters: they are different failures with different causes.

**When to use it**

- **Any video or audio delivered over the public internet**, where bandwidth varies and is not controllable.
- **Mobile playback**, where variance is extreme and stalls are frequent without adaptation.
- **Diverse device populations**, since the same content must serve phones and televisions.
- **Live events**, though with a smaller buffer and therefore more visible adaptation.
- **Anywhere quality degradation is preferable to interruption**, which is nearly all media playback.

**When to avoid it**

- **Do not use it on controlled networks with predictable bandwidth**, where a single rendition is simpler.
- **Do not build a ladder with large gaps between rungs**, which causes oscillation between too high and much too low.
- **Do not use short segments for on-demand**, where the request overhead buys reactivity nobody needs.
- **Do not encode variants with independent keyframes**, which breaks switching.
- **Do not run a large buffer for live**, since buffer is latency.

**Advantages**

- **Playback continues through bandwidth variation**, degrading quality instead of stalling.
- **Each viewer gets the quality their connection supports**, without configuration.
- **Adapts within the session**, not just at its start.
- **Cache-friendly**, since segments are immutable static files that CDNs serve trivially.
- **Device-appropriate**, as a phone and a television select different rungs from the same content.

**Disadvantages**

- **Requires the whole rendition ladder**, multiplying encoding and storage cost.
- **Keyframe alignment constrains encoding**, costing some compression efficiency.
- **Visible quality changes**, which some viewers find distracting.
- **Player complexity** — the switching algorithm is genuinely subtle and easy to get wrong.
- **Live latency** is bounded below by segment duration and buffer.

**Trade-offs**

**Configuration trade-offs**

| Choice | Favours | Cost |
|---|---|---|
| Short segments (2 s) | Fast adaptation; low live latency | Request overhead; compression loss |
| Long segments (10 s) | Efficiency; fewer requests | Slow reaction; high live latency |
| Large buffer (30 s+) | Stall resistance | Latency; wasted download on abandonment |
| Small buffer (5 s) | Low live latency | Little tolerance for variance |
| Aggressive up-switching | Higher average quality | More stalls |
| Conservative up-switching | Fewer stalls | Lower average quality |

The asymmetric default — step down immediately, climb back slowly with sustained evidence — exists because the two errors are not equally costly. Being one rung too low is barely noticed; stalling is the thing viewers abandon over.

**How it fails**

**Adaptive streaming failures**

| Failure | Cause | Fix |
|---|---|---|
| Glitch or audio gap when quality changes | Keyframes not aligned across variants | Encode all variants with identical keyframe placement |
| Quality oscillates constantly | Ladder rungs too far apart, or up-switching too eager | Closer rungs; require sustained evidence before stepping up |
| Stalls despite adequate bandwidth | Algorithm trusts noisy throughput over buffer level | Weight buffer occupancy in the decision |
| Slow start / abandonment | Starting variant too high with no history | Start conservatively; measure time-to-first-frame separately |
| Live stream far behind the event | Long segments plus a large buffer | Shorter segments; smaller buffer; chunked delivery |
| Player references a missing segment | Manifest published before segments existed | Publish the manifest after its segments, atomically |
| Wasted bandwidth on abandoned views | Very large buffer filled before the viewer left | Cap buffer ahead; fill progressively |

**Limits**

> **Typical parameters**
>
> - **Segment duration**: 2 s for low-latency live, 4–6 s for normal live, 6–10 s on demand.
> - **Buffer target**: 30 s or more on demand; a few seconds for live, because buffer is latency.
> - **Rung ratio**: around 1.5–2× between adjacent bitrates, so a single step down is meaningful without overshooting.
> - **Live latency floor**: roughly segment duration multiplied by the number of segments buffered, unless chunked delivery is used.
> - **Switch reaction time**: bounded below by segment duration — you cannot adapt faster than you decide.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Adaptive bitrate (HLS/DASH) | Public-internet media | Ladder cost; player complexity |
| Single fixed rendition | Controlled networks | Stalls or permanently poor quality |
| Progressive download | Short clips | No adaptation; whole file |
| Chunked / low-latency HLS | Live with low latency | More complex delivery; less buffer |
| WebRTC | Sub-second interactivity | Different infrastructure; harder to scale |
| Server-side adaptation | Fine control | Stateful per viewer; loses CDN cacheability |

WebRTC is the alternative to reach for when latency must be sub-second — genuine two-way interaction — but it abandons the property that makes adaptive streaming scale: immutable segments served from ordinary CDN caches. That single difference is why broadcast-scale live uses segmented delivery even though it costs seconds of latency.

**In real systems**

- **HLS and DASH** are the two dominant segmented formats, differing in manifest syntax more than in principle.
- **Buffer-based switching algorithms** largely replaced purely throughput-based ones, because buffer occupancy is a less noisy signal of whether the player is keeping up.
- **Low-latency HLS and chunked CMAF** deliver partial segments so live latency is not bounded by full segment duration.
- **Per-title encoding** tunes the ladder to each video's complexity, which changes which rungs are worth producing.
- **CDN delivery of immutable segments** is what allows a single live event to reach millions — segments are ordinary cacheable files.

**Common mistakes**

- **Unaligned keyframes across variants**, causing glitches on every switch.
- **Ladder rungs too far apart**, producing oscillation.
- **Trusting throughput estimates over buffer level**, stalling despite adequate bandwidth.
- **Starting at too high a rung**, stalling in the first seconds when abandonment is highest.
- **Large buffers on live streams**, trading latency for insurance nobody asked for.
- **Publishing manifests before their segments exist**, causing playback failures.
- **Symmetric switching**, climbing up as fast as it steps down and causing the stalls it avoided.

**The staff-level view**

Adaptive streaming problems reported as player bugs usually originate in the encoding stage or in the ladder's shape.

- **Verify keyframe alignment as an encoding requirement.** Glitching on quality change is almost always misaligned variants, and it is invisible until someone switches at exactly the wrong moment.
- **Design the ladder with step-ability in mind.** Rungs separated by much more than a factor of two force a choice between too high and far too low, which is what produces visible oscillation.
- **Measure startup and rebuffering separately.** They are distinct failures — one is the starting decision, the other is the steady-state algorithm — and a single quality-of-experience number hides which is broken.
- **Accept that live and buffering are in tension.** Any request to reduce live latency is a request to reduce the shock absorber, which will increase stalls; that trade should be made explicitly rather than discovered.
- **Keep segments immutable and cacheable.** Server-side adaptation looks appealing until you realise it makes every viewer's stream unique and destroys CDN efficiency.

**Go deeper**

Adaptive streaming encodes media at several bitrates, cuts each into short keyframe-aligned segments, and lets the player choose a quality before every segment based on measured throughput and buffer level. Bandwidth variance becomes quality variance rather than stalling, which is the trade viewers strongly prefer. Keyframe alignment across variants is a hard encoding requirement — without it, every switch glitches.

The switching algorithm is deliberately asymmetric: step down immediately, often several rungs, because a draining buffer leads to a stall; climb back only on sustained evidence, because an eager up-switch causes the stall it was avoiding. Buffer occupancy is weighted over throughput estimates, since throughput is noisy and backward-looking while buffer directly measures whether the player is keeping up.

Configuration differs sharply between on-demand and live. On demand, a thirty-second buffer is free insurance that absorbs nearly all variance. For live, buffer is latency, so the player runs with a few seconds and therefore adapts visibly and stalls more readily. The property that makes the whole approach scale is that segments are immutable static files, cacheable by ordinary CDNs with no per-viewer server state — which is what server-side adaptation would destroy.

Adaptive bitrate streaming solves an unavoidable problem: the bandwidth that matters is the bandwidth during playback, and it varies continuously in ways no starting decision can anticipate.

**The decision moves into the player and repeats.** Media is encoded at several bitrates, each cut into segments of a few seconds, described by a manifest. Before every segment the player picks a variant. This converts bandwidth variance into quality variance rather than interruption — an exchange viewers accept readily, since a resolution change is barely noticed while a stall is what they abandon over.

**Keyframe alignment is the hidden hard requirement.** A switch means fetching segment N from a different variant than segment N−1, which only works if every variant's segments begin at identical media timestamps. Independent keyframe placement across variants produces glitches, audio gaps or decode errors at exactly the moments the player is trying to help. This constrains the encoder — forcing keyframes at fixed intervals costs some compression efficiency — and misalignment is the single most common cause of adaptive playback appearing broken.

**Buffer is the real control signal.** Early algorithms estimated throughput from recent downloads and chose accordingly, but throughput estimates are noisy, backward-looking, and easily confused by a single slow fetch. Buffer occupancy is a direct measurement of whether download is outpacing playback. Modern algorithms weight buffer heavily and fall back on throughput mainly at startup, when no buffer exists — which is also the moment with no history and the highest abandonment risk, so players start conservatively and climb.

**The asymmetry is the design.** Dropping happens immediately and can skip several rungs, because the cost of a stall vastly exceeds the cost of lower resolution. Climbing requires sustained evidence, because an eager up-switch drains the buffer and causes the very stall the mechanism exists to prevent. Related to this, ladder rungs should sit within roughly a factor of two of each other: wider gaps force a choice between too high and far too low, producing the visible oscillation users complain about.

**Live and on-demand are configured differently because buffer means different things.** On demand, thirty seconds of buffer is free insurance that absorbs almost all variance. For live, buffer is latency — each buffered second is a second behind the event — so live players run with a few seconds and consequently have almost no tolerance for variation. Any request to reduce live latency is a request to reduce the shock absorber, and should be presented as the stall-rate trade it actually is. Chunked delivery of partial segments is what breaks the dependency between latency and segment duration.

**Immutable segments are why it scales.** Every viewer at the same position and quality requests byte-identical files, so ordinary CDN caching serves nearly all traffic from edge nodes and the origin sees a small fraction. All adaptation lives in the player, so servers hold no per-viewer state and the delivery path reduces to static file serving. That property is what permits a single live event to reach millions, and it is precisely what server-side adaptation would spend — an important thing to name when someone proposes personalising the stream.

**Prove it — interview questions**

1. **[Basic] What problem does adaptive bitrate streaming solve?**

   <details><summary>Model answer</summary>

   Bandwidth varies during playback and cannot be predicted at the start. A single-quality stream forces a bad choice: high bitrate and repeated stalling when the connection drops, or low bitrate and permanently poor quality even when it is excellent. Adaptive streaming makes the quality decision per segment rather than per session, so when bandwidth falls the player requests a lower-bitrate segment and keeps playing. It converts bandwidth variance into quality variance instead of interruption, which is the trade viewers overwhelmingly prefer.

   </details>

2. **[Basic] Why must keyframes align across variants?**

   <details><summary>Model answer</summary>

   Because a quality switch means downloading the next segment from a different variant. If each variant was encoded with independent keyframe placement, its segments start at different media timestamps, so the switch produces a visible glitch, an audio gap, or a decode error. Aligned keyframes across the whole ladder ensure every variant's segment N covers exactly the same time range, making switches seamless. It is an encoding-stage requirement, and misalignment is the most common cause of adaptive playback misbehaving.

   </details>

3. **[Senior] Why is buffer level a better signal than throughput?**

   <details><summary>Model answer</summary>

   Throughput estimates are noisy and backward-looking — they describe the recent past on a connection that may already have changed, and a single slow segment download can be congestion, a cache miss, or a routing hiccup. Buffer occupancy is a direct measurement of whether the player is winning or losing the race: a growing buffer means download outpaces playback regardless of what the throughput number says. That is why modern algorithms weight buffer heavily and use throughput mainly at startup, when there is no buffer to read.

   </details>

4. **[Senior] Why step down fast but climb up slowly?**

   <details><summary>Model answer</summary>

   Because the two errors have very different costs. Being one rung lower than necessary is barely noticeable; stalling interrupts playback and is what viewers abandon over. So on any sign of trouble the player drops immediately, often several rungs at once, to stop the buffer draining. Climbing back requires sustained evidence that the higher bitrate is genuinely supportable, because an over-eager up-switch consumes the buffer and causes exactly the stall the algorithm exists to prevent.

   </details>

5. **[Staff] How would you configure adaptive streaming for a live event versus on-demand video?**

   <details><summary>Model answer</summary>

   The structural difference is that for on-demand the buffer is free insurance and for live it is latency. On demand I would use six-to-ten-second segments and a buffer target of thirty seconds or more, which absorbs almost all bandwidth variance so quality rarely needs to drop; the request overhead is low and reaction speed hardly matters because the buffer is doing the work. For live, every buffered second is a second behind the event, so the buffer shrinks to a few seconds and segments to two-to-four, which means the player has almost no shock absorber and will visibly adapt and occasionally stall on exactly the connection that would play the on-demand version flawlessly. If latency must go lower still, chunked delivery sends partial segments so latency is not bounded by full segment duration. The important thing is to state the trade explicitly to whoever is asking for lower latency, because they are asking for more stalls whether they realise it or not.

   </details>

6. **[Principal] What makes adaptive streaming scale to millions of concurrent viewers?**

   <details><summary>Model answer</summary>

   That the segments are immutable static files. Every viewer requesting the 720p segment at position 400 requests byte-identical content, so ordinary CDN caching serves nearly all of it from edge nodes and origin traffic is a tiny fraction of delivered traffic. All the adaptation logic lives in the player, which means the server holds no per-viewer state and the delivery path is just static file serving — that is the property that lets a single live event reach an enormous audience on infrastructure that is, architecturally, a file cache. It is also why server-side adaptation is a trap: it gives finer control but makes every viewer's stream unique, destroying cacheability and turning a cheap static delivery problem into an expensive stateful one. When someone proposes personalising the stream, that cacheability is the thing being spent, and it is usually worth far more than what is being bought.

   </details>

---

### Content delivery networks

*Serve content from locations near the user so latency falls and origin load collapses — with cache invalidation as the central design problem.*

**Flow:** `Client` → `Edge POP` → `Regional shield` → `Origin` → `Invalidation`

> **The 30-second version**  
> Cache content near users so latency drops and the origin serves a few per cent of traffic — with cache-key design driving hit ratio and immutable URLs removing invalidation.

**The problem**

An origin in one region serves users worldwide. A user on another continent pays a round trip of two hundred milliseconds or more per request, and connection setup multiplies that. Meanwhile every request — including millions for identical static files — reaches the origin, which must be provisioned for a load that is almost entirely redundant.

Both problems have the same root: distance and duplication. The same bytes travel the same long path repeatedly for different users.

> **A CDN buys latency and origin protection with one mechanism**  
> Caching copies near users shortens the path — often from two hundred milliseconds to twenty — and simultaneously means the origin serves one request instead of a million. The two benefits come from the same cache, which is why a CDN is usually the highest-leverage single infrastructure change for a globally accessed system.

**Mental model**

A hierarchy of caches sits between users and the origin. Requests reach the nearest edge point of presence; a miss goes to a regional shield; a miss there goes to the origin. Each layer absorbs requests, and the hit ratio at the edge determines nearly everything about the system's economics.

1. **Edge POP** — Many locations close to users. Low latency, small cache, holds the popular subset.
2. **Regional shield** — Fewer, larger caches that absorb misses from many edges so the origin sees far less.
3. **Origin** — The source of truth, reached only on a full miss.
4. **Cache key** — What makes two requests identical — the URL plus whatever headers vary. Getting this wrong is the main cause of poor hit ratios.
5. **Invalidation** — How content changes propagate. The genuinely hard part.

> **Without a shield tier, every edge misses independently**  
> A hundred edge locations each experiencing a cache miss for the same object send a hundred requests to the origin. When a popular object expires everywhere at once, that becomes a thundering herd. A shield tier means edges miss to a regional cache that fetches from the origin once, which is often the difference between an origin that copes and one that falls over during a traffic spike.

**How it works**

**The cache key determines the hit ratio**

```text
A CACHE KEY is what makes two requests "the same request".
Default: the URL. Everything added to it multiplies the
number of cached copies and divides the hit ratio.

COMMON HIT-RATIO DESTROYERS
  analytics query parameters in the URL
    /img.jpg?utm_source=a  and  ?utm_source=b
    -> two cache entries for identical bytes
    -> strip unknown parameters from the key
  Vary: User-Agent
    -> a separate copy per browser version
    -> effectively no caching at all
  Vary: Cookie
    -> a separate copy per user - catastrophic
  cache-busting on every deploy for unchanged files
    -> whole cache cold after each release

GOOD PRACTICE
  include in the key only what genuinely changes the bytes
    Accept-Encoding (gzip vs brotli)   yes
    device class (mobile vs desktop)   sometimes
    User-Agent in full                 almost never
  normalise parameter order and strip tracking parameters

HIT RATIO ARITHMETIC
  95% hit ratio -> origin sees 5% of traffic
  90% hit ratio -> origin sees 10%  = DOUBLE the load
  a few points of hit ratio is a large multiple of
  origin capacity
```

1. **Design the cache key before anything else** — Hit ratio is dominated by key design, and a few percentage points of hit ratio doubles or halves origin load.
2. **Use immutable URLs for versioned assets** — Content-hashed filenames can be cached for a year and never invalidated, which removes the hard problem entirely for most static content.
3. **Add a shield tier for popular content** — It converts many independent edge misses into one origin fetch and prevents thundering herds on expiry.
4. **Prefer stale-while-revalidate over hard expiry** — Serving slightly stale content while refreshing in the background eliminates the latency cliff and the herd at expiry.
5. **Separate browser TTL from CDN TTL** — A short browser TTL keeps clients current while a long CDN TTL protects the origin; purging the CDN is possible, purging browsers is not.
6. **Treat purge as slow and eventual** — Global invalidation takes time and may partially fail, so correctness should not depend on it completing promptly.

**Invalidation strategies, from best to worst**

```text
1  IMMUTABLE URLs  (best - no invalidation at all)
   /app.a3f9c2.js  cached one year
   new content -> new URL
   -> nothing to purge, ever
   -> use for all versioned static assets

2  SHORT TTL  (simple, predictable)
   Cache-Control: max-age=60
   staleness bounded by 60 s; no purge machinery
   -> good for content that changes continuously

3  STALE-WHILE-REVALIDATE  (best for dynamic)
   max-age=60, stale-while-revalidate=3600
   after 60 s: serve stale INSTANTLY, refresh in background
   -> no latency cliff at expiry
   -> no thundering herd
   -> users see content up to 60 s old; nobody waits

4  EXPLICIT PURGE  (necessary but weakest)
   call the CDN API to invalidate a path or tag
   -> takes seconds to minutes to propagate globally
   -> may partially fail
   -> rate-limited on most providers
   -> NEVER build correctness on prompt purge

5  PURGE EVERYTHING  (emergency only)
   -> cache goes cold; all traffic hits the origin
   -> can take down the origin you were protecting

DESIGN RULE
push as much content as possible up this list.
Most invalidation pain comes from using strategy 4
where strategy 1 was available.
```

> **Caching authenticated responses is the classic catastrophic bug**  
> A response containing one user's data, cached at the edge and served to another, is a serious data leak — and it happens through ordinary mistakes: a missing private directive, a cache key that ignores the session, or a framework setting a permissive header by default. Any endpoint returning user-specific data must be explicitly non-cacheable at shared caches, and this deserves an automated check rather than a code-review convention.

**Worked example**

A globally accessed application's CDN configuration, by content type.

**Configuration by content type**

```text
STATIC ASSETS  (js, css, images with content hashes)
  Cache-Control: public, max-age=31536000, immutable
  cache key: URL only
  -> hit ratio ~99%, never purged
  -> new deploy = new hashes = new URLs

MEDIA SEGMENTS  (video, audio)
  Cache-Control: public, max-age=86400
  cache key: URL only
  -> immutable by nature; very high hit ratio
  -> this is what makes streaming economical

API RESPONSES, PUBLIC AND READ-HEAVY
  Cache-Control: public, max-age=60,
                 stale-while-revalidate=600
  cache key: URL + Accept-Encoding
  -> origin sees roughly one request per URL per minute
  -> no herd at expiry, no latency cliff

HTML PAGES, PERSONALISED
  Cache-Control: private, no-store
  -> never cached at shared caches
  -> enforce with an automated check, not a convention

TOPOLOGY
  client -> edge POP (200+ locations)
         -> regional shield (~10)
         -> origin
  shield exists so 200 edges missing the same object
  produce ONE origin fetch, not 200

RESULT
  p50 latency   200 ms -> 25 ms
  origin load   100%   -> ~3%
  origin fleet  sized for misses and writes only
```

| Metric | Value | Note |
|---|---|---|
| Latency | 200 → 25 ms | distance removed |
| Origin load | 100% → 3% | **hit ratio 97%** |
| Static | 1 year, immutable | never purged |
| Dynamic | 60 s + SWR | no herd |

> **Hit ratio is nonlinear in origin load**  
> Going from ninety-five to ninety per cent hit ratio doubles origin traffic; going from ninety-nine to ninety-five multiplies it fivefold. That means a few points of hit ratio — usually lost to cache keys that include tracking parameters or an over-broad Vary header — translates directly into multiples of origin capacity, and is almost always cheaper to fix than to provision around.

**When to use it**

- **Geographically distributed users**, where distance dominates latency.
- **Static assets and media**, which are immutable and cache near-perfectly.
- **Read-heavy public API responses**, where brief staleness is acceptable.
- **Traffic spikes**, since the edge absorbs load the origin could never serve.
- **Egress cost reduction**, as CDN bandwidth is typically far cheaper than origin egress.
- **Absorbing volumetric attacks**, which the edge is built to handle.

**When to avoid it**

- **Do not cache personalised or authenticated responses at shared caches** — this is a data-leak class of bug.
- **Do not cache content requiring strict freshness**, unless a bounded staleness is genuinely acceptable.
- **Do not depend on prompt global purge for correctness**; it is slow, eventual and can partially fail.
- **Do not add request attributes to the cache key casually**, since each one divides the hit ratio.
- **Do not purge everything** except in emergencies — a cold cache can take down the origin.

**Advantages**

- **Large latency reduction** by removing distance from the common path.
- **Origin load collapses**, often to a few per cent of total traffic.
- **Absorbs spikes and attacks** at capacity the origin could not provision.
- **Cheaper bandwidth** than origin egress at scale.
- **Improves availability**, since cached content can be served while the origin is down.

**Disadvantages**

- **Invalidation is genuinely hard** and purge is slow and eventual.
- **Staleness is inherent**, so users may see content that has already changed.
- **Misconfiguration can leak data** when authenticated responses are cached.
- **Debugging is harder**, with behaviour varying by POP and cache state.
- **Another vendor in the critical path**, whose outage becomes yours.

**Trade-offs**

**TTL and invalidation trade-offs**

| Strategy | Freshness | Origin load | Best for |
|---|---|---|---|
| Immutable, 1-year TTL | Perfect via new URLs | Minimal | Versioned static assets |
| Long TTL + purge | Depends on purge speed | Low | Content that changes rarely |
| Short TTL (60 s) | Bounded staleness | Moderate | Continuously changing content |
| Stale-while-revalidate | Bounded, no latency cliff | Low | Dynamic read-heavy content |
| No caching | Always fresh | Full | Personalised or authenticated |

Most invalidation pain comes from using explicit purge where immutable URLs were available. Content-hashed filenames convert the hardest problem in caching into a non-problem, and are worth restructuring a build pipeline to obtain.

**How it fails**

**CDN failure modes**

| Failure | Cause | Fix |
|---|---|---|
| User sees another user's data | Authenticated response cached at a shared cache | Explicit private/no-store; automated check on personalised endpoints |
| Origin overwhelmed despite the CDN | Low hit ratio from a bad cache key | Strip tracking parameters; narrow Vary |
| Origin spike when a popular object expires | Many edges missing simultaneously | Shield tier; stale-while-revalidate; jittered TTLs |
| Stale content after a deploy | Long TTL with slow purge | Immutable content-hashed URLs |
| Origin collapse after a full purge | Cache cold, all traffic passing through | Purge by tag or path; warm before purging broadly |
| Inconsistent behaviour between users | Different POPs with different cache state | Expect it; debug with POP-identifying headers |
| Long browser-cached staleness | Long max-age sent to browsers | Short browser TTL, long CDN TTL — browsers cannot be purged |

**Limits**

> **Reference figures**
>
> - **Latency**: intercontinental round trips of 150–250 ms reduce to 10–30 ms from a nearby POP.
> - **Hit ratio**: 95–99% for static assets and media; each lost point is a large relative increase in origin load.
> - **Purge propagation**: seconds to minutes globally, and it can partially fail — never a correctness mechanism.
> - **Origin load**: at a 97% hit ratio the origin serves roughly 3% of requests, so it is sized for misses and writes.
> - **Browser TTL** cannot be revoked, which is why it should be short even when the CDN TTL is long.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Commercial CDN | Global static and media delivery | Vendor in the path; cost at scale |
| Edge compute at the CDN | Personalisation without origin round trips | Constrained runtime; another deployment target |
| Multi-region origins | Dynamic, non-cacheable workloads | Data replication complexity |
| Reverse proxy cache at origin | Origin protection without distance benefit | No latency improvement |
| Client-side caching | Repeat visits by the same user | No benefit to first visits or other users |
| Peer-assisted delivery | Very large distributions | Complexity; client cooperation |

Edge compute is the interesting recent addition: it lets personalisation happen near the user rather than forcing a round trip to the origin, which recovers cacheability for pages that would otherwise be entirely dynamic — assembling a cached shell with a small personalised fragment at the edge.

**In real systems**

- **Content-hashed asset filenames** are standard in modern build tooling precisely because they make year-long caching safe with no invalidation.
- **Shield or mid-tier caching** is offered by every major CDN because independent edge misses otherwise concentrate on the origin.
- **Stale-while-revalidate** is widely used for dynamic content, since it removes both the latency cliff and the expiry herd.
- **Video segment delivery** depends entirely on CDN caching; immutable segments are what make streaming economically possible at scale.
- **Edge compute platforms** run personalisation logic at POPs so that pages needing user-specific fragments still avoid an origin round trip.

**Common mistakes**

- **Caching authenticated responses**, leaking one user's data to another.
- **Tracking parameters in the cache key**, fragmenting identical content across entries.
- **Vary on User-Agent or Cookie**, effectively disabling caching.
- **No shield tier**, so a popular object's expiry becomes an origin herd.
- **Relying on purge for correctness**, despite it being slow and eventual.
- **Long browser TTLs**, creating staleness that cannot be revoked.
- **Purging everything to fix one object**, taking the origin down with a cold cache.

**The staff-level view**

CDN decisions are usually made once and then quietly determine origin capacity for years.

- **Treat cache-key design as a capacity decision.** Hit ratio maps nonlinearly to origin load, and a few points lost to tracking parameters or a broad Vary header can double the fleet you need.
- **Push content toward immutable URLs wherever possible.** Content hashing eliminates invalidation entirely for static assets and is worth restructuring a build for; explicit purge is the weakest strategy and should be the last resort.
- **Make caching of authenticated responses structurally impossible.** This is a data-leak class of failure, so it needs an automated check on personalised endpoints rather than reliance on review.
- **Require a shield tier before a traffic event.** Independent edge misses on a popular object create a thundering herd precisely when the origin is least able to absorb it.
- **Never let correctness depend on purge.** It is slow, eventual, sometimes partial and often rate-limited; design so that stale content is tolerable for its TTL.

**Go deeper**

A CDN caches content at locations near users, which removes most of the distance from the latency and means the origin serves only cache misses. At a typical hit ratio the origin handles a few per cent of total traffic, so it is sized for misses and writes rather than for the full load — and intercontinental latency falls from hundreds of milliseconds to tens.

Cache-key design dominates the outcome. Anything added to the key multiplies the number of stored copies and divides the hit ratio, so tracking parameters in URLs and broad Vary headers are the usual causes of poor performance. Because origin load is the complement of hit ratio, a few lost points translate into a multiple of origin capacity.

Invalidation is the hard part, and the best strategy is to avoid needing it: content-hashed filenames make static assets cacheable for a year with no purge ever. Where content genuinely changes, short TTLs with stale-while-revalidate give bounded staleness without a latency cliff or an expiry herd. Explicit purge is slow, eventual and sometimes partial, so correctness should never depend on it — and caching authenticated responses at a shared cache is a data-leak class of bug that deserves an automated check.

A content delivery network addresses distance and duplication with a single mechanism: copies of content held close to users.

**Two benefits, one cache.** Latency falls because the round trip is to a nearby point of presence rather than a distant origin — typically from two hundred milliseconds to twenty or thirty. Origin load falls because a request served at the edge never reaches the origin at all. At a ninety-seven per cent hit ratio the origin serves three per cent of requests, so it is provisioned for misses and writes. Secondary benefits follow: cheaper bandwidth, absorption of traffic spikes and volumetric attacks, and continued service of cached content during an origin outage.

**Cache-key design is a capacity decision.** The key defines when two requests are the same request, and anything included in it multiplies stored copies while dividing the hit ratio. Analytics parameters appended to asset URLs create separate entries for byte-identical content; a Vary on User-Agent produces a copy per browser version, which is effectively no caching. Because origin load is the complement of hit ratio, the relationship is nonlinear at the top: ninety-five to ninety per cent doubles origin traffic. Fixing the key is almost always cheaper than provisioning around it.

**Invalidation is the hard problem, and it is mostly avoidable.** Content-hashed filenames make each version a distinct URL, so cached copies are never wrong and a year-long immutable TTL is safe — invalidation simply does not arise. For content that genuinely changes, a short TTL bounds staleness, and stale-while-revalidate serves the stale copy instantly while refreshing in the background, removing both the latency cliff at expiry and the thundering herd it would otherwise cause. Explicit purge remains necessary for occasional corrections but is slow, eventual, sometimes partial and frequently rate-limited, so nothing whose correctness matters should depend on it completing promptly.

**Topology matters more than it appears.** With only edge caches, a hundred locations missing the same object generate a hundred origin requests, and a popular object expiring everywhere produces a coordinated spike at precisely the wrong moment. A shield tier converts those into one origin fetch, and jittered TTLs prevent synchronised expiry. This is usually the difference between an origin that survives a traffic event and one that does not.

**Caching authenticated responses is a different class of failure.** Serving one user's personalised page to another is a privacy incident, not a performance regression, and it arises from ordinary mistakes: a missing private directive, a framework default, a cache key that ignores the session. Endpoints returning user-specific data should be structurally prevented from being cached at shared caches, with an automated check rather than a review convention, because the consequence is disproportionate to the ease of the mistake.

**Browser caches cannot be purged.** A long max-age sent to clients creates staleness with no remedy — the only option is to wait it out or change the URL. The standard pattern separates the two: a short browser TTL keeps clients able to pick up corrections quickly, while a long CDN TTL protects the origin, since the CDN is something you can actually invalidate. Getting this backwards produces the worst kind of stale content: distributed across users' machines, invisible, and outside your control.

**Prove it — interview questions**

1. **[Basic] What does a CDN actually give you?**

   <details><summary>Model answer</summary>

   Two things from one mechanism. Latency: serving from a location near the user removes most of the distance, so an intercontinental request that cost two hundred milliseconds costs twenty. And origin protection: if the edge serves ninety-seven per cent of requests, the origin handles three per cent, so it is sized for misses and writes rather than total traffic. There are secondary benefits — cheaper bandwidth, absorbing spikes and volumetric attacks, serving cached content while the origin is down — but the latency and load reduction are the core.

   </details>

2. **[Basic] Why do immutable URLs make caching easy?**

   <details><summary>Model answer</summary>

   Because they remove invalidation from the problem entirely. If a file's name contains a hash of its contents, then changed content means a different filename, so the cached copy of the old URL is never wrong — it is simply no longer referenced. That lets you cache for a year with an immutable directive and never purge anything. Since invalidation is the genuinely hard part of caching, converting it into a non-problem for all versioned static assets is the highest-value thing a build pipeline can do.

   </details>

3. **[Senior] Why does hit ratio matter so much more than it looks?**

   <details><summary>Model answer</summary>

   Because origin load is the complement of hit ratio, so the relationship is nonlinear at the top end. At ninety-five per cent the origin sees five per cent of traffic; at ninety per cent it sees ten — double the load for a five-point drop. Going from ninety-nine to ninety-five multiplies origin traffic fivefold. Those points are usually lost to cache-key problems: tracking parameters creating separate entries for identical bytes, or a Vary header on User-Agent producing a copy per browser version. Fixing the key is almost always cheaper than provisioning the origin capacity to absorb the miss traffic.

   </details>

4. **[Senior] What is stale-while-revalidate and why prefer it?**

   <details><summary>Model answer</summary>

   It lets the cache serve content past its freshness lifetime while fetching a fresh copy in the background. Without it, expiry creates a latency cliff — the unlucky request that arrives at expiry waits for a full origin fetch — and a thundering herd, since every edge that expires simultaneously goes to the origin at once. With it, users always get an instant response, staleness stays bounded by the max-age, and the origin sees roughly one refresh per object per interval. For dynamic read-heavy content it gives most of the freshness of a short TTL with far better latency and origin behaviour.

   </details>

5. **[Staff] How would you configure a CDN for an application with static assets, media, public APIs and personalised pages?**

   <details><summary>Model answer</summary>

   Different content types get genuinely different treatment. Static assets get content-hashed filenames cached for a year as immutable, with the URL alone as the cache key — hit ratio approaching ninety-nine per cent and no invalidation ever, since a deploy produces new URLs. Media segments are immutable by nature and cache the same way, which is what makes streaming economical. Public read-heavy API responses get a short max-age with stale-while-revalidate, so the origin sees about one request per URL per interval and no user ever waits at expiry. Personalised pages are marked private and no-store so shared caches never hold them, and I would enforce that with an automated check rather than review, because caching an authenticated response is a data leak rather than a performance bug. Topologically I would insist on a shield tier so that hundreds of edges missing the same object produce one origin fetch rather than hundreds, and I would keep browser TTLs short even where CDN TTLs are long, since the CDN can be purged and browsers cannot.

   </details>

6. **[Principal] What are the failure modes that make CDNs risky rather than just beneficial?**

   <details><summary>Model answer</summary>

   Three, and they are all configuration rather than technology. The severe one is caching authenticated responses at a shared cache, which serves one user's data to another — a correctness and privacy failure that arises from ordinary mistakes like a missing directive or a cache key that ignores the session, so it needs to be structurally prevented rather than reviewed for. The expensive one is a low hit ratio from cache-key fragmentation, which silently multiplies origin load and is usually discovered as a capacity problem rather than a caching one. And the sharp one is depending on purge: it is slow, eventual, sometimes partial and often rate-limited, so a system whose correctness assumes prompt global invalidation will be wrong at the worst moment — and the instinctive fix, purging everything, empties the cache and directs full traffic at an origin that has been sized for three per cent of it. The discipline is to push content toward immutable URLs and bounded staleness so that purge is a convenience rather than a dependency.

   </details>

---

### Signed media access

*Grant time-limited, cryptographically verifiable access to private content so the CDN can enforce authorisation without consulting your service.*

**Flow:** `Authorisation check` → `Signed URL` → `Edge verification` → `Expiry` → `Revocation`

> **The 30-second version**  
> Authorise once in your service, encode the decision as a time-limited signature the CDN can verify statelessly — and treat the expiry as your revocation window, because signed URLs cannot be withdrawn.

**The problem**

Private media must be served from a CDN for latency and cost, but the CDN does not know who your users are. The obvious approaches both fail: proxying every media request through your service destroys the CDN's benefit entirely, while making the content public means anyone with the URL can read it forever.

The second failure is worse than it sounds. Media URLs leak constantly — through referrer headers, browser history, shared links, logs, screenshots and support tickets — so a permanent unguessable URL is a permanent unauthenticated grant.

> **The signature moves the authorisation decision to the edge**  
> Your service performs the authorisation check once and encodes the result as a signature over the URL and an expiry time. The CDN verifies that signature with a shared key and needs to know nothing about users, sessions or permissions. Authorisation happens in your service; enforcement happens at the edge; and the two are connected by a token rather than a request.

**Mental model**

A signed URL is a bearer capability with an expiry. It says: whoever holds this may fetch this specific resource until this specific time, and the signature proves the statement was issued by someone holding the key.

1. **Authorise** — Your service checks whether this user may access this resource. This is the only place real authorisation happens.
2. **Sign** — Produce a signature over the path, the expiry and any constraints, using a key shared with the CDN.
3. **Deliver** — Hand the signed URL to the client, which uses it directly against the edge.
4. **Verify** — The edge recomputes the signature and checks the expiry. No call to your service.
5. **Expire** — The capability lapses on its own, which is the primary revocation mechanism.

> **A signed URL is a bearer token, and bearer tokens get shared**  
> Anyone who obtains the URL has the access it grants, for as long as it is valid. URLs leak through referrers, history, shared messages, logs and screenshots, and nothing in the mechanism distinguishes the intended user from anyone they forwarded it to. That is why expiry must be short, why the signature should be constrained as tightly as possible, and why the URL should never appear in anything that logs full request paths.

**How it works**

**What is signed, and what each constraint buys**

```text
SIGNATURE INPUT  (all of it must be verified)
  path or path prefix     what may be fetched
  expiry timestamp        when it stops working
  optional: client IP     binds to one network
  optional: HTTP method   GET only, not PUT
  optional: key id        which key signed it

signature = HMAC(secret, canonical(path, expiry, ...))

VERIFICATION AT THE EDGE
  recompute the HMAC from the request
  constant-time compare
  check expiry against the edge clock
  -> no call to your service, no user database

WHAT EACH CONSTRAINT COSTS AND BUYS
  short expiry
    + a leaked URL is useless quickly
    - breaks resumable downloads and long playback
    - clock skew between your service and the edge
      matters at very short lifetimes
  IP binding
    + stops the URL working from elsewhere
    - breaks mobile network handoff, VPNs, and any
      CDN that presents a different client IP
    - use only when you control the client network
  path prefix
    + one signature covers a whole media ladder
    - a leak exposes the whole prefix, not one file

THE EXPIRY-LENGTH TRADE
  60 s     page-embedded images; refresh on load
  15 m     typical download link
  4-8 h    a video playback session
  days     almost always wrong
```

1. **Authorise before signing, never after** — The signature carries no identity — it only proves a decision was made. If you sign without checking, you have issued access to whoever asked.
2. **Keep expiry as short as the use case tolerates** — Expiry is the main revocation mechanism, and a long-lived signed URL is effectively a permanent public link.
3. **Sign a path prefix for media ladders** — A video has many segments; one signature covering the prefix avoids signing thousands of URLs, at the cost of a leak exposing the whole asset.
4. **Support key rotation from the start** — Include a key identifier so the edge can verify against multiple keys during a rotation, otherwise rotation means an outage.
5. **Keep signed URLs out of logs and referrers** — Full-path logging turns your log store into a collection of live access grants.
6. **Provide a refresh path for long sessions** — A playback session longer than the expiry needs to renew the signature mid-stream rather than fail at the worst moment.

**Revocation is the weak point**

```text
A SIGNED URL CANNOT BE UNSIGNED.
Once issued, it is valid until it expires. There is no
list to remove it from.

IF ACCESS MUST BE REVOKED IMMEDIATELY
  1  short expiry + frequent re-signing
     revocation takes effect within one expiry window
     -> the standard answer; tune the window to the
        tolerance
  2  rotate the signing key
     invalidates EVERY outstanding URL at once
     -> a blunt emergency tool; every active session
        breaks
  3  edge-side denylist
     check a revocation list at the edge
     -> reintroduces state and lookups at the edge,
        which is what signing was avoiding
  4  signed cookies instead of signed URLs
     the cookie is scoped to the browser, not the link
     -> does not leak through shared URLs
     -> but is still a bearer credential

DESIGN CONSEQUENCE
choose the expiry from the revocation requirement,
not from convenience. If access must stop within
five minutes of an entitlement change, the expiry
is five minutes - and the client must be built to
refresh.
```

> **Signed cookies and signed URLs solve slightly different problems**  
> A signed URL travels with the link, so forwarding it forwards the access. A signed cookie is scoped to the browser and covers a whole path prefix, so sharing a link does not share access — which is usually what you want for a streaming session with many segment requests. The trade is that cookies need domain alignment between your site and the media host, and are awkward for native clients.

**Worked example**

A private video platform, showing where signing sits and what expiry each case needs.

**Design with lifetimes justified**

```text
PLAYBACK REQUEST
  1  client asks the API to play video X
  2  API authorises: does this user have this video?
       subscription active, not region-blocked,
       not revoked
  3  API signs the MANIFEST URL and the SEGMENT PREFIX
       expiry: 6 hours (longer than any single session)
       prefix covers all renditions and segments
  4  client plays; every segment request is verified
       at the edge with no further service calls
  5  session longer than 6 h -> client re-requests
       a fresh signature; playback continues

THUMBNAILS ON A BROWSE PAGE
  expiry: 5 minutes
  -> the page is transient; a leaked thumbnail URL
     stops working almost immediately

DOWNLOAD LINK SENT BY EMAIL
  expiry: 24 hours, single asset path
  -> long enough to be useful, short enough that an
     inbox breach is time-limited
  -> NOT a prefix: a download link should grant one file

WHY A PREFIX FOR PLAYBACK
  a 2-hour video at 6 s segments across 5 renditions
  = ~6,000 segment URLs
  signing each is impractical and pointless: the
  authorisation decision is per-video, not per-segment

LEAK EXPOSURE
  playback prefix leaked  -> that one video, 6 hours
  key leaked              -> everything, until rotation
  -> so key handling matters more than any single URL
```

| Metric | Value | Note |
|---|---|---|
| Thumbnails | 5 min | transient page |
| Playback | 6 h prefix | session-length |
| Download | 24 h, one file | email lifetime |
| Revocation | = expiry window | **no unsigning** |

> **The expiry is a revocation SLA, not a convenience setting**  
> Because a signed URL cannot be withdrawn, the maximum time between an entitlement change and access actually stopping equals the expiry. If the product requires access to end within minutes of a cancellation, the expiry must be minutes and the client must refresh — choosing a long expiry because refreshing is inconvenient is quietly choosing a long revocation delay.

**When to use it**

- **Private media served through a CDN**, where proxying through your service would defeat the purpose.
- **Time-limited download links**, such as an export or an invoice sent by email.
- **Direct-to-storage uploads**, where a presigned URL grants write access to one specific object.
- **Third-party access** you want to grant without creating an account.
- **Any case where authorisation is expensive but verification must be cheap and distributed.**

**When to avoid it**

- **Do not use long expiries for sensitive content**, since a signed URL cannot be revoked before it lapses.
- **Do not sign without authorising first** — the signature proves a decision was made, not that it was correct.
- **Do not rely on IP binding** for consumer clients, whose addresses change mid-session.
- **Do not log full request paths** where signed URLs appear, which turns logs into live credentials.
- **Do not use signed URLs where per-request identity is genuinely needed**; use a real auth token instead.

**Advantages**

- **The edge enforces access** with no call to your service, preserving CDN latency and cost benefits.
- **Stateless verification** — nothing to look up, nothing to replicate.
- **Fine-grained scoping** by path, method, time and optionally network.
- **Works with any client** that can fetch a URL, including ones you do not control.
- **Automatic expiry** limits the blast radius of a leak without any cleanup process.

**Disadvantages**

- **No revocation before expiry**, short of rotating the key and breaking everything.
- **Bearer semantics** — whoever has the URL has the access.
- **URLs leak easily** through referrers, logs, history and sharing.
- **Key management becomes critical**, since a leaked signing key compromises everything.
- **Clock skew** between signer and verifier constrains very short expiries.
- **Refresh logic** is required for sessions longer than the expiry.

**Trade-offs**

**Expiry and scope trade-offs**

| Choice | Security | Usability | Best for |
|---|---|---|---|
| Very short expiry (minutes) | Leak window tiny | Needs refresh logic | Page-embedded assets |
| Session expiry (hours) | Bounded by session | Works without refresh | Video playback |
| Long expiry (days) | Effectively a public link | Simple | Rarely appropriate |
| Single-path signature | Leak exposes one file | Many signatures needed | Download links |
| Prefix signature | Leak exposes the asset | One signature per session | Segmented media |
| Signed cookie | Not shareable by link | Domain alignment required | Browser streaming |

The prefix-versus-path choice is decided by the granularity of the authorisation decision. Access to a video is a decision about the video, not about each of its six thousand segments, so signing the prefix matches the semantics — whereas a download link genuinely is about one file.

**How it fails**

**Signed access failures**

| Failure | Cause | Fix |
|---|---|---|
| Access continues after cancellation | Long expiry; no revocation mechanism | Short expiry with client refresh |
| Content leaked via shared link | Bearer URL forwarded | Short expiry; signed cookies for browser sessions |
| Signed URLs found in log aggregation | Full request paths logged | Redact query strings; treat logs as credential stores |
| Playback fails mid-session | Expiry shorter than the session, no refresh | Refresh before expiry; expiry longer than typical sessions |
| Mobile playback breaks on network change | IP-bound signature | Do not bind IP for consumer clients |
| All access breaks during key rotation | No key identifier; single-key verification | Include a key id; verify against current and previous keys |
| Intermittent rejection at short expiries | Clock skew between signer and edge | Add a skew allowance; avoid very short lifetimes |

**Limits**

> **Typical lifetimes**
>
> - **Page-embedded assets**: minutes, refreshed on page load.
> - **Download links**: hours to a day, scoped to a single path.
> - **Playback sessions**: a few hours over a path prefix, with client refresh.
> - **Revocation delay** equals the expiry — there is no faster mechanism short of key rotation.
> - **Clock skew** makes lifetimes under roughly a minute fragile without an explicit allowance.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Signed URLs | Any client; direct CDN access | Bearer; no revocation |
| Signed cookies | Browser streaming sessions | Domain alignment; awkward for native |
| Token auth at edge compute | Real identity checks near the user | Edge runtime complexity |
| Proxy through your service | Full control and per-request auth | Loses CDN benefits entirely |
| Public unguessable URLs | Low-sensitivity content | Permanent access if leaked |
| Per-user encryption (DRM) | High-value content | Substantial complexity and licensing |

Edge compute changes this picture somewhat: running a small authorisation check at the point of presence allows real token validation close to the user without a round trip to the origin, which recovers revocation and identity at the cost of running code at the edge.

**In real systems**

- **Presigned object-storage URLs** are the same mechanism applied to uploads, granting write access to one object for a limited time.
- **CDN signed URLs and signed cookies** are offered by every major provider, with cookies preferred for browser streaming because links do not carry the access.
- **Video platforms** typically sign a path prefix per playback session, since per-segment signing is impractical for thousands of segments.
- **Key rotation with key identifiers** is standard practice, because verifying against only one key makes rotation an outage.
- **DRM systems** sit above this for high-value content, adding per-device key exchange that signed URLs alone cannot provide.

**Common mistakes**

- **Signing before authorising**, issuing access to whoever asks.
- **Expiries measured in days**, making signed URLs effectively public links.
- **Logging full URLs**, storing live credentials in log systems.
- **No key identifier**, so rotation breaks all outstanding access.
- **IP binding on consumer clients**, breaking mobile sessions.
- **No refresh path**, so playback fails mid-session at expiry.
- **Assuming a signed URL can be revoked** — it cannot, only expired.

**The staff-level view**

Signed access is usually implemented correctly and configured carelessly — the mechanism is simple, the lifetimes are where the risk lives.

- **Derive expiry from the revocation requirement.** The time between an entitlement change and access stopping equals the expiry, so a long lifetime chosen for convenience is a long revocation delay chosen silently.
- **Treat signed URLs as credentials in logging and analytics.** Full-path logging turns a log store into a collection of live access grants, and this is discovered far too often during audits.
- **Require a key identifier and multi-key verification.** Without it, rotating a compromised key is an outage, which means it will not be done promptly when it matters most.
- **Prefer signed cookies for browser streaming.** They give the same edge enforcement without the property that forwarding a link forwards the access.
- **Do not bind to client IP for consumer traffic.** It breaks on mobile handoff and VPNs, and the support cost consistently exceeds the security benefit.

**Go deeper**

Signed access lets a CDN enforce authorisation without knowing anything about your users. Your service performs the real check and encodes the result as a signature over the path and an expiry; the edge recomputes the signature, checks the time, and serves. That preserves CDN latency and cost benefits, which proxying every media request through your service would destroy, while avoiding the permanent grant that an unguessable public URL represents.

The central limitation is that a signed URL cannot be revoked — verification is stateless, so there is nothing to remove it from. Access therefore continues until expiry, which makes expiry a revocation SLA rather than a convenience setting: if entitlement changes must take effect within minutes, the lifetime is minutes and clients must refresh. Rotating the signing key is the only faster remedy and it invalidates everything at once.

Practical design centres on scope and lifetime. Segmented media is signed by path prefix, since the authorisation decision concerns the asset rather than its thousands of segments; download links are signed per path. Signed cookies are preferable for browser sessions because forwarding a link does not forward access. Key identifiers make rotation routine rather than an outage, and query strings must be redacted in logs — otherwise ordinary logging accumulates live access grants.

Signed media access resolves a structural tension: private content needs CDN delivery, but the CDN has no knowledge of your users, sessions or permissions.

**Separating the decision from the enforcement.** Your service answers the authorisation question once and expresses the answer as a signature over the path, an expiry and any additional constraints, computed with a key the CDN also holds. The edge recomputes and compares, checks the expiry against its clock, and serves or refuses. No user lookup, no state, no round trip to the origin — which is exactly what preserves the latency and cost benefits that proxying media through your own service would eliminate.

**Bearer semantics are the defining property.** The signature attests that a decision was made, not who made it or for whom. Anyone holding the URL has the access, and URLs escape routinely through referrer headers, browser history, forwarded messages, screenshots and logs. Nothing in the mechanism distinguishes the intended recipient from a stranger, which means the security of the scheme rests almost entirely on how tightly scoped and how short-lived the capability is.

**Expiry is a revocation SLA.** Because verification is stateless there is no list from which to remove an issued URL, so access persists until it lapses. The maximum delay between an entitlement change and access actually ending is therefore the expiry, and choosing a long lifetime because refresh logic is inconvenient is choosing that delay silently. Key rotation invalidates everything outstanding at once, which is why it is an emergency tool rather than a revocation mechanism, and why an edge denylist — while possible — defeats the statelessness that made the design attractive.

**Scope should match the granularity of the decision.** A video's entitlement question is about the video, not about each of its thousands of segments across several renditions, so signing a path prefix per playback session is both practical and semantically correct. A download link genuinely concerns one file and should be signed for that path, so a leak exposes only it. Constraints like client-IP binding sound attractive but break consumer traffic on mobile handoff and VPNs, and the support burden reliably exceeds the security gain.

**Signed cookies are usually better for browsers.** The cookie is scoped to the browser rather than travelling with the link, so sharing a URL does not share access — which removes the most common leak path for streaming sessions with thousands of segment requests. The costs are domain alignment between the site and the media host, and awkwardness for native clients, which is why most systems support both forms.

**The operational failures are mundane and recurrent.** Signed URLs logged in full turn observability infrastructure into a store of live credentials, a finding that appears in audits far more often than any cryptographic weakness. Missing key identifiers make rotating a compromised key an outage, which means it will not happen promptly at the moment it matters most. Absent refresh logic makes long sessions fail exactly at expiry, the worst possible time. And clock skew between the signing service and the edge makes very short lifetimes intermittently unreliable without an explicit allowance. None of these are flaws in the mechanism; all of them are consequences of treating a time-bounded bearer capability as though it were an ordinary permission check.

**Prove it — interview questions**

1. **[Basic] What is a signed URL?**

   <details><summary>Model answer</summary>

   A URL carrying a cryptographic signature over the path and an expiry time, produced with a key shared between your service and the CDN. Your service performs the real authorisation check and then encodes the result as this signature; the edge recomputes it, checks the expiry, and serves or rejects — with no call back to your service and no knowledge of your users. It is a bearer capability: whoever holds the URL has the access it grants until it expires.

   </details>

2. **[Basic] Why not just use unguessable URLs?**

   <details><summary>Model answer</summary>

   Because an unguessable URL is a permanent grant, and URLs leak constantly — through referrer headers, browser history, forwarded messages, server logs, screenshots and support tickets. Once it escapes, there is nothing to expire and nothing to revoke. A signed URL has the same convenience but a bounded lifetime, so a leak is a time-limited exposure rather than a permanent one, and that bound is the entire point.

   </details>

3. **[Senior] How do you revoke a signed URL?**

   <details><summary>Model answer</summary>

   You largely cannot, and that is the mechanism's main limitation. Once issued, it is valid until it expires; there is no list to remove it from because verification is stateless. The practical answer is to choose the expiry as the revocation window — if access must stop within five minutes of an entitlement change, the expiry is five minutes and the client refreshes. The blunt alternative is rotating the signing key, which invalidates every outstanding URL simultaneously and breaks every active session, so it is an emergency tool. An edge denylist is possible but reintroduces exactly the state and lookups that signing was designed to avoid.

   </details>

4. **[Senior] When would you sign a prefix rather than a single path?**

   <details><summary>Model answer</summary>

   When the authorisation decision is about an asset rather than a file. A two-hour video at six-second segments across five renditions is roughly six thousand URLs, and the entitlement question is whether this user may watch this video — not whether they may fetch segment 412 of the 720p rendition. Signing the prefix once per playback session matches that semantics and avoids signing thousands of URLs. The cost is blast radius: a leaked prefix signature exposes the whole asset for its lifetime, whereas a download link, which genuinely concerns one file, should be signed for that path alone.

   </details>

5. **[Staff] Design signed access for a subscription video service that must stop playback within minutes of a cancellation.**

   <details><summary>Model answer</summary>

   The requirement fixes the expiry, because revocation delay equals expiry. So signatures live for a few minutes over the segment prefix, and the player refreshes its signature well before expiry — which means building refresh into the client from the start rather than treating it as an enhancement, since a session that fails mid-playback at expiry is the worst possible failure. Each refresh is an opportunity to re-check entitlement, so cancellation takes effect at the next refresh. I would use signed cookies for browser playback so that forwarding a link does not forward access, and signed URLs for native clients where cookies are awkward. Signatures carry a key identifier and the edge verifies against current and previous keys, so rotation is routine rather than an outage. And I would ensure query strings are redacted in logging and analytics, because otherwise the log pipeline accumulates live access grants — which is the finding that turns up in audits far more often than any cryptographic weakness.

   </details>

6. **[Principal] What are the systemic risks of signed access, as opposed to implementation bugs?**

   <details><summary>Model answer</summary>

   The mechanism is sound and simple; the risk is that its properties are easy to configure away. The first systemic risk is that expiry is chosen for convenience rather than as a revocation SLA, so an organisation believes it can cut off access and discovers it cannot — this is a product and compliance issue that surfaces as a security one. The second is that bearer semantics are invisible in the design: nothing in the architecture diagram shows that forwarding a link forwards the access, so it is not discussed until content appears somewhere unexpected. The third is key handling, because a leaked signing key compromises everything at once and rotation is an outage unless key identifiers were designed in from the beginning — which means the capability to respond quickly must exist before the incident, not after. And the fourth is the quiet one: signed URLs accumulating in logs, analytics, error reports and referrers, turning ordinary observability infrastructure into a store of live credentials. None of these are cryptographic failures; they are all consequences of treating a bearer capability with a fixed lifetime as if it were an ordinary access check.

   </details>

---

### Image transformation services

*Derive resized and reformatted images on demand from a single master, cached at the edge, with a strict allowlist of permitted transformations.*

**Flow:** `Master image` → `Transform request` → `Derivative cache` → `Edge delivery` → `Origin protection`

> **The 30-second version**  
> Keep one master, derive sizes and formats on demand with the spec in the URL, cache the derivatives at the edge, and allowlist the specs so the work stays bounded.

**The problem**

The same photograph is needed as a thumbnail in a list, a medium image in a card, a large hero on desktop, at one and two times pixel density, in modern and legacy formats. Pre-generating every combination for every upload multiplies storage and compute, and a new layout next quarter needs sizes nobody produced.

The alternative — serving the full-resolution master everywhere and resizing in the browser — wastes enormous bandwidth on mobile, where a four-megabyte photograph is downloaded to be displayed at three hundred pixels wide.

> **Derive on demand, cache aggressively, allowlist strictly**  
> Keep one master, generate derivatives when first requested, and cache them at the edge so the second request costs nothing. This gives arbitrary sizes without pre-generation and near-zero marginal cost after the first hit. The essential constraint is an allowlist of permitted transformations, because a URL-driven transform service without one is an invitation to generate unbounded expensive work.

**Mental model**

The transformation is expressed in the URL, which makes it a cache key. The first request for a given transformation computes it; every subsequent request is a cache hit. The master is never served directly.

1. **Master** — The original upload, stored once and never delivered to end users.
2. **Transform spec** — Width, height, fit, format and quality, encoded in the URL so it identifies the derivative.
3. **Cache** — Edge and derivative storage. After the first request, transformation cost is zero.
4. **Allowlist** — The bounded set of permitted specs. Without it, the URL space is infinite and so is the work.
5. **Format negotiation** — Serving modern formats to clients that accept them, which is usually the largest single bandwidth saving.

> **An unbounded transform URL space is a resource-exhaustion vulnerability**  
> If any width from one to ten thousand is permitted, an attacker requests ten thousand distinct widths and each one is a cache miss triggering a real image decode and encode. The cache provides no protection because every request is unique by construction. Signed transform URLs or a fixed allowlist of permitted sizes is not a refinement — it is the difference between a service and an easily exhausted one.

**How it works**

**The request path and where cost lands**

```text
GET /img/photo123?w=400&fmt=auto&q=80
          |
    edge cache?  --- HIT (99% of requests) ---> served
          |                                     ~15 ms
         MISS
          |
    derivative store?  --- HIT ---> served, cached at edge
          |
         MISS
          |
    fetch master, decode, resize, encode, store
    ~200-500 ms, real CPU and memory
          |
    serve + populate both caches

COST DISTRIBUTION
  first request for a spec:  expensive
  all subsequent requests:   free
  -> so the number of DISTINCT specs is what costs money,
     not the number of requests

WHICH IS WHY THE ALLOWLIST MATTERS
  6 widths x 2 densities x 3 formats = 36 derivatives
  per master, bounded and predictable
  vs. arbitrary w= values: unbounded distinct specs,
  every one a miss, every miss a decode
```

1. **Encode the transform in the URL** — It becomes the cache key, so identical requests collapse and the whole thing is CDN-cacheable with no server state.
2. **Allowlist the permitted specs** — A bounded set of sizes and formats keeps derivative count and attack surface finite; sign the URL if arbitrary specs are genuinely needed.
3. **Negotiate format from the client** — Serving a modern format to clients that accept it typically saves a large fraction of bytes for no visual cost.
4. **Never serve the master** — Masters are large and sometimes contain metadata you do not want distributed; all delivery goes through derivatives.
5. **Strip metadata during transformation** — Location data and camera details in uploaded photographs are a privacy exposure that transformation is the natural place to remove.
6. **Cap dimensions and validate the master first** — Decoding an image can consume memory proportional to its declared dimensions, so a malicious file must be rejected before decoding, not during.

**Where the bytes actually go**

```text
SAME PHOTO, DELIVERED TO A PHONE AT 400px WIDE

  master, full resolution, JPEG      4,000 KB
  resized to 400px, JPEG q80            45 KB
  resized to 400px, modern format q80   28 KB

-> resizing saves ~99% of the bytes
-> format saves a further ~40% on top

DERIVATIVE COUNT PER MASTER (allowlisted)
  widths   200, 400, 800, 1200, 1600, 2400
  density  1x, 2x  (folded into the widths above)
  formats  modern, legacy
  = ~12 derivatives, not 36 - collapse where you can

STORAGE
  12 derivatives averaging 60 KB = ~720 KB per master
  master itself 4 MB
  -> derivatives are cheap relative to masters
  -> and can be evicted and regenerated on demand,
     which masters cannot

QUALITY SETTING
  q95 -> q80 typically halves the bytes with no
  perceptible difference at display size
  -> quality is usually the cheapest single win and
     the one most often left at a default
```

> **Image decoding is an untrusted-input parser**  
> An uploaded image is attacker-controlled data fed to a complex native decoder. Historic vulnerabilities in image libraries are numerous, and a file declaring enormous dimensions can exhaust memory before any resize logic runs. Validating declared dimensions before decoding, imposing memory and time limits on the transform process, and isolating it from anything sensitive are baseline requirements rather than hardening.

**Worked example**

A product catalogue's image delivery, with the numbers that justify each choice.

**Configuration and outcomes**

```text
UPLOAD
  validate declared dimensions BEFORE decode
  strip EXIF, including location
  store master once
  -> master is never served to users

DELIVERY  /img/{id}/{w}/{fmt}
  allowlist: w in {200,400,800,1200,1600}
             fmt in {modern, legacy}
  anything else -> 400, not a transform
  -> derivative space bounded at 10 per master

RESPONSIVE MARKUP
  srcset offers the allowlisted widths
  the browser picks by viewport and density
  format chosen by Accept header

CACHING
  Cache-Control: public, max-age=31536000, immutable
  -> derivative URLs are content-addressed by
     (id, w, fmt), so they never change meaning
  -> a new master gets a new id

MEASURED RESULT
  cache hit ratio          ~99%
  transform cost           first request only
  mobile page image bytes  3.2 MB -> 180 KB
  p50 image latency        ~15 ms from edge

WHAT WOULD BREAK IT
  arbitrary w= -> unbounded misses, decode per request
  serving masters -> 4 MB per image on mobile
  no EXIF stripping -> user location published
```

| Metric | Value | Note |
|---|---|---|
| Page bytes | 3.2 MB → 180 KB | resize + format |
| Hit ratio | ~99% | bounded specs |
| Derivatives | 10 per master | **allowlisted** |
| Transform | first request only | then free |

> **Bounded specs are what make the cache work**  
> A cache only helps when requests repeat. Allowlisting a small set of widths and formats guarantees repetition, which is why the hit ratio approaches ninety-nine per cent. Permitting arbitrary dimensions guarantees the opposite: every request is unique, every one misses, and the expensive path becomes the normal path. The allowlist is a caching decision as much as a security one.

**When to use it**

- **User-uploaded images** displayed at many sizes across devices.
- **Responsive image delivery**, where each viewport and density needs a different variant.
- **Product catalogues and media libraries** with predictable display contexts.
- **When layouts change**, since new sizes need no re-processing of existing content.
- **Format modernisation**, serving newer formats to capable clients without touching the masters.

**When to avoid it**

- **Do not permit arbitrary transform parameters** without signing — it is an unbounded work generator.
- **Do not serve masters to end users**, wasting bandwidth and distributing metadata.
- **Do not transform without limits** on memory, time and declared dimensions.
- **Do not use on-demand transformation for a tiny fixed set**, where pre-generating is simpler.
- **Do not skip metadata stripping**, which silently publishes location data from user photographs.

**Advantages**

- **Any size on demand** without pre-generation or re-processing.
- **Marginal cost near zero** after the first request for each spec.
- **Large bandwidth savings** from correct sizing and modern formats.
- **Layout changes are free**, since new sizes are generated when first requested.
- **Derivatives are disposable** and can be evicted and regenerated, unlike masters.
- **Natural place to strip metadata** and normalise formats.

**Disadvantages**

- **First request is slow**, which matters for above-the-fold images.
- **Unbounded specs are a denial-of-service vector** unless allowlisted or signed.
- **Image decoding is risky**, operating on untrusted input with complex native libraries.
- **Derivative storage grows** with the product of masters and specs.
- **Quality settings are subjective**, and regressions are hard to detect automatically.

**Trade-offs**

**Generation strategy trade-offs**

| Strategy | First-request latency | Cost | Best for |
|---|---|---|---|
| On demand, cached | Slow once per spec | Low; bounded by specs | Most applications |
| Pre-generate all specs | None | Storage and compute for unused variants | Small fixed spec sets |
| Pre-generate common, derive rest | None for common | Balanced | Known hot paths |
| Serve master, resize in browser | None | Enormous bandwidth waste | Almost never |
| Arbitrary specs, signed | Slow per unique spec | Bounded by who can sign | Internal or partner tools |

Pre-generating the handful of sizes that appear above the fold, while deriving everything else on demand, removes the one genuinely user-visible cost of the on-demand model without giving up its flexibility.

**How it fails**

**Transformation service failures**

| Failure | Cause | Fix |
|---|---|---|
| Service exhausted by unique requests | Arbitrary transform parameters | Allowlist specs or sign transform URLs |
| Out-of-memory during transform | Image with enormous declared dimensions | Validate dimensions before decoding; cap memory |
| User location data published | EXIF not stripped | Strip metadata during transformation and at upload |
| Slow first paint for hero images | Cold cache on the critical image | Pre-generate above-the-fold specs |
| Huge mobile page weight | Master served instead of a derivative | Never expose master URLs; enforce via routing |
| Derivative storage growing without bound | No eviction; unbounded spec set | Bounded specs; evict and regenerate on demand |
| Security incident in the decoder | Untrusted input in a native library | Isolate the transform process; limit its privileges |

**Limits**

> **Reference figures**
>
> - **Transform cost**: a few hundred milliseconds per derivative on first request, then effectively zero.
> - **Byte savings**: correct sizing typically removes over 90% of a master's bytes; modern formats a further 25–40%.
> - **Quality**: dropping from a near-lossless setting to around 80 usually halves the bytes with no perceptible difference.
> - **Derivative count**: keep it around 10 per master; the product of widths, densities and formats grows quickly.
> - **Hit ratio**: approaches 99% with bounded specs and collapses without them.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| On-demand transform service | Varied sizes; changing layouts | First-request latency; must be bounded |
| Pre-generated fixed set | Stable, small spec sets | Re-processing when layouts change |
| CDN-integrated transformation | Simplicity; no service to run | Vendor coupling; per-transform cost |
| Client-side resizing | Trivial cases | Wastes bandwidth; poor on mobile |
| Build-time processing | Static sites with known images | Not applicable to user uploads |
| Dedicated image CDN | Outsourcing the whole concern | Cost; less control over quality |

CDN-integrated transformation is worth considering seriously before building a service: it removes the operational burden and the decoder-security concern entirely, at the cost of vendor coupling and per-transform pricing that becomes significant at high derivative counts.

**In real systems**

- **Image CDNs** offer URL-driven transformation as a product precisely because the pattern generalises across nearly every application with user uploads.
- **Responsive markup with srcset** pairs with a bounded width allowlist so browsers select from exactly the derivatives that exist.
- **Modern image formats** are served conditionally on the client's Accept header, which is usually the single largest bandwidth win available.
- **Signed transform URLs** are the standard defence where arbitrary specs are genuinely required, bounding who can generate work rather than what work exists.
- **Metadata stripping at upload and transform** is routine in consumer platforms because photograph EXIF commonly contains precise location.

**Common mistakes**

- **Permitting arbitrary dimensions**, creating an unbounded work generator with a useless cache.
- **Serving masters to browsers**, wasting megabytes per image on mobile.
- **Decoding before validating declared dimensions**, inviting memory exhaustion.
- **Leaving EXIF intact**, publishing location data from user photographs.
- **Leaving quality at a near-lossless default**, doubling bytes for no visible benefit.
- **No pre-generation for hero images**, so the most visible image is the slowest.
- **Unbounded derivative storage** with no eviction policy.

**The staff-level view**

Image delivery is usually treated as a detail and is frequently the largest single contributor to mobile page weight.

- **Bound the transform space explicitly.** An allowlist is simultaneously the security control against resource exhaustion and the reason the cache hit ratio is high — those are the same decision viewed twice.
- **Measure image bytes per page on mobile.** Serving masters or oversized derivatives is extremely common and is often a larger performance win than anything in the application code.
- **Treat the decoder as an untrusted-input parser.** Validate declared dimensions before decoding, cap memory and time, and isolate the process; image libraries have a long history of exploitable bugs.
- **Strip metadata as policy.** Publishing users' photograph location data is a privacy incident that costs one line of configuration to prevent.
- **Pre-generate only the above-the-fold specs.** It removes the on-demand model's one user-visible cost while keeping its flexibility.

**Go deeper**

An image transformation service stores one master and generates resized, reformatted derivatives when they are first requested. The transform specification lives in the URL, which makes it a cache key: the first request pays a few hundred milliseconds of decode and encode, and every subsequent one is served from cache. New layout sizes require no re-processing, because derivatives appear as they are asked for.

The permitted specifications must be bounded. An allowlist of a handful of widths and formats keeps derivative count predictable and — more importantly — guarantees that requests repeat, which is what drives the cache hit ratio toward ninety-nine per cent. Permitting arbitrary dimensions does the opposite: every request is unique, every one misses, and each miss triggers real image processing, making the service trivially exhaustible.

The wins are large and the risks are mundane. Correct sizing removes most of a master's bytes and modern formats remove a further quarter to forty per cent, which on mobile is frequently the biggest available performance improvement in a product. Against that, the decoder is a complex native parser handling attacker-controlled input, memory use scales with declared dimensions rather than file size, and uploaded photographs carry location metadata that is published unless stripped.

Image transformation services solve a combinatorial problem: the same source image is needed at many sizes, densities and formats, and the required set changes with every layout revision.

**Deriving on demand with the spec in the URL.** Encoding width, format and quality into the path makes the request self-describing and therefore cacheable by ordinary CDN machinery. The first request for a given specification performs a real decode, resize and encode; every subsequent one is a cache hit costing nothing. The consequence is that cost scales with the number of distinct specifications rather than the number of requests, which reframes the whole design around keeping that number small.

**The allowlist is two decisions in one.** Bounding permitted widths and formats guarantees that requests repeat, which is the precondition for a cache being useful at all — hit ratios near ninety-nine per cent follow directly. The same bound is the security control: with arbitrary parameters permitted, an adversary generates unlimited unique requests, each a guaranteed miss triggering real image processing, and the cache provides no protection because nothing ever repeats. Where arbitrary specifications are genuinely required, signing the transform URL bounds who may generate work instead of what work is possible.

**The bandwidth win is usually the largest available.** A full-resolution photograph delivered to a phone that displays it at a few hundred pixels wastes well over ninety per cent of its bytes, and serving a modern format to clients that accept it removes a further quarter to forty per cent. Quality settings compound this: moving from a near-lossless default to around eighty typically halves the bytes with no perceptible difference at display size. For image-heavy products these three adjustments frequently outweigh every optimisation available in the application code.

**The decoder is an untrusted-input parser.** Uploaded images are attacker-controlled data processed by complex native libraries with a long history of exploitable defects, so the transform process deserves isolation and minimal privilege. Memory consumption during decode scales with declared dimensions rather than file size, which means a small file claiming enormous dimensions exhausts memory before any resize logic runs — validation of declared dimensions must therefore happen before decoding, not as part of it.

**Metadata is a privacy surface.** Photographs from consumer devices routinely embed precise capture location alongside camera details, and serving the file unmodified publishes that to everyone who downloads it. Transformation is the natural interception point, and stripping metadata there — as well as at upload, so the master itself does not retain it — converts a recurring incident class into a configuration line.

**Derivatives are disposable in a way masters are not.** Because any derivative can be regenerated from the master, derivative storage can be evicted under pressure without data loss, which makes its growth manageable in a way that master storage is not. The one genuinely user-visible cost of the on-demand model is first-request latency, and it is fully addressed by pre-generating only the specifications used above the fold — preserving the flexibility of lazy derivation everywhere else while ensuring the most visible image on the page is never the one paying for a cold cache.

**Prove it — interview questions**

1. **[Basic] Why transform on demand rather than pre-generating?**

   <details><summary>Model answer</summary>

   Because the set of sizes you need is not known in advance and changes with every layout revision. Pre-generating means deciding upfront, re-processing the entire library whenever a new size is needed, and paying storage for variants nobody requests. On-demand generation with aggressive caching gives any size when it is first asked for and costs nothing thereafter, so a new breakpoint next quarter requires no migration — the derivatives simply appear as they are requested.

   </details>

2. **[Basic] Why does the transform go in the URL?**

   <details><summary>Model answer</summary>

   So it becomes the cache key. Every request for the same image at the same width and format is byte-identical, which means ordinary CDN caching collapses them into one stored object and one generation. It also makes the service stateless: there is no session, no lookup, and nothing to coordinate — the URL fully describes the desired output, which is what allows the whole path to be static file serving after the first request.

   </details>

3. **[Senior] Why must transform parameters be bounded?**

   <details><summary>Model answer</summary>

   For two reasons that turn out to be the same reason. Security: if any width is permitted, an attacker issues requests with ten thousand distinct widths, every one is a cache miss by construction, and every miss triggers a real decode, resize and encode — the cache offers no protection because no request ever repeats. Performance: a cache only helps when requests repeat, so bounding the spec set is precisely what produces a hit ratio near ninety-nine per cent. The allowlist is a caching decision and a denial-of-service control simultaneously. Where arbitrary specs are genuinely needed, signing the transform URL bounds who can generate work instead.

   </details>

4. **[Senior] What are the security concerns in image transformation?**

   <details><summary>Model answer</summary>

   The decoder is a complex native parser operating on entirely attacker-controlled input, and image libraries have a long history of exploitable vulnerabilities — so the transform process should be isolated and minimally privileged. Separately, memory consumption during decode is proportional to declared dimensions rather than file size, so a small file claiming enormous dimensions exhausts memory before any resize logic executes; dimensions must be validated before decoding. And there is a privacy dimension: uploaded photographs routinely carry EXIF metadata including precise location, which is published to everyone who downloads the image unless it is stripped.

   </details>

5. **[Staff] Design image delivery for a product catalogue with heavy mobile traffic.**

   <details><summary>Model answer</summary>

   One master per image, never served to users, with metadata stripped and declared dimensions validated at upload before anything decodes it. Delivery through URLs that encode image id, width and format, with a strict allowlist of perhaps five widths and two formats — that bounds derivatives to around ten per master, makes the transform space finite, and is what gets the cache hit ratio to ninety-nine per cent. Responsive markup offers exactly those widths so browsers select from derivatives that exist, and format is chosen from the Accept header, which alone typically removes a quarter to forty per cent of bytes. Derivative URLs are immutable, so they cache for a year and a new master gets a new id rather than an invalidation. I would pre-generate the above-the-fold specs so the most visible image is never the one paying first-request latency, and derive everything else lazily. The measurable target is image bytes per mobile page view — for a catalogue this is usually the dominant component of page weight, and getting it from megabytes to low hundreds of kilobytes is normally a larger win than anything available in the application code.

   </details>

6. **[Principal] Why is image delivery so often a major performance problem despite being conceptually simple?**

   <details><summary>Model answer</summary>

   Because it sits in nobody's area of ownership and its costs are invisible on the machines where it is developed. The application team treats it as a detail, the infrastructure team sees only bandwidth, and the failure mode — a four-megabyte photograph rendered at three hundred pixels — looks completely correct on a desktop connection. Meanwhile it is frequently the largest single component of mobile page weight, so the highest-leverage performance work in a product is often not in the code at all. The compounding factor is that the easy implementations are also the dangerous ones: permitting arbitrary dimensions is the most natural API and is simultaneously a resource-exhaustion vector and a guarantee of cache misses, while skipping metadata stripping is the default and is a privacy incident waiting to be discovered. So the discipline I would push for is to measure image bytes per page as a first-class metric, to make the transform space bounded by construction, and to treat the decoder as the untrusted-input parser it is — three decisions that are cheap at the start and expensive to retrofit.

   </details>

---

### Live streaming ingest

*Accept a continuous broadcaster feed, transcode it in real time, and segment it for delivery — with no ability to slow down and no second chance.*

**Flow:** `Broadcaster` → `Ingest endpoint` → `Real-time transcode` → `Segmenter` → `Edge delivery`

> **The 30-second version**  
> Accept a continuous feed and transcode, segment and publish it at real time forever — with headroom, because falling behind never recovers, and with CDN delivery, because that is what makes the audience free.

**The problem**

A broadcaster streams continuously to an endpoint. Unlike an uploaded file, the media arrives at exactly the rate it was captured and cannot be paused, retried or re-read. Every stage — ingest, transcode, segment, publish — must keep pace with real time, permanently, because falling behind is unrecoverable.

The consequences of that constraint are unusual. There is no backlog to work through, because a backlog means growing latency that never shrinks. There is no retry, because the moment has passed. And there is no way to provision for the average, because a live event's audience arrives all at once.

> **Live ingest is a real-time pipeline with no back pressure valve**  
> Ordinary pipelines absorb load spikes by queueing and catching up later. A live pipeline cannot: if transcoding takes longer than real time, latency grows without bound and the only remedies are to shed quality, shed renditions, or drop the stream. That single property — that falling behind is permanent — shapes every design decision in the system.

**Mental model**

A continuous connection carries media to an ingest endpoint, which transcodes it into a ladder in real time, cuts each rendition into segments, and publishes them for CDN delivery. Every stage runs at the speed of the source, forever.

1. **Ingest** — A long-lived connection from the broadcaster, authenticated at connect and sustained for the whole event.
2. **Real-time transcode** — The ladder produced faster than real time per rendition, or the pipeline falls behind permanently.
3. **Segment** — Cut into short aligned chunks and written to storage as they complete.
4. **Publish** — Manifest updated as segments appear, so players discover new content.
5. **Deliver** — Ordinary CDN distribution of immutable segments, which is what makes audience scale free.

> **Falling behind real time is unrecoverable**  
> If a stage processes at 0.98 times real time, latency grows by roughly a minute every fifty minutes and never recovers, because there is no idle period to catch up in — the source keeps producing. Live transcoding must therefore be provisioned with genuine headroom and must degrade by dropping renditions rather than by slowing down. A pipeline that is merely fast enough on average is a pipeline that will drift.

**How it works**

**The latency budget, end to end**

```text
CAPTURE -----> VIEWER

  encoder buffer at the broadcaster    1-2 s
  network to ingest                      0.5 s
  ingest buffer                          0.5 s
  transcode                              1-2 s
  segmentation (must complete a segment) = segment length
  upload to storage + CDN propagation    0.5-1 s
  player buffer (a few segments)         2-3 segments

TOTAL with 6 s segments:  ~30-45 s
TOTAL with 2 s segments:  ~10-15 s
TOTAL with chunked delivery: 3-6 s

WHERE THE TIME ACTUALLY GOES
  segment length appears TWICE: once because a segment
  must be complete before it can be published, and again
  because the player buffers several of them
  -> segment length is the dominant lever, and it is
     multiplied, not added

REDUCING IT
  shorter segments      -> less compression efficiency,
                           more requests, less player
                           buffer, MORE STALLS
  chunked delivery      -> publish partial segments so
                           latency stops depending on
                           full segment duration
  smaller player buffer -> lower latency, less tolerance
                           for network variance

EVERY LEVER TRADES LATENCY FOR STALL RESISTANCE.
There is no configuration that improves both.
```

1. **Authenticate at connect, not per frame** — The connection is long-lived, so authorisation happens once — which means a revoked broadcaster keeps streaming unless connections are actively terminated.
2. **Provision transcoding with real headroom** — Anything slower than real time drifts permanently. Headroom is not waste; it is the only defence against unrecoverable lag.
3. **Degrade by dropping renditions, never by slowing** — Under pressure, produce fewer qualities at real time rather than the full ladder behind real time.
4. **Handle broadcaster reconnection explicitly** — Consumer uplinks drop. Deciding whether a reconnect continues the same stream or starts a new one determines whether viewers see a glitch or an ending.
5. **Publish segments and manifest in the right order** — The manifest must never reference a segment that has not been written, and players poll it continuously.
6. **Plan the audience spike, not the average** — A live event's viewers arrive within a minute of the start; the CDN absorbs this, the origin cannot.

**Failure handling in a system that cannot pause**

```text
BROADCASTER CONNECTION DROPS
  brief (< a few seconds)
    -> hold the stream open, wait for reconnect
    -> viewers see a short stall, then continuity
  extended
    -> end the stream, publish what exists as a recording
    -> a stream held open indefinitely for a broadcaster
       who has gone strands viewers on a frozen player

TRANSCODER FAILS MID-EVENT
  -> the ladder is producing nothing; every viewer stalls
  -> restart quickly and accept a gap, OR run redundant
     transcoders on the same input
  -> redundancy doubles cost and is justified only for
     events where a gap is unacceptable

INGEST NODE FAILS
  -> the broadcaster must reconnect, which they will
     only do if the client is built for it
  -> anycast or DNS failover moves the reconnect to a
     healthy node

STORAGE OR PUBLISH SLOW
  -> segments arrive late; players run out of buffer
  -> this looks like a network problem to viewers and
     like nothing at all in origin metrics unless
     segment publish latency is measured directly

THE RULE
every failure is visible to viewers within seconds,
because there is no buffer of unprocessed work to
hide behind. Detection must be faster than a
viewer noticing.
```

> **The audience arrives all at once and mostly at the start**  
> A scheduled live event goes from no viewers to peak within a minute or two. Autoscaling cannot react on that timescale, so capacity must be provisioned ahead of the event rather than scaled into. The saving grace is that segments are immutable and identical for all viewers, so the CDN absorbs essentially the entire audience — which is why live streaming scales at all, and why anything that makes segments viewer-specific is catastrophic.

**Worked example**

A live event platform, with the numbers that drive each decision.

**Design and its trade-offs**

```text
INGEST
  long-lived connection, stream key authenticated
    at connect
  regional endpoints so broadcasters connect nearby
  reconnection window: 10 s
    -> shorter would end streams on ordinary mobile
       hiccups; longer strands viewers on a frozen
       player

TRANSCODE  (the constrained stage)
  ladder: 1080p, 720p, 480p, 360p
  hardware acceleration - required, not optional:
    software encoding of 4 renditions in real time
    is marginal on general-purpose CPU
  headroom target: process at 1.5x real time
    -> absorbs a transient slowdown without drifting
  degradation: under pressure, drop 1080p first
    -> fewer qualities at real time beats all
       qualities behind it

SEGMENT AND PUBLISH
  4 s segments, keyframe-aligned across the ladder
  write segment, THEN update manifest
  -> a manifest entry for a segment that does not
     exist is an immediate playback error

DELIVERY
  CDN with a shield tier
  manifest: max-age 2 s (it changes constantly)
  segments: max-age 1 hour (immutable once written)
  -> players poll the manifest; segments cache hard

RESULT
  glass-to-glass latency   ~20 s
  origin load              ~1 request per segment
                           per POP
  audience scale           bounded by CDN, not origin
```

| Metric | Value | Note |
|---|---|---|
| Latency | ~20 s | 4 s segments |
| Headroom | 1.5× real time | **drift defence** |
| Degrade | drop renditions | never slow down |
| Reconnect | 10 s window | mobile reality |

> **Live is on-demand with every safety margin removed**  
> The delivery mechanism is identical to on-demand adaptive streaming — a ladder of keyframe-aligned segments served from a CDN. What differs is that every buffer that makes on-demand robust has been spent on latency: the player holds seconds instead of half a minute, the pipeline cannot queue, and processing cannot fall behind. Understanding live as on-demand minus its margins explains why the same content stalls in live and plays flawlessly as a recording.

**When to use it**

- **Broadcasts, sports and events**, where content is consumed as it happens.
- **Creator streaming platforms**, with many concurrent broadcasters of varying quality.
- **Interactive streams with chat**, where latency of tens of seconds is tolerable but minutes are not.
- **Surveillance and monitoring feeds**, where continuous ingest is the requirement.
- **Any case where the audience must see events close to when they occur.**

**When to avoid it**

- **Do not use segmented live streaming for sub-second interaction**, where WebRTC is the appropriate tool.
- **Do not run live transcoding without headroom**, since falling behind real time never recovers.
- **Do not rely on autoscaling for the audience spike**, which arrives faster than any scaler reacts.
- **Do not make segments viewer-specific**, which destroys the CDN cacheability that makes live scale.
- **Do not stream live when a recording would serve**, since the operational burden is far higher.

**Advantages**

- **Scales to enormous audiences** through ordinary CDN caching of immutable segments.
- **Reuses on-demand delivery infrastructure**, since the player and format are the same.
- **Adaptive quality per viewer**, so varied connections are all served.
- **Produces a recording for free**, since the segments already exist.
- **Broadcaster requirements are modest** — one outbound connection.

**Disadvantages**

- **No recovery from falling behind**, making headroom mandatory rather than prudent.
- **Latency of tens of seconds** without chunked delivery.
- **Every failure is immediately viewer-visible**, with no queue to hide behind.
- **Transcoding cost scales with concurrent broadcasters**, not with viewers.
- **Capacity must be pre-provisioned** for spikes that arrive faster than autoscaling.
- **Reconnection semantics are genuinely ambiguous** and must be decided deliberately.

**Trade-offs**

**Live configuration trade-offs**

| Choice | Latency | Robustness | Best for |
|---|---|---|---|
| 6 s segments, 3-segment buffer | ~40 s | High | Broadcasts where delay is fine |
| 2 s segments, 3-segment buffer | ~15 s | Moderate | Interactive streams with chat |
| Chunked delivery | 3–6 s | Lower | Latency-sensitive events |
| WebRTC | Sub-second | Different infrastructure | Two-way interaction |
| Redundant transcoders | Unchanged | Much higher | Events where a gap is unacceptable |

Every row except the last trades robustness for latency, and the trade is direct: lower latency means a smaller player buffer, which means less tolerance for the network variance that will certainly occur. Anyone requesting lower live latency is requesting a higher stall rate, and it is worth stating that plainly.

**How it fails**

**Live pipeline failures**

| Failure | Cause | Fix |
|---|---|---|
| Latency grows steadily through the event | Transcoding slower than real time | Hardware acceleration; headroom; drop renditions under pressure |
| All viewers stall simultaneously | Transcoder or publish path failed | Fast detection; rapid restart; redundancy for critical events |
| Playback errors on new segments | Manifest updated before the segment was written | Write the segment first, then the manifest |
| Stream ends on a brief mobile hiccup | Reconnection window too short | A window of several seconds; explicit reconnect handling |
| Viewers stranded on a frozen player | Stream held open for an absent broadcaster | Bounded reconnection window, then end the stream |
| Origin overwhelmed at event start | Audience spike outpacing autoscaling | Pre-provision; CDN shield tier |
| Glitches when quality changes | Renditions not keyframe-aligned | Align keyframes across the ladder in the encoder |

**Limits**

> **Latency and capacity figures**
>
> - **Glass-to-glass latency**: 30–45 s with 6 s segments; 10–15 s with 2 s segments; 3–6 s with chunked delivery.
> - **Segment length enters the budget twice** — once for completion, once for player buffering — so it is the dominant lever.
> - **Transcode headroom**: target meaningfully faster than real time; anything at or below 1× drifts permanently.
> - **Audience arrival**: peak within a minute or two of a scheduled start, faster than autoscaling reacts.
> - **Reconnection window**: several seconds — short enough not to strand viewers, long enough to survive a mobile hiccup.

**Alternatives**

| Approach | Latency | Scale | Best for |
|---|---|---|---|
| Segmented live (HLS/DASH) | 10–45 s | Millions via CDN | Broadcast and events |
| Low-latency chunked | 3–6 s | Millions via CDN | Interactive broadcast |
| WebRTC | Sub-second | Harder; needs media servers | Two-way interaction |
| RTMP direct to viewers | ~5 s | Poor; per-viewer connections | Legacy; small audiences |
| Record and publish afterwards | Minutes to hours | Trivial | When live is not actually required |

The last row deserves genuine consideration. Live streaming carries a large operational burden — pre-provisioned capacity, no recovery from lag, every failure instantly visible — and a surprising number of requirements described as live are satisfied by publishing a recording shortly afterwards.

**In real systems**

- **Creator streaming platforms** run thousands of concurrent ingest sessions, where transcoding cost scales with broadcasters rather than viewers.
- **Low-latency HLS and chunked CMAF** publish partial segments so latency is no longer bounded by full segment duration.
- **Hardware-accelerated encoding** is standard for live, because software encoding of a full ladder in real time is marginal on general-purpose CPU.
- **CDN shield tiers** are essential for live, since a scheduled event's audience arrives simultaneously and misses would otherwise concentrate on the origin.
- **Recordings produced from live segments** are effectively free, since the segments and manifest already exist.

**Common mistakes**

- **Running transcoding at barely real time**, guaranteeing permanent drift.
- **Degrading by slowing down** rather than by dropping renditions.
- **Updating the manifest before writing the segment**, causing playback errors.
- **Reconnection windows that are too short or unbounded**, ending streams prematurely or freezing players.
- **Relying on autoscaling** for an audience that arrives within a minute.
- **Making segments viewer-specific**, destroying CDN cacheability.
- **Not measuring segment publish latency**, leaving the commonest silent failure undiagnosed.

**The staff-level view**

Live systems fail differently from everything else because there is no queue to absorb trouble and no idle time to recover in.

- **Insist on transcoding headroom as a hard requirement.** A pipeline that keeps up on average will drift during the event it matters for, and the drift is permanent.
- **Define degradation before the event, not during it.** Dropping the top rendition is a decision that should already be automated; discovering it under load means discovering it too late.
- **Decide reconnection semantics explicitly.** Whether a broadcaster's reconnect continues or ends the stream determines whether viewers see a hiccup or a frozen player, and it is usually left to a default nobody chose.
- **Pre-provision for the spike.** Live audiences arrive faster than any autoscaler reacts, and the CDN is what absorbs them — anything that makes segments viewer-specific removes that protection entirely.
- **Measure segment publish latency directly.** A slow publish path looks like a network problem to viewers and shows nothing in ordinary origin metrics, so it is the failure most likely to go undiagnosed.

**Go deeper**

Live ingest accepts a broadcaster's continuous feed, transcodes it into a rendition ladder in real time, segments it, and publishes segments and manifest for CDN delivery. The delivery path is identical to on-demand adaptive streaming; what differs is that nothing can pause. Media arrives at capture rate, cannot be re-read, and offers no retry, so every stage must sustain real time permanently.

The defining constraint is that falling behind is unrecoverable. There is no idle period in which to work through a backlog, so a stage running below real time accumulates latency monotonically. Transcoding therefore needs genuine headroom, and degradation under pressure must mean producing fewer renditions at full speed rather than the whole ladder more slowly.

Latency is dominated by segment length, which enters the budget twice — once because a segment must complete before publication and again because the player buffers several. Reducing it also reduces the player's tolerance for network variance, so lower live latency directly means more stalling; chunked delivery of partial segments is what breaks that dependency. On the capacity side, a scheduled event's audience arrives within a minute, faster than autoscaling reacts, which is survivable only because immutable segments let the CDN absorb essentially all of it.

Live streaming ingest is an ordinary media pipeline operating under a constraint that changes everything: the input cannot be slowed, buffered indefinitely, or replayed.

**No back pressure, no recovery.** Conventional pipelines tolerate slow stages by queueing and catching up during quieter periods. A live pipeline has no quieter period — the source produces at real time indefinitely — so any stage running below real time accumulates latency that is never repaid. A transcoder at ninety-eight per cent of real time adds roughly a minute of delay every fifty minutes of broadcast. The design consequence is that headroom is mandatory, and that the correct response to load is to shed work, not to slow down: fewer renditions produced at full speed always beats the full ladder produced behind real time.

**The latency budget is dominated by segment length, counted twice.** A segment must be complete before it can be published, and the player buffers several segments before playing, so segment duration contributes at both points. This is why halving it helps more than proportionally, and why chunked delivery — publishing partial segments as they are produced — is the technique that breaks latency's dependence on segment duration altogether. Everything else in the budget, from encoder buffers to CDN propagation, is seconds at most.

**Every latency reduction is a stall-rate increase.** The player's buffer is the only protection against network variance, and in live it is also the latency. On-demand playback holds thirty seconds and absorbs nearly all variation; live holds a few and absorbs almost none, which is exactly why the same content stalls live and plays flawlessly as a recording. When someone asks for lower live latency, they are asking for a higher stall rate, and presenting that trade explicitly is more useful than quietly tuning toward it.

**Reconnection semantics are a genuine product decision.** Broadcaster uplinks drop routinely, particularly on consumer connections. Too short a reconnection window ends streams on ordinary hiccups; an unbounded one leaves every viewer on a frozen player when a broadcaster disappears for good. Neither default is safe, and the window has to be chosen with both failure modes in view — along with whether a reconnection continues the existing stream or begins a new one, which determines whether viewers experience a stall or an ending.

**Capacity must precede the event.** A scheduled broadcast goes from nothing to peak audience within a minute or two, far faster than any autoscaler responds. What makes this survivable is that segments are immutable and identical for every viewer, so the CDN serves essentially the whole audience and the origin sees roughly one request per segment per point of presence. That property is the entire scaling story for live, and anything that makes segments viewer-specific — server-side personalisation, per-viewer watermarking done naively — removes it and converts a cacheable broadcast into a per-viewer workload.

**Failures are immediately visible and easy to misattribute.** With no queue of unprocessed work to hide behind, any stage failing reaches viewers within seconds, so detection must outpace a human noticing. The subtlest case is a slow publish path: segments arrive late, players exhaust their buffers, and the symptom presents to viewers as a network problem and to operators as normal origin metrics — which is why segment publish latency deserves direct measurement rather than inference. And the question worth asking before any of this is built is whether the requirement is genuinely live: a substantial fraction of stated live requirements are satisfied by publishing a recording a few minutes later, at a small fraction of the operational cost.

**Prove it — interview questions**

1. **[Basic] How does live ingest differ from uploading a file?**

   <details><summary>Model answer</summary>

   The media arrives at exactly the rate it was captured and cannot be paused, re-read or retried. An upload can be buffered, processed slowly, and retried on failure; a live stream offers none of those, because the source keeps producing regardless of what the pipeline is doing. That means every stage must keep pace with real time permanently, and there is no backlog to work through later — a backlog is simply latency that never goes away.

   </details>

2. **[Basic] Why is segment length the main latency lever?**

   <details><summary>Model answer</summary>

   Because it enters the budget twice. A segment cannot be published until it is complete, which costs one segment duration, and the player buffers several segments before playing, which costs several more. So halving segment length reduces latency by more than half a segment — it reduces both terms. The cost is that shorter segments compress less efficiently, generate more requests, and leave the player with a smaller buffer, which directly increases stalling.

   </details>

3. **[Senior] Why can a live pipeline never catch up after falling behind?**

   <details><summary>Model answer</summary>

   Because there is no idle period. An ordinary batch pipeline that falls behind processes its backlog when load drops; a live pipeline's input continues at real time forever, so any stage running slower than real time accumulates delay monotonically. A stage at ninety-eight per cent of real time adds about a minute of latency every fifty minutes, and nothing ever removes it. That is why live transcoding needs genuine headroom rather than adequacy, and why degradation must take the form of producing fewer renditions at full speed rather than the full ladder more slowly.

   </details>

4. **[Senior] How should a broadcaster disconnection be handled?**

   <details><summary>Model answer</summary>

   It depends on duration, and both extremes are bad. A brief drop — the ordinary case on a mobile or home uplink — should hold the stream open for several seconds so a reconnection continues the same stream; viewers see a short stall rather than the broadcast ending. An extended absence should end the stream and publish what exists as a recording, because a stream held open indefinitely for a broadcaster who has gone leaves every viewer on a frozen player with no indication anything is wrong. The window in between is a product decision, and leaving it to a default nobody chose is how both failure modes end up in production.

   </details>

5. **[Staff] Design the live pipeline for a platform hosting thousands of concurrent broadcasters.**

   <details><summary>Model answer</summary>

   The economics differ from on-demand because transcoding cost scales with broadcasters, not viewers, so the transcode tier is the whole cost model. Regional ingest endpoints so broadcasters connect nearby, authenticated at connect with a stream key. Hardware-accelerated transcoding as a requirement rather than an optimisation, provisioned to run comfortably faster than real time, with an automated degradation policy that drops the top rendition under pressure — fewer qualities at real time always beats the full ladder behind it. Four-second keyframe-aligned segments, written to storage before the manifest is updated, since a manifest entry for a segment that does not exist is an immediate playback error. Delivery through a CDN with a shield tier, manifests cached for a couple of seconds because they change constantly and segments for an hour because they never do. For a scheduled event I would pre-provision rather than rely on autoscaling, since the audience arrives within a minute. And I would measure segment publish latency explicitly, because a slow publish path presents to viewers as a network problem and to operators as nothing at all.

   </details>

6. **[Principal] What makes live streaming operationally harder than on-demand, given the delivery path is identical?**

   <details><summary>Model answer</summary>

   That every safety margin has been spent on latency. The delivery mechanism is the same — keyframe-aligned segments on a CDN — but on-demand playback holds thirty seconds of buffer that absorbs almost all variance, while live holds a few seconds because buffer is latency. So the same content that plays flawlessly as a recording stalls as a live stream on the same connection, and that is not a bug but the trade the product asked for. The pipeline side has the same character: there is no queue to absorb a slow stage and no idle time to recover in, so a problem that would be a brief latency blip in a batch system becomes permanent drift. The practical consequences are that detection must be faster than a viewer noticing, that capacity must exist before the event rather than be scaled into, and that degradation policies must be decided and automated in advance because there is no time to think during the event. And the honest observation to raise early is that a meaningful fraction of requirements described as live are satisfied by publishing a recording minutes later, at a fraction of the operational cost — that question is worth asking before building any of this.

   </details>

---

### Content addressing and deduplication

*Name data by the hash of its contents so identical data is stored once, references are immutable, and integrity is verifiable by construction.*

**Flow:** `Content hash` → `Address` → `Reference count` → `Dedup store` → `Garbage collection`

> **The 30-second version**  
> Address data by the hash of its content so identical data is stored once, references are immutable and permanently cacheable, and integrity is verifiable — at the cost of deletion becoming a shared-ownership problem.

**The problem**

A thousand users upload the same popular file. Stored by filename or identifier, that is a thousand copies of identical bytes. In a backup system the waste is worse: the same file appears in every daily snapshot, unchanged, for a year.

A second problem hides behind the first. Names assigned by users or systems say nothing about content, so there is no way to know two objects are identical without comparing them, no way to verify an object was not corrupted or substituted, and no safe way to cache indefinitely because a name's meaning can change.

> **Naming by content makes identity, integrity and immutability the same property**  
> If an object's address is the hash of its bytes, then identical content has the same address — so deduplication is automatic rather than a process. The address also verifies the content, since recomputing the hash checks it. And the mapping from address to content can never change, so any cache holding it is permanently correct. Three separate concerns collapse into one decision about naming.

**Mental model**

Storage becomes a map from hash to bytes. Writing computes the hash and stores only if absent; reading fetches by hash and can verify. Human-meaningful names live in a separate layer that points at hashes.

1. **Hash** — A cryptographic digest of the content, which is its address.
2. **Content store** — Immutable map from hash to bytes. Writing the same content twice is a no-op.
3. **Name layer** — Mutable mapping from human names or identifiers to hashes — this is where change happens.
4. **References** — Which names or snapshots point at which hashes, needed to know what is still in use.
5. **Garbage collection** — Removing content nothing references, which is the hard operational part.

> **Deleting deduplicated content is dangerous by construction**  
> Because one stored object may back many logical references, deleting it on behalf of one owner destroys data for all the others. Every content-addressed store therefore needs reference counting or reachability analysis, and both are subject to races: content can gain a reference between the moment you determine it is unreferenced and the moment you delete it. Getting this wrong is silent data loss, which is why conservative collection with a grace period is the standard approach.

**How it works**

**Write and read paths**

```text
WRITE
  hash = SHA-256(content)
  if store contains hash:
      increment reference / add to reference set
      RETURN hash        <- no bytes written at all
  else:
      write bytes at hash
      RETURN hash

-> the thousandth upload of the same file writes nothing
-> the client can hash first and skip the upload entirely

READ
  bytes = store[hash]
  verify SHA-256(bytes) == hash    (optional but cheap)
  -> corruption and substitution are DETECTABLE, which
     is not true of name-addressed storage

DELETE  (the dangerous one)
  this reference goes away
  if no references remain:
      content MAY be collectable
  -> but a new reference can appear between the check
     and the delete
  -> hence grace periods and conservative collection

WHAT YOU GET FOR FREE
  dedup           identical content has one address
  integrity       the address verifies the content
  immutability    the mapping can never change
  cacheability    a cached entry is correct forever
```

1. **Choose the chunking strategy before anything else** — Whole-file hashing deduplicates only identical files; content-defined chunking deduplicates across similar files and is what makes backup systems effective.
2. **Use a cryptographic hash** — A collision means one object silently overwriting another, so the hash must be collision-resistant against adversaries, not merely unlikely to collide by accident.
3. **Separate the name layer from the content store** — Content is immutable; names change. Conflating them loses the property that makes content addressing valuable.
4. **Track references explicitly** — Without knowing what points at each object, nothing can ever be deleted safely.
5. **Collect garbage conservatively with a grace period** — Races between reference creation and collection cause silent data loss, so err toward retaining.
6. **Consider the cross-tenant privacy implication** — Deduplication across users lets an observer infer whether content already exists, which is a real information leak.

**Chunking determines how well dedup works**

```text
WHOLE-FILE HASHING
  dedup only when files are byte-identical
  a 1 GB file with one byte changed = fully distinct
  -> good for: identical uploads, shared assets
  -> useless for: backups of changing files

FIXED-SIZE CHUNKS  (e.g. 4 MB)
  insert one byte at the start of a file
  -> every subsequent chunk boundary shifts
  -> every chunk hashes differently
  -> dedup destroyed by a single insertion

CONTENT-DEFINED CHUNKING  (rolling hash)
  boundaries chosen where the rolling hash of a sliding
  window hits a pattern
  -> boundaries depend on CONTENT, not position
  -> an insertion changes only the chunk containing it
  -> the rest still deduplicates

THIS IS WHY BACKUP SYSTEMS ACHIEVE 10-20x
  daily snapshots of a mostly unchanged dataset store
  only the chunks that actually changed
  with fixed chunks, a small edit early in a large file
  would re-store the entire remainder

COST
  average chunk 1-8 MB
  smaller chunks -> better dedup, more metadata
  metadata can dominate if chunks are too small
```

> **Cross-user deduplication leaks the existence of content**  
> If uploading a file completes instantly when the content already exists, a user can determine whether some specific file is present in the system without having access to it. That is a genuine information leak with real consequences for sensitive material. The mitigations are to deduplicate only within a trust boundary such as a single account, or to make the timing indistinguishable — and the fact that the naive implementation leaks by default makes this worth deciding explicitly.

**Worked example**

A backup system, where content addressing does the most work.

**Backup with content-defined chunking**

```text
DATASET  500 GB, backed up daily, ~1% changes per day

NAIVE DAILY FULL BACKUP
  500 GB x 365 = 182 TB per year

CONTENT-ADDRESSED WITH CDC
  day 1:  500 GB stored (all chunks new)
  day 2:  ~5 GB of changed chunks stored
            the rest already exist -> no bytes written
  day 3+: ~5 GB/day
  year:   500 GB + 364 x 5 GB = ~2.3 TB

-> roughly 80x reduction, and every snapshot is still
   a complete point-in-time view

WHY EVERY SNAPSHOT IS COMPLETE
  a snapshot is a LIST OF CHUNK HASHES
  chunks are shared across snapshots
  -> no incremental chain to replay
  -> no full-plus-increments restore procedure
  -> restoring any day is equally cheap

DELETION  (the hard part)
  deleting a 90-day-old snapshot removes its hash list
  chunks referenced by other snapshots must survive
  -> mark-and-sweep over reachable chunks
  -> a grace period before collection, because a new
     snapshot may reference a chunk mid-sweep
  -> conservative: retaining garbage is cheap,
     deleting live data is not

INTEGRITY
  verify by rehashing chunks
  silent corruption is DETECTED, not discovered
  during a restore
```

| Metric | Value | Note |
|---|---|---|
| Naive | 182 TB/year | daily fulls |
| Content-addressed | ~2.3 TB/year | **~80× less** |
| Restore | any day, equal cost | no chain |
| Integrity | verify by rehash | corruption detected |

> **Content addressing makes every snapshot both complete and incremental**  
> A snapshot is a list of chunk hashes, and chunks are shared. That means each snapshot is a full point-in-time view — restoring any day requires no chain of increments — while consuming only the storage of what actually changed. Traditional backup forces a choice between full backups that are expensive and incremental chains that are fragile; content addressing removes the choice entirely, which is the single clearest demonstration of what the technique buys.

**When to use it**

- **Backup and snapshot systems**, where the same data recurs across versions.
- **Version control**, where content addressing gives immutable history and cheap branching.
- **Container image layers**, deduplicated across images and cached by digest.
- **Build artefact and dependency caches**, where the hash identifies the exact input.
- **Any storage with substantial duplicate content**, such as shared uploads.
- **Immutable asset URLs**, where the hash in the name makes year-long caching safe.

**When to avoid it**

- **Do not use it for frequently mutated data**, where every change writes a new object and churns references.
- **Do not deduplicate across trust boundaries** without considering the existence leak.
- **Do not use a non-cryptographic hash**, where an adversarial collision means silent data replacement.
- **Do not build it without reference tracking**, since nothing can then be deleted safely.
- **Do not use small chunks indiscriminately**, where metadata can exceed the data it describes.

**Advantages**

- **Automatic deduplication** — identical content is stored once with no separate process.
- **Integrity is verifiable** by recomputing the hash, so corruption and substitution are detectable.
- **Immutable references** that can be cached indefinitely with no invalidation.
- **Snapshots are complete and cheap simultaneously**, removing the full-versus-incremental trade.
- **Safe concurrent writes**, since writing identical content twice is idempotent.
- **Efficient synchronisation**, as peers exchange hashes to determine what is missing.

**Disadvantages**

- **Deletion is genuinely hard**, requiring reference counting or reachability with race protection.
- **No locality**, since hashes are random and related content is scattered.
- **Metadata overhead** grows with chunk count and can dominate for small chunks.
- **Cross-tenant dedup leaks existence** unless deliberately mitigated.
- **Poor fit for mutable data**, where every change creates new objects.
- **Hash computation costs CPU** on every write.

**Trade-offs**

**Chunking trade-offs**

| Strategy | Dedup effectiveness | Metadata cost | Best for |
|---|---|---|---|
| Whole-file hashing | Identical files only | Minimal | Shared assets; identical uploads |
| Fixed-size chunks | Destroyed by insertions | Low | Append-only data |
| Content-defined chunking | Survives insertions and edits | Moderate | Backups; changing files |
| Small chunks (< 1 MB) | Highest | Can dominate | Small datasets with fine-grained change |
| Large chunks (8 MB+) | Coarse | Low | Large files; bandwidth-limited transfer |

The distinction between fixed and content-defined chunking is the one that matters most in practice: a single byte inserted at the start of a file shifts every fixed boundary and destroys deduplication entirely, while content-defined boundaries move with the data and leave everything after the edit still matching.

**How it fails**

**Content-addressed store failures**

| Failure | Cause | Fix |
|---|---|---|
| Data lost when one owner deletes | No reference tracking | Reference counts or reachability analysis |
| Live data collected as garbage | Race between new reference and sweep | Grace period; conservative collection |
| Storage grows despite deduplication | Fixed chunking defeated by insertions | Content-defined chunking |
| Metadata larger than the data | Chunks too small | Increase average chunk size |
| Silent replacement of content | Weak hash with feasible collisions | Cryptographic hash |
| Users infer presence of private files | Cross-tenant dedup with instant completion | Dedup within a trust boundary; mask timing |
| Poor read performance | Related chunks scattered by random addresses | Locality hints; pack related chunks together |

**Limits**

> **Reference figures**
>
> - **Backup deduplication**: 10–20× typical for daily snapshots of slowly changing data, higher over long retention.
> - **Chunk size**: 1–8 MB average is the usual range; below that metadata overhead grows sharply.
> - **Hash**: SHA-256 or similar; collision resistance must hold against adversaries, not just chance.
> - **Garbage collection**: conservative with a grace period — retaining garbage is cheap, deleting live data is not.
> - **Locality**: hash addresses are random, so sequential reads of related content require explicit packing.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Content addressing + CDC | Backups; versioned data | Deletion complexity; no locality |
| Whole-file hash dedup | Shared identical uploads | No dedup across similar files |
| Delta encoding | Linear version chains | Restore requires replaying deltas |
| Compression only | Single-copy data | No cross-object savings |
| Name-addressed storage | Mutable data | No dedup, no integrity guarantee |
| Block-level dedup in storage | Transparent to applications | Less effective; opaque behaviour |

Delta encoding is the natural comparison for backups: it stores differences against a previous version, which is efficient but creates a chain, so restoring the newest version means replaying everything before it. Content addressing achieves comparable savings while keeping every snapshot independently complete, which is why it has largely displaced delta chains in modern backup design.

**In real systems**

- **Version control systems** address objects by content hash, which is what makes commits immutable, history verifiable and identical files stored once.
- **Container image layers** are identified by digest, so layers shared between images are pulled and stored once and can be cached permanently.
- **Deduplicating backup tools** use content-defined chunking to achieve large reductions while keeping every snapshot independently restorable.
- **Content-hashed asset filenames** in web builds are the same idea applied to caching: the hash in the name makes year-long immutable caching safe.
- **Peer-to-peer and distributed storage systems** use content addresses so any peer's copy is verifiably the right bytes regardless of who served it.

**Common mistakes**

- **No reference tracking**, making deletion unsafe or impossible.
- **Aggressive garbage collection** without a grace period, losing live data to races.
- **Fixed-size chunking** for data that gets edited, destroying deduplication.
- **Chunks too small**, letting metadata exceed the data.
- **Non-cryptographic hashes**, permitting adversarial collisions.
- **Cross-tenant deduplication** without considering the existence leak.
- **Expecting locality**, when hash addresses scatter related content.

**The staff-level view**

Content addressing is usually adopted for deduplication and turns out to be more valuable for immutability and verifiability.

- **Design deletion before adopting it.** Deduplication makes deletion a shared-ownership problem, and a store without reference tracking is one where nothing can ever be removed safely — which is discovered when storage costs force the question.
- **Choose chunking deliberately.** Whole-file and fixed-size chunking both fail on exactly the workload deduplication is usually bought for; content-defined chunking is what delivers the order-of-magnitude reductions people expect.
- **Decide the cross-tenant dedup question explicitly.** Instant completion for content that already exists leaks whether a specific file is present, which matters for sensitive material and is invisible in the naive implementation.
- **Collect garbage conservatively.** Retaining unreferenced data costs money; deleting referenced data costs the data, and the race between reference creation and sweeping is real.
- **Exploit the immutability, not just the dedup.** Content-addressed references can be cached forever with no invalidation, which is often worth more than the storage saved.

**Go deeper**

Content addressing names data by the cryptographic hash of its bytes. Identical content therefore has the same address, so deduplication happens automatically rather than as a process; the address doubles as a checksum, so integrity is verifiable; and the mapping can never change, so references are immutable and anything cached under them stays correct forever. Three concerns collapse into one naming decision.

How well deduplication works depends entirely on chunking. Whole-file hashing catches only byte-identical files, and fixed-size chunks are destroyed by a single insertion because every subsequent boundary shifts. Content-defined chunking picks boundaries from a rolling hash over the content, so an edit invalidates only the chunk containing it — which is what lets backup systems store a year of daily snapshots for a small multiple of one full copy, with every snapshot independently restorable.

The cost is deletion. Because one object may back many references, removing it for one owner destroys it for all, so the store needs reference counting or reachability analysis plus protection against the race between a reference appearing and a sweep removing the object. Collection is therefore conservative with grace periods. A further consideration is that deduplicating across users leaks whether specific content already exists, which for sensitive material is a real disclosure and is invisible in the naive implementation.

Content addressing replaces assigned names with derived ones: an object's address is the cryptographic hash of its contents.

**One decision, three properties.** Identical bytes produce an identical address, so duplicate storage is impossible by construction rather than eliminated by a process. The address is a checksum, so corruption or substitution is detectable by recomputation — a guarantee name-addressed storage cannot offer. And because the mapping from address to content can never change, references are immutable, which means any cache holding one is permanently correct and no invalidation mechanism is needed. Writes also become idempotent, so concurrent writers of the same content require no coordination at all.

**Chunking decides whether deduplication actually works.** Hashing whole files deduplicates only byte-identical uploads, which is useful for shared assets and useless for anything that changes. Fixed-size chunking appears to solve that until a single byte is inserted near the start of a file, shifting every subsequent boundary so that no chunk matches its predecessor — deduplication collapses on exactly the workload it was bought for. Content-defined chunking selects boundaries where a rolling hash over a sliding window meets a condition, so boundaries follow content rather than position: an insertion perturbs one chunk and everything after it still matches. This is the difference between a backup system achieving an order of magnitude and achieving nothing.

**Snapshots stop being a trade-off.** Traditionally a backup is either a full copy, which is complete but expensive, or an incremental, which is cheap but requires a chain to restore. A content-addressed snapshot is a list of chunk hashes with chunks shared between snapshots, so it is simultaneously a complete point-in-time view and incremental in storage cost. Restoring any day costs the same as any other, with no chain to replay and no dependency on the integrity of intervening increments.

**Deletion becomes a shared-ownership problem.** Since a single stored object may back many logical references, removing it on one owner's behalf destroys data belonging to others. The store must therefore track what points at each object, through reference counts or periodic reachability analysis, and must contend with the race in which a new reference is created between the determination that an object is unreferenced and its actual removal. The standard resolution is conservative collection with a grace period, biased toward retention, because unreferenced data costs storage while wrongly collected data costs the data itself.

**Deduplication across trust boundaries discloses existence.** When an upload completes instantly because the content is already present, the uploader learns that someone, somewhere, holds that exact file. This allows testing for the presence of specific sensitive content without any access to it, and it arises automatically from the most natural implementation. Restricting deduplication to within an account, or equalising completion timing, are the available mitigations — but the point is that this must be decided rather than inherited.

**The secondary properties often outweigh the primary one.** Content addressing is typically proposed for storage savings, yet in many systems the immutability and verifiability matter more: content-hashed asset names make year-long browser caching safe with no invalidation machinery; container layers identified by digest can be pulled once and trusted indefinitely; distributed systems can accept data from untrusted peers because the address proves the bytes. When evaluating the technique, those benefits belong in the assessment alongside the deduplication ratio — and against them sits the one real cost, which is that deletion must be designed for up front rather than discovered later.

**Prove it — interview questions**

1. **[Basic] What does content addressing mean?**

   <details><summary>Model answer</summary>

   Naming data by the cryptographic hash of its contents rather than by an assigned identifier. The address is derived from the bytes, so identical content always has the same address — which makes deduplication automatic rather than a separate process — and the address verifies the content, since recomputing the hash confirms you received what you asked for. It also means the mapping from address to content can never change, so anything cached under that address is correct permanently.

   </details>

2. **[Basic] Why is deletion hard in a deduplicated store?**

   <details><summary>Model answer</summary>

   Because one stored object may back many logical references. If a thousand users uploaded the same file, there is one copy; deleting it when one user removes their reference destroys the data for the other nine hundred and ninety-nine. So the store must know what still points at each object — through reference counts or reachability analysis — and even then there is a race, since a new reference can appear between determining that something is unreferenced and actually deleting it. That is why collection is conservative and uses grace periods: retaining garbage costs money, deleting live data costs the data.

   </details>

3. **[Senior] Why does content-defined chunking matter?**

   <details><summary>Model answer</summary>

   Because fixed boundaries are fragile in exactly the way real data changes. Insert one byte at the start of a file and every fixed-size chunk boundary after it shifts, so every chunk hashes differently and deduplication is completely destroyed even though the data barely changed. Content-defined chunking picks boundaries using a rolling hash over a sliding window, so boundaries follow the content rather than the offset: an insertion changes only the chunk containing it and everything after still matches. That is what makes backup deduplication achieve order-of-magnitude reductions instead of nearly nothing.

   </details>

4. **[Senior] What is the privacy concern with cross-user deduplication?**

   <details><summary>Model answer</summary>

   If an upload completes instantly because the content already exists in the store, that timing difference tells the uploader that someone else has the same file. A user can therefore test whether a specific file exists in the system without having any access to it, which for sensitive material is a genuine leak — and it comes for free with the naive implementation, so it is easy not to notice. Mitigations are to deduplicate only within a trust boundary such as a single account, or to make completion timing indistinguishable regardless of whether the content was already present.

   </details>

5. **[Staff] Design a backup system using content addressing.**

   <details><summary>Model answer</summary>

   Content-defined chunking with an average chunk of a few megabytes, so an edit anywhere in a file invalidates only the chunks it touches rather than everything after it. Each chunk stored once under its SHA-256, and each snapshot represented as an ordered list of chunk hashes — which gives the property I most want: every snapshot is a complete point-in-time view requiring no chain to restore, while consuming only the storage of what actually changed. For a five-hundred-gigabyte dataset changing one per cent daily, a year of daily snapshots costs a couple of terabytes instead of a hundred and eighty. Integrity comes free by rehashing on read, so silent corruption is detected rather than discovered during a restore. The genuinely hard part is retention: expiring old snapshots means removing their hash lists and then collecting chunks nothing references, which needs mark-and-sweep with a grace period because a new snapshot can reference a chunk mid-sweep. I would deliberately bias toward retaining — unreferenced chunks cost storage, wrongly collected chunks cost the backup.

   </details>

6. **[Principal] Content addressing is often adopted for deduplication. What else does it give you?**

   <details><summary>Model answer</summary>

   The storage saving is usually the reason it is proposed and rarely the most valuable property. Immutability is the big one: because the mapping from address to bytes can never change, any reference is permanently valid and anything cached under it is correct forever — which is why content-hashed filenames let web assets be cached for a year with no invalidation, and why container layers can be pulled once and trusted indefinitely. Verifiability is the second: the address is a checksum, so you can confirm that what you received is what you asked for regardless of who served it, which is what makes distributed and peer-to-peer distribution safe without trusting the source. And idempotent writes fall out too — writing the same content twice is a no-op, so concurrent writers need no coordination. The practical lesson is that when evaluating content addressing, the storage saving should be weighed alongside cacheability, integrity and coordination-free writes, because in many systems those are worth more than the bytes. The cost to weigh against all of it is that deletion becomes a shared-ownership problem that must be designed for before adoption, not after storage growth forces the question.

   </details>

---
