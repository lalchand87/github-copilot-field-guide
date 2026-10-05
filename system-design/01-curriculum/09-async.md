# Curriculum · Async

[← System Design index](../README.md)

> 8 lessons in **Async**. Part of the Curriculum — Build the fundamentals.

## Contents

- **Async** (8): [Background job lifecycle](#background-job-lifecycle) · [Retry, backoff and jitter](#retry-backoff-and-jitter) · [Backpressure](#backpressure) · [Batching and microbatching](#batching-and-microbatching) · [Durable workflow orchestration](#durable-workflow-orchestration) · [Sagas and compensation](#sagas-and-compensation) · [Fanout and aggregation](#fanout-and-aggregation) · [Distributed scheduling](#distributed-scheduling)

## Async

### Background job lifecycle

*A job is a durable record with an explicit state machine — accepted, queued, running, terminal — so its progress is observable and its failure is recoverable.*

**Flow:** `Accepted` → `Queued` → `Running` → `Terminal state` → `Result`

> **The 30-second version**  
> Model background work as a durable record with named states, bounded transitions, leases with heartbeats, and a reaper — so progress is visible and crashes are recoverable.

**The problem**

A user uploads a video, clicks generate report, or triggers an export. The work takes minutes. Holding an HTTP connection open is unacceptable, so the request returns immediately — and now the user has no idea whether anything is happening, the operations team cannot tell what is stuck, and a crashed worker leaves the job in a state nobody can name.

Treating background work as fire-and-forget makes it invisible. The fix is to model the job as a first-class durable entity with an explicit state machine, so that at every moment the system can answer three questions: what state is it in, how did it get there, and what happens next.

> **The job record is the contract**  
> The queue message is transport; it can be redelivered, duplicated and lost. The job record in the database is the truth: it holds the state, the attempt history, the result and the failure reason. Systems that rely only on the queue have no way to answer “what happened to my export?” because the message is gone once acknowledged.

**Mental model**

A job moves through a small set of named states via explicit transitions. Every transition is recorded. Terminal states are final. Anything not in a terminal state is either progressing or stuck, and the difference is measurable.

1. **Accepted** — The request was validated and a durable job record created. The client gets an id immediately. This is what makes the operation observable.
2. **Queued** — The job is ready for a worker. Created transactionally with the record, so a job cannot exist without being scheduled.
3. **Running** — A worker has claimed it, with a lease and a heartbeat so a crash is detectable rather than indefinite.
4. **Terminal** — Succeeded, failed permanently, or cancelled. No further transitions. The result or failure reason is stored here.
5. **Retrying** — A transient failure occurred and the job will be attempted again after a delay. Distinct from running and from failed.

> **A job with no terminal state is a job nobody can reason about**  
> If a worker crashes and the record stays in `running` forever, no dashboard can distinguish it from genuine progress, no alert fires, and the user waits indefinitely. Every non-terminal state needs a bound — a lease with a heartbeat, a maximum duration, an expiry — after which the job transitions somewhere explicit rather than lingering.

**How it works**

**The state machine, with the transitions that matter**

```text
              +-----------+
request ----> | accepted  |  durable record created, id returned
              +-----+-----+
                    v
              +-----------+
              |  queued   |  visible to workers
              +-----+-----+
                    v
              +-----------+   lease expires / heartbeat lost
              |  running  |--------------------+
              +--+-----+--+                    |
                 |     |                       v
        success  |     | transient failure  (back to queued,
                 |     v                    attempt + 1)
                 |  +-----------+
                 |  | retrying  | -- delay --> queued
                 |  +-----+-----+
                 |        | attempts exhausted
                 v        v
            +---------+ +---------+
            |succeeded| | failed  |   TERMINAL
            +---------+ +---------+
                             |
                        (also: cancelled)

EVERY non-terminal state has a TIMEOUT that forces a
transition. Nothing lingers.
```

1. **Create the record and enqueue in one transaction** — Otherwise a crash between them leaves a job the user can see but nobody will run, or a message referring to a job that does not exist. This is the same dual-write problem the transactional outbox solves.
2. **Return an id immediately, and expose a status endpoint** — `202 Accepted` with a `Location` pointing at the job resource makes the asynchronous nature part of the contract rather than a surprise.
3. **Lease and heartbeat long-running jobs** — A worker claims a job for a bounded period and extends it while working. If it dies, the lease expires and the job returns to the queue rather than sitting in `running` forever.
4. **Distinguish transient from permanent failure** — A timeout retries; a malformed input never will. Retrying permanent failures wastes the budget and delays the moment anyone learns about the real problem.
5. **Store the result and the failure reason on the record** — A job that failed with no explanation generates a support ticket. The reason, the attempt count and the timestamps should all be queryable.
6. **Make cancellation explicit** — Users start expensive jobs by mistake. A cancellation state that workers check periodically is far better than letting the work run to completion and discarding it.

**Idempotency and the at-least-once reality**

```text
the worker claims job 4711, does the work, then crashes
BEFORE marking it succeeded

-> lease expires -> job returns to queued -> claimed again
-> the work runs a SECOND time

THIS IS UNAVOIDABLE. The fix is in the handler:

  if job already produced its output (check by job id):
      mark succeeded; return
  perform the work
  record output AND mark succeeded    <- one transaction

FOR EXTERNAL EFFECTS (email, payment, webhook)
  pass the job id as an idempotency key so the remote
  system deduplicates - a local transaction cannot cover
  an external call

THE STATE MACHINE DOES NOT PREVENT DUPLICATES.
It makes them detectable and recoverable.
```

> **Expose progress, not just state, for long jobs**  
> A video transcode that takes eight minutes should report percentage or stage — “transcoding 720p, 3 of 5 renditions” — not merely `running`. It costs one extra column and transforms the user experience from an opaque wait into a visible process, while also giving operators a way to distinguish a slow job from a stuck one.

**Worked example**

A data export job: the full lifecycle including the failure paths teams usually omit.

**Export job, end to end**

```text
POST /exports  {filters}
  validate synchronously (fail fast on bad input)
  BEGIN
    INSERT jobs (id, type='export', state='accepted',
                 params, owner, created_at)
    INSERT outbox (job_queued event)     <- same transaction
  COMMIT
  return 202 + Location: /exports/{id}

GET /exports/{id}
  { state: "running", progress: 0.42,
    stage: "querying", startedAt: ..., attempts: 1 }

WORKER
  claim: UPDATE jobs SET state='running', worker=?,
         lease_until=now()+2min
   WHERE id=? AND state IN ('queued','retrying')
  -> conditional claim; two workers cannot both win

  heartbeat every 30s: extend lease_until
  update progress as stages complete

  on success:
    write the file to object storage
    BEGIN
      UPDATE jobs SET state='succeeded',
             result_url=?, finished_at=now()
    COMMIT

  on transient failure (timeout, 503):
    attempts < 5 ?  state='retrying', next_attempt_at=
                    now() + backoff(attempts) + jitter
                 :  state='failed', error='...'

  on permanent failure (invalid filter, deleted dataset):
    state='failed' IMMEDIATELY, error='...'
    -> do not burn 5 attempts on something that cannot work

REAPER (scheduled)
  UPDATE jobs SET state='queued', attempts=attempts+1
   WHERE state='running' AND lease_until < now()
  -> recovers jobs whose workers died
```

| Metric | Value | Note |
|---|---|---|
| States | 6 named | all bounded |
| Claim | conditional update | no double-claim |
| Lease | 2 min + heartbeat | **crash recovery** |
| Reaper | scheduled | no stuck jobs |

> **The reaper is the component teams forget**  
> Everything works until a worker is killed mid-job — by a deploy, an OOM, a node preemption — and the record sits in `running` with an expired lease. Without a scheduled process that finds those and returns them to the queue, the job is lost silently and the user waits forever. The reaper is what converts “a worker died” from a data-loss event into a slightly delayed retry.

**When to use it**

- **Any work too slow for a synchronous response** — exports, imports, transcoding, report generation, bulk operations.
- **Work that must survive a crash**, where a durable record is the only way to resume.
- **User-visible long operations**, where progress and status are part of the product.
- **Scheduled and recurring work**, which is the same lifecycle with a different trigger.
- **Operations requiring audit**, since the job record is a natural record of what was requested, by whom, and what happened.

**When to avoid it**

- **Do not use it for work that completes in milliseconds** — the record, the queue and the polling cost more than the work.
- **Do not rely on the queue message as the source of truth**; it disappears on acknowledgement and cannot answer status queries.
- **Do not leave non-terminal states unbounded**, which produces jobs that are neither progressing nor failed.
- **Do not retry permanent failures**, which delays the alert and wastes the budget.
- **Do not omit the reaper**, or every worker crash silently loses a job.

**Advantages**

- **Requests return immediately** while long work proceeds independently.
- **Progress and status are queryable**, which is both a product feature and an operational one.
- **Crashes are recoverable**, because the durable record plus lease expiry brings the job back.
- **Failures are explainable**, since the reason, attempts and timing are stored rather than lost in logs.
- **Backpressure is natural** — queue depth and job age show whether capacity is sufficient.

**Disadvantages**

- **More moving parts**: a job table, a queue, workers, a reaper, a status endpoint.
- **At-least-once execution** means every handler must be idempotent.
- **Clients must poll or subscribe**, which is more complex than a synchronous response.
- **State machine bugs are subtle**, particularly around claiming and lease expiry.
- **Job records accumulate** and need a retention policy.

**Trade-offs**

**How clients learn the outcome**

| Mechanism | Latency to notice | Cost | Best for |
|---|---|---|---|
| Polling the status endpoint | Poll interval | Repeated requests | Simple; most cases |
| Long polling | Near-immediate | Held connections | Moderate job counts |
| Webhook callback | Immediate | Delivery reliability, retries | Server-to-server |
| Server-sent events / WebSocket | Immediate | Connection management | Live UI progress |
| Email or notification | Minutes | Minimal | Long jobs; user not waiting |

Polling is underrated: with a sensible interval and an `ETag`, it is cheap, requires no persistent connections, survives client restarts, and works through every proxy. Push mechanisms are better user experience for jobs measured in seconds, and unnecessary for jobs measured in minutes.

**How it fails**

**Job lifecycle failures**

| Failure | Cause | Fix |
|---|---|---|
| Job stuck in running forever | Worker died; no lease expiry or reaper | Lease with heartbeat; scheduled reaper |
| Job exists but never runs | Record created, enqueue failed | Transactional outbox: record and message in one transaction |
| Work performed twice | Redelivery after a crash before marking success | Idempotent handler keyed on job id |
| Permanent failure retried five times | No failure classification | Classify; fail fast on non-retryable errors |
| User cannot tell what happened | Failure reason not stored on the record | Store error, attempts and timestamps; expose them |
| Cancelled job keeps running | No cancellation check in the worker | Workers check a cancellation flag between stages |
| Jobs table grows unbounded | No retention for terminal jobs | Archive or delete after a retention window |

**Limits**

> **Operating parameters**
>
> - **Lease duration**: 2–3× the p99 stage duration, extended by heartbeat rather than set long.
> - **Reaper interval** should be well under the lease duration, so recovery is prompt.
> - **Retry attempts**: 3–10 with exponential backoff and jitter, then terminal failure.
> - **Poll interval**: a few seconds for interactive jobs; back off for long ones.
> - **Retention**: keep terminal job records long enough for support and audit — typically weeks — then archive.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Job record + queue | General background work | Several moving parts |
| Queue only | Fire-and-forget with no status need | No observability or recovery |
| Durable workflow engine | Multi-step processes with branching | Heavier; another platform |
| Synchronous with a long timeout | Work under a few seconds | Holds connections; poor failure behaviour |
| Scheduled batch | Naturally periodic work | Latency; burstiness |
| Streaming pipeline | Continuous high-volume processing | Different operational model |

**In real systems**

- **Video and image processing pipelines** universally expose a job id and status, because the work takes minutes and users need to see progress.
- **Cloud provider long-running operation APIs** standardise on returning an operation resource that clients poll, which is exactly this pattern.
- **`SELECT ... FOR UPDATE SKIP LOCKED`** is how many systems implement safe job claiming directly in a database without a separate queue.
- **CI/CD systems** are elaborate job lifecycles, with states, logs, progress, cancellation and retention as first-class product features.
- **Export and report generation** in SaaS products almost always uses this shape, often with email notification for jobs measured in minutes.

**Common mistakes**

- **Treating the queue message as the source of truth**, so status cannot be queried.
- **Creating the record and enqueuing separately**, losing jobs on a crash between them.
- **No lease or heartbeat**, so a dead worker's job sits in `running` forever.
- **No reaper**, so crashed jobs are never recovered.
- **Non-idempotent handlers** under at-least-once execution.
- **Retrying permanent failures**, wasting the budget and delaying the alert.
- **No retention policy**, so the jobs table grows without bound.

**The staff-level view**

Background jobs are where observability is easiest to omit and hardest to retrofit, because the work is invisible by construction.

- **Insist on a durable job record, not just a queue message.** Status queries, failure reasons and audit all depend on it, and none of them can be added later from messages that no longer exist.
- **Require a bound on every non-terminal state.** A job that can sit in `running` indefinitely is invisible to every dashboard and alert.
- **Build the reaper before launch.** Worker crashes are routine — deploys, preemptions, OOM kills — and without recovery each one silently loses work.
- **Classify failures explicitly.** Retrying a permanent failure ten times delays the alert and teaches the team that retries mean nothing.
- **Alert on the oldest non-terminal job**, not on queue depth, because age is what the user experiences and depth conceals it.

**Go deeper**

Background work becomes invisible if the only artifact is a queue message, which disappears on acknowledgement. A durable job record holds the state, attempts, progress, result and failure reason, so users can query status and operators can see what is stuck. The request returns `202 Accepted` with a job id immediately, making the asynchronous nature part of the contract.

The state machine matters: accepted, queued, running, retrying, and terminal states of succeeded, failed or cancelled. Every non-terminal state must be bounded, because a worker killed by a deploy leaves its record in `running` and, without a lease expiry, nothing ever notices. Workers claim jobs with a conditional update so two cannot both win, extend a lease by heartbeat while working, and a scheduled reaper returns expired-lease jobs to the queue.

Two things the state machine does not solve. Execution is at-least-once — a worker that finishes the work and crashes before recording success will re-run it — so handlers must be idempotent, keyed on job id, with the same key passed to external systems. And failures must be classified: retrying a malformed input five times wastes the budget and delays the moment anyone learns the real problem. Alert on the age of the oldest non-terminal job rather than on queue depth.

Background jobs are invisible by construction, so the discipline is entirely about making their state and failures observable — decisions that must be made up front because the evidence is otherwise discarded.

**The record, not the message, is the truth.** A queue message is transport: redeliverable, duplicable, and gone once acknowledged. A durable job record carries the state, attempt history, progress, result and failure reason, which is what allows a status endpoint, a support investigation and an audit trail. Systems built on messages alone cannot answer “what happened to my export?” months later, and cannot answer it at all once the message is acknowledged. The record and the enqueue must be created in one transaction, or a crash between them leaves either a job nobody will run or a message referring to nothing — the same dual-write problem the transactional outbox exists to solve.

**Bounded states make stuck work detectable.** Accepted, queued, running, retrying, and the terminal states succeeded, failed and cancelled. The critical rule is that every non-terminal state has a bound. A worker killed by a deploy, an out-of-memory event or a node preemption leaves its record in `running`; without a lease expiry it is indistinguishable from progress, no alert fires, and the user waits forever. Leases extended by heartbeat plus a scheduled reaper that returns expired-lease jobs to the queue converts “a worker died” from silent loss into a slightly delayed retry — and the reaper is the component most commonly omitted.

**Claiming must be atomic.** A conditional update — set state to running with a lease, where state is currently queued or retrying — lets the database serialise the claim, with the affected row count telling the worker whether it won. No lock service, no coordination, no possibility of two workers running the same job simultaneously under normal operation.

**At-least-once execution is unavoidable.** A worker that performs the work, produces its output, and crashes before recording success will have the job re-run. The state machine makes that recoverable; it does not prevent it. Handlers must therefore check whether their output already exists, keyed by job id, and record completion atomically with the effect where the effect is local. Where the effect is external — an email, a payment, a webhook — the job id must be passed as an idempotency key so the remote system deduplicates, because no local transaction can cover a remote call.

**Failure classification changes the operational experience.** A timeout or a 503 may succeed on retry; a malformed input or a deleted referent never will. Retrying the latter consumes the retry budget, delays the moment anyone learns about the real defect, and teaches the team that retries are meaningless. Classifying failures so permanent ones move straight to a terminal state with a stored reason makes the failure explicable and the alert prompt.

**Alert on age, not depth.** Queue depth and throughput both conceal the user experience: a million queued jobs with a two-second age is a healthy high-throughput system, while a hundred jobs with a twenty-minute age means someone has been waiting twenty minutes. The age of the oldest non-terminal job, broken down by job type, is the signal that corresponds to what users feel — and per-type breakdown matters, because an aggregate view hides the one job type that is broken while the rest are fine.

**Prove it — interview questions**

1. **[Basic] Why model a background job as a durable record rather than just a queue message?**

   <details><summary>Model answer</summary>

   Because the message disappears once acknowledged, so it cannot answer any question afterwards. A durable record holds the state, the attempt history, the progress, the result and the failure reason, which is what makes the work observable to users and operators. It also survives worker crashes, which is what allows recovery: the queue message may be gone, but the record shows a job in `running` with an expired lease, and something can act on that.

   </details>

2. **[Basic] Why does every non-terminal state need a timeout?**

   <details><summary>Model answer</summary>

   Because otherwise a job that gets stuck is indistinguishable from one making progress. A worker killed by a deploy or an OOM leaves its record in `running`, and with no lease expiry nothing will ever notice — no alert fires, no dashboard flags it, and the user waits indefinitely. Bounding every non-terminal state with a lease or a maximum duration means the job always transitions somewhere explicit, which is what makes stuck work detectable.

   </details>

3. **[Senior] How do you safely claim a job so two workers cannot both run it?**

   <details><summary>Model answer</summary>

   With a conditional update: set the state to `running` with a lease expiry, but only where the current state is `queued` or `retrying`. The affected row count tells the worker whether it won the claim. That makes the claim atomic with no lock service and no coordination — the database serialises it. The complement is the lease: the worker extends it with a heartbeat while working, so a crash causes it to expire, and a scheduled reaper returns expired-lease jobs to the queue with an incremented attempt count.

   </details>

4. **[Senior] Why is idempotency still necessary if the state machine is correct?**

   <details><summary>Model answer</summary>

   Because the state machine cannot make the work and the state transition atomic when the work has external effects. A worker that completes a transcode, uploads the output, and then crashes before marking the job succeeded will have its lease expire and the job re-run — performing the work twice. The state machine makes that detectable and recoverable; it does not prevent it. The handler must therefore check whether its output already exists, keyed by job id, and for external effects like emails or payments it must pass the job id as an idempotency key so the remote system deduplicates.

   </details>

5. **[Staff] What operational signals would you expose for a background job system?**

   <details><summary>Model answer</summary>

   The one that matters most is the age of the oldest non-terminal job, because that is what a user actually experiences and queue depth conceals it entirely — a million queued jobs with a two-second age is healthy, while a hundred with a twenty-minute age is not. Alongside that: the count of jobs whose lease has expired, which indicates worker instability; the rate of terminal failures by error class, which distinguishes a code defect from a dependency outage; retry rate, since a rising one predicts exhaustion; and per-stage progress for long jobs so a slow job can be distinguished from a stuck one. I would also expose all of it per job type, because a single aggregated view hides the one job type that is broken while the others are fine.

   </details>

6. **[Principal] What makes background job systems hard to operate well?**

   <details><summary>Model answer</summary>

   The work is invisible by construction, so every failure is quiet unless someone deliberately made it loud. A synchronous failure surfaces as an error to a caller who notices; an asynchronous one surfaces as a record in a state nobody is watching, or as work that simply never happened. That means the operational design has to be done up front rather than added later — the durable record, the bounded states, the reaper, the failure reasons, the age-based alerting. Retrofitting observability is particularly hard here because the evidence has already been discarded: you cannot reconstruct why jobs failed last month from messages that were acknowledged and deleted. So my priority would be making the job record and its lifecycle a platform primitive rather than something each team implements, since the correct shape is identical everywhere and the failure modes of a hand-rolled version — unbounded states, missing reaper, non-idempotent handlers — are both predictable and severe.

   </details>

---

### Retry, backoff and jitter

*Retry only what can succeed, wait longer each time, randomise the wait, and cap the total — because uncoordinated retries turn a degradation into an outage.*

**Flow:** `Failure` → `Error classification` → `Retry budget` → `Jittered delay` → `New attempt`

> **The 30-second version**  
> Retry only transient failures, on idempotent operations, with exponentially growing randomised delays, bounded by a total deadline and a budget — or retries become the outage.

**The problem**

A dependency returns errors for thirty seconds. Ten thousand clients retry immediately. The dependency, already struggling, now receives double its normal load precisely when it has least capacity. It fails more, more clients retry, and a brief degradation becomes a sustained outage that outlives its original cause.

Retries are the correct response to transient failure and the most reliable way to convert a small problem into a large one. The difference is entirely in the policy: what you retry, how long you wait, whether the waits are randomised, and how much retrying you permit in total.

> **Retry amplification is multiplicative**  
> Each layer that retries multiplies the load of the layer below. A client retrying three times, calling a gateway that retries three times, calling a service that retries three times, produces twenty-seven requests for one user action. During an incident that is the difference between recovery and collapse — and each layer's policy looks reasonable in isolation.

**Mental model**

A retry is a bet that the same request will succeed later. That bet is only worth making if the failure was transient, if the operation is safe to repeat, and if the system can afford the extra load right now. All three conditions must hold.

1. **Classification** — Is this failure transient? A timeout or 503 may succeed; a 400 or a validation error never will.
2. **Safety** — Is the operation idempotent, or protected by an idempotency key? Retrying a non-idempotent operation after an ambiguous timeout duplicates it.
3. **Backoff** — Exponentially increasing delays, so the retry rate falls as the failure persists rather than staying constant.
4. **Jitter** — Randomisation, so independent clients do not synchronise into waves.
5. **Budget** — A cap on retries as a proportion of traffic, so a sustained failure cannot produce unbounded extra load.

> **Jitter matters more than backoff**  
> Exponential backoff reduces the rate from one client. Jitter prevents ten thousand clients from retrying at the same instant. Without jitter, all clients that failed together back off together and retry together, producing synchronised waves that hammer the recovering service at exactly the wrong moments — and each wave can knock it back down. Full jitter, choosing a random delay between zero and the exponential bound, is the standard and it is markedly better than adding a small random offset.

**How it works**

**Backoff strategies compared**

```text
base = 100ms, cap = 30s, attempt n

NO BACKOFF        100ms, 100ms, 100ms...
  constant retry rate; maximum amplification

EXPONENTIAL       100, 200, 400, 800, 1600ms...
  rate falls, but all clients follow the SAME schedule
  -> synchronised waves

EXPONENTIAL + SMALL JITTER   backoff +/- 10%
  slightly spread; waves still visible

FULL JITTER       sleep = random(0, min(cap, base * 2^n))
  each client picks independently across the whole window
  -> arrivals spread smoothly; no waves
  -> this is the recommended default

DECORRELATED JITTER
  sleep = min(cap, random(base, previous_sleep * 3))
  spreads well and recovers faster than full jitter when
  the dependency comes back
```

1. **Classify before retrying** — 4xx-class errors, validation failures and permanent rejections must not be retried. Retrying them consumes budget, delays the alert, and teaches the team that retries are meaningless.
2. **Only retry idempotent operations, or supply an idempotency key** — Retrying after an ambiguous timeout is exactly the case where the first attempt may have succeeded.
3. **Use full jitter as the default** — Randomising the whole delay window, not a small offset, is what prevents synchronisation.
4. **Enforce a retry budget** — Cap retries at a proportion of successful requests — commonly around ten per cent. When exceeded, stop retrying entirely rather than continuing to amplify.
5. **Retry at one layer, not every layer** — Nested retries multiply. Decide which layer owns retrying — usually the one closest to the caller with enough context — and make the others fail fast.
6. **Honour `Retry-After`** — When a server tells you when to come back, guessing instead is both rude and counterproductive.

**Why nested retries are so dangerous**

```text
client: 3 attempts
  -> gateway: 3 attempts
    -> service A: 3 attempts
      -> service B

ONE user action, worst case:
  3 x 3 x 3 = 27 requests to service B

DURING AN INCIDENT
  B is failing -> every layer retries -> 27x normal load
  B cannot recover because the load never drops
  -> the retry logic prevents the recovery it exists to enable

THE FIX
  choose ONE layer to retry - usually the outermost that
  has enough context and a deadline
  every other layer fails fast and propagates
  + a retry BUDGET at each layer as a backstop:
    "retries may not exceed 10% of successful requests"
    -> during a sustained failure, retrying simply stops
```

> **A retry after a timeout may duplicate work that succeeded**  
> A timeout means the response was lost, not that the operation failed. Retrying a payment, an email or a stateful update after a timeout can perform it twice. This is why retry policy and idempotency are inseparable: a system that retries without idempotency keys has chosen duplicate side effects over lost ones, usually without realising it made a choice.

**Worked example**

A checkout flow calling a payment provider, designing the retry policy end to end.

**Policy, layer by layer**

```text
FAILURE CLASSIFICATION
  retry:      connection error, timeout, 502/503/504,
              429 (honour Retry-After)
  do not:     400 invalid card, 402 declined,
              409 already captured, 422 validation
  -> a declined card retried five times is five declines
     and a five-fold slower failure for the user

WHERE RETRIES HAPPEN
  browser:          NO retry (user-visible; would duplicate)
  checkout service: YES - it owns the deadline and the
                    idempotency key
  payment client:   NO - fail fast, propagate
  -> exactly one layer retries

POLICY AT THE CHECKOUT SERVICE
  max attempts:  3
  backoff:       full jitter, base 200ms, cap 5s
  total deadline: 10s   <- bounds ALL attempts together
  idempotency:   the same key on every attempt, so a
                 retry after an ambiguous timeout returns
                 the original result instead of charging again
  budget:        retries capped at 10% of successful calls;
                 above that, fail immediately

CIRCUIT BREAKER
  if the failure rate stays high, open the circuit and
  stop calling entirely for a period
  -> retries handle blips; the breaker handles outages

WHAT THE USER SEES
  transient blip: a slightly slower success
  sustained outage: a fast, clear failure - not a 30s hang
```

| Metric | Value | Note |
|---|---|---|
| Attempts | 3 | one layer only |
| Jitter | full | no synchronised waves |
| Deadline | 10 s total | **bounds all attempts** |
| Budget | 10% | stops amplification |

> **A total deadline is what bounds the user's experience**  
> Three attempts with a ten-second timeout each means a thirty-second wait before failure — which is worse for the user than failing immediately. A single deadline covering all attempts means retries happen only if there is time, and the user gets an answer within a bounded period whether it succeeds or not. Per-attempt timeouts without a total deadline are the most common way retry logic degrades user experience while trying to improve it.

**When to use it**

- **Transient failures**: timeouts, connection resets, 503s, rate limits, brief dependency unavailability.
- **Idempotent operations**, or non-idempotent ones protected by an idempotency key.
- **Background and asynchronous work**, where a delayed success is far better than a failure.
- **With a circuit breaker**, so sustained failures stop generating load rather than being retried indefinitely.
- **At exactly one layer** of a call chain, chosen deliberately.

**When to avoid it**

- **Do not retry permanent failures** — validation errors, declines, malformed input. They will fail identically.
- **Do not retry without jitter**, which produces synchronised waves against a recovering service.
- **Do not retry at every layer**; the multiplication turns a degradation into an outage.
- **Do not retry non-idempotent operations** without an idempotency key.
- **Do not use per-attempt timeouts without a total deadline**, which multiplies the user's wait.

**Advantages**

- **Transient failures become invisible**, which is most failures in a distributed system.
- **Exponential backoff reduces pressure** on a struggling dependency over time.
- **Jitter prevents synchronisation**, which is the difference between smooth recovery and repeated collapse.
- **Budgets bound the worst case**, so a sustained failure cannot produce unbounded load.
- **Cheap to implement** relative to the availability it buys.

**Disadvantages**

- **Amplifies load during failures**, exactly when capacity is scarcest.
- **Multiplies across layers**, often unintentionally, because each layer's policy looks reasonable alone.
- **Can duplicate side effects** without idempotency.
- **Increases user-visible latency** when attempts are serialised without a total deadline.
- **Masks genuine problems**, since a dependency failing ten per cent of the time looks healthy behind retries.

**Trade-offs**

**Retry policy dimensions**

| Dimension | Too little | Too much |
|---|---|---|
| Attempts | Transient failures surface to users | Amplification; long user waits |
| Backoff | Constant pressure on a failing dependency | Slow recovery after it returns |
| Jitter | Synchronised waves | None — full jitter is nearly always right |
| Budget | Unbounded amplification | Transient failures not absorbed |
| Layers retrying | Failures propagate unnecessarily | Multiplicative load |

> **The framing that shows judgement**  
> “One retry layer, at the service that owns the deadline and the idempotency key. Three attempts with full jitter, all bounded by a single ten-second total deadline so the user never waits thirty seconds for a failure. Retries capped at ten per cent of successful calls, and a circuit breaker for sustained failure — retries handle blips, the breaker handles outages.”

**How it fails**

**Retry-related failures**

| Failure | Cause | Fix |
|---|---|---|
| Brief degradation becomes an outage | Uncapped retries amplifying load | Retry budget; circuit breaker |
| Recovery repeatedly knocked down | Synchronised retry waves | Full jitter |
| One user action generates 27 requests | Every layer retrying | Retry at one layer; others fail fast |
| Duplicate charges or emails | Retrying non-idempotent operations after a timeout | Idempotency keys on every attempt |
| Users wait 30 seconds to see a failure | Per-attempt timeouts with no total deadline | Single deadline covering all attempts |
| Declines retried five times | No failure classification | Retry only transient error classes |
| A failing dependency looks healthy | Retries masking the underlying error rate | Monitor pre-retry failure rate separately |

> **Retries hide the failure rate they are absorbing**  
> A dependency failing five per cent of the time looks perfectly healthy from the caller's dashboards, because retries absorb it — until load rises and the absorbed failures become visible all at once. Always measure the failure rate *before* retries as a separate metric, or you will be unaware of a degradation until it exceeds what retries can hide.

**Limits**

> **Working parameters**
>
> - **Full jitter**: `sleep = random(0, min(cap, base × 2^attempt))` — randomise the whole window, not an offset.
> - **Attempts**: 3–5 for interactive paths; more for background work where latency is flexible.
> - **Retry budget**: around 10% of successful requests is a common cap; beyond it, stop retrying.
> - **Total deadline** must bound all attempts together, not each one separately.
> - **Nested retries multiply**: three layers of three attempts is twenty-seven requests for one action.

**Alternatives**

| Mechanism | Handles | Limitation |
|---|---|---|
| Retry with backoff and jitter | Transient failures | Amplifies load; needs idempotency |
| Circuit breaker | Sustained failure | Does not help with brief blips |
| Hedged requests | Tail latency | Extra load; idempotent only |
| Fallback / degraded response | Dependency unavailable | Requires a sensible default |
| Queue and retry asynchronously | Work that need not be synchronous | Latency; a queue to operate |
| Fail fast | When retrying cannot help | Failure is user-visible |

Retries and circuit breakers are complements rather than alternatives: retries absorb brief blips that would otherwise be visible, while the breaker detects that failures have stopped being transient and cuts off the load entirely. A system with only retries keeps hammering a dead dependency; one with only a breaker fails users on momentary glitches.

**In real systems**

- **AWS's exponential backoff and jitter analysis** established full jitter as the recommended default and is widely cited for demonstrating how much better it is than naive backoff.
- **gRPC and Envoy retry budgets** cap retries as a fraction of traffic, which is the mechanism that bounds amplification during sustained failure.
- **Retry storms** are a documented cause of major outages, where each layer's individually reasonable policy compounded into a self-sustaining load multiplier.
- **`Retry-After` on 429 and 503** is the cooperative mechanism that lets servers coordinate client behaviour instead of being guessed at.
- **Service meshes** centralise retry policy so it can be configured per route rather than reimplemented in every client, which also makes nested retries visible.

**Common mistakes**

- **Retrying permanent failures**, wasting budget and delaying the alert.
- **No jitter**, producing synchronised waves that repeatedly knock down a recovering service.
- **Retrying at every layer**, multiplying load during an incident.
- **No idempotency key**, so retries after timeouts duplicate side effects.
- **Per-attempt timeouts with no total deadline**, making users wait for the sum.
- **No retry budget**, allowing unbounded amplification.
- **Measuring only post-retry success**, hiding the real failure rate.

**The staff-level view**

Retry policy is where individually sensible decisions compose into a system-level hazard, which makes it a platform concern rather than a per-service one.

- **Decide which layer retries and make the others fail fast.** Nested retries multiply, and each layer's policy looks reasonable in isolation, so this cannot be left to individual teams.
- **Make full jitter and a retry budget the shared client defaults.** Both are easy to omit and both are the difference between graceful degradation and amplification.
- **Require a total deadline covering all attempts.** Per-attempt timeouts are the most common way retries make user experience worse.
- **Pair retries with a circuit breaker**, since retries are for blips and breakers are for outages, and a system with only one of them fails in a predictable way.
- **Monitor the pre-retry failure rate separately.** Retries hide degradation until it exceeds what they can absorb, which is precisely when you would have wanted the warning.

**Go deeper**

Retrying is the right response to transient failure and the most reliable way to turn a brief degradation into a sustained outage. The determining factors are what you retry, how you wait, and how much retrying you permit in total. Permanent failures — validation errors, declines, malformed input — must not be retried, since they fail identically while consuming budget and delaying the alert.

Exponential backoff reduces the rate from one client; jitter prevents many clients from synchronising. Without it, everyone who failed together backs off together and retries together, producing waves that repeatedly knock down a recovering service. Full jitter — a random delay anywhere between zero and the exponential bound — is the recommended default and is markedly better than a small random offset.

Two structural rules matter most. Retry at exactly one layer, because nested retries multiply: three layers of three attempts is twenty-seven requests for one user action, arriving precisely when capacity is scarcest. And bound all attempts with a single total deadline rather than per-attempt timeouts, or three ten-second attempts make the user wait thirty seconds for a failure. Add a retry budget capping retries at around ten per cent of successful traffic, pair with a circuit breaker for sustained failure, and always retry with an idempotency key.

Retry policy is the clearest example of a system-level hazard composed entirely from locally reasonable decisions, which is why it belongs in shared infrastructure rather than in each service.

**Three conditions must hold before retrying.** The failure must be transient — a timeout, connection reset, 503 or rate limit, not a validation error or a decline, which will fail identically while consuming the budget and delaying the alert. The operation must be safe to repeat, either naturally idempotent or protected by an idempotency key, because a timeout means the response was lost rather than the operation failing, and the first attempt may well have succeeded. And the system must be able to afford the extra load, which during an incident is exactly when it cannot.

**Jitter matters more than backoff.** Exponential backoff reduces the rate at which a single client retries. Jitter prevents many clients from doing so simultaneously. Ten thousand clients failing together and following the same exponential schedule retry at the same instants, producing waves that can knock a recovering dependency back down repeatedly. Full jitter — sleeping a random duration anywhere between zero and the exponential bound — spreads arrivals smoothly, and the published analysis showing how much better it is than naive backoff or small offsets is one of the more actionable results in this area.

**Nesting is multiplicative and invisible.** A client retrying three times, calling a gateway retrying three times, calling a service retrying three times, produces twenty-seven requests to the bottom dependency for a single user action. Each policy is individually defensible; the composition is not. This cannot be resolved by individual teams because no team can see the whole chain, so the platform must designate which layer retries — normally the one that owns the deadline and the idempotency key — and require the others to fail fast and propagate.

**Budgets and deadlines bound the worst case.** A retry budget capping retries at a proportion of successful requests, typically around ten per cent, means that during a sustained failure retrying simply stops rather than continuing to amplify. A single total deadline covering all attempts means the user waits a bounded time regardless of outcome — per-attempt timeouts are the most common way retry logic degrades the experience it was meant to protect, turning a three-second failure into a thirty-second one. Deadlines also propagate naturally, letting each hop decline work it cannot finish.

**Retries and circuit breakers are complements.** Retries absorb blips that would otherwise be user-visible; breakers detect that failures have stopped being transient and cut off load entirely. A system with only retries hammers a dead dependency indefinitely; one with only a breaker fails users on momentary glitches. Both are needed, and their thresholds should be set together rather than independently.

**Retries hide the degradation they absorb.** A dependency failing five per cent of the time appears perfectly healthy from the caller's dashboards, because retries mask it — until load rises and the absorbed failures surface all at once. Exporting the pre-retry failure rate as a distinct metric is what converts this from an invisible accumulating risk into an ordinary signal, and it is routinely omitted because the post-retry number looks reassuring.

**Prove it — interview questions**

1. **[Basic] Why is jitter necessary if you already have exponential backoff?**

   <details><summary>Model answer</summary>

   Because backoff reduces the rate from one client while jitter prevents many clients from synchronising. If ten thousand clients fail at the same moment and all follow the same exponential schedule, they all retry at the same instants, producing waves that hit a recovering service hard enough to knock it down again. Full jitter — choosing a random delay anywhere between zero and the exponential bound — spreads arrivals smoothly and is markedly better than adding a small random offset to a fixed schedule.

   </details>

2. **[Basic] What should you not retry?**

   <details><summary>Model answer</summary>

   Anything that will fail identically: validation errors, malformed input, authorisation failures, declined payments, references to deleted resources. Retrying them consumes the retry budget, delays the moment someone learns about the real problem, and multiplies the user's wait for a failure that was certain. Retries are for transient conditions — timeouts, connection resets, 503s, rate limits — where the same request may succeed later.

   </details>

3. **[Senior] Why are nested retries dangerous?**

   <details><summary>Model answer</summary>

   Because they multiply. A client retrying three times, calling a gateway that retries three times, calling a service that retries three times produces twenty-seven requests to the bottom dependency for one user action. During an incident, that means the failing service receives many times its normal load precisely when it has least capacity, which prevents the recovery the retries were meant to enable. Each layer's policy looks perfectly reasonable in isolation, which is why this has to be decided centrally: one layer owns retrying, and every other layer fails fast and propagates.

   </details>

4. **[Senior] Why does a total deadline matter more than a per-attempt timeout?**

   <details><summary>Model answer</summary>

   Because the user experiences the sum. Three attempts at ten seconds each means a thirty-second wait before seeing a failure, which is worse than failing immediately — the retries made the experience worse while trying to improve it. A single deadline covering all attempts means retries only happen if time remains, and the user gets an answer within a bounded period regardless of outcome. It also propagates naturally: each hop knows how much budget is left and declines to start work it cannot finish.

   </details>

5. **[Staff] Design a retry policy for a checkout calling a payment provider.**

   <details><summary>Model answer</summary>

   First, classification: retry connection errors, timeouts and 5xx, and honour `Retry-After` on 429; never retry declines, invalid cards or validation errors, because those fail identically and retrying just makes the user wait longer for the same answer. Second, placement: exactly one layer retries — the checkout service, because it owns both the deadline and the idempotency key — while the payment client and the browser fail fast. Third, policy: three attempts with full jitter, all bounded by a single ten-second total deadline so the user never waits thirty seconds, with the same idempotency key on every attempt so a retry after an ambiguous timeout returns the original result rather than charging again. Fourth, a retry budget capping retries at around ten per cent of successful calls, and a circuit breaker so that sustained failure stops generating load entirely — retries handle blips, the breaker handles outages.

   </details>

6. **[Principal] Why should retry policy be a platform concern rather than left to teams?**

   <details><summary>Model answer</summary>

   Because the hazard is emergent rather than local. Every team makes a reasonable decision — retry three times with backoff — and the composition produces a multiplicative load amplifier that appears only during incidents, which is exactly when it is most harmful and least diagnosable. No individual team can see the nesting, and no individual policy is wrong. So the platform needs to own three things: which layer retries for a given call path, full jitter and retry budgets as non-optional client defaults, and a total deadline propagated through the chain so attempts are bounded collectively. I would also insist that pre-retry failure rates are exported as a standard metric, because retries systematically hide degradation — a dependency failing five per cent of the time looks perfectly healthy until load rises and the absorbed failures surface all at once, and by then the warning you wanted has been invisible for weeks.

   </details>

---

### Backpressure

*Let a slow consumer tell a fast producer to slow down, so overload is bounded and visible instead of accumulating in a buffer until something bursts.*

**Flow:** `Producer` → `Bounded buffer` → `Demand signal` → `Consumer` → `Capacity feedback`

> **The 30-second version**  
> Let the consumer's capacity govern the producer's rate. Bound every buffer, decide what happens when it fills, and make sure the signal reaches whoever can actually slow down.

**The problem**

A producer generates work faster than a consumer can process it. The difference has to go somewhere: into memory, onto disk, into a queue. Unbounded, it accumulates until the process runs out of memory, the disk fills, or latency grows until every result is useless — and all three failures arrive suddenly, after a long period of apparent health.

Backpressure inverts the flow of control: instead of the producer pushing as fast as it can, the consumer signals how much it can accept. Overload then manifests as a producer slowing down or rejecting work, which is bounded, visible and recoverable — rather than as an invisible accumulation that ends in a crash.

> **Buffers do not solve overload, they delay and disguise it**  
> Adding a buffer converts an immediate throughput problem into a delayed memory problem. If the producer is persistently faster than the consumer, no buffer size is sufficient — it only changes how long the system looks healthy before failing. Buffers absorb *bursts*, which is valuable; they cannot absorb a sustained rate mismatch, and treating them as if they can is the central mistake.

**Mental model**

Think of it as a demand signal travelling backwards against the flow of data. The consumer says how much it wants; the producer sends no more than that. The buffer between them is small and bounded, existing to smooth jitter rather than to store a backlog.

1. **Bounded buffer** — A queue with a hard maximum. Reaching the maximum is the signal, not a failure.
2. **Blocking** — The producer waits when the buffer is full. Simple, and propagates naturally up a synchronous chain.
3. **Dropping** — The producer discards work when the buffer is full. Correct when data is disposable — telemetry, sensor samples.
4. **Rejecting** — The producer refuses new work, returning an error. Correct for request-response, where the caller can retry or degrade.
5. **Demand signalling** — The consumer explicitly requests N items; the producer sends at most N. This is what reactive streams formalise.

> **Unbounded queues are the most common form of this bug**  
> An unbounded queue looks like resilience and behaves like a delay-action failure. It absorbs load gracefully right up to memory exhaustion, at which point everything buffered is lost at once — the maximum possible damage, during an incident that was already in progress. Every queue, channel and buffer in a system needs an explicit bound and a defined behaviour on reaching it.

**How it works**

**What happens when the buffer fills: four options**

```text
BLOCK          producer waits for space
  + no loss; pressure propagates upstream naturally
  - can deadlock if the chain is circular
  - a blocked producer may hold resources (threads, locks)
  use: synchronous pipelines, in-process channels

DROP NEWEST    discard the incoming item
  + bounded memory; consumer keeps working on older data
  - newest data lost, which is usually the most valuable
  use: rarely

DROP OLDEST    evict the head to make room
  + bounded memory; consumer always sees recent data
  - historical data lost
  use: live telemetry, dashboards, presence

REJECT         return an error to the caller
  + caller learns immediately and can retry or degrade
  - caller must handle rejection
  use: request-response APIs, the usual correct answer

THE CHOICE IS A PRODUCT DECISION, not a technical default.
What is the cost of losing this item versus delaying it?
```

1. **Bound every buffer, queue and channel** — Including ones you did not create explicitly: thread pool queues, connection pool wait queues, socket buffers, library-internal channels.
2. **Propagate pressure to the true source** — Blocking a worker is useless if the producer is an external client. The signal must reach whoever can actually slow down — often via rejection at the API boundary.
3. **Prefer rejecting fast to accepting slowly** — A request that waits thirty seconds and then fails has consumed resources and delivered nothing. A request rejected in one millisecond lets the caller retry elsewhere or degrade.
4. **Use concurrency limits as the practical mechanism** — Capping in-flight work is backpressure expressed as a number, and it is easier to reason about than buffer sizes because it maps onto Little's law.
5. **Distinguish bursts from sustained mismatch** — Size buffers for expected burst duration. If the producer is persistently faster, no buffer helps and the fix is capacity or shedding.
6. **Make the pressure observable** — Buffer occupancy, rejection rate and wait time should be first-class metrics, because backpressure that is working looks like errors and needs to be interpretable.

**Why pressure must reach the actual source**

```text
SYSTEM  HTTP API -> queue -> workers -> database

DATABASE SLOWS DOWN
  workers block on the database
  queue fills                      <- pressure reaches here
  API keeps accepting requests and enqueuing
  -> the queue is unbounded, or bounded and now rejecting
  -> but the CLIENT never learned anything until now

WITH PRESSURE PROPAGATED
  workers slow -> queue reaches its bound
  API sees the queue full -> returns 503 + Retry-After
  client backs off                 <- the SOURCE slows
  -> system operates at the database's actual capacity
  -> latency stays bounded; nothing accumulates

THE PRINCIPLE
  backpressure only works if the signal reaches something
  that can genuinely reduce its rate. Blocking an internal
  worker while an external client keeps pushing just moves
  the accumulation somewhere else.
```

> **Backpressure and buffering serve opposite purposes**  
> Buffering smooths variance so that a brief burst does not cause rejection. Backpressure limits sustained rate so that a persistent mismatch does not cause accumulation. A system needs both, sized differently: a small buffer for jitter, and a firm bound whose breach triggers the pressure signal. Large buffers are not more backpressure — they are less.

**Worked example**

An ingestion pipeline receiving telemetry faster than it can be written, and how the design changes with backpressure.

**Before and after, with the numbers**

```text
SYSTEM  HTTP ingest -> in-memory queue -> batch writer -> store

WITHOUT BACKPRESSURE
  ingest accepts 50,000 events/s
  writer sustains 30,000 events/s
  queue grows at 20,000/s
  each event ~500 B -> 10 MB/s of growth
  with 8 GB of headroom: ~13 minutes of "healthy"
  then: OOM, process restart, ENTIRE QUEUE LOST
  -> 13 minutes of accepted data gone, during an incident

WITH BACKPRESSURE
  queue bounded at 200,000 events (~100 MB, ~6s of buffer)
  on full:
    ingest returns 429 + Retry-After
    clients back off and retry
  result:
    accepted rate settles at the writer's real capacity
    buffer absorbs bursts up to 6 seconds
    sustained excess is REJECTED, not accumulated
    nothing is ever lost silently

WHAT THE CLIENT SEES
  without: everything succeeds, then 13 minutes vanish
  with:    some requests rejected with clear guidance,
           and nothing that was accepted is lost

THE SECOND IS OBVIOUSLY BETTER, AND REQUIRES ONLY A BOUND
AND A DEFINED BEHAVIOUR ON REACHING IT.
```

| Metric | Value | Note |
|---|---|---|
| Rate mismatch | 20,000/s | unsustainable |
| Unbounded | 13 min then total loss | **worst case** |
| Bounded | 6 s of burst | then reject |
| Client sees | 429 + Retry-After | actionable |

> **The comparison that makes the case**  
> Without backpressure, every request succeeds for thirteen minutes and then thirteen minutes of accepted data disappears. With it, some requests are rejected with clear guidance and nothing accepted is ever lost. Stakeholders sometimes resist rejection because it looks like failure — but the alternative is not success, it is a delayed and much larger failure that also destroys data the system promised to keep.

**When to use it**

- **Any producer-consumer relationship** where rates can differ — which is all of them.
- **Streaming and ingestion pipelines**, where sustained mismatch is the normal failure mode.
- **Request-response services**, where rejecting fast is far better than queueing slowly.
- **In-process channels and thread pools**, which are buffers people forget to bound.
- **Anywhere a queue exists**, since an unbounded queue is a deferred outage.

**When to avoid it**

- **Do not use unbounded buffers**, ever. They convert overload into total loss.
- **Do not block in a chain that can be circular**, which deadlocks.
- **Do not size buffers to absorb sustained mismatch**; they cannot, and trying delays the failure rather than preventing it.
- **Do not apply pressure only internally** when the source is an external client that keeps pushing.
- **Do not drop silently** — a dropped item must be counted, or the loss is invisible.

**Advantages**

- **Bounded memory and latency** under any load, rather than graceful-looking accumulation.
- **Failure becomes visible and early** — rejections appear immediately rather than as a later crash.
- **The system operates at the real capacity of its slowest stage**, which is the honest throughput.
- **Callers can react**, retrying elsewhere, degrading, or shedding their own load.
- **No catastrophic loss**, since nothing accepted is discarded by a buffer overflowing.

**Disadvantages**

- **Rejections are user-visible**, and stakeholders often prefer the appearance of success.
- **Blocking can deadlock** in circular dependencies and can hold scarce resources.
- **Requires bounding everything**, including buffers inside libraries you do not control.
- **Choosing the overflow behaviour is a product decision** that engineering cannot make alone.
- **Correct behaviour looks like errors**, so monitoring must distinguish healthy shedding from real failure.

**Trade-offs**

**Overflow behaviour by data type**

| Data | Cost of loss | Cost of delay | Correct behaviour |
|---|---|---|---|
| Payment request | Very high | Moderate | Reject with a clear error; client retries |
| Live dashboard metric | Low | High (stale is useless) | Drop oldest |
| Audit log entry | Very high | Low | Block, or persist to durable storage |
| Sensor sample | Low | High | Drop; count the drops |
| User-submitted job | High | Low | Reject at submission; do not accept and lose |

The rows differ entirely in which cost dominates. Getting this wrong in either direction is damaging: dropping payment requests loses money, while blocking on dashboard metrics makes the dashboard both stale and a source of pressure on the system it monitors.

**How it fails**

**Backpressure failures**

| Failure | Cause | Fix |
|---|---|---|
| Out of memory during an incident | Unbounded buffer absorbing a sustained mismatch | Bound every buffer; define overflow behaviour |
| All buffered work lost at once | Buffer overflowed or process restarted | Bound and reject early; durable buffering if loss is unacceptable |
| Latency grows until results are useless | Queue absorbing rather than signalling | Bound the queue; alert on age, not depth |
| Deadlock | Circular blocking between stages | Avoid blocking in cycles; use bounded rejection instead |
| Pressure does not reach the client | Signal stops at an internal boundary | Propagate to the API edge; return 429/503 |
| Silent data loss | Dropping without counting | Count and alert on every drop |
| Healthy dashboards, growing backlog | Monitoring throughput rather than buffer occupancy and age | Expose occupancy, wait time and rejection rate |

**Limits**

> **Sizing guidance**
>
> - **Buffer size = expected burst duration × rate mismatch.** A six-second burst at 20,000 events/s is 120,000 items.
> - **No buffer absorbs a sustained mismatch** — if the producer is persistently faster, the only fixes are capacity or shedding.
> - **Concurrency limits** are backpressure expressed as in-flight count, and size from Little's law.
> - **Acquisition and wait timeouts** should be short, so pressure manifests as fast rejection rather than slow acceptance.
> - **Every drop must be counted**, or loss is invisible and unmeasurable.

**Alternatives**

| Mechanism | Effect | Best for |
|---|---|---|
| Bounded buffer + reject | Fast, visible shedding | Request-response APIs |
| Bounded buffer + block | Pressure propagates upstream | Synchronous in-process pipelines |
| Drop oldest | Freshness preserved | Live telemetry and dashboards |
| Concurrency limit | Bounds in-flight work | Service-to-service calls |
| Durable queue | Absorbs much larger bursts | Work that must not be lost; adds latency |
| Autoscaling consumers | Raises capacity | Slow to react; bounded by downstream |

A durable queue is the honest way to absorb a genuinely large burst: it trades memory for disk and latency for durability, and it can hold hours rather than seconds. But it is still bounded, still needs a defined behaviour when full, and still cannot absorb a sustained mismatch — it simply moves the deadline further out.

**In real systems**

- **TCP flow control** is backpressure in the transport: the receiver advertises a window and the sender may not exceed it, which is why TCP does not overwhelm slow receivers.
- **Reactive Streams and its implementations** formalise demand signalling, where the consumer requests N items and the producer emits at most N.
- **Kafka consumers pull rather than being pushed**, which makes backpressure inherent — a slow consumer simply fetches less often.
- **Envoy and service meshes** implement concurrency limits and outstanding-request caps, which is backpressure applied to service-to-service calls.
- **Thread pool queue bounds** in application servers are the most commonly misconfigured instance, with unbounded defaults that turn overload into memory exhaustion.

**Common mistakes**

- **Unbounded queues**, converting overload into memory exhaustion and total loss.
- **Sizing buffers to absorb a sustained mismatch**, which only delays the failure.
- **Blocking internally** while the external producer continues unaffected.
- **Dropping without counting**, making loss invisible.
- **Long acquisition timeouts**, turning pressure into a stall rather than a rejection.
- **Blocking in a cycle**, producing deadlock.
- **Treating rejections as failures to eliminate** rather than as the system working correctly.

**The staff-level view**

Backpressure is unglamorous and its absence is the single most common cause of overload turning into total failure.

- **Audit for unbounded buffers, including ones you did not write.** Thread pool queues, channel constructors, client library internals and connection pool wait queues all have defaults that are frequently unbounded.
- **Make the overflow behaviour an explicit, documented decision per data type**, since it is a product judgement about the relative cost of loss and delay, not a technical default.
- **Ensure the pressure signal reaches something that can slow down.** Internal blocking while an external client keeps pushing simply relocates the accumulation.
- **Prefer fast rejection with `Retry-After` to slow acceptance.** A request that waits and then fails consumed capacity and delivered nothing.
- **Instrument occupancy, wait time and rejection rate.** Working backpressure looks like errors, so it must be interpretable or teams will disable it during an incident.

**Go deeper**

When a producer is faster than a consumer, the difference accumulates somewhere. Unbounded, it grows until memory is exhausted or latency makes results useless — and both failures arrive suddenly after a long period of apparent health. Backpressure inverts control so the consumer's capacity governs the flow, making overload bounded and visible rather than deferred and catastrophic.

The key insight is that buffers absorb bursts, not sustained mismatch. If the producer is persistently faster, no buffer size helps — a larger one simply produces a later and bigger failure, since everything buffered is lost when it overflows or the process restarts. So buffers should be sized for expected burst duration, with a hard bound whose breach triggers the signal, and the behaviour on reaching it — block, drop oldest, drop newest, or reject — chosen per data type according to whether loss or delay costs more.

Two failures recur. The signal must reach something that can genuinely slow down: blocking internal workers while an external client keeps pushing merely relocates the accumulation, so pressure must propagate to the API edge as a 429 or 503 with `Retry-After`. And working backpressure looks like errors, so occupancy, wait time and rejection rate must be instrumented — otherwise teams disable the bounds during an incident and convert a bounded degradation into total loss.

Backpressure is the mechanism that keeps overload bounded, and its absence is the most common reason a degradation becomes a total failure.

**Buffers and backpressure solve different problems.** A buffer absorbs variance so that a brief burst does not cause rejection. Backpressure limits sustained rate so that a persistent mismatch does not cause accumulation. Conflating them produces the characteristic mistake of enlarging a buffer in response to overload, which changes nothing about the underlying rate difference and merely extends the period of apparent health before a larger failure. If a producer is persistently faster than its consumer, the queue grows at the difference regardless of its size.

**Unbounded queues are delayed-action failures.** They absorb load gracefully right up to memory exhaustion, at which point everything buffered is lost simultaneously — the maximum possible damage, occurring during an incident already in progress, and destroying work the system had already acknowledged. Every buffer, channel, thread pool queue, connection pool wait queue and library-internal queue needs an explicit bound, and many of these have unbounded defaults in widely used frameworks.

**The overflow behaviour is a product decision.** Blocking propagates pressure naturally up a synchronous chain but can deadlock in cycles and can hold scarce resources such as threads. Dropping the oldest preserves freshness and suits live telemetry and dashboards. Dropping the newest is rarely right, since the newest item is usually the most valuable. Rejecting is the usual correct answer for request-response, because the caller learns immediately and can retry elsewhere or degrade. The determining question is the relative cost of losing the item versus delaying it, which differs completely between a payment request, a dashboard metric and an audit entry — and engineering cannot decide it unilaterally.

**The signal must reach a source that can slow down.** If a database degrades, workers block and the internal queue fills, but an HTTP API that keeps accepting and enqueuing has merely moved the accumulation. Backpressure works only when the pressure propagates to something capable of reducing its rate — in practice, to the edge, where the service returns a 429 or 503 with `Retry-After` and external clients back off. Internal blocking alone relocates the problem, usually somewhere less observable.

**Prefer fast rejection to slow acceptance.** A request that waits thirty seconds and then fails has consumed capacity, held a connection, occupied a thread, and delivered nothing — while also denying the caller the chance to try an alternative. Short acquisition and wait timeouts turn pressure into a prompt, actionable rejection. Concurrency limits express the same idea as a number rather than a buffer size, and are easier to reason about because they map directly onto Little's law.

**The organisational difficulty is that working backpressure looks like failure.** Rejections show as errors on dashboards and as “the system isn't working” in stakeholder conversations, while an unbounded queue produces a flawless success rate until the moment it does not. The instinct during an incident is therefore to raise limits, which enlarges the eventual failure. Countering this requires instrumenting occupancy, wait time and rejection rate so shedding is interpretable, making bounded buffers a platform default so teams inherit correct behaviour, and settling the overflow policy per data type as a documented decision in advance — so the argument happens in design review rather than at three in the morning.

**Prove it — interview questions**

1. **[Basic] What is backpressure?**

   <details><summary>Model answer</summary>

   A mechanism by which a slow consumer signals a fast producer to reduce its rate, so the difference between them does not accumulate. Instead of the producer pushing as fast as it can and the excess piling up in a buffer, the consumer's capacity determines the flow. Overload then appears as the producer slowing or rejecting work — bounded and visible — rather than as an invisible accumulation that ends in memory exhaustion.

   </details>

2. **[Basic] Why doesn't a bigger buffer solve overload?**

   <details><summary>Model answer</summary>

   Because a buffer absorbs bursts, not sustained rate mismatch. If the producer is persistently faster than the consumer, the buffer fills at the difference between them regardless of size, so doubling it merely doubles how long the system looks healthy before failing. Worse, when it does fail, everything buffered is lost at once — so the larger buffer produces a later and bigger failure. Buffers should be sized for expected burst duration, with a firm bound whose breach triggers the pressure signal.

   </details>

3. **[Senior] What should happen when a bounded buffer is full?**

   <details><summary>Model answer</summary>

   It depends entirely on the relative cost of losing the item versus delaying it, which is a product decision rather than a technical default. A payment request should be rejected with a clear error so the client can retry — losing it is unacceptable, delaying it is tolerable. A live dashboard metric should drop the oldest item, because stale data is useless and freshness matters more than completeness. An audit entry should block or spill to durable storage, because losing it is unacceptable and delay is not. Whatever the choice, drops must be counted, or the loss is invisible.

   </details>

4. **[Senior] Why must the pressure signal reach the original producer?**

   <details><summary>Model answer</summary>

   Because only something that can genuinely reduce its rate can relieve the pressure. If a database slows, workers block, and the queue fills — but the HTTP API keeps accepting requests and enqueuing them, the accumulation has simply moved. The signal has to propagate to the edge, where the service returns a 429 or 503 with `Retry-After`, so the external client backs off. Blocking an internal stage while an external producer continues at full rate relocates the problem rather than solving it.

   </details>

5. **[Staff] Walk through the consequences of running an ingestion pipeline without backpressure.**

   <details><summary>Model answer</summary>

   Suppose ingest accepts fifty thousand events per second while the writer sustains thirty thousand. The queue grows at twenty thousand per second, roughly ten megabytes per second at typical event sizes, so with eight gigabytes of headroom the system looks completely healthy for about thirteen minutes. Then it runs out of memory, the process restarts, and everything buffered is lost — thirteen minutes of data the system had already told clients it accepted, discarded during an incident that was already in progress. With a bounded queue of a few seconds' capacity and a 429 on overflow, the accepted rate settles at the writer's genuine capacity, bursts are absorbed, sustained excess is rejected with guidance, and nothing accepted is ever lost. The second outcome is obviously better and requires only a bound and a defined overflow behaviour.

   </details>

6. **[Principal] Why do teams resist backpressure, and how do you address that?**

   <details><summary>Model answer</summary>

   Because working backpressure looks like failure. Rejections appear on dashboards as errors and in stakeholder conversations as the system not working, whereas an unbounded queue produces a perfect success rate right up until it does not. The instinct during an incident is therefore to raise limits or remove bounds, which makes the eventual failure larger. The way to address it is to reframe the comparison honestly: the alternative to rejecting some requests is not accepting all of them, it is accepting them and then losing them, at a worse moment, along with everything else in flight. Concretely, I would make bounded buffers and defined overflow behaviour a platform default so teams inherit them, export occupancy and rejection rate as standard metrics so shedding is interpretable rather than alarming, and make the overflow policy a documented product decision per data type — because once someone has explicitly agreed that dropping sensor samples is fine and dropping payments is not, the incident-time argument has already been had.

   </details>

---

### Batching and microbatching

*Group many small operations into one, amortising fixed per-operation cost across the group — and pay for it in latency that must be explicitly bounded.*

**Flow:** `Incoming items` → `Size or timer threshold` → `Batch execution` → `Item outcomes`

> **The 30-second version**  
> Group operations to amortise fixed per-call cost, flushing on whichever of a size or time threshold fires first — and demand per-item results so one bad item cannot block the rest.

**The problem**

Writing one row costs a round trip, a transaction, a write-ahead log flush and an index update. Writing a thousand rows individually costs a thousand of each. But writing a thousand rows in one statement costs one round trip, one transaction, and one flush — the per-row work remains, while the per-operation overhead is divided by a thousand.

That amortisation is where nearly all the gain lives, and it applies everywhere there is fixed cost per call: database writes, API requests, message publishes, disk flushes, cache lookups. The cost is that the first item in a batch waits for the rest, which is latency added deliberately.

> **Batching is latency traded for throughput, always**  
> Every batching decision adds waiting to buy amortisation, so the only question is how much waiting is acceptable. That makes the maximum wait a requirement rather than a tuning parameter — and a batch that fills only on size, with no time bound, can stall indefinitely when traffic drops.

**Mental model**

Collect items until a threshold, then process them together. Two thresholds are needed: a size cap so batches do not grow unboundedly, and a time cap so a partially filled batch still departs.

1. **Size trigger** — Flush when N items have accumulated. Bounds batch size, memory and failure granularity.
2. **Time trigger** — Flush after T milliseconds regardless of size. Bounds the latency added to the first item.
3. **Whichever fires first** — Both triggers together. Under load, size dominates; under light traffic, time dominates.
4. **Per-item outcomes** — A batch operation must report success or failure per item, not just for the batch, or a single bad item fails nine hundred and ninety-nine good ones.
5. **Microbatching** — Batches measured in milliseconds rather than minutes — the streaming variant, giving near-real-time latency with batch efficiency.

> **Batch failure granularity is the detail teams underestimate**  
> If a batch of a thousand fails as a unit, one malformed item means retrying nine hundred and ninety-nine good ones — and if the malformed item is deterministic, the batch fails forever. Batch APIs must report per-item results, and the retry path must be able to split a failing batch and isolate the offending items rather than retrying the whole thing.

**How it works**

**Where the gain comes from, quantified**

```text
SINGLE-ROW INSERTS
  per row: network round trip 0.5ms
           transaction begin/commit
           WAL flush (fsync)
  1,000 rows -> 1,000 round trips, 1,000 commits
  ~ 1,000 x 1ms = 1 second

BATCHED INSERT (1,000 rows, one statement)
  1 round trip, 1 transaction, 1 flush
  per-row work (parse, index, write) remains
  ~ 20-50ms total
  -> 20-50x improvement

WHAT DID NOT CHANGE
  the per-row cost. Batching amortises FIXED cost,
  not variable cost.

SO THE GAIN DEPENDS ENTIRELY ON THE RATIO
  high fixed cost per call (network, fsync, transaction)
    -> batching is transformative
  low fixed cost, high per-item cost (heavy computation)
    -> batching changes little
```

1. **Set both a size cap and a time cap** — Size alone stalls under light traffic; time alone allows unbounded batches during a burst. Whichever fires first is the correct rule.
2. **Derive the time cap from the latency budget** — If the pipeline may add fifty milliseconds, the linger time must be well under that. This is a requirement, not a knob to tune for throughput.
3. **Require per-item results** — A batch API returning a single success or failure forces all-or-nothing retry, which is unworkable at scale.
4. **Split on failure** — When a batch fails, retry in halves to isolate the offending item rather than retrying the whole batch or discarding it.
5. **Bound memory, not just count** — A thousand small items and a thousand large ones are very different. Cap by total bytes as well as count.
6. **Measure the fill distribution** — If batches consistently fill by time rather than size, the size cap is irrelevant; if always by size, latency may be higher than intended. The distribution tells you which threshold is actually binding.

**Choosing the thresholds from requirements**

```text
START FROM THE LATENCY BUDGET, NOT THE THROUGHPUT TARGET

  "this pipeline may add at most 100ms"
    -> linger time <= 50ms, leaving headroom for the
       batch operation itself

THEN SIZE THE BATCH FROM WHAT THE DOWNSTREAM HANDLES
  a 10,000-row insert may:
    hold locks too long
    exceed statement size limits
    spike memory
    make failure granularity unusable
  -> typically 100-1,000 rows, measured not assumed

THEN CHECK WHICH THRESHOLD BINDS
  at 5,000 items/s with a 50ms linger:
    250 items accumulate per window
    if the size cap is 1,000 -> TIME always fires
    -> latency is 50ms, batches are 250
    if the size cap is 100 -> SIZE always fires
    -> latency is 20ms, batches are 100

THE BINDING THRESHOLD TELLS YOU WHICH REGIME YOU ARE IN,
and it changes with traffic. Measure it.
```

> **Adaptive batching handles variable load better than fixed thresholds**  
> A fixed linger time is too slow at high volume and too aggressive at low volume. Adaptive schemes flush as soon as the previous batch completes — so under light load batches are small and latency is minimal, while under heavy load batches naturally grow because more items accumulate during the previous operation. This self-tunes without any threshold at all.

**Worked example**

An event pipeline writing to a database, worked through from requirement to configuration.

**From latency budget to observed behaviour**

```text
REQUIREMENT
  50,000 events/s peak
  end-to-end latency budget: 200ms
  database sustains ~2,000 individual inserts/s
  -> unbatched, this is 25x over capacity

BATCHING CONFIGURATION
  linger:       50ms      (within the 200ms budget)
  max size:     500 rows
  max bytes:    1 MB
  flush on whichever fires first

OBSERVED BEHAVIOUR
  at 50,000/s: 2,500 events accumulate in 50ms
               -> SIZE fires at 500, every 10ms
               -> 100 batches/s of 500 rows
               -> latency added: ~10ms
  at 500/s:    25 events accumulate in 50ms
               -> TIME fires at 50ms with 25 rows
               -> latency added: 50ms
  -> the system self-adjusts; both regimes are acceptable

DATABASE LOAD
  100 batched inserts/s of 500 rows
  vs 50,000 individual inserts/s
  -> 500x fewer statements, well within capacity

FAILURE HANDLING
  batch insert returns per-row results
  on a constraint violation: retry the batch in halves
  to isolate the bad rows, dead-letter those, commit
  the rest
  -> one malformed event does not block 499 valid ones
```

| Metric | Value | Note |
|---|---|---|
| Unbatched | 50,000 inserts/s | 25× over capacity |
| Batched | 100 statements/s | **500× fewer** |
| Latency added | 10–50 ms | within budget |
| Failure | per-row results | isolate, not discard |

> **Measure which threshold is binding, because it changes with load**  
> At high volume the size cap fires and latency is low; at low volume the time cap fires and latency is at its maximum. That means the latency you observe in a load test is not the latency users experience at three in the morning. Monitoring the fill distribution — what fraction of batches flush on size versus time, and their sizes — tells you which regime you are in and whether either threshold needs changing.

**When to use it**

- **High fixed cost per operation**: database writes, network calls, disk flushes, message publishes.
- **High volume of small items**, where per-call overhead dominates per-item work.
- **Protecting a downstream with limited operation capacity**, by reducing statement or request count rather than data volume.
- **Streaming pipelines**, where microbatching gives near-real-time latency with batch efficiency.
- **Bulk APIs**, where a client naturally has many items to send at once.

**When to avoid it**

- **Do not batch on a synchronous user-facing path** without a strict time bound; the first request waits for the batch.
- **Do not batch when per-item cost dominates**, since there is little fixed overhead to amortise.
- **Do not use a size trigger alone**, which stalls indefinitely when traffic drops.
- **Do not batch operations whose failures are coupled**, unless per-item results and splitting are available.
- **Do not batch across tenants or security contexts**, which complicates isolation and error attribution.

**Advantages**

- **Amortises fixed cost**, often producing order-of-magnitude throughput improvements.
- **Reduces load on the downstream in operation count**, which is frequently the binding constraint rather than data volume.
- **Improves compression and locality**, since batched writes are contiguous and similar.
- **Self-adjusts with dual thresholds**, giving low latency at low volume and high efficiency at high volume.
- **Reduces connection and transaction overhead**, which is often invisible but substantial.

**Disadvantages**

- **Adds latency by construction**, which must be explicitly bounded.
- **Coarsens failure granularity** unless per-item results and splitting are supported.
- **Increases memory usage**, since items are held until flush.
- **Loss window**: items buffered in memory are lost on a crash unless the buffer is durable.
- **Larger operations hold resources longer**, such as locks or transaction duration.

**Trade-offs**

**Threshold behaviour**

| Configuration | Latency | Efficiency | Risk |
|---|---|---|---|
| Size only | Unbounded at low volume | Maximum | Stalls when traffic stops |
| Time only | Bounded | Unbounded batch size at peak | Memory spikes; long operations |
| Size and time | Bounded by the time cap | Good in both regimes | Two parameters to choose |
| Adaptive (flush on completion) | Self-minimising | Self-maximising | Less predictable; harder to reason about |

> **Framing the configuration**  
> “I'd set the linger from the latency budget — fifty milliseconds within a two-hundred-millisecond allowance — and the size cap from what the database handles well, around five hundred rows. At peak the size cap fires every ten milliseconds so latency is low; at low volume the time cap fires and latency is fifty milliseconds, which is still within budget. And I'd require per-row results so one bad row doesn't fail the batch.”

**How it fails**

**Batching failures**

| Failure | Cause | Fix |
|---|---|---|
| Items stuck indefinitely | Size trigger with no time trigger and traffic stopped | Always include a time cap |
| One bad item fails the whole batch forever | All-or-nothing batch semantics | Per-item results; split-on-failure retry |
| Memory spike under load | Time trigger with no size or byte cap | Cap both count and bytes |
| User-facing latency regression | Batching added to a synchronous path | Bound the linger tightly, or do not batch there |
| Data lost on crash | In-memory buffer with no durability | Durable buffer, or accept and document the loss window |
| Locks held too long | Batch size too large for the transaction | Smaller batches; measure lock duration |
| Latency worse than load tests suggested | Time threshold binds at low volume, size at high | Measure the fill distribution across traffic levels |

**Limits**

> **Sizing guidance**
>
> - **Gain ≈ fixed cost per operation ÷ per-item cost.** High round-trip or fsync cost means large gains; heavy per-item computation means small ones.
> - **Linger time** must be well inside the latency budget — typically a quarter to a half of it.
> - **Batch size**: usually 100–1,000 for database writes, limited by lock duration, statement limits and failure granularity.
> - **Cap bytes as well as count**, since item sizes vary widely.
> - **Monitor the size-versus-time fill ratio** to know which threshold is binding at current traffic.

**Alternatives**

| Approach | Gains | Costs |
|---|---|---|
| Batching | Amortised fixed cost | Latency; failure granularity |
| Pipelining | Overlaps round trips without waiting | Requires protocol support; no amortisation of commits |
| Connection pooling | Removes connection setup cost | Does not reduce operation count |
| Async fire-and-forget | No waiting at all | No confirmation; loss risk |
| Bulk load APIs | Maximum throughput for large volumes | Not for incremental writes |
| Increase downstream capacity | No latency added | Cost; may not be possible |

Pipelining is worth distinguishing: it sends multiple requests without waiting for each response, which hides round-trip latency but does not amortise per-operation costs like transaction commits or fsyncs. Where the fixed cost is the network, pipelining suffices; where it is the commit, only batching helps.

**In real systems**

- **Kafka producers** expose `linger.ms` and `batch.size` as exactly this pair of thresholds, making the latency-throughput trade an explicit configuration.
- **Database group commit** batches transaction commits so that one fsync serves many transactions, which is what makes synchronous durability affordable at volume.
- **Spark Streaming's microbatching** applies batch processing at second-scale intervals, trading a little latency for the efficiency of batch execution.
- **GraphQL DataLoader** batches per-field lookups within one request, which is the same amortisation applied to data access rather than to writes.
- **Bulk APIs** in most SaaS products exist because per-item request overhead would otherwise dominate for clients with large datasets.

**Common mistakes**

- **Size trigger with no time trigger**, stalling items indefinitely when traffic stops.
- **Time trigger with no size or byte cap**, producing memory spikes and very long operations at peak.
- **All-or-nothing batch semantics**, so one bad item fails the rest forever.
- **Batching on a synchronous path** without a tight latency bound.
- **In-memory buffering** of data that must not be lost, with no durability and no stated loss window.
- **Batches large enough to hold locks too long**, converting a throughput gain into a contention problem.
- **Tuning for throughput** and quietly exceeding the latency budget.

**The staff-level view**

Batching is a straightforward win that goes wrong in two predictable ways: unbounded latency and unusable failure granularity.

- **Derive the linger time from the latency budget and treat it as a requirement**, not a throughput tuning parameter that can be raised when someone wants more efficiency.
- **Require per-item results in every batch API you build.** All-or-nothing semantics make retry unworkable at scale and guarantee that one poison item eventually blocks a pipeline.
- **Insist on both thresholds.** Size alone stalls when traffic drops, which is exactly when nobody is watching; time alone allows memory spikes at peak.
- **Ask whether the buffer needs durability.** An in-memory batch is a loss window, and whether that is acceptable is a product decision rather than an implementation detail.
- **Measure the fill distribution**, because the binding threshold changes with load and the latency observed in a load test is not what users experience at low traffic.

**Go deeper**

Batching amortises fixed per-operation cost: a thousand single-row inserts pay a thousand round trips, transactions and log flushes, while one thousand-row insert pays each once. The per-row work is unchanged, so the gain depends entirely on the ratio of fixed to variable cost — transformative where round trips or fsyncs dominate, negligible where per-item computation does.

Two thresholds are required. A size cap bounds batch size, memory and failure granularity; a time cap bounds the latency added to the first item. Size alone stalls indefinitely when traffic drops; time alone produces memory spikes and very long operations at peak. Flushing on whichever fires first means the system self-adjusts, with the size cap binding under load and the time cap binding at low volume — which also means the latency seen in a load test is not the latency at quiet hours.

The detail teams underestimate is failure granularity. A batch that fails as a unit makes one malformed item fail nine hundred and ninety-nine good ones, and if that item fails deterministically the batch fails forever, blocking everything behind it. Batch operations need per-item results and a retry path that splits a failing batch to isolate offenders. Derive the linger time from the latency budget rather than treating it as a throughput knob.

Batching is one of the highest-return optimisations available, and its failure modes are entirely predictable: unbounded latency at low traffic, and unusable failure granularity at scale.

**The gain is amortisation of fixed cost.** A single-row insert pays a network round trip, a transaction begin and commit, and a write-ahead log flush; batching a thousand rows pays each of those once while the per-row parsing, indexing and writing remains. So the improvement is governed by the ratio of fixed to variable cost — twenty to fifty times where round trips and fsyncs dominate, and close to nothing where per-item work is heavy computation. That ratio should be estimated before assuming batching will help.

**Both thresholds are necessary and they bind in different regimes.** A size trigger alone means a partially filled batch waits indefinitely once traffic drops, which typically happens overnight when nobody is watching, and items sit unprocessed for hours. A time trigger alone means that at peak an unbounded number of items accumulate within the window, producing memory spikes and operations long enough to hold locks and exhaust statement limits. With both, the size cap binds under load and the time cap binds at low volume, so the system self-adjusts — but it also means the latency measured in a load test is the best case, not the typical one, and the fill distribution must be monitored to know which regime is current.

**Failure granularity determines whether batching survives contact with real data.** All-or-nothing semantics mean one malformed item fails the entire batch, and a deterministically failing item fails it forever — stalling the pipeline behind it indefinitely. Batch operations must therefore report per-item outcomes, and the retry path must split a failing batch, typically by halving, to isolate offenders so they can be dead-lettered while valid items commit. This is straightforward to design in and painful to retrofit, because by the time you need it a poison record is already blocking production.

**Derive the linger from the latency budget.** Treating it as a throughput tuning parameter quietly consumes the budget, and because the time threshold only binds at low traffic the regression does not appear under load testing. A pipeline permitted two hundred milliseconds end to end should linger for a fraction of that, leaving headroom for the batch operation and downstream stages. Batch size should come from what the downstream handles well — lock duration, statement limits, memory, failure granularity — rather than from a theoretical maximum, with a byte cap alongside the count cap since item sizes vary widely.

**The buffer is a loss window.** Items held in memory awaiting flush are lost if the process dies, which for telemetry is fine and for accepted user work is not. Whether that window is acceptable is a product decision, and if it is not, the buffer must be durable — which reintroduces some of the fixed cost batching was avoiding, though typically far less than per-item durability would.

**Adaptive batching handles variable load better than fixed thresholds.** Flushing as soon as the previous batch completes means batches are small and latency minimal when traffic is light, while under load more items naturally accumulate during the previous operation and batches grow — self-tuning with no threshold at all. The cost is less predictable behaviour, which makes it harder to reason about and to explain, so fixed thresholds remain the common choice despite being worse at both extremes.

**Prove it — interview questions**

1. **[Basic] Where does the gain from batching come from?**

   <details><summary>Model answer</summary>

   From amortising fixed per-operation cost. A single-row insert pays a network round trip, a transaction, and a write-ahead log flush; a thousand-row insert pays those once while the per-row work remains unchanged. So the improvement depends on the ratio of fixed to variable cost — where round trips or fsyncs dominate it can be twenty to fifty times, while for heavy per-item computation there is little overhead to amortise and batching changes almost nothing.

   </details>

2. **[Basic] Why do you need both a size and a time threshold?**

   <details><summary>Model answer</summary>

   Because each alone fails in a different regime. A size threshold with no time bound means that when traffic drops, a partially filled batch waits indefinitely — items can sit unprocessed for hours, and this typically happens overnight when nobody is watching. A time threshold with no size cap means that at peak, a huge number of items accumulate within the window, producing memory spikes and very long operations. Flushing on whichever fires first gives bounded latency at low volume and bounded size at high volume.

   </details>

3. **[Senior] Why is failure granularity the detail that matters most?**

   <details><summary>Model answer</summary>

   Because a batch that fails as a unit makes retry unworkable. One malformed item means retrying nine hundred and ninety-nine valid ones, and if the malformed item fails deterministically the batch fails forever — blocking the pipeline behind it. Batch operations therefore need per-item results, and the retry path needs to split a failing batch, typically in halves, to isolate the offending items so they can be dead-lettered while the rest commit. Designing this in from the start is far easier than discovering it when one poison record stalls ingestion.

   </details>

4. **[Senior] How do you choose the linger time?**

   <details><summary>Model answer</summary>

   From the latency budget, not from a throughput target. If the pipeline is allowed to add two hundred milliseconds end to end, the linger should be a fraction of that — perhaps fifty milliseconds — leaving headroom for the batch operation itself and for downstream stages. Treating it as a tuning knob to be raised for efficiency quietly consumes the latency budget, and because the time threshold only binds at low traffic, the regression is invisible in load testing and appears when volume drops.

   </details>

5. **[Staff] How would you configure batching for a 50,000 events per second pipeline?**

   <details><summary>Model answer</summary>

   First the linger, from the latency budget: fifty milliseconds within a two-hundred-millisecond allowance. Then the batch size from what the database handles well — around five hundred rows, bounded by lock duration, statement limits and failure granularity rather than by a theoretical maximum — plus a byte cap, since item sizes vary. At peak, fifty thousand events per second means five hundred items accumulate in ten milliseconds, so the size cap fires and latency is low; at five hundred events per second the time cap fires with twenty-five items and latency is fifty milliseconds, still within budget. That self-adjustment is the point of having both thresholds. I would then require per-row results with split-on-failure retry, and monitor the fill distribution so I know which threshold is binding at any given traffic level.

   </details>

6. **[Principal] When is batching the wrong answer to a throughput problem?**

   <details><summary>Model answer</summary>

   When per-item cost dominates, because there is little fixed overhead to amortise — batching a thousand expensive computations saves one round trip and nothing else. It is also wrong on genuinely synchronous user-facing paths where the added wait is not affordable, and in those cases pipelining is often the better tool: it overlaps round trips without making anyone wait for a batch to fill, though it does not amortise commits or flushes. The subtler wrong case is using batching to avoid confronting a capacity shortfall. If a downstream is persistently unable to keep up, batching buys a multiple and then the same problem returns at higher volume, while the added latency and coarser failure granularity are permanent. I would want to know whether batching moves the system comfortably inside capacity or merely inside it temporarily, because the second is a deferral rather than a fix.

   </details>

---

### Durable workflow orchestration

*Record every step's outcome so a multi-step process can resume exactly where it stopped, surviving crashes, deploys and week-long waits.*

**Flow:** `Workflow history` → `Scheduler` → `Activity` → `Recorded outcome` → `Resume`

> **The 30-second version**  
> Record each step's outcome durably so the process can be replayed and resumed exactly where it stopped — making crashes, deploys and week-long waits non-events.

**The problem**

An order fulfilment process charges a card, reserves inventory, books a courier, sends a confirmation and, three days later, requests a review. Implemented as a sequence of function calls, a crash after the charge and before the reservation leaves the system in a state no code path anticipates — money taken, nothing reserved, and no record of where it stopped.

The usual response is a chain of queues with a message per step, which works but scatters the process across many handlers with no single place showing what state it is in. And it cannot express a three-day wait, a timeout, a compensating rollback, or a human approval without building each of those mechanisms by hand.

> **The insight is to make the history the program**  
> Rather than the process being a running function that can be lost, the process is a durable record of every step attempted and its outcome. Execution becomes replaying that history to determine what to do next. A crash therefore loses nothing — the history is the state, and resuming means reading it.

**Mental model**

Workflow code looks like ordinary sequential code but is executed differently: each external call is an activity whose result is recorded, and after any interruption the code is re-executed from the beginning, with recorded results returned instantly instead of re-invoking the activities.

1. **Workflow** — The orchestration logic. Must be deterministic, because it is replayed.
2. **Activity** — A single unit of work with side effects — a charge, an API call, a database write. Its result is durably recorded.
3. **History** — The append-only log of activity invocations and results. This is the true state of the process.
4. **Replay** — After a crash or a scheduling gap, the workflow is re-run from the start; recorded activities return their stored results without executing again.
5. **Timer** — A durable sleep. Waiting three days costs nothing but a row, and survives restarts and deploys.

> **Workflow code must be deterministic, and this is the main trap**  
> Because the workflow is replayed, anything non-deterministic diverges from the recorded history: calling `now()`, generating a random value or a UUID, reading a mutable global, or iterating a map with unstable ordering. All of these must go through the framework's deterministic equivalents or be moved into activities. The failure mode is subtle — a workflow that behaves correctly until it is replayed after a crash, then takes a different path.

**How it works**

**How replay produces resumption**

```text
WORKFLOW CODE (looks sequential)
  charge  = await chargeCard(order)
  reserve = await reserveInventory(order)
  courier = await bookCourier(order)
  await sleep(3 days)
  await requestReview(order)

FIRST EXECUTION
  chargeCard invoked      -> result recorded in history
  reserveInventory        -> result recorded
  CRASH

RECOVERY (workflow re-executed from the top)
  chargeCard      -> found in history, returns instantly
                     NOT re-invoked, so no double charge
  reserveInventory-> found in history, returns instantly
  bookCourier     -> NOT in history -> actually invoked
  ...continues from exactly where it stopped

THE SLEEP
  durable timer; the process is not running during it
  three days later the workflow is rehydrated and continues
  deploys, restarts and failovers in between are irrelevant
```

1. **Keep side effects in activities, never in workflow code** — Workflow code runs many times; activities run once and are recorded. Anything with an effect must be an activity.
2. **Make activities idempotent anyway** — An activity can be invoked, succeed, and crash before its result is recorded — so it will be retried. The framework guarantees at-least-once activity execution, not exactly-once.
3. **Use durable timers rather than sleeping a process** — Waiting for days, waiting for a human, or waiting for a deadline costs a database row rather than a held thread or a scheduled job.
4. **Model compensation explicitly** — A failure at step four requires undoing steps one to three. Workflow code can express that directly as a sequence of compensating activities, which is a saga with the bookkeeping handled for you.
5. **Version workflow code carefully** — A workflow started under one version may replay under a newer one. Changing the sequence of activities breaks replay, so frameworks provide versioning primitives that must actually be used.
6. **Keep histories bounded** — Long-running or high-volume workflows accumulate history. Continue-as-new, or archiving completed workflows, prevents unbounded growth.

**What this replaces, and why it is better**

```text
HAND-ROLLED: QUEUE PER STEP
  charge -> queue -> reserve -> queue -> book -> queue -> ...
  you must build:
    state tracking across steps
    correlation between messages and the process
    timeout handling per step
    compensation when a later step fails
    the three-day wait (a scheduler + a durable timer table)
    visibility ("where is order 4711?")
    retry with backoff per step
  each of these is a component to build, test and operate

DURABLE WORKFLOW
  the same logic as sequential code
  state, correlation, timers, retries and history are the
  framework's job
  visibility comes free: the history IS the answer to
  "where is order 4711 and what happened?"

WHAT YOU PAY
  a workflow engine to run (or a managed service)
  determinism constraints on workflow code
  a versioning discipline
  a new operational model for the team to learn
```

> **Replay makes debugging different, not harder**  
> A workflow that fails after a crash may behave differently from its first execution if determinism was violated, and the stack trace shows the replay rather than the original run. The compensating advantage is that the history is a complete, queryable record of every step and outcome — so the question “what happened to this order?” has an exact answer, which a queue-chain architecture cannot provide at all.

**Worked example**

Order fulfilment with compensation and a human approval step.

**A workflow that would be a project to hand-build**

```text
workflow fulfilOrder(order):
  payment = await activity.chargeCard(order)          // retried
  try:
    reservation = await activity.reserveInventory(order)
  catch OutOfStock:
    await activity.refund(payment)                    // compensate
    await activity.notifyCustomer(order, "out of stock")
    return Cancelled

  if order.total > 10_000:
    approval = await waitForSignal("manager_approval",
                                    timeout = 48 hours)
    if approval is timeout or rejected:
      await activity.releaseInventory(reservation)
      await activity.refund(payment)
      return Cancelled

  courier = await activity.bookCourier(order)
  await activity.sendConfirmation(order)
  await sleep(3 days)
  await activity.requestReview(order)
  return Completed

WHAT IS HANDLED FOR YOU
  each activity retried with backoff, recorded once
  a crash anywhere resumes at the next unrecorded activity
  the 48-hour human wait holds no resources
  the 3-day sleep survives every deploy in between
  "where is order 4711?" is answered by the history
  compensation is ordinary code, not a bespoke saga engine

WHAT YOU MUST STILL DO
  make each activity idempotent (at-least-once execution)
  keep the workflow deterministic
  version the code when the activity sequence changes
```

| Metric | Value | Note |
|---|---|---|
| Crash recovery | automatic | replay from history |
| Human wait | 48 h | no resources held |
| Long sleep | 3 days | **survives deploys** |
| Visibility | the history | exact answer |

> **The visibility is worth as much as the durability**  
> Teams adopt workflow engines for crash recovery and keep them for observability. A queue-chain architecture cannot answer “where is this order and what has happened to it?” without correlating messages across several systems and hoping the logs are still there. A workflow history answers it exactly, including every retry, every failure reason and every wait — which changes support and incident response more than it changes the happy path.

**When to use it**

- **Multi-step processes with side effects** where partial completion is unacceptable — fulfilment, onboarding, provisioning, payouts.
- **Long-running processes** measured in hours, days or weeks, where holding a process or scheduling jobs is impractical.
- **Processes requiring compensation**, where a late failure must undo earlier steps.
- **Human-in-the-loop steps**, such as approvals with timeouts.
- **Where visibility matters** — processes that support will be asked about.

**When to avoid it**

- **Do not use it for simple single-step async work**, where a queue and a job record are sufficient and far lighter.
- **Do not use it for high-volume, low-value events**, since per-workflow overhead is significant compared to a queue message.
- **Do not put side effects in workflow code**, which is replayed and will repeat them.
- **Do not use non-deterministic constructs** in workflow code without the framework's equivalents.
- **Do not adopt it without committing to versioning discipline**, since replay under changed code is the main operational hazard.

**Advantages**

- **Crash recovery is automatic**, resuming at exactly the step that had not completed.
- **Long waits cost nothing** — days or weeks of waiting are a durable timer, not a held resource.
- **Compensation is ordinary code**, rather than a bespoke saga framework.
- **Complete visibility** of every step, retry, failure and wait, which transforms support and debugging.
- **Complex control flow is expressible** — branching, parallel steps, signals, timeouts — as normal code rather than as message choreography.

**Disadvantages**

- **Determinism constraints on workflow code** are unfamiliar and violations fail subtly, only on replay.
- **Versioning is a permanent discipline**, since in-flight workflows replay under newer code.
- **A workflow engine to operate**, or a dependency on a managed service.
- **Per-workflow overhead** makes it unsuitable for very high volume, low-value events.
- **History growth** requires continue-as-new or archiving for long-running workflows.
- **A new mental model** that the whole team must learn before it can be operated confidently.

**Trade-offs**

**Orchestration approaches**

| Approach | State | Long waits | Visibility | Complexity |
|---|---|---|---|---|
| Sequential function calls | In memory — lost on crash | Impossible | None | Lowest |
| Queue per step (choreography) | Scattered across messages | Needs a scheduler | Poor; requires correlation | Deceptively high |
| State machine in a database | Explicit, queryable | Needs a scheduler | Good | Moderate; hand-built |
| Durable workflow engine | History, durable | Native | Excellent | Engine to operate |

The second row is the one that misleads: a queue per step looks simple and cheap, and then the team builds state tracking, correlation, timers, timeouts, compensation and visibility one at a time over a year — arriving at a worse version of the fourth row without having decided to.

**How it fails**

**Workflow failure modes**

| Failure | Cause | Fix |
|---|---|---|
| Duplicate side effects | Activity invoked, succeeded, crashed before recording | Idempotent activities; at-least-once is the guarantee |
| Workflow takes a different path after a crash | Non-determinism in workflow code | Use framework time and random; move logic into activities |
| In-flight workflows break on deploy | Activity sequence changed without versioning | Versioning primitives; keep old paths until drained |
| Side effect repeated on every replay | Side effect in workflow code rather than an activity | Move all effects into activities |
| History grows unbounded | Long-running or looping workflow | Continue-as-new; archive completed workflows |
| Workflow stuck waiting forever | Signal never arrives and no timeout set | Always pair waits with timeouts and a timeout path |
| Engine becomes a bottleneck | Very high volume of short workflows | Use a queue instead for high-volume simple work |

**Limits**

> **Practical considerations**
>
> - **Activity execution is at-least-once**, so idempotency is required regardless of the engine's guarantees about workflow state.
> - **Workflow code must be deterministic** — no wall clock, randomness, or unordered iteration outside framework primitives.
> - **Durable timers** make waits of days or weeks essentially free, unlike scheduled jobs or held processes.
> - **History size** bounds how long a single workflow can run; continue-as-new resets it.
> - **Per-workflow overhead** is far higher than a queue message, which sets the lower bound on sensible use.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Durable workflow engine | Multi-step, long-running, compensating processes | Engine; determinism; versioning |
| Queue chain | Simple linear pipelines | State and visibility must be hand-built |
| Database state machine + scheduler | Moderate complexity, existing infrastructure | You build timers, retries, compensation |
| Saga library | Compensation without full orchestration | Narrower; still needs state |
| Single transaction | Steps within one database | Only works inside one system |
| Synchronous orchestration | Fast, few steps, failure tolerable | Crash loses everything |

A database state machine plus a scheduler is the honest middle ground and is what most teams build incrementally. It works well for a handful of workflows and becomes a maintenance burden once every team needs their own — which is the point at which adopting an engine stops being over-engineering.

**In real systems**

- **Temporal and its predecessor Cadence** popularised the replay-based model, where workflow code is ordinary code executed deterministically against a recorded history.
- **AWS Step Functions** takes a declarative approach with a state machine definition rather than code, trading expressiveness for a managed service and visual tooling.
- **Azure Durable Functions** implements the same replay model in a serverless context, showing that the pattern is independent of the hosting style.
- **Order fulfilment, KYC onboarding and provisioning pipelines** are the archetypal use cases, since all involve multiple side-effecting steps, long waits and compensation.
- **Uber's Cadence origins** came from exactly the problem described here: dozens of teams independently building state machines, timers and compensation on top of queues.

**Common mistakes**

- **Side effects in workflow code**, repeated on every replay.
- **Non-deterministic constructs** — wall clock, randomness, map iteration — diverging on replay.
- **Assuming activities execute exactly once**, so they are not made idempotent.
- **Changing the activity sequence** without using versioning primitives, breaking in-flight workflows.
- **Waits with no timeout**, leaving workflows stuck indefinitely.
- **Unbounded history** in long-running or looping workflows.
- **Using an engine for high-volume trivial work**, where a queue is far cheaper.

**The staff-level view**

The decision is less about capability than about whether you want to build the same five mechanisms repeatedly across teams.

- **Recognise the incremental trap.** A queue per step looks cheap and then accumulates state tracking, correlation, timers, compensation and visibility over a year — arriving at a worse workflow engine without a decision ever being made.
- **Treat determinism and versioning as the real adoption cost**, not the engine itself. Both fail subtly and only on replay, so they need explicit team education and review rules.
- **Insist activities remain idempotent.** The engine guarantees workflow state, not exactly-once activity execution, and teams routinely assume otherwise.
- **Weigh the visibility benefit explicitly.** For processes that support and operations are asked about constantly, the queryable history is often worth more than the durability.
- **Set a volume floor.** Per-workflow overhead makes engines unsuitable for high-volume, low-value events, and using one there is a genuine mistake rather than a conservative choice.

**Go deeper**

A durable workflow makes the history the state. Every activity invocation and result is recorded, and after any interruption the workflow code is re-executed from the top with recorded activities returning their stored results instantly instead of running again. Execution fast-forwards to the first unrecorded step and continues, so a crash mid-process loses nothing and a three-day wait is a durable timer that survives every deploy in between.

This replaces a set of mechanisms teams otherwise build individually: state tracking across steps, correlation between messages and processes, per-step retry and timeout, durable timers, compensation when a late step fails, and visibility into where a given process is. Compensation becomes ordinary code — catch the failure, call the undo activities — rather than a bespoke saga engine, and the history answers “what happened to order 4711?” exactly.

The costs are two disciplines that fail subtly. Workflow code must be deterministic, because it is replayed — wall clocks, randomness and unordered iteration must go through framework equivalents or move into activities. And versioning matters, because in-flight workflows replay under newer code, so changing the activity sequence breaks processes already running. Activities also remain at-least-once, so they must be idempotent regardless of what the engine guarantees about workflow state.

Durable workflow orchestration solves a specific and common problem: a multi-step process with side effects, where a crash between steps leaves the system in a state nobody planned for and nothing records where it stopped.

**The core idea is that the history is the program's state.** Rather than a running function that can be lost, the process is an append-only record of activities attempted and their outcomes. Recovery re-executes the workflow code from the beginning, but activities already in the history return their recorded results without being invoked, so execution fast-forwards to the first incomplete step. The consequence is that crashes, deploys, failovers and multi-day waits are all the same thing — a gap between replays — and none of them requires special handling.

**Determinism is the price and the main hazard.** Because the code is replayed, any non-determinism causes it to diverge from the recorded history: calling the wall clock, generating a UUID, reading mutable global state, or iterating a map with unstable ordering. Frameworks provide deterministic substitutes for time and randomness, and anything genuinely non-deterministic must live in an activity whose result is recorded. The failure mode is unusually unpleasant: the workflow behaves correctly until a replay occurs, which may be weeks later and in production, so ordinary testing does not surface it.

**Activities are at-least-once, not exactly-once.** An activity can be invoked, complete its external side effect, and crash before its result reaches the history — after which the replay finds nothing recorded and invokes it again. The engine guarantees durability of workflow state, not exactly-once execution of effects on the outside world. So charges, emails and external API calls must be idempotent, typically by passing the workflow and activity identifiers as an idempotency key. Teams routinely assume the engine covers this and it does not.

**What it replaces is a pile of hand-built mechanisms.** A queue per step looks cheap and then accumulates, over a year, state tracking, message-to-process correlation, per-step timeout and retry, a scheduler and timer table for long waits, compensation logic when a late step fails, and some way of answering where a given process is. Each is built separately, tested separately and operated separately, and the result is a worse workflow engine that nobody decided to build. The signal that the balance has tipped is a team about to build a durable timer table for a multi-day wait in a process that already has hand-rolled state tracking.

**Visibility is often the larger benefit.** Crash recovery is why teams adopt these engines; the queryable history is why they keep them. A queue-chain architecture cannot answer “where is this order, what has happened to it, and why did it fail?” without correlating messages across systems and hoping logs are retained. A workflow history answers exactly, including every retry, failure reason and wait — which changes support and incident response far more than it changes the happy path.

**Versioning is the permanent discipline.** In-flight workflows replay under whatever code is deployed, so changing the sequence of activities breaks processes that started days earlier. Frameworks provide versioning primitives, but they only help if the team has a convention for using them, agreed before the second workflow exists. Together with determinism, this is the real adoption cost — two rules that produce no test failures and must therefore be enforced by review and education rather than by tooling.

**Prove it — interview questions**

1. **[Basic] How does a durable workflow survive a crash?**

   <details><summary>Model answer</summary>

   Because the history, not the running process, is the state. Every activity invocation and its result are durably recorded. After a crash the workflow code is re-executed from the beginning, but activities already present in the history return their stored results immediately rather than being invoked again — so execution fast-forwards to exactly the point where it stopped and continues from there. Nothing is lost because nothing important was ever only in memory.

   </details>

2. **[Basic] Why must workflow code be deterministic?**

   <details><summary>Model answer</summary>

   Because it is replayed. If the code calls the wall clock, generates a random value, or iterates a collection with unstable ordering, the replay can take a different path from the original execution — and then the recorded history no longer matches what the code is doing. The failure is subtle: the workflow behaves perfectly until it is replayed after a crash, at which point it diverges. Frameworks provide deterministic equivalents for time and randomness, and anything genuinely non-deterministic belongs inside an activity, whose result is recorded.

   </details>

3. **[Senior] Why do activities still need to be idempotent?**

   <details><summary>Model answer</summary>

   Because the engine guarantees at-least-once activity execution, not exactly-once. An activity can be invoked, complete its side effect, and then crash before its result is written to the history — in which case the replay finds nothing recorded and invokes it again. The workflow state is durable; the activity's effect on the outside world is not covered by that. So a charge, an email or an external API call must be idempotent, typically by passing the workflow and activity id as an idempotency key so the remote system deduplicates.

   </details>

4. **[Senior] What does a durable timer give you that a scheduled job does not?**

   <details><summary>Model answer</summary>

   It makes waiting free and reliable. A three-day wait is a durable timer entry, so no process is held, no thread blocked, and deploys, restarts and failovers during that period are irrelevant — the workflow is simply rehydrated when the timer fires. A scheduled job approach requires a separate scheduler, a correlation mechanism to find the right process state, and handling for what happens if the job fires while the system is mid-deploy. The workflow model collapses all of that into one line of ordinary-looking code.

   </details>

5. **[Staff] When is a workflow engine over-engineering, and when is it under-engineering not to have one?**

   <details><summary>Model answer</summary>

   It is over-engineering for high-volume, low-value events and for single-step asynchronous work, where per-workflow overhead is far higher than a queue message and the benefits do not apply. It is under-engineering to avoid it once several teams each have multi-step processes with side effects, long waits and compensation — because each will independently build state tracking, correlation, durable timers, per-step retry, compensation and visibility, arriving at a worse engine incrementally without anyone deciding to. The specific signal I look for is a team about to build a timer table and a scheduler to handle a multi-day wait in a process that already has hand-rolled state tracking; that is the point where the accumulated cost has exceeded the adoption cost.

   </details>

6. **[Principal] What is the real cost of adopting a workflow engine?**

   <details><summary>Model answer</summary>

   Not the engine — it is the two disciplines that fail subtly. Determinism, because violations behave perfectly until a replay occurs, which may be weeks after the code was written and in production rather than in testing; and versioning, because in-flight workflows replay under newer code, so changing the sequence of activities breaks processes that started days earlier. Both need explicit team education and review rules rather than documentation, since neither produces a failure a test will catch. Beyond that there is a genuine mental model shift: engineers must internalise that workflow code runs many times and activity code runs once, which is unlike anything else they write. I would plan adoption around those costs specifically — a pilot team, explicit review criteria for determinism, and a versioning convention agreed before the second workflow exists — rather than around the engine's operational footprint, which is usually the easier part.

   </details>

---

### Sagas and compensation

*When a transaction cannot span services, commit each step locally and undo the earlier ones with compensating actions if a later step fails.*

**Flow:** `Local commit A` → `Local commit B` → `Failure` → `Compensation B` → `Compensation A`

> **The 30-second version**  
> Commit each step locally and undo completed steps with compensating business actions when a later one fails — with no isolation, so intermediate states are visible and must be designed for.

**The problem**

Booking a trip means reserving a flight, a hotel and a car, each owned by a different service with its own database. There is no transaction spanning them. If the car reservation fails after the flight and hotel succeeded, something must undo those — but a rollback is not available, because they are already committed.

Distributed transactions using two-phase commit exist but are rarely viable: they hold locks across services for the duration, they require every participant to support the protocol, and a coordinator failure can leave participants blocked indefinitely. Most organisations cannot use them across service boundaries.

> **Compensation is not rollback**  
> A rollback makes it as though the operation never happened. A compensation is a new business action that undoes the effect — refunding a charge, cancelling a booking, restocking an item. The intermediate state was visible to the outside world, and the compensating action is itself visible. That is a semantic difference the business must accept, not an implementation detail.

**Mental model**

A saga is a sequence of local transactions, each with a defined compensating action. Forward progress commits each step; failure runs the compensations for completed steps in reverse order.

1. **Local transaction** — One step, committed atomically within a single service. Immediately visible.
2. **Compensating action** — A business operation that undoes the effect of a step. Defined alongside the step, not invented during an incident.
3. **Orchestration** — A coordinator that calls each step and drives compensation. Central, explicit, easy to reason about.
4. **Choreography** — Each service reacts to events and emits its own. No coordinator; harder to see the whole process.
5. **Pivot step** — The point after which the saga cannot be compensated — a payment captured, an email sent, a physical item shipped. Order steps so this comes as late as possible.

> **Sagas expose intermediate state**  
> Between the flight booking and the compensation, the flight genuinely is booked and anyone looking can see it. A customer may receive a confirmation for something that is about to be cancelled. There is no isolation — sagas give atomicity of a kind and abandon isolation entirely, which means the product must be designed around visible intermediate states rather than assuming they are hidden.

**How it works**

**Forward path and compensation order**

```text
FORWARD
  T1 reserve flight     -> committed, visible
  T2 reserve hotel      -> committed, visible
  T3 reserve car        -> FAILS

COMPENSATE IN REVERSE
  C2 cancel hotel       (undo T2)
  C1 cancel flight      (undo T1)
  -> saga ends in "cancelled", not "rolled back"

WHY REVERSE ORDER
  later steps may depend on earlier ones. Undoing the
  hotel before the flight keeps dependencies valid, the
  same reason you unwind a stack rather than a queue.

PIVOT STEP
  once payment is CAPTURED (not just authorised), some
  steps cannot be cleanly undone - a refund is visible,
  costs fees, and may take days
  -> order the saga so irreversible steps come LAST
  -> authorise early, capture late
```

1. **Define the compensation with the step, not later** — Every forward action needs a named, implemented undo at design time. Discovering during an incident that a step has no compensation is the characteristic failure.
2. **Make compensations idempotent and always-succeed** — A compensation that can fail leaves the saga stuck in an inconsistent state with nothing to do. They must be retried until they succeed, which means they must be designed to be retryable indefinitely.
3. **Order steps so irreversible ones come last** — Authorise a payment early and capture it late; send emails at the end; ship physical goods only after everything else has committed.
4. **Persist saga state durably** — The saga's position and step outcomes must survive a crash, or a failure mid-saga leaves orphaned reservations nobody knows about.
5. **Prefer orchestration for anything non-trivial** — Choreography scatters the process across services with no single place showing its state, and the coupling reappears as event ordering assumptions.
6. **Accept and design for visible intermediate state** — Show pending status to users, avoid sending confirmations before the pivot, and make cancellation messages part of the product rather than an apology.

**Orchestration versus choreography**

```text
ORCHESTRATION
  coordinator calls each service in turn and records results
  coordinator -> flight.reserve()
              -> hotel.reserve()
              -> car.reserve()   FAILS
              -> hotel.cancel()
              -> flight.cancel()
  + the whole process is in one place, readable and testable
  + saga state is explicit and queryable
  + adding a step is a change in one file
  - the coordinator is a component to build and operate
  - it knows about every participant

CHOREOGRAPHY
  each service listens for events and emits its own
  FlightReserved -> hotel reacts -> HotelReserved -> car reacts
  CarFailed -> hotel compensates -> car reacts...
  + no coordinator; services are decoupled in code
  - the process exists nowhere; you read five services to
    understand it
  - adding a step means changing several services
  - compensation logic is duplicated and easy to get wrong
  - cycles and ordering assumptions are invisible

IN PRACTICE, ORCHESTRATION IS USUALLY RIGHT for sagas
with more than two or three steps.
```

> **Compensations must be able to succeed unconditionally**  
> If cancelling a hotel booking can fail because the hotel service is down, the saga cannot complete its unwind. Compensations therefore need to be retried indefinitely with backoff, must be idempotent so repeated attempts are safe, and need a dead-letter path with human escalation for the cases that genuinely cannot resolve — because a saga stuck half-compensated is worse than one that never started.

**Worked example**

An order saga, with the ordering decision that makes it workable.

**Step ordering around the pivot**

```text
NAIVE ORDER (fragile)
  1 capture payment        <- irreversible-ish, costs fees
  2 reserve inventory      <- may fail
  3 book courier           <- may fail
  4 send confirmation      <- irreversible, user-visible
  failure at 3 means refunding a captured payment and
  explaining a cancelled order after confirming it

BETTER ORDER
  1 authorise payment      <- reversible: void the auth
  2 reserve inventory      <- reversible: release
  3 book courier           <- reversible: cancel
  4 CAPTURE payment        <- PIVOT: from here, forward only
  5 send confirmation      <- after the pivot
  6 schedule review email  <- after the pivot

  failure at 1-3: void the authorisation, release stock,
  cancel the courier. The customer sees "order failed",
  no money moved, no confirmation was sent.

COMPENSATIONS
  void authorisation   - idempotent, always succeeds
  release inventory    - idempotent (release by reservation id)
  cancel courier       - idempotent; retried until success;
                         dead-letters to ops if the courier
                         API is down for hours

THE PIVOT IS THE DESIGN DECISION
  everything before it is cheaply undoable
  everything after it must succeed or be retried forever
  -> put as much as possible before the pivot
```

| Metric | Value | Note |
|---|---|---|
| Before pivot | 3 steps | all reversible |
| Pivot | capture payment | **forward only** |
| After pivot | retry forever | no compensation |
| User sees | clean failure | no false confirmation |

> **Designing the pivot is most of the work**  
> A saga where the irreversible step comes first is technically a saga and practically a source of refunds, apologies and manual reconciliation. Moving the capture after the reservations converts every pre-pivot failure into a clean, invisible cancellation. That reordering is usually possible — authorise then capture, reserve then confirm, stage then publish — and it is where the real design effort belongs.

**When to use it**

- **Multi-service processes** where a distributed transaction is unavailable or unacceptable.
- **Long-running processes** whose steps cannot hold locks for their duration.
- **Processes with natural business compensations** — cancel, refund, release, restock.
- **Where partial completion is genuinely unacceptable**, so unwinding is required rather than optional.
- **With a durable workflow engine**, which handles saga state, retries and compensation ordering for you.

**When to avoid it**

- **Do not use a saga when a single transaction would do.** If all steps are in one database, use a transaction — sagas are for crossing boundaries you cannot avoid.
- **Do not use it where intermediate visibility is unacceptable**, since sagas provide no isolation at all.
- **Do not use it where steps have no meaningful compensation**, such as an email already sent or a physical item already dispatched.
- **Do not use choreography for complex sagas**, where the process becomes invisible and compensation logic is duplicated.
- **Do not start a saga without durable state**, or a crash mid-saga leaves orphaned effects nobody will find.

**Advantages**

- **No distributed locks**, so services stay independent and available.
- **Each step commits locally**, using ordinary transactions within each service.
- **Works across heterogeneous systems**, including third parties that will never support two-phase commit.
- **Long-running processes are natural**, since nothing is held between steps.
- **Compensations are business-meaningful**, which is often what the domain actually needs — a cancellation rather than an erasure.

**Disadvantages**

- **No isolation**: intermediate states are visible and can be acted upon by others.
- **Compensations must be designed for every step**, which is real work and often overlooked.
- **Compensation can itself fail**, requiring indefinite retry and a human escalation path.
- **Some steps cannot be compensated**, forcing careful ordering around a pivot.
- **Reasoning about partial states is hard**, particularly when concurrent sagas touch the same entities.
- **Choreographed sagas become invisible**, with the process existing in no single place.

**Trade-offs**

**Cross-service consistency options**

| Approach | Isolation | Availability | Practicality |
|---|---|---|---|
| Single transaction | Full | One database | Best — if it fits |
| Two-phase commit | Full | Blocks on coordinator failure | Rarely viable across services |
| Saga with compensation | None | High — no locks held | The common answer |
| Eventual consistency, no compensation | None | Highest | Only if partial completion is acceptable |
| Redraw the boundary | Full within one service | Depends | Often the best answer |

The last row deserves serious consideration before any saga is built. If two services must frequently participate in the same transaction, the boundary between them may be wrong — and merging them, or moving the shared data to one owner, removes the distributed problem entirely rather than managing it.

**How it fails**

**Saga failure modes**

| Failure | Cause | Fix |
|---|---|---|
| Orphaned reservations | Crash mid-saga with no durable state | Persist saga position and outcomes; a reaper resumes or compensates |
| Stuck half-compensated | A compensation failed and was not retried | Compensations retried indefinitely; dead-letter with escalation |
| Money captured for a cancelled order | Irreversible step ordered before fallible ones | Reorder around the pivot; authorise early, capture late |
| Customer confirmed then cancelled | Notification sent before the pivot | Move user-visible commitments after the pivot |
| Compensation applied twice | Non-idempotent compensating action | Key compensations by the step's identifier |
| Nobody can see the process state | Choreographed saga with no coordinator | Orchestrate; or maintain an explicit saga record |
| Concurrent sagas interfere | No isolation; both act on the same entity | Reservations with expiry; optimistic concurrency on shared entities |

> **Concurrent sagas have no mutual isolation**  
> Two sagas reserving the last item can both read availability, both proceed, and both commit — because there is no transaction spanning them. Sagas solve the multi-step problem, not the concurrency problem. Shared contended resources still need their own protection: a conditional reservation write, an expiring hold, or ownership by a single service that serialises access.

**Limits**

> **Design guidance**
>
> - **Every step needs a named compensation**, implemented and tested, before the saga ships.
> - **Compensations must be idempotent and retryable indefinitely**, with a dead-letter path for the genuinely stuck.
> - **Order steps so the pivot is as late as possible** — everything before it should be cheaply reversible.
> - **Saga state must be durable**, or crashes leave orphaned effects with no record.
> - **Sagas provide atomicity of a sort and no isolation** — contended resources still need conditional writes or holds.

**Alternatives**

| Approach | When | Trade |
|---|---|---|
| Single transaction | All steps in one database | Requires the right service boundary |
| Saga (orchestrated) | Multi-service, compensable steps | No isolation; compensation design |
| Saga (choreographed) | Two or three simple steps | Process becomes invisible |
| Two-phase commit | Homogeneous systems supporting it | Locks; coordinator blocking |
| Durable workflow engine | Complex sagas with long waits | Engine to operate |
| Redraw the service boundary | Steps frequently transact together | Organisational change |

A durable workflow engine is the natural implementation of an orchestrated saga: the workflow code expresses the forward path and the compensations as ordinary try-and-catch logic, while the engine supplies the durable state, retries and resumption that a hand-built orchestrator would otherwise need.

**In real systems**

- **Travel and e-commerce booking flows** are the archetype, reserving several independently owned resources with cancellation as the compensation.
- **Payment authorise-then-capture** exists precisely to create a reversible pre-pivot step, which is why it is the standard shape in commerce.
- **Temporal and Step Functions** implement orchestrated sagas directly, with compensation expressed as ordinary error handling in workflow code.
- **Inventory reservation with expiry** is the standard way to handle the isolation gap — a hold that lapses if the saga does not complete.
- **Financial reversal transactions** are compensations in the accounting sense: the original entry stays and a correcting entry is added, because erasure is not permitted.

**Common mistakes**

- **Irreversible steps ordered first**, so every failure means a refund and an apology.
- **Compensations designed during an incident** rather than alongside the forward step.
- **Non-idempotent compensations**, applied twice on retry.
- **No durable saga state**, so a crash leaves orphaned reservations nobody knows about.
- **Choreography for complex sagas**, scattering the process across services with no visible state.
- **Assuming sagas provide isolation**, so concurrent sagas over-allocate shared resources.
- **Sending user-visible confirmations before the pivot.**

**The staff-level view**

Sagas are frequently adopted as a technical pattern when the harder questions are about business semantics and service boundaries.

- **Ask first whether the boundary is wrong.** If two services must constantly transact together, merging them or moving the shared data to one owner eliminates the problem rather than managing it, and that is usually the better answer.
- **Make the pivot an explicit design decision**, and push it as late as possible. Most of a saga's practical quality comes from step ordering, not from the mechanism.
- **Require a named, implemented, tested compensation for every step** before the saga ships. Discovering a missing one during an incident is the characteristic failure.
- **Insist compensations are idempotent and retried indefinitely**, with human escalation for the stuck cases — a half-compensated saga is worse than one that never started.
- **Name the intermediate states as product decisions.** Users will see pending, partially committed and cancelled states, and designing for them is part of the work.

**Go deeper**

A saga replaces an unavailable distributed transaction with a sequence of local commits, each paired with a compensating action. If a later step fails, the compensations for completed steps run in reverse order. Crucially, compensation is not rollback: the intermediate state was real and visible, and the undo is itself a visible business action — a refund rather than an un-charge, a cancellation rather than an erasure.

Most of the practical design is step ordering around the pivot: the point after which compensation is no longer clean. Authorising a payment is reversible while capturing it is not, so reserving inventory and booking a courier should come before the capture, not after. Everything before the pivot fails invisibly and cleanly; everything after it must be retried forward rather than unwound.

Two properties must hold. Compensations have to be idempotent and retryable indefinitely, because a saga stuck half-compensated is worse than one that never started. And saga state must be durable, or a crash leaves orphaned reservations nobody will find. What sagas do not provide is isolation — two concurrent sagas can both allocate the last item — so shared contended resources still need conditional writes or expiring holds of their own.

Sagas exist because a transaction cannot span services, and two-phase commit is rarely viable across organisational or technology boundaries. They trade isolation for availability and accept that intermediate states are visible.

**Compensation is a business action, not a rollback.** A rollback makes it as though nothing happened; a compensation adds a new effect that counteracts the previous one. The flight was genuinely booked, the charge genuinely made, and both the original and the undo are visible to the customer and to any other system watching. That is a semantic property the product must accommodate — pending states, cancellation notifications, and the possibility of a confirmation followed by a cancellation — rather than an implementation detail that can be hidden.

**The pivot is where the design effort belongs.** Some steps cannot be compensated cleanly: a captured payment costs fees to refund and takes days, an email cannot be unsent, a dispatched parcel cannot be un-dispatched. The point after which the saga must go forward is the pivot, and the quality of a saga is largely determined by how late it sits. Authorise-then-capture exists precisely to create a reversible pre-pivot step, and the same reordering usually applies elsewhere — reserve before confirming, stage before publishing, hold before committing. A saga whose irreversible step comes first is technically correct and practically a generator of refunds and apologies.

**Compensations must be unconditionally completable.** If a compensating action can fail and is not retried, the saga is left neither committed nor undone, which is the worst available state. So compensations must be idempotent, retried indefinitely with backoff, and backed by a dead-letter path with human escalation for cases a third-party outage makes genuinely impossible. Designing a compensation that may simply fail is designing a saga that can become permanently stuck, and those stuck sagas are what eventually force a manual reconciliation process.

**Orchestration is usually right.** Choreography — each service reacting to events and emitting its own — avoids a coordinator but scatters the process across services so that no single place describes it, duplicates compensation logic, makes adding a step a multi-service change, and hides ordering assumptions and cycles. For anything beyond two or three steps, an orchestrator that calls each step and records outcomes is more readable, more testable, and gives queryable saga state. A durable workflow engine is the natural implementation, since the forward path and compensations become ordinary code while the engine supplies durability, retries and resumption.

**Sagas give no isolation, and this surprises people.** Two sagas can each read that the last item is available and both proceed, because nothing serialises them. Solving the multi-step problem does not solve the concurrency problem. Contended resources still need their own protection — a conditional write, or more commonly an expiring reservation that holds the item for the saga's duration and lapses automatically if the saga crashes, which conveniently covers both the concurrency gap and the failure-to-compensate case.

**A saga proposal is a prompt to check the boundary.** Occasional cross-service processes genuinely need one. But if the same services transact together on every order or every signup, the boundary is probably wrong, and the saga imposes a permanent cost — compensation logic per step, no isolation, visible intermediate states, durable tracking, and reconciliation for stuck cases. Merging the services or moving the shared data to one owner makes the invariant local and removes all of it. The question worth asking is what invariant spans the boundary and why, because the answer is often an organisational split now imposing a distributed-systems tax on every transaction.

**Prove it — interview questions**

1. **[Basic] What is a saga?**

   <details><summary>Model answer</summary>

   A sequence of local transactions, each committed independently in its own service, with a compensating action defined for each. If a later step fails, the compensations for the completed steps run in reverse order to undo their effects. It exists because there is no transaction spanning several services, so atomicity has to be approximated with forward commits and explicit undo rather than with rollback.

   </details>

2. **[Basic] How is compensation different from rollback?**

   <details><summary>Model answer</summary>

   A rollback erases the operation as though it never occurred. A compensation is a new business action that counteracts the effect — refunding rather than un-charging, cancelling rather than un-booking. The crucial consequence is that the intermediate state was real and visible: a customer may have received a booking confirmation for something that is about to be cancelled, and the compensation itself is visible too. That is a semantic difference the business has to accept, not something the implementation can hide.

   </details>

3. **[Senior] What is the pivot step and why does its placement matter?**

   <details><summary>Model answer</summary>

   The pivot is the point after which the saga can no longer be compensated cleanly — typically capturing a payment, sending a customer email, or dispatching a physical item. Before it, every step can be cheaply undone and a failure results in a clean cancellation nobody notices. After it, failures must be resolved by retrying forward rather than unwinding. So the design work is ordering steps to push the pivot as late as possible: authorise rather than capture, reserve rather than confirm, stage rather than publish. Most of a saga's practical quality comes from that ordering rather than from the mechanism itself.

   </details>

4. **[Senior] Why must compensations be able to succeed unconditionally?**

   <details><summary>Model answer</summary>

   Because if a compensation fails, the saga cannot finish unwinding and is left in a state that is neither committed nor undone — which is worse than either. That means compensations must be idempotent so they can be retried safely, must be retried indefinitely with backoff rather than abandoned after a few attempts, and must have a dead-letter path with human escalation for the genuinely stuck cases, such as a third-party API that has been down for hours. Designing a compensation that can simply fail is designing a saga that can get permanently stuck.

   </details>

5. **[Staff] What do sagas not give you, and what fills the gap?**

   <details><summary>Model answer</summary>

   Isolation. Two sagas can both read that the last item is available, both proceed, and both commit, because nothing serialises them — sagas solve the multi-step problem and not the concurrency problem. The gap is filled by the contended resource protecting itself: a conditional write on the availability row, an expiring reservation that holds the item for the saga's duration, or routing all operations on that resource to a single owner. The second is the most common pattern in practice, because the expiry also handles the case where the saga crashes and never compensates. The important framing is that adopting a saga does not remove the need for ordinary concurrency control on shared entities.

   </details>

6. **[Principal] When should a saga make you question the architecture rather than build one?**

   <details><summary>Model answer</summary>

   When the same two or three services participate in a transaction constantly. A saga is the right answer for occasional cross-boundary processes, but a boundary that is crossed transactionally on every order, every signup, or every update is probably drawn in the wrong place — and the cost of the saga is permanent: compensation logic for every step, no isolation, visible intermediate states, durable saga tracking, and reconciliation for the cases that get stuck. Merging the services, or moving the shared data to a single owner so the invariant becomes local, removes all of that at once. I would treat a proposal to build a saga as a prompt to ask what invariant is being protected and why it spans services, because the answer is quite often an organisational split that was made for team reasons and is now imposing a distributed-systems cost on every transaction.

   </details>

---

### Fanout and aggregation

*Call many dependencies in parallel and combine the results under a deadline — bounding concurrency, tolerating partial failure, and returning what arrived.*

**Flow:** `Incoming request` → `Bounded fanout` → `Dependency results` → `Deadline join` → `Response`

> **The 30-second version**  
> Call independent dependencies in parallel under one shared deadline, classify which may be missing, and return what arrived — bounding latency and multiplying availability rather than failure.

**The problem**

A product page needs pricing, inventory, reviews, recommendations, shipping estimates and personalisation — six services. Called sequentially at fifty milliseconds each, the page takes three hundred milliseconds before any rendering. Called in parallel, it takes as long as the slowest one.

But parallelism introduces its own problems: the response now waits for the slowest of six, so the tail latency of every dependency becomes the page's typical latency; a single failing dependency can fail the whole page; and unbounded fan-out means one request can consume dozens of connections and threads.

> **Fanout converts a sum into a maximum, and that is both the gain and the cost**  
> Sequential latency is the sum of the parts; parallel latency is the maximum. That is a large win on the median and a large loss on the tail, because the probability that at least one of N calls is slow grows rapidly with N. Fanout therefore requires a deadline and a partial-result strategy, not just a parallel loop.

**Mental model**

Issue all the calls, wait until either everything returns or the deadline expires, then assemble a response from whatever arrived. What is missing is either omitted, defaulted, or causes a failure — and which of those is a per-dependency decision.

1. **Fanout** — Issuing N concurrent requests from one incoming request. The amplification factor is N.
2. **Deadline** — A hard bound on the total wait, shared across all calls, after which the join proceeds without stragglers.
3. **Criticality** — Per-dependency classification: required, or optional with a fallback. This determines what happens when it is missing.
4. **Concurrency bound** — A cap on in-flight fan-out calls, so one request cannot exhaust connections or threads.
5. **Partial result** — A response assembled from what arrived, with missing sections marked degraded rather than failing the whole request.

> **Fanout amplifies both load and tail latency**  
> One incoming request becomes N outgoing ones, so a modest request rate becomes a large dependency load. And if each dependency is slow one per cent of the time, a page fanning out to six is slow about six per cent of the time — to forty, about a third of the time. The component's p99 becomes the page's typical experience, which is why tail latency work matters disproportionately behind a fan-out.

**How it works**

**Latency arithmetic for fanout**

```text
SIX DEPENDENCIES, each p50 = 20ms, p99 = 300ms

SEQUENTIAL
  latency = sum of medians = 120ms typical
  one slow call adds 300ms -> 400ms

PARALLEL
  latency = MAX of six draws
  P(at least one slow) = 1 - 0.99^6 = 5.9%
  so ~6% of requests take 300ms instead of ~20ms
  -> median much better, tail much worse

PARALLEL WITH A 100ms DEADLINE
  all fast calls complete
  slow calls are abandoned at 100ms
  latency is bounded at 100ms ALWAYS
  6% of responses are missing one section
  -> predictable latency, graceful degradation

THE DEADLINE IS WHAT MAKES FANOUT SAFE.
Without it, the response waits for the worst dependency
on every single request.
```

1. **Classify every dependency as required or optional** — Pricing is required; recommendations are not. Optional dependencies get a fallback — cached, default, or omitted — and never fail the request.
2. **Set one deadline for the whole fan-out, not per call** — The user cares about total time. A shared deadline propagated to each call means each dependency knows how long it has and can refuse work it cannot finish.
3. **Bound the concurrency** — A per-request cap and a global in-flight cap, so one request cannot consume unlimited connections and a burst cannot exhaust the pool.
4. **Return partial results with explicit degradation** — A page missing its recommendations is far better than an error page. Mark the missing section so the client can render appropriately.
5. **Reduce N before optimising the calls** — Amplification is multiplicative in N. Batching six calls into two, or denormalising to remove a call, beats making each call faster.
6. **Consider hedging only for idempotent, critical calls** — A duplicate request to a second replica after the p95 delay tightens the tail, at the cost of extra load and a hedge budget.

**Criticality classification drives the failure behaviour**

```text
DEPENDENCY        CRITICAL?  MISSING BEHAVIOUR
----------------  ---------  --------------------------------
pricing           YES        fail the request (a page without
                             a price is not a product page)
inventory         YES        fail, or show "checking..."
reviews           no         omit the section
recommendations   no         omit, or show a static fallback
shipping estimate no         show "calculated at checkout"
personalisation   no         fall back to the generic view

RESULT
  only 2 of 6 can fail the page
  the other 4 degrade invisibly or near-invisibly
  -> the probability of a FAILED page is governed by two
     dependencies, not six

THIS RECLASSIFICATION IS OFTEN THE LARGEST AVAILABILITY
IMPROVEMENT AVAILABLE, and it costs no engineering at all -
only a product decision about what a degraded page may show.
```

> **Unbounded fan-out is a self-inflicted overload**  
> A request that fans out to one call per item in a list — per order, per friend, per row — has an amplification factor determined by data rather than by design. A user with a thousand items generates a thousand calls from one request. Fan-out width must be bounded by pagination or batching, or a single legitimate request becomes a denial of service against your own dependencies.

**Worked example**

A product page fan-out, designed for bounded latency and graceful degradation.

**Design and measured effect**

```text
SIX DEPENDENCIES, page latency budget 150ms

DESIGN
  deadline:    120ms for the whole fan-out, propagated
  concurrency: all six in parallel (bounded per request)
  criticality: pricing and inventory required;
               the other four optional with fallbacks
  batching:    reviews + ratings served by one call
               recommendations + personalisation by one call
               -> six calls become four

BEHAVIOUR
  typical:   all four return in ~25ms -> page in ~30ms
  one slow:  abandoned at 120ms, section marked degraded
             -> page still renders in ~125ms
  pricing
  unavailable: request fails with a clear error
             -> the only case that produces an error page

AVAILABILITY ARITHMETIC
  if each dependency is 99.9% available:
    all six required:  0.999^6 = 99.40%  -> 4.3h/month down
    two required:      0.999^2 = 99.80%  -> 1.4h/month
  -> reclassifying four dependencies as optional TRIPLED
     the page's effective availability, with no code change
     to any dependency

FALLBACKS
  reviews:          omit the section
  recommendations:  cached popular items, refreshed hourly
  shipping:         "calculated at checkout"
  personalisation:  generic view
```

| Metric | Value | Note |
|---|---|---|
| Calls | 6 → 4 | batching |
| Deadline | 120 ms | bounded always |
| Required | 2 of 6 | **3× availability** |
| Degraded | invisible | fallbacks |

> **Criticality classification is the cheapest availability win there is**  
> Multiplying six dependencies at 99.9% availability gives 99.4% — four hours of downtime a month caused entirely by treating every dependency as required. Deciding that four of them may be missing costs no engineering work on any dependency and triples the page's effective availability. It is a product decision about what a degraded page may look like, and it is routinely skipped because nobody asks the question.

**When to use it**

- **Composite views** assembled from several services — product pages, dashboards, feeds, profiles.
- **Independent dependencies**, where nothing needs another's result.
- **Latency-sensitive paths**, where sequential calls would exceed the budget.
- **Where partial results are acceptable**, which is most user-facing rendering.
- **Scatter-gather queries**, where a request must reach many shards and merge their answers.

**When to avoid it**

- **Do not fan out when calls are dependent**, since the second needs the first's result and parallelism is impossible.
- **Do not fan out without a deadline**, or the response waits for the slowest dependency every time.
- **Do not fan out per item** over an unbounded list; bound the width by pagination or batching.
- **Do not treat every dependency as required**, which multiplies their failure probabilities into your availability.
- **Do not hedge non-idempotent calls**, and not without a budget.

**Advantages**

- **Latency becomes the maximum rather than the sum**, which is a large improvement on the median.
- **Partial results give graceful degradation**, turning a dependency outage into a missing section rather than an error page.
- **Criticality classification improves availability dramatically** at no engineering cost.
- **Deadlines make latency predictable**, bounded regardless of dependency behaviour.
- **Independent dependencies can be scaled and operated separately**, which is why the architecture exists.

**Disadvantages**

- **Tail latency dominates**, because the response waits for the slowest of N.
- **Load amplification**: one request becomes N, so dependency load is a multiple of request rate.
- **Resource consumption per request** is N connections and N slots, which bounds concurrency.
- **Partial results complicate clients**, which must render a response with missing sections.
- **Debugging is harder**, since a slow page could be any of N dependencies.

**Trade-offs**

**Handling a slow or failed dependency**

| Strategy | Latency effect | Result quality | Cost |
|---|---|---|---|
| Wait indefinitely | Unbounded | Complete | Unacceptable |
| Per-call timeout | Sum of timeouts in the worst case | Complete or error | Poor bound |
| Shared deadline + partial result | Bounded | Degraded section | Client must handle it |
| Cached fallback | Bounded | Stale but complete | Cache to maintain |
| Hedged request | Tighter tail | Complete | Extra load; idempotent only |

> **The framing that shows judgement**  
> “Four calls in parallel under a shared 120-millisecond deadline, with only pricing and inventory able to fail the page — the other two degrade to a cached or generic fallback. That bounds latency regardless of dependency behaviour and takes the page's availability from the product of four dependencies to the product of two, which is the largest single improvement available and costs no engineering on any dependency.”

**How it fails**

**Fanout failures**

| Failure | Cause | Fix |
|---|---|---|
| Page latency equals the worst dependency | No deadline; waiting for all | Shared deadline; partial results |
| One dependency outage takes down the page | Every dependency treated as required | Criticality classification with fallbacks |
| Dependency overwhelmed by fan-out load | Amplification of N per request | Reduce N by batching; rate limit; cache |
| Connection pool exhausted | Unbounded concurrent fan-out | Per-request and global concurrency caps |
| One request generates a thousand calls | Fan-out per item over an unbounded list | Paginate; batch; cap the width |
| Availability much lower than each dependency | Multiplying required dependencies | Reclassify as optional where the product allows |
| Cannot tell which dependency is slow | No per-dependency instrumentation | Per-dependency latency and error metrics on the fan-out path |

**Limits**

> **Arithmetic to carry**
>
> - **Parallel latency = max of N draws**, so P(slow) = 1 − (1 − p)^N — at p = 1% and N = 6 that is 5.9%.
> - **Availability multiplies**: six dependencies at 99.9% required gives 99.4%.
> - **Deadline** should be shared across the fan-out and propagated, not set per call.
> - **Amplification**: request rate × N is the load each dependency sees.
> - **Hedge delay ≈ p95** with a budget of a few per cent, and only for idempotent calls.

**Alternatives**

| Approach | Latency | Trade |
|---|---|---|
| Sequential calls | Sum of all | Simple; slow |
| Parallel fan-out + deadline | Max, bounded | Amplification; partial results |
| Batched calls | Fewer round trips | Requires batch APIs |
| Denormalised read model | One read | Write amplification; staleness |
| Client-side composition | Parallel from the browser | More round trips over the internet |
| GraphQL / BFF aggregation | One client round trip | Server-side fan-out still happens |

A denormalised read model is the structural answer when a fan-out is on a very hot path: precompute the composite view so the read is a single lookup, accepting write amplification and staleness. It removes the fan-out entirely rather than optimising it, which is why feed and timeline systems converge on it.

**In real systems**

- **Search engines** fan out to many shards with a strict deadline and merge whatever returns, which is why results are occasionally incomplete rather than slow.
- **Product and profile pages** at large retailers classify dependencies by criticality so that recommendations or reviews failing never blocks the purchase path.
- **Backend-for-frontend layers** exist largely to perform this fan-out server-side, where latency between services is low, rather than from the client over the internet.
- **The tail-at-scale literature** established hedged and tied requests as the standard technique for tightening fan-out tails at a few per cent extra load.
- **Scatter-gather in distributed databases** is the same pattern applied to sharded queries, with the same deadline and partial-result considerations.

**Common mistakes**

- **No deadline**, so the response always waits for the slowest dependency.
- **All dependencies treated as required**, multiplying their failure rates into the page's availability.
- **Per-item fan-out** over an unbounded list.
- **No concurrency bound**, exhausting connections under burst.
- **Per-call timeouts instead of a shared deadline**, giving an unbounded total.
- **Optimising individual call latency** rather than reducing the number of calls.
- **No per-dependency metrics**, making slow pages undiagnosable.

**The staff-level view**

Fan-out designs are usually built for latency and then fail on availability, because nobody classified which dependencies may be missing.

- **Insist on criticality classification as a product decision.** Multiplying availabilities is the largest self-inflicted availability loss in most composite pages, and reclassifying dependencies costs no engineering at all.
- **Require a shared, propagated deadline.** Per-call timeouts give an unbounded worst case and waste downstream capacity on work whose caller has already given up.
- **Bound fan-out width by design, not by data.** Per-item fan-out over an unbounded collection is a denial of service one large customer away.
- **Reduce N before optimising individual calls**, since amplification is multiplicative and batching or denormalising beats making each call faster.
- **Instrument per dependency on the fan-out path**, or a slow page is an unsolvable mystery among N candidates.

**Go deeper**

Fan-out converts sequential latency, the sum of the parts, into parallel latency, the maximum. That is a large gain on the median and a loss on the tail, because the probability that at least one of N calls is slow rises quickly with N. A shared deadline across the whole fan-out — propagated so each dependency knows its remaining budget — bounds the response time regardless of how any dependency behaves.

The larger issue is availability. Treating every dependency as required multiplies their failure probabilities: six services at 99.9% give 99.4%, roughly four hours of downtime a month created purely by composition. Classifying most of them as optional with fallbacks — omit the reviews, show cached recommendations, defer the shipping estimate — triples the effective availability with no engineering work on any dependency. It is a product decision that is routinely never posed.

Two further constraints. Amplification means one request becomes N, so dependency load is a multiple of request rate, and reducing N by batching beats making individual calls faster. And fan-out width must be bounded by design rather than by data — one call per item over an unbounded list means a single large customer can generate thousands of calls from one request.

Fan-out is how composite views are assembled, and its design is dominated by two arithmetic facts: parallel latency is a maximum, and composed availability is a product.

**Latency becomes the maximum of N draws.** Six sequential twenty-millisecond calls take a hundred and twenty milliseconds; in parallel they take roughly twenty. But the probability that at least one is slow is 1 − (1 − p)^N, so with a one per cent tail and six calls, nearly six per cent of requests experience the slow path — the component's p99 becoming the page's common case. A shared deadline across the whole fan-out is what converts this from unbounded to bounded, and propagating it means each dependency can decline work it cannot complete in the remaining time rather than producing a result nobody will wait for.

**Availability multiplies, and this is the larger effect.** Six dependencies at 99.9% availability, all required, compose to 99.4% — about four hours of monthly downtime generated entirely by composition rather than by any dependency being unreliable. Classifying dependencies by criticality changes the arithmetic fundamentally: if only two can fail the page and the rest have fallbacks, the composed availability is the product of two rather than six. This is usually the single largest availability improvement available in a composite view, it requires no engineering work on any dependency, and it is skipped because it is a product question — what may a degraded page show — that engineering does not think to ask.

**Amplification bounds what the architecture can support.** One incoming request becomes N outgoing ones, so each dependency sees request rate times N. Reducing N by batching related calls or denormalising away a dependency is multiplicative and therefore worth more than making individual calls faster. The pathological case is fan-out per item over an unbounded collection, where the amplification factor is set by data rather than design — one customer with a thousand records generating a thousand calls from a single legitimate request. Width must be bounded by pagination or batching.

**Partial results are the mechanism that makes degradation graceful.** When the deadline expires, the join proceeds with whatever arrived and marks missing sections explicitly, so the client renders a page without recommendations rather than an error page. This requires the client to handle incomplete responses, which is a real cost, and it requires fallbacks to be defined per dependency — cached values, static defaults, or deferred computation such as showing shipping cost at checkout instead of on the page.

**Hedging is the last lever, not the first.** Duplicating a request to a second replica after roughly the p95 delay and taking whichever returns first tightens the tail substantially, but it costs extra load, requires idempotency and cancellation, and needs a budget capping hedges at a few per cent — otherwise a fleet-wide slowdown causes every request to hedge and doubles load exactly when capacity is scarcest. Reducing N, setting a deadline, and classifying criticality should all come first.

**Know when to stop optimising and restructure.** On a very hot path where amplification has become the dominant load on dependencies, the structural answer is a denormalised read model: precompute the composite view so the read is one lookup, accepting write amplification and staleness. Feed and timeline systems converge on this. Persistent fan-out problems are also worth reading as a signal about service boundaries — a view that must always call six services suggests the data was divided along lines that do not match how it is consumed.

**Prove it — interview questions**

1. **[Basic] What does fan-out change about latency?**

   <details><summary>Model answer</summary>

   It converts a sum into a maximum. Six sequential calls at twenty milliseconds each take a hundred and twenty milliseconds; the same six in parallel take as long as the slowest one, typically around twenty. That is a large improvement on the median. The cost is on the tail: the probability that at least one of six calls is slow is much higher than for a single call, so the component's p99 becomes something close to the page's typical experience unless a deadline bounds it.

   </details>

2. **[Basic] Why does a shared deadline matter more than per-call timeouts?**

   <details><summary>Model answer</summary>

   Because the user cares about total time, and per-call timeouts give an unbounded worst case — six calls with a one-second timeout each could take six seconds if they fail sequentially. A single deadline covering the whole fan-out means the response is bounded regardless of how any individual dependency behaves, and propagating it means each dependency knows how much time remains and can decline work it cannot finish rather than producing a result nobody will wait for.

   </details>

3. **[Senior] How does treating every dependency as required affect availability?**

   <details><summary>Model answer</summary>

   It multiplies their failure probabilities. Six dependencies each at 99.9% availability give a combined 99.4% if all are required — about four hours of downtime a month caused entirely by composition rather than by any dependency being unreliable. Classifying four of them as optional with fallbacks takes it to 99.8%, a threefold improvement in downtime, with no engineering work on any dependency. It is a product decision about what a degraded page may show, and it is routinely skipped because nobody poses the question.

   </details>

4. **[Senior] Why is per-item fan-out dangerous?**

   <details><summary>Model answer</summary>

   Because the amplification factor is determined by data rather than by design. A request that makes one call per item in a user's list generates as many calls as that user has items, so one customer with a thousand records produces a thousand dependency calls from a single request — a denial of service against your own dependencies triggered by an entirely legitimate user. Fan-out width must be bounded by pagination or batching so it is a property of the design rather than of whoever happens to be the largest customer.

   </details>

5. **[Staff] Design a product page that assembles data from six services.**

   <details><summary>Model answer</summary>

   First reduce the count: batching related data — reviews with ratings, recommendations with personalisation — often takes six calls to four, and since amplification is multiplicative that is worth more than making any individual call faster. Then classify criticality as a product decision: pricing and inventory are required because a product page without a price is not a product page, while reviews, recommendations, shipping estimates and personalisation each get a fallback and may be omitted. That alone takes the availability from the product of six dependencies to the product of two. Then a shared deadline of around a hundred and twenty milliseconds within a hundred-and-fifty-millisecond budget, propagated so each dependency knows its remaining time, with the join proceeding on expiry and missing sections marked degraded. Finally per-dependency latency and error instrumentation, because otherwise a slow page is an unsolvable mystery among several candidates.

   </details>

6. **[Principal] When should a fan-out be replaced rather than optimised?**

   <details><summary>Model answer</summary>

   When it sits on a very hot path and the amplification has become the dominant load on the dependencies. At that point the structural answer is a denormalised read model: precompute the composite view so the read is a single lookup, accepting write amplification and a staleness window in exchange for eliminating the fan-out entirely. Feed and timeline systems converge on this for exactly that reason. The judgement is about read-to-write ratio and tolerance for staleness — a page read thousands of times per write is a strong candidate, while one read rarely is not worth the maintenance. I would also treat persistent fan-out latency problems as a signal to check whether the service boundaries match the access patterns, because a page that must always call six services to render one view suggests the data was split along lines that do not reflect how it is used.

   </details>

---

### Distributed scheduling

*Run work at a time, exactly once across a fleet — which requires durable schedules, claimed execution, and an honest answer about what a missed window means.*

**Flow:** `Durable schedule` → `Due-time scan` → `Run claim` → `Job execution` → `Completion record`

> **The 30-second version**  
> Store schedules durably, create one run record per window with a unique constraint, claim it for execution, and decide explicitly what a missed window means.

**The problem**

A nightly billing run, an hourly reconciliation, a reminder in three days, a subscription renewal at a specific instant. A cron entry on one machine does this — until that machine is replaced, scaled to three instances, or restarts during the scheduled minute. Then the job runs three times, or not at all, and nothing records which.

Distributed scheduling is deceptively hard because it combines three separate problems: durably remembering what should run and when, ensuring exactly one instance runs it, and deciding what to do about windows that were missed while the system was unavailable.

> **The three failures of naive cron on a fleet**  
> **Duplicate execution**: every instance has the same crontab, so all of them run the job. **Silent omission**: the one instance with the crontab was down at the scheduled minute, and nothing ever runs it. **No record**: neither case produces evidence, so a billing run that never happened looks identical to one that did.

**Mental model**

Separate the schedule from the execution. The schedule is durable data describing what should happen and when; a scanner finds due entries; a claim ensures exactly one worker executes each; and a completion record proves it happened.

1. **Schedule record** — A durable row: what to run, the recurrence or the specific time, the next due time, and the last completed run.
2. **Scanner** — A process that periodically finds entries whose due time has passed. Leader-elected or itself using claimed execution.
3. **Claim** — A conditional update that moves a due entry to running, so concurrent scanners cannot both win.
4. **Execution record** — A row per intended run, with its outcome — which is what makes “did the billing run happen?” answerable.
5. **Catch-up policy** — What happens to windows missed during downtime: run once, run all of them, or skip. This must be decided per schedule.

> **The catch-up policy is the question nobody asks in advance**  
> The system is down from 23:50 to 00:30 and misses the midnight billing run. Should it run late, at 00:30? Should it skip, because the window has passed? If four hourly runs were missed, should all four execute, or only the most recent? Each is correct for some schedules and disastrous for others, and the choice must be made per schedule rather than by the scheduler's default.

**How it works**

**Durable schedule plus claimed execution**

```text
SCHEDULE TABLE
  id, name, cron_expression, timezone,
  next_due_at, last_completed_at,
  catch_up_policy, enabled

RUN TABLE  (one row per intended execution)
  schedule_id, scheduled_for, state, claimed_by,
  lease_until, started_at, finished_at, outcome

SCANNER (runs on every instance, safely)
  find schedules where next_due_at <= now()
  for each, atomically:
    INSERT INTO runs (schedule_id, scheduled_for, state)
      VALUES (?, next_due_at, 'pending')
    ON CONFLICT (schedule_id, scheduled_for) DO NOTHING
    -- the unique constraint makes duplicate creation
    -- impossible, no matter how many scanners race
    UPDATE schedules SET next_due_at = <next occurrence>

WORKER
  UPDATE runs SET state='running', claimed_by=?,
                  lease_until=now()+10min
   WHERE id=? AND state='pending'
  -- 0 rows affected means another worker won

  ... execute ...
  UPDATE runs SET state='succeeded', finished_at=now()

THE UNIQUE CONSTRAINT ON (schedule_id, scheduled_for)
IS WHAT GUARANTEES ONE RUN PER WINDOW.
```

1. **Make the run record the unit of exactly-once, not the trigger** — A unique constraint on schedule plus scheduled time means any number of racing scanners produce exactly one run record. That is far more robust than trying to elect a single scheduler.
2. **Store the schedule durably, not in a crontab** — A crontab lives on a machine, is invisible to the rest of the system, and disappears with it. A schedule table can be queried, audited, disabled and reasoned about.
3. **Decide the catch-up policy per schedule** — A billing run should catch up; a cache warm should skip; an hourly report that missed four windows probably wants only the most recent.
4. **Lease and heartbeat long runs** — A worker that dies mid-execution must have its run recovered, which needs a lease expiry and a reaper — the same mechanism as any background job.
5. **Handle timezones and daylight saving explicitly** — Two in the morning happens twice in autumn and not at all in spring. A schedule expressed in local time is ambiguous on exactly those days, and billing systems notice.
6. **Alert on absence, not just on failure** — The dangerous case is a run that never started. Monitoring must assert that each expected run occurred, which requires knowing what was expected.

**Catch-up policies and where each belongs**

```text
SKIP MISSED
  the window passed; do nothing
  use: cache warming, metric snapshots, health summaries
  -> a missed run costs nothing and running late is pointless

RUN ONCE ON RECOVERY
  run the most recent missed occurrence only
  use: reports, syncs, reconciliation
  -> being current matters; the intermediate runs do not

RUN ALL MISSED
  execute every missed window in order
  use: billing, accruals, anything where each window has
       its own output
  -> skipping a window means a customer is not billed
  -> CAUTION: a long outage produces a thundering herd
              of catch-up runs

THE DEFAULT MATTERS
  most schedulers default to "skip", which is safe for
  cache warming and catastrophic for billing.
  The policy must be explicit per schedule.
```

> **Timezone-aware schedules are ambiguous twice a year**  
> A job scheduled at 02:30 local time does not run on the spring-forward day and runs twice on the autumn day, because that local time either does not exist or occurs twice. Systems that schedule in UTC avoid this entirely but drift relative to business hours. Systems that schedule in local time must define behaviour for both cases explicitly, and financial and reporting jobs are exactly where the ambiguity is noticed.

**Worked example**

A subscription billing scheduler: the failure cases that matter and how the design addresses each.

**Design and failure analysis**

```text
REQUIREMENT
  each subscription bills on its own anniversary date
  exactly one charge per period, ever
  a missed window must still be charged
  200,000 subscriptions, spread across every hour

SCHEDULE
  per-subscription row with next_bill_at
  rather than one cron job scanning everything
  -> load spreads naturally; no midnight spike

EXACTLY-ONCE
  runs table with UNIQUE (subscription_id, billing_period)
  -> a racing scanner, a retry, or a duplicate trigger
     all collapse to one run record
  -> the charge itself carries the run id as an
     idempotency key, so even a retried execution
     cannot double-charge

CATCH-UP
  policy: RUN ALL MISSED
  an outage from Friday to Monday must still bill
  everyone whose anniversary fell in between
  -> but cap the catch-up rate, or Monday morning
     produces 3 days of billing in one burst against
     the payment provider

FAILURE HANDLING
  payment declined -> not a scheduler failure; the run
    SUCCEEDED and produced a "declined" outcome, which
    starts a dunning workflow
  payment provider down -> run fails, retried with
    backoff, dead-letters to ops after N attempts
  worker crashes mid-run -> lease expires, reaper
    returns it to pending, idempotency key prevents
    a double charge

MONITORING
  alert if any subscription's next_bill_at is more than
  an hour past due
  -> detects ABSENCE, which failure alerts cannot
```

| Metric | Value | Note |
|---|---|---|
| Exactly-once | unique constraint | on (sub, period) |
| Catch-up | run all missed | rate-limited |
| Crash | lease + reaper | **idempotent charge** |
| Alert | overdue schedules | absence, not failure |

> **Alert on absence, because failure alerts cannot see a job that never started**  
> Every monitoring system will tell you when a job failed. Almost none will tell you when a job never ran, because there is nothing to report on. The only way to detect it is to know what was expected and assert that it happened — an overdue-schedule check. A billing run that silently never executed is indistinguishable from a quiet night unless something is actively looking for its absence.

**When to use it**

- **Recurring work across a fleet**: billing, reconciliation, reports, cleanup, index rebuilds.
- **Per-entity schedules**, such as subscription anniversaries or user-set reminders, where one cron entry cannot express it.
- **Delayed actions**, like a reminder in three days or a timeout on a pending approval.
- **Anywhere exactly-once execution matters**, particularly when money or notifications are involved.
- **When missed windows must be recoverable**, which a stateless cron cannot provide.

**When to avoid it**

- **Do not use a crontab on instances in an autoscaled group**, which either duplicates or silently omits.
- **Do not rely on a single scheduler instance** without leader election and failover, which makes it a single point of failure.
- **Do not leave the catch-up policy at a default**, since the right answer differs completely between billing and cache warming.
- **Do not schedule in local time** for financial or reporting jobs without defining daylight-saving behaviour.
- **Do not monitor only for failures**, which cannot detect a run that never started.

**Advantages**

- **Exactly-once per window** via a unique constraint, robust against racing scanners and retries.
- **Schedules are queryable and auditable**, unlike crontabs scattered across machines.
- **Missed windows are recoverable**, with an explicit policy per schedule.
- **Per-entity scheduling** is expressible, which cron fundamentally cannot do.
- **Execution history exists**, so “did this run and what happened?” is answerable.

**Disadvantages**

- **More machinery** than a crontab: tables, a scanner, claiming, leases, a reaper.
- **Scanner frequency bounds precision**, so sub-second scheduling needs a different design.
- **Catch-up bursts** after an outage can overwhelm downstream systems without rate limiting.
- **Timezone handling is genuinely intricate**, particularly around daylight saving.
- **Run records accumulate** and need retention management.

**Trade-offs**

**Scheduling approaches**

| Approach | Exactly-once | Missed windows | Per-entity |
|---|---|---|---|
| Crontab on one host | Yes, if the host is up | Lost silently | No |
| Crontab on every host | No — runs N times | Lost silently | No |
| Leader-elected scheduler | Yes, while the leader is healthy | Depends on implementation | Possible |
| Durable schedule + claimed runs | Yes, by constraint | Policy-driven | Yes |
| Workflow engine timers | Yes | Native | Yes |

A durable workflow engine subsumes this entirely: a timer is a first-class primitive, exactly-once execution and resumption are the engine's job, and a missed window is simply a timer that fires late. Where one is already in use, building a separate scheduler is usually redundant.

**How it fails**

**Scheduling failures**

| Failure | Cause | Fix |
|---|---|---|
| Job ran on every instance | Crontab deployed to all replicas | Durable schedule with claimed runs |
| Job never ran and nobody noticed | Host down at the scheduled minute; no absence monitoring | Overdue-schedule alerting |
| Double charge | Retry without idempotency on the effect | Unique run record; idempotency key on the external call |
| Catch-up burst overwhelms a dependency | All missed windows executed at once | Rate-limited catch-up; policy per schedule |
| Job ran twice in autumn | Local-time schedule during daylight saving | Schedule in UTC, or define DST behaviour explicitly |
| Run stuck in running forever | Worker died; no lease expiry | Lease with heartbeat plus a reaper |
| Cannot tell whether last night's run happened | No execution record | One run row per intended execution, with outcome |

**Limits**

> **Design parameters**
>
> - **Scanner interval** bounds scheduling precision; a one-minute scan means up to a minute of lateness.
> - **Unique constraint on (schedule, scheduled_for)** is what makes exactly-once robust — not leader election.
> - **Lease duration** 2–3× the p99 run time, extended by heartbeat.
> - **Catch-up rate limit** must be set, or a multi-day outage produces a burst against downstream systems.
> - **Schedule in UTC** unless business-hours alignment genuinely requires local time, in which case define DST behaviour.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Durable schedule + claimed runs | General recurring and per-entity work | Machinery to build |
| Workflow engine timers | Where an engine already exists | Engine dependency |
| Managed scheduler service | Simple fleet-wide cron | Limited per-entity scheduling |
| Delay queue | One-off delayed actions | Not for recurrence; limited delay |
| Crontab on a single host | Genuinely non-critical jobs | Single point of failure; no record |
| Time-bucketed polling | Per-entity due checks at scale | Polling cost; coarse precision |

For per-entity schedules at large scale, time-bucketed polling is a common pragmatic design: entities are indexed by their due time bucket, and workers scan only the current bucket. It avoids a row per scheduled run while keeping the scan bounded, at the cost of coarser precision.

**In real systems**

- **Kubernetes CronJobs** provide fleet-level scheduling with a concurrency policy and a missed-deadline setting, exposing exactly the catch-up decision as configuration.
- **Subscription billing systems** universally use per-entity schedules with idempotency keyed on the billing period, because duplicate charges are unrecoverable reputationally.
- **Durable workflow engines** treat timers as first-class, making a three-day wait or a monthly recurrence a single line of workflow code.
- **Quartz and similar schedulers** use a database-backed job store with row locking to achieve exactly-once execution across a cluster.
- **Daylight-saving incidents** are a recurring class of production bug in scheduled financial jobs, which is why most such systems schedule in UTC.

**Common mistakes**

- **Crontabs on autoscaled instances**, running N times or zero times.
- **No execution record**, so nobody can tell whether last night's run happened.
- **Default catch-up policy**, silently skipping a billing window.
- **Monitoring failures but not absence**, missing the worst case entirely.
- **Local-time schedules** for financial jobs, breaking twice a year.
- **Unbounded catch-up**, producing a burst that overwhelms downstream systems after an outage.
- **No lease or reaper**, leaving crashed runs stuck in running forever.

**The staff-level view**

Scheduling looks like infrastructure plumbing and is actually a correctness surface, because the failures are silent and the effects are often financial.

- **Make exactly-once a constraint, not an election.** A unique key on schedule and scheduled time is robust against racing scanners, retries and restarts in a way that leader election is not.
- **Require an explicit catch-up policy per schedule.** The default is wrong for half your jobs, and the wrong default for billing is a customer who is never charged.
- **Alert on absence.** Failure monitoring cannot see a job that never started, and that is the dangerous case — which means something must know what was expected.
- **Schedule in UTC by default**, and treat any local-time schedule as requiring an explicit decision about the two ambiguous days a year.
- **Check whether a workflow engine already covers this.** Where one exists, building a parallel scheduler duplicates durability, exactly-once execution and history for no benefit.

**Go deeper**

A crontab is per-machine, so on a fleet it either runs on every instance or fails silently when the one instance holding it is down — and neither case leaves a record. Distributed scheduling separates the durable schedule from execution: a schedule table describes what should run and when, a scanner finds due entries, and a claim ensures exactly one worker executes each.

Exactly-once comes from a unique constraint on schedule plus scheduled time rather than from electing a single scheduler. Any number of racing scanners attempting to create the same run record collapse to one row because the database enforces it, which survives races, retries and restarts without depending on any process being uniquely alive. Workers then claim a pending run with a conditional update, and a lease plus reaper recovers runs whose worker died.

Two decisions are routinely left at defaults and should not be. The catch-up policy — skip, run once, or run all missed — differs completely between cache warming and billing, and the common default of skipping is financially damaging for the latter. And monitoring must alert on *absence*, since a job that never started produces nothing for failure alerting to see; that requires knowing what was expected, which is precisely what the durable schedule provides.

Distributed scheduling combines three problems that are usually solved separately: durable memory of what should happen, exactly-once execution across a fleet, and a policy for windows that were missed.

**The durable schedule replaces the crontab.** A crontab lives on a machine, is invisible to the rest of the system, cannot be queried or audited, and disappears with the host. A schedule table holds what to run, the recurrence, the next due time, the last completion, and the catch-up policy — all of which can be inspected, disabled, and reasoned about. It also makes per-entity scheduling expressible, which cron fundamentally cannot do: two hundred thousand subscriptions each billing on their own anniversary is a natural schedule table and an impossible crontab.

**Exactly-once should be a constraint, not an election.** Leader election gives one scheduler, which then becomes a single point of failure and still races with itself across restarts. A unique constraint on schedule identifier plus scheduled time means that any number of concurrent scanners, retries or restarts produce exactly one run record, enforced by the database. Workers claim a pending run with a conditional update whose affected row count decides the winner. This is robust in a way coordination is not, because it does not depend on any process being uniquely alive at any moment.

**The catch-up policy is a per-schedule decision.** Skipping missed windows is correct for cache warming and metric snapshots, where a late run is pointless. Running only the most recent is correct for reports and syncs, where currency matters but intermediate runs do not. Running every missed window is correct for billing and accruals, where each window produces output that someone depends on. Most schedulers default to skipping, which is safe for the first case and means an un-billed customer in the last. The secondary hazard is that running all missed windows after a multi-day outage produces a burst, so catch-up needs its own rate limit.

**Absence is the failure that monitoring misses.** Every alerting system reports failures; almost none report non-events, because there is nothing to observe. A job that silently never ran is indistinguishable from a quiet period unless something knows what was expected and asserts it happened — which is exactly what the schedule table enables, via an alert on any schedule whose due time has passed without a completed run. For financial jobs this is the single most important monitor, and it is the one most often absent.

**Time itself is a source of production bugs.** A schedule expressed in local time does not occur on the spring-forward day and occurs twice on the autumn day, because that local time either does not exist or exists twice. Financial and reporting jobs are precisely where this is noticed. Scheduling in UTC removes the ambiguity at the cost of drifting relative to business hours, so local-time schedules should be treated as requiring an explicit, documented decision about both ambiguous days.

**Consider whether the schedule should exist at all.** Where a durable workflow engine is already deployed, timers are a first-class primitive and a separate scheduler duplicates durability, claiming, leases, reapers and history for no benefit. More broadly, a great deal of scheduled work exists because someone found periodic polling easier than building a notification — a nightly reconciliation is often a missing event in disguise. The schedules genuinely worth building are the time-driven ones: billing anniversaries, regulatory windows, retention expiry.

**Prove it — interview questions**

1. **[Basic] Why doesn't cron work on a fleet?**

   <details><summary>Model answer</summary>

   Because a crontab is per-machine. Deployed to every instance, the job runs on all of them; deployed to one, it fails silently whenever that instance is down or replaced during the scheduled minute. Neither case leaves a record, so a job that ran three times and a job that never ran are both invisible. Autoscaling makes it worse, since the set of machines changes without anyone updating the schedule.

   </details>

2. **[Basic] How do you guarantee a scheduled job runs exactly once?**

   <details><summary>Model answer</summary>

   With a unique constraint rather than with coordination. Create one run record per intended execution, keyed on the schedule identifier and the scheduled time, with a unique index on that pair. Any number of racing scanners attempting to create it will collapse to a single row, because the database enforces it. Workers then claim a pending run with a conditional update, so only one can execute it. That is far more robust than electing a single scheduler, because it survives races, retries and restarts without relying on any process being uniquely alive.

   </details>

3. **[Senior] What should happen to windows missed during an outage?**

   <details><summary>Model answer</summary>

   It depends entirely on the schedule, which is why it must be an explicit per-schedule policy rather than a system default. A cache warm should skip, because running it late is pointless. A report or a sync should run once on recovery, since being current matters but the intermediate runs do not. A billing run should execute every missed window, because each one produces output that a customer depends on and skipping means someone is never charged. Most schedulers default to skipping, which is safe for the first case and financially damaging for the last.

   </details>

4. **[Senior] Why is alerting on absence harder than alerting on failure?**

   <details><summary>Model answer</summary>

   Because a job that never started produces nothing to alert on. Error monitoring, exception tracking and log alerting all require something to have happened. Detecting absence requires the system to know what was expected — which is exactly what a durable schedule provides — and to assert that each expected run occurred. Without that, a billing job that silently never executed looks identical to a quiet night, and the discovery comes from a customer noticing they were not charged.

   </details>

5. **[Staff] Design scheduling for subscription billing across 200,000 subscriptions.**

   <details><summary>Model answer</summary>

   Per-subscription schedule records rather than one cron job scanning everything, so each subscription has its own next-due time and the load spreads naturally across the day instead of spiking at midnight. Exactly-once comes from a unique constraint on subscription and billing period, so racing scanners, retries and restarts all collapse to a single run record — and the charge itself carries that run id as an idempotency key, so even a re-executed run cannot double-charge. The catch-up policy is to run all missed windows, since a weekend outage must still bill everyone whose anniversary fell in it, but rate-limited so Monday morning does not send three days of billing at the payment provider at once. Crashed runs are recovered by lease expiry and a reaper. And the critical monitoring is an overdue-schedule alert, because a subscription whose next billing time passed hours ago is the failure that failure alerting cannot see.

   </details>

6. **[Principal] When should you not build a scheduler at all?**

   <details><summary>Model answer</summary>

   When a durable workflow engine is already in use, because it subsumes the entire problem: timers are a first-class primitive, exactly-once execution and resumption are the engine's responsibility, missed windows are simply timers firing late, and the execution history is already recorded. Building a separate scheduler alongside one duplicates durability, claiming, leases, reapers and history for no benefit, and creates two places where scheduled work can live. More generally, I would push back on scheduled work as a design choice where an event-driven alternative exists — a nightly reconciliation often exists because nobody built the event that would make it unnecessary, and polling on a schedule is frequently a workaround for a missing notification. The scheduled jobs worth building are the ones that are genuinely time-driven: billing anniversaries, regulatory reporting windows, retention expiry — not the ones that are periodic because it was easier than being reactive.

   </details>

---
