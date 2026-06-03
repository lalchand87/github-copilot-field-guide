# Case Study 20 — Distributed Job Scheduler

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Leader Election, At-Least-Once vs Exactly-Once, Job State Machine.

---

## Business Context

A distributed job scheduler executes recurring and one-off tasks across a fleet of workers. For a payment platform: daily reconciliation jobs, monthly statement generation, interest computation, fraud model retraining, S3 data archiving, and compliance report generation. A missed job can mean a delayed customer statement or a failed regulatory submission.

The hardest problem is not scheduling — it is ensuring a job runs **exactly once** even when the scheduler has multiple instances for HA, and even when workers crash mid-execution.

**Business goals driving architecture:**
- Every scheduled job runs at least once (no silent misses)
- Idempotent jobs run exactly once (or at-least-once for idempotent jobs)
- Scheduler itself is HA: no single point of failure
- Workers can be added or removed dynamically (elastic compute)
- Job history queryable: what ran when, how long, success/failure, logs

---

## Recognition Framework

### Signals from the Problem
```
- Multiple scheduler instances needed for HA → leader election required
- Worker crash mid-job → job must be retried → at-least-once delivery
- At-least-once + idempotent job design = effectively exactly-once
- Cron expressions for recurring jobs → need a cron parser
- Job priorities: compliance report > analytics report
- Job dependencies: "run reconciliation AFTER data pipeline completes"
- Backpressure: more jobs than workers → queue with priority
- Visibility: operators need to see job status, history, and manually trigger/cancel
```

### Leader Election — Why Required

```
Without leader election:
  3 scheduler instances all evaluate "daily_reconciliation should run at 00:00"
  All 3 trigger the job simultaneously
  Result: 3 reconciliation jobs run in parallel → data corruption / duplicate work

With leader election:
  One scheduler instance is the "leader" at any time
  Only the leader evaluates cron expressions and triggers jobs
  If leader fails: another instance is elected leader within seconds
  Result: exactly one scheduler triggers each job

Leader election options:

Option A: PostgreSQL advisory locks
  SELECT pg_try_advisory_lock(12345)  → returns true for one instance; false for others
  Winning instance is leader; renews lock every 30s
  On release or timeout: other instances race for lock
  Pros: no additional infrastructure (uses existing PostgreSQL)
  Cons: lock granularity is coarse; single PostgreSQL = SPOF for leader election

Option B: Redis SETNX (SET if Not eXists)
  SET leader-lock {instance_id} NX EX 30
  Winner is leader for 30s; must renew every 15s
  On Redis failure: all instances may fail to renew → no leader → jobs pause
  Pros: simple; Redis already in use
  Cons: Redis failure = no leader; clock skew can cause brief multi-leader

Option C: etcd / ZooKeeper / Consul
  Purpose-built for distributed coordination
  etcd: Raft consensus; distributed lease with TTL
  Pros: purpose-built; correct even under network partition (Raft guarantees)
  Cons: additional infrastructure to operate

CHOSEN: PostgreSQL advisory locks for simplicity
  Most payment platforms already have PostgreSQL as core infrastructure
  Adding etcd/ZooKeeper for scheduler alone is operational overhead
  If PostgreSQL goes down: scheduler pauses (acceptable; PostgreSQL downtime ≈ payment downtime)
```

### At-Least-Once vs Exactly-Once

```
The fundamental constraint: distributed systems cannot guarantee exactly-once delivery
without careful design.

At-Least-Once Delivery:
  Job execution model: claim job → execute → mark complete
  If worker crashes after execute but before mark complete:
    → Job still shows as "RUNNING" or "CLAIMED"
    → Scheduler re-assigns after timeout
    → Job runs again → AT LEAST ONCE

Exactly-Once Execution (requires idempotent job design):
  Option A: Idempotent job design
    Job itself is designed to be safe to run multiple times:
    "Compute merchant_id=X daily_totals for 2024-01-15"
    → Uses UPSERT → second run produces same result → safe
    Design principle: EVERY scheduled job should be idempotent

  Option B: Distributed transaction (expensive, complex)
    Job execution + mark-complete in the same transaction
    Not practical for jobs that call external systems

  Option C: Exactly-once via dedup key
    Job generates output with deterministic key (date + job_type + target)
    Output store rejects duplicate with same key (idempotent write)
    → Even if job runs twice, output appears once

CHOSEN APPROACH: At-least-once delivery + idempotent job design
  Scheduler guarantees at-least-once (simpler; achievable)
  Job developers guarantee idempotency (their responsibility)
  Combined = effective exactly-once without distributed transaction complexity
```

---

## Problem Statement

Distributed job scheduler for a payment platform. Executes cron-scheduled and one-off jobs across a worker pool. HA scheduler (leader election). At-least-once execution with idempotent job design. Job history, prioritization, and dependency management.

---

## Functional Requirements
- Cron-scheduled jobs (standard cron expressions)
- One-off jobs (submit and run once)
- Job priorities (HIGH, NORMAL, LOW)
- Job dependencies (job B runs only after job A succeeds)
- At-least-once execution (re-run on worker failure)
- Idempotency key per job execution (prevent duplicate side effects)
- Job history: status, duration, logs, output artifacts
- Manual controls: trigger now, cancel, pause, resume
- Worker management: register/deregister workers; heartbeat

## Non-Functional Requirements
- Scheduler HA: **failover < 10 seconds**
- Job dispatch latency: **< 5 seconds** from scheduled time to worker start
- Scale: **10,000 jobs/day**, **100 concurrent workers**
- Job timeout detection: worker failure detected within **60 seconds**

---

## Capacity Estimation
```
Jobs:
  10,000 jobs/day = ~7 jobs/sec
  Concurrent jobs: 100 workers × 1 job each = 100 concurrent
  Job metadata: ~1 KB per job execution record
  Daily storage: 10,000 × 1 KB = 10 MB/day → negligible

Workers:
  100 workers × heartbeat every 30s = ~3 heartbeats/sec → trivial load

Scheduler state:
  10,000 job definitions × cron expressions + metadata = ~10 MB in PostgreSQL
  Upcoming job queue: next 1 hour's jobs = ~420 jobs (7/sec × 3600) → trivial
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| PostgreSQL Advisory Lock | Leader election; simple; no extra infrastructure |
| Job State Machine | PENDING → CLAIMED → RUNNING → COMPLETED / FAILED |
| Heartbeat + Timeout Detection | Detect crashed workers; reclaim their jobs |
| Priority Queue | HIGH priority jobs dispatched before NORMAL and LOW |
| Idempotency Key per Execution | Prevent duplicate side effects on retry |
| Job Dependency Graph | DAG: job B waits for job A before becoming PENDING |
| Dead Letter (FAILED) Queue | After N retries: mark as DEAD; alert; no more retries |

---

## Data Model

```sql
-- Job definitions (what jobs exist and their schedule)
CREATE TABLE job_definitions (
  job_id           UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
  name             VARCHAR(256) NOT NULL UNIQUE,
  job_type         VARCHAR(64)  NOT NULL,     -- PAYMENT_RECONCILIATION / STATEMENT_GEN / etc.
  cron_expression  VARCHAR(64),               -- "0 1 * * *" = daily at 01:00 UTC; null for one-off
  priority         VARCHAR(16)  DEFAULT 'NORMAL',  -- HIGH / NORMAL / LOW
  max_retries      INTEGER      DEFAULT 3,
  timeout_seconds  INTEGER      DEFAULT 3600,  -- 1 hour default
  payload          JSONB,                       -- job-specific config (idempotent keys, params)
  is_active        BOOLEAN      DEFAULT TRUE,
  created_at       TIMESTAMP    DEFAULT NOW(),
  updated_at       TIMESTAMP    DEFAULT NOW()
);

-- Job executions (instances of a job definition running)
CREATE TABLE job_executions (
  execution_id     UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
  job_id           UUID         REFERENCES job_definitions(job_id),
  idempotency_key  VARCHAR(256) UNIQUE,      -- prevents duplicate scheduling of same instance
  status           VARCHAR(32)  NOT NULL DEFAULT 'PENDING',
                                             -- PENDING / CLAIMED / RUNNING / COMPLETED / FAILED / DEAD
  priority         VARCHAR(16)  DEFAULT 'NORMAL',
  scheduled_for    TIMESTAMP    NOT NULL,    -- when this execution is due
  claimed_at       TIMESTAMP,               -- when a worker picked it up
  started_at       TIMESTAMP,
  completed_at     TIMESTAMP,
  worker_id        VARCHAR(64),             -- which worker is executing this
  attempt_num      INTEGER      DEFAULT 1,
  error_message    TEXT,
  output_artifact  TEXT,                    -- S3 URL of job output (if any)
  created_at       TIMESTAMP    DEFAULT NOW()
);

-- Indexes for efficient scheduler queries
CREATE INDEX idx_executions_status_scheduled ON job_executions(status, scheduled_for)
  WHERE status = 'PENDING';                 -- dispatcher polls this index
CREATE INDEX idx_executions_worker_claimed ON job_executions(worker_id, claimed_at)
  WHERE status IN ('CLAIMED', 'RUNNING');   -- timeout detection queries this

-- Worker registry
CREATE TABLE workers (
  worker_id        VARCHAR(64)  PRIMARY KEY,
  hostname         VARCHAR(256),
  status           VARCHAR(16)  DEFAULT 'ACTIVE',  -- ACTIVE / DRAINING / OFFLINE
  capacity         INTEGER      DEFAULT 5,          -- max concurrent jobs
  current_load     INTEGER      DEFAULT 0,
  last_heartbeat   TIMESTAMP    DEFAULT NOW(),
  registered_at    TIMESTAMP    DEFAULT NOW()
);

-- Job dependencies (DAG edges)
CREATE TABLE job_dependencies (
  job_id           UUID,                            -- dependent job
  depends_on_job_id UUID,                           -- must complete first
  PRIMARY KEY (job_id, depends_on_job_id)
);
```

---

## Job State Machine

```
PENDING ──────────────────────────────────────────────► CLAIMED
  │                                                      │
  │ (scheduled_for <= now; no worker yet)                │ (worker picks up job)
  │                                                      ▼
  │                                                  RUNNING
  │                                                  │       │
  │                                     job success  │       │  job fails / timeout
  │                                                  ▼       ▼
  │                                             COMPLETED  FAILED
  │                                                          │
  │                                              attempt_num < max_retries
  │                                                          │
  │                             ◄────────────── back to PENDING (with backoff)
  │                                                          │
  │                                              attempt_num >= max_retries
  │                                                          ▼
  │                                                        DEAD
  │                                                   (alert on-call)
  │
  │ (job skipped: dependency not met; DND window; etc.)
  ▼
SKIPPED

Timeout detection:
  RUNNING job with started_at + timeout_seconds < now → mark as FAILED → retry logic
  CLAIMED job with claimed_at + 60s < now → worker died before starting → back to PENDING

Heartbeat:
  Worker sends heartbeat every 30s: UPDATE workers SET last_heartbeat = now()
  Scheduler checks: workers with last_heartbeat < now() - 90s → mark OFFLINE
  Their running jobs: moved back to PENDING for reassignment
```

---

## API Design

```
Management API:

POST /api/v1/jobs                          → Create job definition
  Body: { name, job_type, cron_expression, priority, payload, max_retries }
  Response 201: { job_id, next_run_at }

POST /api/v1/jobs/{job_id}/trigger         → Trigger one-off execution now
  Response 202: { execution_id }

GET /api/v1/jobs/{job_id}/executions       → Execution history
  Response 200: { executions: [...] }

GET /api/v1/executions/{execution_id}      → Execution status + logs
  Response 200: { execution_id, status, started_at, completed_at, logs_url }

POST /api/v1/executions/{execution_id}/cancel
  Response 200: (only if PENDING; running jobs must handle cancellation signal)

GET /api/v1/workers                        → Active worker list + load
  Response 200: { workers: [{ worker_id, hostname, current_load, capacity, last_heartbeat }] }

Worker API (called by workers; not by operators):

POST /api/v1/workers/register
  Body: { worker_id, hostname, capacity }
  Response 200: { registered: true }

POST /api/v1/workers/{worker_id}/heartbeat
  Response 200: { next_heartbeat_in: 30 }

POST /api/v1/executions/{execution_id}/claim
  Body: { worker_id }
  Response 200: { claimed: true, idempotency_key, payload }
  Response 409: { claimed: false }  → another worker claimed it first (use SELECT FOR UPDATE SKIP LOCKED)

POST /api/v1/executions/{execution_id}/complete
  Body: { status: "COMPLETED", output_artifact: "s3://..." }

POST /api/v1/executions/{execution_id}/fail
  Body: { error_message: "Connection timeout to DB" }
  → Scheduler re-queues with backoff based on attempt_num
```

---

## High-Level Architecture

```
Scheduler Service (N instances; one is leader at a time)
  │
  ├── Leader Election Thread (runs in all instances):
  │     Every 15s: try to acquire PostgreSQL advisory lock
  │     pg_try_advisory_lock(SCHEDULER_LOCK_ID)
  │     Winner: set is_leader = true; proceeds with scheduling work
  │     Loser: set is_leader = false; standby; only does heartbeat/worker monitoring
  │
  ├── Cron Evaluator (leader only; runs every 30s):
  │     Load all active job definitions with cron_expression
  │     For each: compute next_run_time using cron parser
  │     If next_run_time <= now AND no PENDING/RUNNING execution for this slot:
  │       INSERT INTO job_executions with status=PENDING and idempotency_key
  │       Idempotency key: {job_id}:{scheduled_for_epoch}  → prevents double-insert
  │
  ├── Job Dispatcher (leader only; polls every 5s):
  │     SELECT * FROM job_executions
  │     WHERE status = 'PENDING' AND scheduled_for <= now
  │     ORDER BY priority DESC, scheduled_for ASC
  │     LIMIT 100
  │     FOR EACH: find available worker with capacity > current_load
  │     Dispatch: notify worker via internal API or worker polling
  │
  ├── Timeout Monitor (leader only; runs every 30s):
  │     Find stuck jobs:
  │       RUNNING: started_at + timeout_seconds < now → FAILED
  │       CLAIMED: claimed_at + 60s < now → back to PENDING
  │     Find dead workers:
  │       workers with last_heartbeat < now - 90s → OFFLINE
  │       Their jobs: RUNNING/CLAIMED → back to PENDING
  │
  └── Dependency Manager (leader only):
        When job A completes: check job_dependencies for jobs that depend on A
        If all dependencies met: update status from BLOCKED → PENDING

Worker (N instances; stateless; register with scheduler)
  │
  ├── On startup: POST /api/v1/workers/register
  ├── Heartbeat loop: POST /api/v1/workers/{id}/heartbeat every 30s
  ├── Job polling OR listening for push notification:
  │     Poll: GET /api/v1/executions?worker_id={id}&status=DISPATCHED
  │     Push: SSE or webhook from scheduler (same pattern as Feature Flag CS08)
  ├── On job assignment:
  │     Claim: POST /api/v1/executions/{id}/claim (SELECT FOR UPDATE SKIP LOCKED)
  │     Execute: run job code; log to structured log (sent to ELK/Loki)
  │     Heartbeat during execution: POST every 30s (proves job is still alive)
  │     On success: POST /api/v1/executions/{id}/complete
  │     On failure: POST /api/v1/executions/{id}/fail
  └── Graceful shutdown: DRAINING state → finish current jobs → deregister

PostgreSQL:
  job_definitions, job_executions, workers, job_dependencies
  Advisory lock for leader election
  SKIP LOCKED for race-free job claiming
```

---

## Detailed Components

### Job Claiming with SKIP LOCKED (Preventing Double-Execution)
```sql
-- Worker claims a pending job atomically
-- SKIP LOCKED: skips rows locked by other workers (no waiting; no deadlock)
BEGIN;

SELECT execution_id, job_id, idempotency_key, payload, priority
FROM job_executions
WHERE status = 'PENDING'
  AND scheduled_for <= NOW()
  AND priority = 'HIGH'  -- try HIGH first; fallback to NORMAL and LOW
ORDER BY scheduled_for ASC
LIMIT 1
FOR UPDATE SKIP LOCKED;

-- If a row is returned:
UPDATE job_executions
SET status = 'CLAIMED',
    worker_id = $worker_id,
    claimed_at = NOW()
WHERE execution_id = $execution_id;

COMMIT;

-- SKIP LOCKED ensures: if 100 workers poll simultaneously for jobs,
-- each worker gets a DIFFERENT job (or no job if none available)
-- No worker gets the same job (no double-execution at claim time)
-- If worker crashes after CLAIMED but before RUNNING:
--   Timeout monitor detects: claimed_at + 60s < now → back to PENDING
```

### Cron Expression Evaluation + Idempotent Scheduling
```java
public void evaluateCronJobs() {
    if (!isLeader) return;  // only leader evaluates

    List<JobDefinition> jobs = jobDefRepo.findAllActive();
    Instant now = Instant.now();

    for (JobDefinition job : jobs) {
        if (job.getCronExpression() == null) continue;

        // Compute when this job was last due
        CronExpression cron = CronExpression.parse(job.getCronExpression());
        Instant nextDue = cron.previousFireTime(now);  // last time it should have run

        if (nextDue == null || nextDue.isBefore(now.minus(Duration.ofHours(1)))) {
            continue;  // too far in the past; skip (avoid catch-up storm on restart)
        }

        // Idempotency key: job_id + the specific scheduled slot
        String idempotencyKey = job.getJobId() + ":" + nextDue.getEpochSecond();

        // INSERT IF NOT EXISTS — safe to call multiple times (idempotent)
        try {
            jobExecRepo.insertIfNotExists(JobExecution.builder()
                .jobId(job.getJobId())
                .idempotencyKey(idempotencyKey)
                .status(JobStatus.PENDING)
                .scheduledFor(nextDue)
                .priority(job.getPriority())
                .build());
        } catch (DuplicateKeyException e) {
            // Already scheduled (idempotency key conflict) → skip silently
        }
    }
}
// This method is safe to call multiple times in the same second (idempotent)
// Works correctly even if two scheduler instances temporarily both think they are leader
```

### Retry with Exponential Backoff
```java
public void handleJobFailure(String executionId, String errorMessage) {
    JobExecution exec = execRepo.findById(executionId);
    JobDefinition def = defRepo.findById(exec.getJobId());

    if (exec.getAttemptNum() >= def.getMaxRetries()) {
        // No more retries: mark as DEAD
        execRepo.updateStatus(executionId, JobStatus.DEAD, errorMessage);
        alertService.page(
            "Job " + def.getName() + " has permanently failed after " +
            def.getMaxRetries() + " attempts. Execution: " + executionId
        );
        return;
    }

    // Schedule retry with exponential backoff
    int attemptNum = exec.getAttemptNum() + 1;
    long backoffSeconds = (long) Math.pow(2, attemptNum) * 30;  // 60s, 120s, 240s
    Instant retryAt = Instant.now().plusSeconds(backoffSeconds);

    // Insert NEW execution record for the retry attempt
    execRepo.insert(JobExecution.builder()
        .jobId(exec.getJobId())
        .idempotencyKey(exec.getIdempotencyKey() + ":retry:" + attemptNum)
        .status(JobStatus.PENDING)
        .scheduledFor(retryAt)
        .attemptNum(attemptNum)
        .build());

    log.info("Scheduled retry {} for job {} at {}", attemptNum, exec.getJobId(), retryAt);
}
```

### Job Dependency Resolution
```sql
-- Check if all dependencies for job B are satisfied
-- Called when any job completes (to unblock dependent jobs)

WITH completed_deps AS (
  SELECT jd.job_id, COUNT(*) as completed_count
  FROM job_dependencies jd
  JOIN job_executions je
    ON je.job_id = jd.depends_on_job_id
   AND je.status = 'COMPLETED'
   AND je.scheduled_for >= CURRENT_DATE  -- only today's run counts
  GROUP BY jd.job_id
),
required_deps AS (
  SELECT job_id, COUNT(*) as required_count
  FROM job_dependencies
  GROUP BY job_id
)
SELECT r.job_id
FROM required_deps r
JOIN completed_deps c ON r.job_id = c.job_id
WHERE r.required_count = c.completed_count;
-- Returns job_ids where all dependencies are now satisfied
-- → Scheduler moves these from BLOCKED to PENDING
```

---

## Scaling Strategy
```
Phase 1 (0 → 100 jobs/day):
  Single scheduler instance (no HA needed at this scale)
  PostgreSQL for all state
  Workers = application threads (same process as scheduler)

Phase 2 (100 → 1,000 jobs/day):
  Separate scheduler and worker processes
  Multiple workers (horizontal scaling)
  PostgreSQL advisory lock for HA scheduler

Phase 3 (1K → 10K jobs/day):
  Multiple scheduler instances (1 leader + 2 standbys)
  Worker pool (100 workers)
  Priority queues in PostgreSQL (priority index on job_executions)
  Web UI for job management (Airflow-style)

Phase 4 (10K → 100K jobs/day):
  Separate job queue into Redis (for dispatcher hot path)
  PostgreSQL for durable state + history
  Redis for real-time worker availability and dispatch
  Webhook/SSE for push notification to workers (instead of polling)

Phase 5 (Uber/Airflow scale):
  Use battle-tested open source: Apache Airflow (Python DAGs),
  Temporal (durable workflow execution), or Quartz (Java)
  Build on top of them instead of rolling custom scheduler
```

---

## Reliability Strategy
- **Leader HA**: PostgreSQL advisory lock released on crash (PostgreSQL connection drops); standby acquires within 15s
- **Worker crash detection**: heartbeat + timeout (90s no heartbeat → worker offline → jobs re-queued)
- **Job re-queuing**: RUNNING job with timed-out worker → back to PENDING → re-dispatched
- **Idempotency**: every job implements idempotent logic; re-run on retry produces same result
- **Job state durability**: PostgreSQL (not in-memory); scheduler restart picks up where it left off
- **DEAD alert**: any job reaching DEAD status → immediate on-call page + Slack alert

---

## Security Considerations
- **Job payload**: may contain sensitive config (DB credentials, API keys) → store in Secrets Manager; payload contains reference, not value
- **Worker authentication**: workers authenticate with mTLS to scheduler API (same as gateway-to-backend in Case 05)
- **Audit log**: every job execution, status change, and manual trigger → Case 10 audit service
- **Job isolation**: workers run in separate containers/pods; one job cannot access another job's state
- **Rate limiting on manual trigger API**: prevent accidental or malicious job flooding

---

## Observability
```
Key metrics:
  jobs_scheduled_total              (counter; by job_type; should match expected schedule)
  jobs_completed_total              (counter; by job_type, priority)
  jobs_failed_total                 (counter; alert if > 0 for critical jobs)
  jobs_dead_total                   (counter; IMMEDIATE alert; max_retries exhausted)
  job_execution_duration_seconds    (histogram; by job_type; alert on P95 spike)
  job_dispatch_latency_seconds      (scheduled_for → claimed_at; target < 5s)
  worker_utilization                (current_load / capacity per worker; alert if > 90%)
  pending_jobs_queue_depth          (by priority; alert if HIGH queue > 10)
  leader_election_changes_total     (counter; frequent changes = instability)
  timeout_reclaims_total            (counter; jobs reclaimed due to worker timeout)

Alerts:
  jobs_dead_total > 0              → CRITICAL: permanent failure; manual intervention
  high_priority_queue_depth > 10   → Worker capacity insufficient; scale out
  leader_election_changes > 5/hr   → Scheduler instability; PostgreSQL connection issue
  job_dispatch_latency > 30s       → Dispatcher stuck; investigate leader
  worker_utilization > 90%         → All workers busy; new jobs will queue
  timeout_reclaims > 10/hr         → Workers crashing frequently; investigate

Dashboard:
  Gantt-style view: all jobs and their execution timeline
  Queue depth by priority (real-time)
  Worker pool utilization (per worker, rolling 1h)
  Failed jobs timeline (when, which job, error)
```

---

## Failure Scenarios
| Failure | Impact | Mitigation |
|---|---|---|
| Leader scheduler crashes | No new jobs dispatched until failover | PostgreSQL lock released; standby acquires within 15s |
| Worker crashes mid-job | Job stuck in RUNNING | Timeout monitor reclaims after 60s; re-queued |
| PostgreSQL down | No scheduling; no job dispatch | PostgreSQL is the SPOF; HA PostgreSQL (Case 03/07 patterns) |
| All workers down | Jobs accumulate in PENDING queue | Alert fires; spin up new workers; jobs processed when workers recover |
| Job causes DB deadlock | Job fails repeatedly → DEAD | Retry with backoff; job code must handle deadlock and retry internally |
| Clock skew (server time out of sync) | Cron job runs at wrong time | NTP sync on all scheduler instances; use UTC always |
| Dependency cycle in DAG | Circular dep → jobs blocked forever | Validate DAG on job definition creation; reject cycles |

---

## Chaos Testing
```
Experiment 1: Leader Scheduler Crash
  Inject:    Kill the leader scheduler process (SIGKILL)
  Expected:  PostgreSQL advisory lock released immediately (connection drop releases lock)
  Expected:  Standby scheduler acquires lock within 15s
  Expected:  Pending jobs dispatched by new leader within 20s
  Expected:  In-flight jobs continue on workers (workers don't need leader during execution)
  Red flag:  No failover within 30s (standbys not trying to acquire lock)

Experiment 2: Worker Crash During Job Execution
  Inject:    Kill a worker process that is RUNNING a job
  Expected:  Worker heartbeat stops; timeout monitor detects after 90s
  Expected:  Job moved back to PENDING status
  Expected:  Another available worker claims the job
  Expected:  Job runs again (at-least-once); idempotent design means same result
  Red flag:  Job stays in RUNNING state indefinitely (timeout monitor not working)

Experiment 3: Double Scheduling Prevention
  Inject:    Temporarily make two scheduler instances both believe they are leader
             (network partition where advisory lock appears released to both)
  Expected:  Both attempt INSERT for same idempotency_key → one gets UNIQUE violation
  Expected:  Only one job execution record created
  Expected:  Only one worker executes the job
  Red flag:  Two job executions for same cron slot (duplicate run)

Experiment 4: Job Dependency Chain
  Inject:    3-job chain: A → B → C; kill worker executing A mid-job
  Expected:  A fails → retries → eventually completes (or goes DEAD with alert)
  Expected:  B remains BLOCKED until A succeeds
  Expected:  C remains BLOCKED until B succeeds
  Expected:  If A goes DEAD: B and C remain BLOCKED forever; alert fires for DEAD + BLOCKED
  Red flag:  B starts executing despite A having failed (dependency not enforced)

Experiment 5: Priority Queue Starvation Prevention
  Inject:    Fill queue with 1,000 LOW priority jobs; then add 10 HIGH priority jobs
  Expected:  HIGH priority jobs dispatched immediately (before any LOW jobs)
  Expected:  LOW jobs execute after all HIGH jobs complete (or as workers free up)
  Expected:  LOW jobs eventually execute (no starvation after reasonable wait time)
  Red flag:  HIGH jobs starve behind LOW queue (priority ordering not working)
```

---

## Monthly Cost Estimate
```
Scale: 10,000 jobs/day, 100 concurrent workers

Scheduler instances (3 for HA; only 1 active as leader):
  3 × t3.medium ($34/mo)                              = $102/mo

Workers (100 instances; assuming mix of small and large):
  100 × m5.large ($75/mo)                             = $7,500/mo
  (worker cost dominated by actual job computation, not scheduler overhead)

PostgreSQL:
  db.r6g.large ($200/mo) + read replica               = $400/mo
  (job_executions grows: 10K/day × 1 KB × 365 = ~3.5 GB/year; tiny)

Redis (real-time dispatch; optional at this scale):
  cache.t3.medium ($30/mo)                            = $30/mo

Monitoring + logging:
  Loki/Elasticsearch for job logs                     = $200/mo
─────────────────────────────────────────────────────────────────
Total scheduler infrastructure: ~$8,232/month

Note: Worker cost ($7,500) is NOT the scheduler cost — it is the cost of
the compute that actually runs the jobs. The scheduler itself costs $732/mo.
This is why build-vs-buy matters:
  Temporal.io (managed): $0.025/workflow execution × 10K/day × 30 = $7,500/mo
  Self-built: $732/mo scheduler overhead (workers are the same either way)
  At 10K jobs/day: self-built is cheaper; at 1M/day: managed may be worth it
```

---

## Migration Story

### Current State
Cron jobs defined in Linux crontab on a single server. Server goes down → all jobs miss silently. No job history. No retry. Multiple teams adding cron jobs = crontab conflicts.

### Migration Plan

```
Step 1: Build scheduler with single instance + DB job log (no HA yet)
  Action:
    Build job_definitions and job_executions tables
    Migrate crontab entries to job_definitions table
    Replace crontab with scheduler process that reads DB
    Single instance only; no leader election yet
  Monitor:
    Job execution history now visible (was invisible in crontab)
    alert if any job does not run within 5 min of scheduled time
  Rollback: restore crontab; stop scheduler process

Step 2: Add HA with leader election (zero downtime)
  Action:
    Deploy 3 scheduler instances
    Add PostgreSQL advisory lock for leader election
    Deploy 2 standbys alongside running primary
    Test failover: kill leader; verify standby takes over within 15s
  Monitor:
    leader_election_changes_total (should be 0 normally; 1 on intentional failover)
  Rollback: stop extra instances; single scheduler continues

Step 3: Add distributed workers (zero downtime)
  Action:
    Extract job execution from scheduler process into separate worker processes
    Workers register and poll for jobs via SKIP LOCKED
    Scheduler dispatches; workers execute
    Test: kill a worker mid-job; verify timeout detection and re-queue
  Monitor:
    worker_utilization (should be < 80% under normal load)
    timeout_reclaims_total (should be near 0 under normal conditions)
  Rollback: revert to in-process execution in scheduler

Step 4: Add priority queues and dependencies (zero downtime)
  Action:
    Add priority field to job_definitions; update dispatch query to ORDER BY priority
    Add job_dependencies table; update dependency resolution logic
    Migrate existing implicit dependencies to explicit DAG entries
  Monitor:
    High-priority jobs always dispatched before LOW (verify with test scenario)
  Rollback: remove priority ordering from dispatch query; ignore dependency table

Step 5: Add monitoring, alerting, and web UI (additive, zero risk)
  Action:
    Emit metrics to Prometheus (Case 17)
    Build simple web UI (job list, execution history, manual trigger, cancel)
    Configure alerts for DEAD jobs and dispatch latency
  Rollback: not needed (additive only)
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → PostgreSQL: job_definitions (what jobs exist) and job_executions (all runs, all statuses).
  → Redis (if used): dispatcher hot path; cached from PostgreSQL; expendable.
  → Workers: stateless; no local persistent state; all state in PostgreSQL.
  → If PostgreSQL is lost: job history lost; job_definitions can be re-entered from code.

□ Consistency model?
  → Job scheduling: strong (UNIQUE idempotency_key prevents double-scheduling).
  → Job claiming: strong (SELECT FOR UPDATE SKIP LOCKED; atomic claim).
  → Leader election: strong (PostgreSQL advisory lock; one winner at a time).
  → Worker heartbeat: eventual (90s detection window for crashed workers).

□ Failure mode?
  → Leader crash: standby acquires lock in 15s; brief scheduling gap acceptable.
  → Worker crash: timeout reclaim in 60s; job re-queued; at-least-once execution.
  → PostgreSQL down: ALL scheduling stops (not degraded; completely paused).
    PostgreSQL is the SPOF — intentional; acceptable because it's always HA anyway.
  → Job permanent failure (DEAD): immediate alert; manual investigation required.

□ At-least-once vs exactly-once:
  → Scheduler guarantees: at-least-once (re-run on worker failure).
  → Job developers guarantee: idempotency (safe to run multiple times).
  → Combined: effective exactly-once without distributed transaction complexity.
  → Verify: every job in the system must have an idempotency strategy documented.

□ Preventing the hardest failures:
  → Double scheduling: UNIQUE idempotency_key on (job_id, scheduled_for).
  → Double claiming: SELECT FOR UPDATE SKIP LOCKED.
  → Stale leader: advisory lock TTL = connection lifetime (no manual expiry).
  → Missing job run: cron evaluator looks back 1 hour; catches up if restarted.

□ SCC LENS:
  STATE: PostgreSQL (job definitions, execution history, worker registry — authoritative)
  COORDINATION: advisory lock (leader election), SKIP LOCKED (worker claiming)
  CONCENTRATION: PostgreSQL is both the coordination point AND the state store → HA PostgreSQL required
```

---

## Staff Engineer Discussion Points

**"What's the difference between a job scheduler and a workflow orchestrator like Temporal?"**
A job scheduler dispatches isolated jobs: each job is independent, short, and stateless. Temporal orchestrates workflows: multi-step processes where step N depends on step N-1's output, steps can wait for external events (human approval, webhook), and state is durable across machine restarts. Temporal is the right tool when: jobs have complex conditional branching, long-running processes (hours/days), need to wait for external signals, or require compensation on failure (saga pattern). Our scheduler is right when: jobs are simple, independent, cron-triggered, and under a few hours in duration. For payment platform: "monthly statement generation" → scheduler. "New user onboarding workflow (email → KYC → account creation → welcome offer)" → Temporal.

**"How do you handle a job that runs for 8 hours — how do you know it's still alive vs stuck?"**
Two mechanisms: (1) worker heartbeat: worker sends heartbeat every 30s during execution (not just at start); scheduler resets the "last seen alive" timer on each heartbeat. Job is declared stuck only if heartbeat missed for 90s — not based on total duration. (2) Job-level progress reporting: long-running jobs emit progress checkpoints ("processed 50K of 1M records") to the scheduler. This enables: dashboard shows progress, not just "RUNNING"; scheduler can distinguish "stuck at 50K for 2 hours" from "actively processing". Set timeout_seconds appropriately per job type — reconciliation might need 4h timeout, not 1h default.

**"How would you design this to support at-most-once execution (the opposite requirement)?"**
At-most-once means: if there's any doubt whether the job ran, do NOT retry. Use case: jobs with non-idempotent side effects that you cannot make idempotent (e.g., sending a physical cheque, deducting a unique fee). Implementation: (1) Mark job as RUNNING before executing (not after); (2) NEVER move a RUNNING job back to PENDING (even on worker crash); (3) Alert on timeout; require manual investigation and explicit re-trigger; (4) Worker crash → job moves to NEEDS_REVIEW state, not PENDING. Trade-off: jobs may be permanently missed on worker crash. Usually the right answer is: make the job idempotent instead of choosing at-most-once.

---

## Cheat Sheet Tie-in

```
PROBLEM SIGNALS → PATTERNS:
  "Multiple schedulers needed for HA"              → Leader Election (advisory lock)
  "Worker crash must not lose job"                 → Heartbeat + Timeout Reclaim
  "Two workers must not run same job"              → SELECT FOR UPDATE SKIP LOCKED
  "Job runs twice on retry"                        → Idempotency Key + idempotent job design
  "Job B must wait for job A"                      → Job Dependency DAG

LEADER ELECTION OPTIONS:
  PostgreSQL advisory lock: simple; uses existing DB; good for most cases
  Redis SETNX + TTL:        simpler but Redis = SPOF for election
  etcd/ZooKeeper:           purpose-built; complex; justified for many distributed components

AT-LEAST-ONCE + IDEMPOTENT = EFFECTIVELY EXACTLY-ONCE:
  Scheduler: guarantees at-least-once (re-run on failure)
  Job developer: guarantees idempotency (UPSERT, dedup key, deterministic output)
  Combined: same result whether job runs 1 or 5 times

JOB STATE MACHINE (memorize):
  PENDING → CLAIMED → RUNNING → COMPLETED
                   ↘           ↘
                   (timeout)    → FAILED → (retry) → PENDING
                                         → (max retries) → DEAD → ALERT

SKIP LOCKED PATTERN (prevents double-execution at claim time):
  SELECT ... FOR UPDATE SKIP LOCKED
  Worker A locks row X; Worker B skips X, takes row Y
  No deadlock; no waiting; each worker gets unique job

SCC LENS:
  STATE: PostgreSQL (ALL authoritative state: definitions, executions, workers)
  COORDINATION: advisory lock (leader), SKIP LOCKED (claiming), heartbeat (liveness)
  CONCENTRATION: PostgreSQL is single coordination point → HA PostgreSQL is mandatory

INTERVIEW TRIGGER: "Design a distributed job scheduler" →
  1. Leader election (advisory lock) → only one instance schedules
  2. Job state machine (PENDING → CLAIMED → RUNNING → COMPLETED/FAILED/DEAD)
  3. SKIP LOCKED for race-free worker claiming
  4. Heartbeat + timeout → reclaim crashed worker's jobs
  5. At-least-once + idempotent jobs = effectively exactly-once
  6. SCC: State=PostgreSQL (all), Coordination=lock+SKIP LOCKED, Concentration=PostgreSQL HA
  7. Reject: in-memory state (lost on restart), crontab (no HA, no history)
```

---

*Part 2 Complete: Case Studies 11–20 (Intermediate Track)*
*Next: Part 3 — Advanced (Case Studies 21–30)*
