# Pattern Gym · Part 2 of 3

[← System Design index](../README.md)

> Coordination · Partitioning · Replication · Feeds · Traffic Control · Service Architecture · Data Processing (29 patterns). Part of the Pattern Gym — Learn to hear the pattern in the problem.

## Contents

- **Coordination** (5): [Leader Election](#leader-election) · [Distributed Lock](#distributed-lock) · [Leases](#leases) · [Heartbeat Failure Detection](#heartbeat-failure-detection) · [Gossip Membership](#gossip-membership)
- **Partitioning** (3): [Consistent Hashing](#consistent-hashing) · [Sharding](#sharding) · [Hot Key Mitigation](#hot-key-mitigation)
- **Replication** (4): [Quorum Replication](#quorum-replication) · [Geographic Replication](#geographic-replication) · [Read Repair](#read-repair) · [Anti Entropy](#anti-entropy)
- **Feeds** (3): [Fanout On Write](#fanout-on-write) · [Fanout On Read](#fanout-on-read) · [Hybrid Fanout](#hybrid-fanout)
- **Traffic Control** (6): [Token Bucket](#token-bucket) · [Leaky Bucket](#leaky-bucket) · [Fixed Window Rate Limit](#fixed-window-rate-limit) · [Sliding Window Rate Limit](#sliding-window-rate-limit) · [Backpressure](#backpressure) · [Load Shedding](#load-shedding)
- **Service Architecture** (6): [API Gateway](#api-gateway) · [Backend For Frontend](#backend-for-frontend) · [Service Discovery](#service-discovery) · [Sidecar](#sidecar) · [Aggregator](#aggregator) · [Scatter Gather](#scatter-gather)
- **Data Processing** (2): [Batch Processing](#batch-processing) · [Stream Processing](#stream-processing)

## Coordination

### Leader Election

*Have a group of identical processes agree on one of them to perform singleton work, with automatic replacement when that one fails.*

> **When you hear…** Work that must happen exactly once across a fleet · a coordinator role that cannot be duplicated · a singleton that must survive its host dying

**Flow:** `Candidates compete` → `Consensus picks one` → `Leader holds term` → `Heartbeats renew` → `Failure triggers re-election`

**The problem**

A nightly reconciliation job runs on every instance of a service, so with twelve instances it runs twelve times and produces twelve sets of adjustments. Pinning it to one named host fixes the duplication and creates a new problem: when that host is replaced during a deploy, the job simply stops happening and nobody notices for a week.

The requirement is genuinely contradictory on its face — exactly one process must do the work, and no specific process can be required to survive. Something has to decide which one, and that decision must itself survive failures.

> **Make the singleton a role, not a machine**  
> If the group can agree on which member currently holds a role, then the role survives any individual member. The hard part is the agreement: two processes both believing they hold it is far worse than nobody holding it, so the mechanism has to make split-brain impossible rather than unlikely.

**Mental model**

A contested, time-bounded role granted by a consensus system. Holding it requires continuous renewal; losing contact means losing it.

1. **Campaign** — Candidates attempt to acquire the role through a store that can decide atomically.
2. **Grant** — Exactly one wins, and receives a term identifier that increases with every election.
3. **Renew** — The leader repeatedly refreshes its claim within the lease period.
4. **Expire** — A leader that stops renewing loses the role automatically.
5. **Re-elect** — The remaining members compete again, under a higher term.

> **The leader can be deposed without knowing it**  
> A process that is paused, partitioned or simply slow may believe it is still the leader long after its lease expired and a successor was elected. It cannot detect this locally — from inside, a network partition looks exactly like a quiet period. Every design must assume the previous leader is still running and still acting, which is why the term number has to be checked by whatever the leader writes to.

**How it works**

**Terms, fencing and the split-brain window**

```text
ELECTION
  candidates attempt an atomic acquire against a
  consensus store (etcd, ZooKeeper, Consul)
  winner receives:
    role      = leader
    term      = 7          (monotonically increasing)
    lease     = 10 s

RENEWAL
  leader refreshes every 3 s
  miss the renewal window -> lease expires
  -> a new election begins, term becomes 8

THE DANGEROUS SEQUENCE
  leader (term 7) pauses for 15 s (GC, host stall)
  lease expires, new leader elected at term 8
  old leader resumes, still believes it is leader
  -> two leaders exist simultaneously
  -> this is UNAVOIDABLE; it is detected, not prevented

FENCING MAKES IT HARMLESS
  every write carries the term
  the resource records the highest term it has seen
    write(term = 8) -> accepted, highest = 8
    write(term = 7) -> REJECTED, stale term
  -> the deposed leader cannot corrupt anything

WITHOUT FENCING
  the old leader's writes land after the new
  leader's, silently overwriting them
  -> the election was correct and the data is wrong
```

1. **Use a real consensus system for the election** — Leadership decided by a single database row inherits that database's failure modes as split-brain.
2. **Give every term a monotonically increasing number** — It is the only thing that lets a resource distinguish current from deposed.
3. **Fence every side effect with the term** — Election alone does not prevent a stale leader from writing.
4. **Set the lease from realistic pause durations** — A lease shorter than a garbage collection pause produces constant spurious elections.
5. **Make the leader's work resumable** — A successor inherits partially completed work and must be able to continue or restart it cleanly.
6. **Expect leaderless intervals** — Between expiry and election nothing is running; the design must tolerate that gap.

**Choosing the lease, and what it costs**

```text
LEASE TOO SHORT  (2 s)
  a 3 s GC pause deposes a healthy leader
  -> elections every few minutes
  -> work restarted constantly, nothing completes

LEASE TOO LONG  (60 s)
  a crashed leader is not replaced for a minute
  -> the singleton work stalls for that long

CHOOSING
  lease > worst realistic pause (GC, disk stall,
  network blip) x a safety factor
  typical: 10-15 s lease, renew every 3-5 s
  -> detection of a dead leader takes up to one lease

AVAILABILITY OF THE ROLE
  failover time = lease remaining + election time
  10 s lease -> up to ~12 s with no leader
  -> if the work cannot pause for 12 s, leader
     election is the wrong pattern

WHAT LEADER ELECTION IS NOT
  it is not a throughput technique; the leader is a
  bottleneck by construction
  -> use it for coordination, not for data paths
```

| Metric | Value | Note |
|---|---|---|
| Guarantee | one leader | **with fencing** |
| Failover | lease + election | ~12 s typical |
| Split brain | possible | made harmless |
| Throughput | one process | not a scaling tool |

> **Election without fencing solves the wrong half of the problem**  
> Choosing a leader correctly is straightforward; preventing the previous one from acting is the difficult part. A system that elects through a robust consensus store and then writes to a database without checking terms has a correct election and a corruptible resource, because a paused leader will eventually wake and write. The fencing token is not a refinement — it is where the guarantee actually lives.

**Technologies**

| Option | Strength | Watch for |
|---|---|---|
| etcd, ZooKeeper, Consul | Purpose-built; leases and watches included | Another distributed system to operate |
| Kubernetes lease objects | Already present in the cluster | Tied to cluster availability |
| Database row with a lease column | No new infrastructure | Split-brain if the database fails over |
| Kafka consumer group leadership | Reuses existing coordination | Semantics tied to partitions |
| Cloud-managed locks | Fully operated | Latency, and service-specific limits |
| Avoid it: partition the work | No coordination at all | Only where work can be split by key |

The last row deserves consideration first. Work that can be partitioned by key and assigned by consistent hashing needs no leader at all, and removes the failover gap along with the coordination system — which is often a better answer than electing a leader well.

**Trade-offs**

**Ways to run singleton work**

| Approach | Survives host failure | Split-brain risk | Cost |
|---|---|---|---|
| Pin to one host | No | None | None |
| Run everywhere with idempotent effects | Yes | n/a | Requires idempotent design |
| Partition work by key | Yes | None | Needs a partitionable workload |
| Leader election with fencing | Yes | Handled | Consensus system, failover gap |
| Leader election without fencing | Yes | Real | Looks correct, is not |
| External scheduler triggering one run | Yes | Low | Depends on the scheduler |

Running everywhere with idempotent effects is underused: if the work can be made naturally repeatable, twelve instances doing it produces the same outcome as one, and no coordination is needed at all.

> **Ask before choosing it**  
> Can the work be partitioned or made idempotent instead? Leader election adds a consensus dependency and a failover window to every operation that needs the role, and both alternatives remove the problem rather than managing it.

**How it fails**

**How leader election goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Two leaders write conflicting data | No fencing token on writes | Carry the term; reject stale terms at the resource |
| Constant re-elections | Lease shorter than realistic pauses | Size the lease above worst-case stalls |
| Work never completes | Every election restarts it from the beginning | Make the work checkpointed and resumable |
| Split-brain from the election store | Leadership held in a non-consensus store | Use a system designed for this |
| Singleton work stops silently | Leader crashed, nobody monitoring the role | Alert on leaderless duration and election rate |
| Leader becomes a bottleneck | Data path routed through it | Use it for coordination only |
| Failover too slow for the workload | Lease duration exceeds tolerance | Shorter lease, or a different pattern |

> **The deposed leader is not a rare edge case**  
> Long garbage collection pauses, host-level stalls and brief partitions are ordinary operational events, and each one can produce a leader that is still running and still convinced. Any design that treats this as improbable will encounter it, typically under load when the pauses are longest — which is also when the work being duplicated matters most.

**Where it is used**

- **Scheduled jobs across a fleet**, the most common application and the one that motivates it.
- **Kubernetes controllers**, which use lease objects so that exactly one replica reconciles at a time.
- **Database primaries**, where election decides which replica accepts writes and fencing prevents the old one from continuing.
- **Partition assignment in messaging systems**, where a coordinator distributes work among consumers.
- **Cluster management planes**, which elect a controller to make cluster-wide decisions.

**In the interview**

The answer that stands out is about the deposed leader, not about how the election works.

- **Say that split-brain is detected, not prevented** — a paused leader cannot know it was deposed — and build the design around that.
- **Introduce fencing tokens** and show the resource rejecting a stale term; this is where the guarantee actually lives.
- **Size the lease against real pauses** and state the failover window it implies, since that is what the workload must tolerate.
- **Make the work resumable**, because every failover hands a successor something half-finished.
- **Offer partitioning or idempotence as alternatives** before electing anything; removing the singleton is better than coordinating it.

**Practice drill**

A nightly reconciliation job must run exactly once across twelve instances. Design leader election: the store, the lease duration, the renewal interval, and the failover window it produces. Then trace a 15-second garbage collection pause on the leader: what happens, who is leader afterwards, and what the paused process does when it resumes. Show the fencing mechanism that makes its late writes harmless, and say what the job must do to resume safely after a mid-run failover.

**Go deeper**

Leader election lets a group of identical processes agree on one member to perform work that must happen exactly once, and to replace it automatically when it fails.

**It converts a machine into a role, which is what makes the singleton survivable.** Pinning work to a named host is correct until that host is replaced; electing a holder means the work continues across restarts, deploys and hardware failures without anyone intervening. The cost is a dependency on a coordination system and a window during which no one holds the role.

**The difficult part is not choosing, it is unchoosing.** A process that is paused by garbage collection or cut off by a partition cannot distinguish that from a quiet period, so it continues believing it is the leader while a successor is already operating. No protocol prevents this, because the deposed process has no way to learn the truth in time — which means the design must assume two leaders can exist simultaneously and make that harmless.

**Fencing tokens are where the guarantee lives.** Each term carries a number that only increases, every write carries its term, and the resource remembers the highest it has accepted. A stale leader's writes are then rejected by the thing being written to, rather than being prevented by the election. Systems that elect carefully and then write without fencing have built the easy half of the pattern and left the failure mode intact.

**Lease duration is a direct trade between false failovers and slow ones.** Short leases depose healthy leaders whenever a pause exceeds them, producing churn in which work restarts continually and nothing finishes; long leases leave a crashed leader unreplaced for their full duration. The lease must exceed realistic worst-case stalls, and the resulting failover window — lease remainder plus election time — is the interruption the workload has to tolerate.

**Every failover inherits partial work, so resumability is part of the design.** A successor takes over mid-run and must be able to determine what has already been done, which means checkpointing progress durably rather than holding it in memory. Without that, each election restarts from the beginning, and a workload longer than the interval between failures never completes at all.

**It is a coordination mechanism and a throughput bottleneck.** Routing a data path through the leader means all traffic passes through one process that can be deposed at any moment, which is both slower and more fragile than the alternatives. The better instinct is to avoid needing a leader — partitioning work by key or making effects idempotent removes the singleton entirely, along with the consensus dependency and the failover gap that come with it.

**Related patterns:** Distributed Lock · Leases · Heartbeat Failure Detection · Consistent Hashing

---

### Distributed Lock

*Grant exclusive access to a resource across processes using a shared store, with an expiry so a crashed holder cannot block it forever.*

> **When you hear…** Two processes must not act on the same resource simultaneously · a shared external resource with no locking of its own · coordination across machines

**Flow:** `Acquire with TTL` → `Holder does work` → `Renew or expire` → `Release explicitly` → `Fencing token on writes`

**The problem**

Two workers pick up the same account and both apply a balance adjustment. In one process a mutex would prevent this trivially; across processes there is no shared memory and no scheduler, so nothing stops them.

The obvious fix — a lock row in a shared store — introduces a worse failure than the one it solves: a worker that crashes while holding the lock blocks the resource permanently, and a worker that is merely slow cannot be distinguished from one that has died.

> **A distributed lock must expire, and expiry is what breaks mutual exclusion**  
> Because a holder can die silently, every distributed lock needs a time limit — and the moment a lock can expire, the holder can still be working when it does. Mutual exclusion is therefore not guaranteed by the lock itself; it is guaranteed by the resource rejecting writes from an expired holder. The lock reduces contention, and fencing provides the correctness.

**Mental model**

A key in a shared store, set conditionally with a time to live, owned by whoever set it and released only by them.

1. **Acquire** — Set the key only if absent, with an expiry and a unique owner identifier.
2. **Work** — Perform the exclusive operation, ideally briefly.
3. **Renew** — Extend the expiry if the work runs longer than expected.
4. **Release** — Delete the key, but only if still the owner.
5. **Fence** — Carry a monotonic token so late writes from a lapsed holder are rejected.

> **Releasing a lock you no longer hold releases someone else's**  
> If the holder's lease expired and another process acquired the lock, a plain delete removes the new owner's lock and lets a third process in. Release must be conditional on ownership — compare the owner identifier and delete atomically — or the lock actively causes the concurrency it was meant to prevent.

**How it works**

**Acquire, renew, release safely**

```text
ACQUIRE
  SET lock:account:42 <owner-uuid> NX PX 10000
  NX  -> only if it does not exist
  PX  -> expires in 10 s automatically
  -> atomic; exactly one caller succeeds

RENEW  (while working)
  if GET lock == my-uuid:
    PEXPIRE lock 10000
  -> extend only if still the owner

RELEASE  (must be conditional)
  if GET lock == my-uuid:
    DEL lock
  -> as a single atomic script, not two commands
  -> a plain DEL can delete someone else's lock

THE EXPIRY PROBLEM
  holder pauses 12 s (GC)
  lock expires at 10 s
  another process acquires it and starts working
  original holder resumes and continues working
  -> both are inside the critical section
  -> the lock did not fail; it did what it must

FENCING
  the store returns an increasing token on acquire
    acquire -> token 41
    acquire -> token 42
  every write carries the token; the resource keeps
  the highest seen and rejects lower
  -> the lapsed holder's writes are rejected
```

1. **Always set an expiry** — A lock without one becomes permanent the first time a holder crashes.
2. **Store a unique owner identifier** — Release and renewal must both be conditional on ownership.
3. **Make release atomic** — Check-then-delete as two operations can delete a successor's lock.
4. **Renew while working, do not simply pick a long TTL** — A long TTL delays recovery from a genuine crash.
5. **Use a fencing token for correctness** — Expiry makes overlapping holders possible; only the resource can reject them.
6. **Keep critical sections short** — Long ones make expiry likely and contention expensive.

**What the lock actually guarantees**

```text
EFFICIENCY USE  (common, and fine)
  goal: avoid doing the same work twice
  occasional overlap = wasted effort, no damage
  -> a plain lock with TTL is sufficient
  -> example: regenerating a cache entry

CORRECTNESS USE  (rare, and demanding)
  goal: the operation must NEVER happen twice
  overlap = corrupted data or double spend
  -> a lock alone is NOT sufficient
  -> needs fencing at the resource, or the operation
     must be conditional (compare-and-set)

ASKING WHICH ONE YOU HAVE
  if two holders overlap, what is the damage?
    wasted CPU        -> efficiency lock, ship it
    wrong balance     -> you need fencing, or you
                         need the database to enforce
                         it with a transaction

THE SIMPLER ANSWER
  if the resource is a database, a row lock or a
  conditional update enforces exclusion better than
  any external lock, because the enforcement is at
  the place the change happens
```

| Metric | Value | Note |
|---|---|---|
| Must have | expiry | or it wedges |
| Consequence | overlap possible | **by construction** |
| Correctness | fencing token | at the resource |
| Better option | database constraint | where available |

> **Most distributed locks are guarding something that could enforce exclusion itself**  
> If the protected resource is a row in a database, a conditional update or a transaction provides mutual exclusion at the point of change, with no expiry, no renewal and no split holder. An external lock is genuinely needed when the resource cannot arbitrate — an external API, a filesystem, a third-party system — and reaching for one when the database was already capable adds a failure mode for nothing.

**Technologies**

| Option | Strength | Watch for |
|---|---|---|
| Redis SET NX PX | Fast, simple, ubiquitous | Failover can lose the lock; no fencing token natively |
| etcd or ZooKeeper | Consensus-backed, leases, watches | Heavier; higher latency per acquire |
| Database row lock | Transactional with the work | Scope limited to that database |
| Conditional update or version column | Exclusion without a lock at all | Requires the work to be a single update |
| Cloud lock services | Managed, with fencing tokens | Latency and service limits |
| Redlock across Redis nodes | Attempts safety without consensus | Contested; use consensus if correctness matters |

Redis is the most common choice and the least safe for correctness use: a failover can promote a replica that never received the lock, allowing a second acquisition while the first holder is still working. That is acceptable for efficiency locks and not for anything where overlap causes damage.

**Trade-offs**

**Ways to prevent concurrent action**

| Approach | Guarantees exclusion | Failure mode | Cost |
|---|---|---|---|
| Database transaction or row lock | Yes | Contention, deadlocks | None extra |
| Conditional update with a version | Yes | Retry on conflict | None extra |
| Distributed lock with TTL | Not under pauses | Overlapping holders | A store to operate |
| Distributed lock plus fencing | Yes, at the resource | Complexity | Store plus resource support |
| Partition work by key | Yes, structurally | Needs partitionable work | Routing |
| Make the operation idempotent | Not needed | n/a | Design effort |

Partitioning is the structural answer worth reaching for: if every account is handled by exactly one worker determined by a hash, no two workers ever contend for it and no lock is required at all.

> **Ask before choosing it**  
> If two holders overlapped, what would actually break? Wasted work means a simple lock is enough; incorrect data means the lock is not the mechanism you need and the resource must enforce it — either with a fencing token or with its own transactional guarantee.

**How it fails**

**How distributed locks go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Resource blocked forever | Lock acquired with no expiry | Always set a TTL |
| Two holders in the critical section | Holder paused past the expiry | Fencing tokens; or a conditional resource update |
| A process releases another's lock | Unconditional delete | Atomic check-owner-and-delete |
| Lock lost during failover | Non-consensus store promoting a stale replica | Consensus store where correctness matters |
| Constant lock expiry | TTL shorter than the work | Renew during work rather than lengthening the TTL |
| Throughput collapses | Long critical sections under contention | Shorten the section or partition the keys |
| Deadlock across several locks | Inconsistent acquisition order | Global ordering, or acquire one lock only |

> **A distributed lock cannot provide mutual exclusion on its own**  
> Expiry is mandatory because holders die silently, and expiry means a slow holder can still be working when its claim lapses. Every distributed lock therefore permits overlapping holders under pauses, and the only place that can prevent damage is the resource itself — through a fencing token it checks, or through its own transactional exclusion. A design whose correctness depends solely on the lock is relying on a guarantee the lock does not offer.

**Where it is used**

- **Cache regeneration**, where an efficiency lock prevents many processes rebuilding the same value.
- **Scheduled task coordination**, ensuring one instance runs a job — effectively lightweight leader election.
- **External API rate coordination**, serialising access to a third party that cannot arbitrate for itself.
- **Filesystem and object-store operations**, where the storage layer offers no locking primitive.
- **Migration and maintenance scripts**, preventing two operators from running the same destructive operation.

**In the interview**

The expected answer explains why expiry and mutual exclusion are in tension, rather than presenting the lock as sufficient.

- **State the dilemma directly**: without expiry a crash wedges the resource, with expiry a slow holder overlaps a new one.
- **Require conditional release** with an owner identifier, since a plain delete can free someone else's lock.
- **Introduce fencing tokens** and show the resource rejecting stale ones — that is where correctness comes from.
- **Separate efficiency from correctness use** and choose the mechanism accordingly; most locks are the former.
- **Suggest the database or partitioning instead** where the resource can enforce exclusion itself, because that removes the failure mode entirely.

**Practice drill**

Two workers must not adjust the same account balance concurrently. Write the acquire, renew and release operations, including what makes release safe. Then trace a 12-second pause on the holder of a 10-second lock: who holds the lock, what the paused worker does on waking, and what the balance ends up as. Add fencing and show what changes. Finally, argue whether an external lock was needed at all given that the balance lives in a database.

**Go deeper**

A distributed lock grants exclusive access to a resource across processes, using a shared store that can decide atomically who holds it.

**Expiry is mandatory and it is what undermines the guarantee.** A holder that crashes cannot release, so without a time limit the first failure blocks the resource permanently. With a time limit, a holder that is merely slow — paused by garbage collection, stalled on a disk — remains inside the critical section after its claim has lapsed and a successor has acquired it. Overlapping holders are therefore a structural property of distributed locking, not a bug in a particular implementation.

**Correctness has to be enforced where the change happens.** Since the lock cannot prevent overlap, the resource must reject work from a holder whose claim has expired, which requires a monotonically increasing fencing token carried on every write and remembered by the resource. Without it, the late writes of a lapsed holder land after the new holder's and silently overwrite them — the lock behaved exactly as designed and the data is still wrong.

**Distinguishing efficiency from correctness decides how much machinery is warranted.** A lock that prevents duplicate cache regeneration can tolerate occasional overlap, because the cost is wasted CPU; a lock guarding a balance adjustment cannot, because the cost is money. The first needs only a TTL and a conditional release; the second needs fencing or a resource that arbitrates for itself, and conflating them is how systems end up with a plausible-looking lock protecting something that genuinely needed a transaction.

**Release must be conditional on ownership, and this is a common omission.** A process whose lease expired and who then deletes the key is removing whatever lock currently exists — including a successor's — which admits a third process and produces exactly the concurrency the lock existed to prevent. Comparing the owner identifier and deleting in one atomic operation is what makes release safe, and splitting it into a read followed by a delete reintroduces the race.

**The choice of store determines what failures are possible.** A single-node cache with asynchronous replication can promote a replica that never saw the lock, permitting a second acquisition while the first holder works — fine for efficiency, unacceptable for correctness. A consensus-backed store costs more per acquisition and does not lose the lock on failover, which is the property that matters when overlap causes damage.

**Most distributed locks are guarding a resource that could enforce exclusion itself.** When the protected state is a database row, a transaction or a conditional update provides mutual exclusion at the point of change, with no expiry, no renewal and no possibility of two holders. External locks earn their place where the resource cannot arbitrate — third-party APIs, filesystems, systems outside your control — and partitioning work by key removes the contention entirely where the workload allows it, which is better than coordinating access well.

**Related patterns:** Leader Election · Leases · Idempotency Key · Consistent Hashing

---

### Leases

*Grant a right for a bounded period rather than indefinitely, so a holder that dies silently loses it automatically without anyone having to detect the failure.*

> **When you hear…** A holder can die without releasing · ownership must recover without operator action · caches or claims that must not outlive their validity

**Flow:** `Grant with deadline` → `Holder renews` → `Renewal stops` → `Deadline passes` → `Right released`

**The problem**

A process claims a resource and then vanishes — the host is terminated, the network drops, the kernel kills it. It never releases, and nothing else can tell whether it is dead or merely quiet. Every claim without an expiry becomes permanent the first time this happens.

Detecting the failure explicitly is harder than it looks: an unanswered heartbeat is indistinguishable from a partition, and a system that waits for certainty waits forever.

> **Make the right decay unless actively renewed**  
> If a grant carries a deadline, then a holder that stops functioning stops renewing, and the right expires on its own. Nobody has to decide that the holder is dead — the absence of renewal is the decision, and it is made by the passage of time rather than by a judgement that could be wrong.

**Mental model**

A time-bounded grant with a deadline both parties can reason about. Continuing to hold it requires continuing to ask.

1. **Grant** — The issuer gives a right valid until a specific time.
2. **Use** — The holder acts, knowing when its right expires.
3. **Renew** — The holder extends the deadline before it arrives.
4. **Lapse** — Renewal stops, the deadline passes, and the right ends with no action required.
5. **Reassign** — The issuer is free to grant it to someone else.

> **The holder's clock and the issuer's clock disagree**  
> A lease is a deadline, and two machines rarely agree precisely on what time it is. Clock skew means a holder may believe it has two seconds left when the issuer already considers the lease expired. Safe designs have the holder stop acting well before the nominal deadline, and never rely on the two sides computing the same instant.

**How it works**

**Sizing a lease and using it safely**

```text
GRANT
  lease = { holder: worker-7, expires_at: T + 10 s }

HOLDER
  renew at T + 4 s  -> expires_at = T + 14 s
  renew at T + 8 s  -> expires_at = T + 18 s
  ...
  renewal fails     -> STOP WORKING IMMEDIATELY
                       do not wait for expiry

THE SAFETY MARGIN
  issuer considers it expired at expires_at
  holder should stop at expires_at - skew - latency
  typical: stop at 80% of the lease
  -> a holder acting right up to the deadline will
     sometimes act after it

RENEWAL INTERVAL
  renew every lease/3
  -> two renewals can fail before the lease lapses
  -> tolerates transient blips without churn

SIZING THE LEASE
  too short: normal pauses cause spurious expiry
  too long:  dead holders block the resource
  lease > worst realistic pause x 2
  typical 10-30 s for coordination
          seconds to minutes for caches
```

1. **Renew at roughly a third of the lease period** — Two missed renewals then become survivable rather than fatal.
2. **Stop working when renewal fails, not when the deadline passes** — By the deadline the issuer may already have reassigned the right.
3. **Leave a safety margin for clock skew and latency** — Both sides compute the deadline independently and will not agree exactly.
4. **Use durations rather than absolute timestamps across machines** — Relative time is far less sensitive to clock disagreement.
5. **Pair with fencing where the right protects a resource** — Expiry admits overlap; only the resource can reject the lapsed holder.
6. **Size from real pause durations** — A lease shorter than a garbage collection pause guarantees churn.

**Where leases appear**

```text
COORDINATION
  leader election   the leadership term is a lease
  distributed lock  the TTL is a lease
  -> both inherit the expiry-versus-exclusion tension

CACHING
  a cache entry TTL is a lease on the right to serve
  that value without rechecking
  -> read leases: a client may cache until the lease
     expires; the server must not change the value
     without first waiting out or revoking the lease

RESOURCE ASSIGNMENT
  DHCP addresses, partition ownership, shard
  assignment
  -> a node that vanishes releases its share
     automatically

MEMBERSHIP
  a node is a member while its lease is renewed
  -> failure detection with no explicit detector

THE COMMON SHAPE
  every one of these replaces the question
  "is the holder alive?" with
  "did the holder renew?"
  -> the second question always has an answer
```

| Metric | Value | Note |
|---|---|---|
| Failure handling | automatic | **no detector needed** |
| Renewal | every lease/3 | tolerates blips |
| Stop acting | on failed renewal | not at deadline |
| Hazard | clock skew | keep a margin |

> **A holder that acts right up to the deadline will sometimes act after it**  
> Network latency on the renewal, a scheduling delay, and a small clock difference each move the effective boundary. A holder that keeps working until its own clock shows expiry has no margin for any of them, and will occasionally perform an action that the issuer has already reassigned. Stopping early is not caution — it is the only way the arithmetic works out.

**Technologies**

| Use | Implementation | Note |
|---|---|---|
| Coordination | etcd and ZooKeeper leases | Sessions expire when renewal stops |
| Locking | Redis SET with PX | TTL is the lease; renewal is an explicit extend |
| Membership | Kubernetes lease objects | Controllers renew to signal liveness |
| Caching | HTTP max-age and TTLs | A lease on serving without revalidating |
| Networking | DHCP leases | The original consumer-facing example |
| Read leases | Distributed filesystems and databases | The server must respect them before mutating |

Read leases are the least familiar and most instructive: a server that has granted one cannot change the underlying value until it expires or is revoked, which is what makes client-side caching safe without a validation round trip on every read.

**Trade-offs**

**Ways to reclaim a right from a failed holder**

| Approach | Detection needed | Recovery time | Risk |
|---|---|---|---|
| Explicit release only | n/a | Never, if the holder died | Permanent block |
| Lease with expiry | None | Up to one lease | Overlap under pauses |
| Heartbeat with a detector | Yes | Detector interval | False positives |
| Operator intervention | Human | Minutes to hours | Slow and error-prone |
| Lease plus fencing | None | Up to one lease | Overlap made harmless |

The lease row and the heartbeat row are closer than they look: a lease is a heartbeat whose failure has a predefined consequence, which is why it needs no separate detector and no decision about what to do when a heartbeat is missed.

> **Ask before choosing it**  
> What is the longest pause a healthy holder can experience? That number sets the lease floor, and teams routinely underestimate it — garbage collection, disk stalls and host-level freezes are all longer in production than in testing, and a lease below them produces constant unnecessary failovers.

**How it fails**

**How leases go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Constant spurious expiry | Lease shorter than real pauses | Size above worst-case stalls |
| Two holders acting | Holder continued past expiry | Stop on failed renewal; add fencing |
| Right held long after death | Lease far too long | Shorten, accepting more renewal traffic |
| Expiry mid-operation | Operation longer than the lease | Renew during work, or checkpoint and resume |
| Clock skew causes disagreement | Absolute timestamps across machines | Use durations and a safety margin |
| Renewal storms | Very short leases across many holders | Longer leases, or jittered renewal |
| Stale data served | Cache lease outlived the value | Revoke, or keep leases short for volatile data |

> **Leases turn failure detection into a timing problem, and timing problems need margins**  
> Because the whole mechanism rests on deadlines computed independently on two machines, every source of delay — clock drift, network latency, scheduler jitter, a long pause — erodes the gap between what the holder believes and what the issuer believes. Without a deliberate margin the two sides will eventually disagree at exactly the wrong moment, and the resource will be held by two parties who each believe they are the only one.

**Where it is used**

- **Coordination systems**, where sessions are leases and losing one relinquishes every right it held.
- **Kubernetes**, which expresses both leader election and node liveness as renewable lease objects.
- **Distributed filesystems**, granting read and write leases so clients can cache without revalidating.
- **Partition and shard assignment**, where a vanished node's share is reassigned automatically on expiry.
- **DHCP**, the original example most people have encountered, reclaiming addresses from devices that left.

**In the interview**

The insight to articulate is that a lease replaces failure detection with a deadline, and that deadlines need margins.

- **Say it removes the need for a detector.** Absence of renewal is the failure signal, and it always has an answer where liveness checks do not.
- **Renew at a third of the period** so transient blips do not cause failover, and explain the arithmetic.
- **Stop working when renewal fails**, not at the nominal deadline, because the issuer may already have reassigned.
- **Raise clock skew** and argue for durations and a safety margin rather than shared absolute timestamps.
- **Connect it to locks and leader election** as the same mechanism, inheriting the same expiry-versus-exclusion tension.

**Practice drill**

A worker holds a lease on a partition. Choose the lease duration and renewal interval, justifying both from realistic pause durations. Say exactly when the worker must stop processing and why that is earlier than the deadline. Then trace a network blip that causes two renewals to fail and a third to succeed, and a longer partition that causes expiry — describing what the issuer does and what the worker does on reconnecting.

**Go deeper**

A lease is a right granted for a bounded period, which expires unless the holder actively renews it.

**It replaces failure detection with the passage of time.** Deciding whether a silent process is dead or partitioned is undecidable from the outside, and any system that waits for certainty waits indefinitely. A lease sidesteps the question entirely: the holder either renewed or it did not, and that has a definite answer at every moment. The failure handling is therefore automatic rather than the result of a judgement that might be wrong.

**The renewal interval sets the tolerance for transient trouble.** Renewing at roughly a third of the lease means two consecutive renewals can fail without losing the right, which absorbs ordinary network blips and brief scheduling delays. Renewing only just before expiry makes every hiccup fatal and produces churn in which rights bounce between holders and no work completes.

**The holder must stop earlier than the deadline, and this is the subtle part.** Renewal latency, scheduler delay and clock disagreement all shift the effective boundary, so a holder that works right up to its own computed expiry will sometimes act after the issuer has already reassigned the right. Stopping at a fraction of the lease is what keeps the two views from overlapping, and treating the deadline as a hard stop rather than a target is the difference between a safe implementation and one that fails rarely and confusingly.

**Sizing is a direct trade between spurious expiry and slow recovery.** A lease below realistic pause durations deposes healthy holders whenever garbage collection or a disk stall runs long; a very long lease leaves a dead holder's right unavailable for its full duration. Production pauses are consistently longer than the ones seen in testing, which is why leases sized from measurements in a quiet environment tend to cause churn once real load arrives.

**Expiry admits overlap, so leases guarding a resource need fencing.** A holder paused past its deadline resumes believing it still holds the right while a successor is already acting, and nothing in the lease mechanism prevents both from writing. The resource itself must reject the lapsed holder, using a monotonically increasing token, which is the same conclusion distributed locks and leader election reach by the same route — because all three are leases wearing different names.

**Once recognised, the pattern appears everywhere.** Cache TTLs are leases on the right to serve a value without rechecking; DHCP grants leases on addresses; partition assignment, cluster membership and leader terms are all rights that decay unless renewed. Seeing them as one mechanism is what makes the recurring questions — how long, how often to renew, what margin, what happens on expiry — transferable rather than rediscovered separately in each system.

**Related patterns:** Leader Election · Distributed Lock · Heartbeat Failure Detection · Cache Invalidation

---

### Heartbeat Failure Detection

*Have nodes send periodic signals and declare a node failed when they stop, accepting that partitions and pauses make the verdict a guess.*

> **When you hear…** Failed nodes must be removed from rotation · work must be reassigned · membership has to reflect reality automatically

**Flow:** `Periodic beat` → `Monitor tracks arrivals` → `Beats stop` → `Timeout elapses` → `Declared failed`

**The problem**

A node stops responding. It may have crashed, it may be paused, it may be perfectly healthy behind a broken network link. From the outside these are indistinguishable, and yet the system has to decide whether to reassign its work.

Waiting longer improves accuracy and delays recovery. Deciding quickly recovers fast and sometimes evicts a node that was fine, potentially causing two nodes to do the same work. There is no setting that avoids both errors.

> **Failure detection is a guess with a tunable error profile**  
> No protocol can distinguish a crashed node from an unreachable one, because the evidence is identical. What a detector can do is choose where to sit between detecting too eagerly and detecting too late, and make the consequences of a wrong guess survivable — which is why detection is always paired with fencing or idempotence rather than trusted on its own.

**Mental model**

A periodic signal, a monitor that tracks arrival times, and a timeout beyond which absence is treated as failure.

1. **Beat** — Each node sends a signal at a fixed interval.
2. **Observe** — A monitor records the time of the last arrival.
3. **Suspect** — Missing beats raise suspicion without immediate action.
4. **Declare** — After a threshold, the node is treated as failed.
5. **Recover** — A returning node rejoins, and must handle having been declared dead.

> **A node that was declared dead does not know it**  
> The evicted node is frequently still running, still holding resources and still processing. It learns of its eviction only when it next communicates, which may be long after its work has been reassigned. Every system using heartbeats must therefore assume a declared-dead node is still active, which is exactly the assumption that makes fencing necessary.

**How it works**

**Interval, threshold and what they cost**

```text
FIXED THRESHOLD
  interval  1 s
  threshold 5 missed beats
  -> detection in ~5 s
  -> a 6 s GC pause causes a FALSE positive

THE TRADE
  threshold 3  -> detect in 3 s, more false positives
  threshold 10 -> detect in 10 s, fewer false
                  positives, slower recovery
  -> there is no value that avoids both errors

ADAPTIVE (phi accrual)
  track the DISTRIBUTION of inter-arrival times
  output a suspicion level, not a boolean
    phi = 1   probably fine
    phi = 8   very likely dead
  act at different thresholds for different actions:
    phi 2  -> stop routing new work
    phi 8  -> reassign its partitions
  -> cheap actions on weak evidence, expensive
     actions on strong evidence

WHO MONITORS WHOM
  central monitor  simple, single point of failure,
                   and its own network view
  all-to-all       n^2 traffic, does not scale
  gossip           each node monitors a few, and
                   suspicion spreads
```

1. **Separate suspicion from declaration** — Cheap precautions on weak evidence, disruptive actions only on strong evidence.
2. **Size the threshold against real pause durations** — A threshold below the worst garbage collection pause guarantees false positives.
3. **Prefer adaptive detection where pause behaviour varies** — A fixed timeout tuned for the median is wrong for the tail.
4. **Do not let one monitor's network view decide alone** — A monitor behind a broken link declares healthy nodes dead.
5. **Require confirmation from several observers** — Indirect probing distinguishes a node failing from a link failing.
6. **Make reassignment safe against the node returning** — Detection will be wrong sometimes; fencing is what makes that survivable.

**Why false positives are the dangerous direction**

```text
FALSE NEGATIVE  (dead node not yet detected)
  requests to it fail or time out
  -> bad, but bounded and self-correcting
  -> callers retry elsewhere

FALSE POSITIVE  (healthy node declared dead)
  its partitions are reassigned
  it is still running and still writing
  -> TWO nodes own the same data
  -> divergence, duplicate work, corruption

-> a detector that is slightly slow is far safer than
   one that is slightly eager

THE CASCADE
  network congestion delays heartbeats
  -> nodes declared dead
  -> their work is reassigned
  -> reassignment causes data movement
  -> data movement increases congestion
  -> more heartbeats delayed
  this is how a partial network problem becomes a
  cluster-wide outage

THE CONTROL
  cap how many nodes may be evicted at once
  if more than X% appear dead, assume the network is
  at fault, not the nodes
```

| Metric | Value | Note |
|---|---|---|
| Detection time | interval × threshold | tunable |
| False positive | two owners | **the dangerous one** |
| Better detector | adaptive suspicion | graded response |
| Safety valve | cap evictions | stops cascades |

> **Mass eviction is almost always the network, not the nodes**  
> When a third of a cluster appears to fail simultaneously, the overwhelmingly likely explanation is a network event rather than a third of the hardware dying at once. A detector that acts on that observation reassigns enormous amounts of work, generates traffic on an already-degraded network, and deepens the problem. Capping how many nodes may be evicted in a window converts a cascade into a degradation.

**Technologies**

| Approach | Used by | Note |
|---|---|---|
| Fixed timeout heartbeats | Most schedulers and load balancers | Simple; badly suited to variable pauses |
| Phi accrual detection | Cassandra and similar | Suspicion level rather than a boolean |
| SWIM-style gossip | Consul, Serf | Scales; indirect probes cut false positives |
| Lease expiry | etcd, ZooKeeper, Kubernetes | Detection folded into the lease |
| Load balancer health checks | Every fleet | Removes from rotation without cluster-wide consequences |
| Indirect probing | SWIM | Asks others to confirm before declaring |

Indirect probing is the cheapest large improvement available: before declaring a node dead, ask a few other nodes to probe it. A node that answers them is clearly alive and the problem is the link between it and the original observer — which is a different fault with a different remedy.

**Trade-offs**

**Detector designs**

| Design | Detection speed | False positives | Scalability |
|---|---|---|---|
| Short fixed timeout | Fast | Many | Fine |
| Long fixed timeout | Slow | Few | Fine |
| Adaptive suspicion | Tunable per action | Few | Fine |
| All-to-all heartbeats | Fast | Few | Poor beyond ~100 nodes |
| Gossip with indirect probes | Moderate | Few | Very good |
| Central monitor | Fast | Depends on its view | Single point of failure |

Central monitoring hides a subtle bias: the monitor's own network position determines what it sees, so a monitor behind a degraded link declares perfectly healthy nodes dead and the cluster acts on one machine's misperception.

> **Ask before choosing it**  
> What happens if this verdict is wrong in each direction? A slow detection that leaves requests failing is usually recoverable; an eager detection that gives two nodes the same partition may not be. The asymmetry should decide the threshold, not the desire for fast failover.

**How it fails**

**How failure detection goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Healthy nodes evicted | Threshold below real pause durations | Raise it; use adaptive detection |
| Two owners for one partition | False positive with no fencing | Fencing tokens on every write |
| Cluster-wide cascade | Mass eviction during congestion | Cap concurrent evictions |
| Dead node serves for minutes | Threshold far too high | Balance against the false-positive cost |
| Monitor's view taken as truth | Single observer | Indirect probes and confirmation |
| Detection traffic dominates | All-to-all heartbeats at scale | Gossip-based membership |
| Returning node causes chaos | No handling of rejoin after eviction | Fence it out; require a clean rejoin |

> **Detection and reassignment together can amplify a network problem into an outage**  
> Congestion delays heartbeats, delayed heartbeats cause evictions, evictions trigger data movement, and data movement adds traffic to the congested network. Each step is a correct local response and the composition is a positive feedback loop. The only reliable brake is a limit on how much reassignment may happen at once, applied before the loop has a chance to run.

**Where it is used**

- **Cluster membership in distributed databases**, deciding which nodes hold which replicas.
- **Kubernetes node liveness**, where a node that stops renewing has its pods rescheduled.
- **Load balancer health checks**, the narrowest and safest form — removal from rotation only.
- **Consumer group coordination**, detecting a dead consumer and rebalancing its partitions.
- **Gossip-based systems**, where detection is distributed and suspicion spreads rather than being centrally decided.

**In the interview**

State the impossibility first; candidates who present detection as solvable miss the entire design problem.

- **Say a crash and a partition are indistinguishable**, so detection is a guess with a tunable error profile.
- **Argue the asymmetry.** A false positive gives two owners and can corrupt; a false negative merely fails requests.
- **Offer graded suspicion** — cheap actions on weak evidence, reassignment only on strong evidence.
- **Raise the cascade** where eviction causes data movement that delays more heartbeats, and cap concurrent evictions.
- **Pair detection with fencing**, because the verdict will sometimes be wrong and only the resource can make that harmless.

**Practice drill**

A 50-node cluster uses 1-second heartbeats with a 5-beat threshold. Say what happens during a 6-second garbage collection pause on one node, including what its partitions do and what the node does on waking. Then a network event delays heartbeats from 20 nodes at once: describe the sequence that follows, why it can worsen, and the specific control you would add. Finally, choose between a fixed threshold and adaptive suspicion for this cluster and justify it.

**Go deeper**

Heartbeat failure detection infers that a node has failed from the absence of its periodic signals, and acts on that inference.

**The verdict is unavoidably a guess.** A crashed node and a healthy but unreachable node produce identical evidence — silence — and no protocol can separate them, because the information needed is on the far side of the failure. What a detector controls is not accuracy but the balance between deciding too early and deciding too late, and understanding that the choice is between error types rather than between right and wrong is the foundation of every sound design here.

**The two errors are not equally costly, and that asymmetry should drive the threshold.** Failing to detect a dead node leaves requests timing out, which is bad and self-correcting as callers retry elsewhere. Wrongly declaring a live node dead reassigns its work while it continues to operate, producing two owners for the same data — a divergence that is not self-correcting and may not be detectable afterwards. A detector tuned slightly slow is therefore much safer than one tuned slightly fast.

**Graded suspicion is better than a boolean.** Tracking the distribution of arrival times and producing a suspicion level allows cheap precautions on weak evidence — stop sending new work — and expensive ones only on strong evidence. A single timeout forces every consequence to share one threshold, so the value that is right for pausing traffic is simultaneously wrong for reassigning partitions, and one of the two is always badly served.

**A single observer's view is not the cluster's view.** A monitor sitting behind a degraded link sees silence from healthy nodes and, if trusted alone, causes the whole cluster to act on its private misperception. Asking other nodes to probe before declaring distinguishes a failing node from a failing link — a different fault with a different remedy — and it is among the cheapest accuracy improvements available.

**Detection and recovery can form a feedback loop.** Congestion delays heartbeats, delays cause evictions, evictions trigger data movement, and that movement congests the network further. Every step is locally reasonable and the composition turns a partial network problem into a cluster-wide outage. Capping how many nodes may be evicted in a window is the brake, and it encodes a useful prior: when many nodes appear to fail simultaneously, the network is the likelier culprit.

**Because the verdict will sometimes be wrong, detection must be paired with something that makes it harmless.** A node declared dead does not know it, keeps running, and keeps writing until it next communicates. Fencing tokens let the resource reject those writes, idempotent handling makes duplicate work benign, and leases fold the detection into the grant itself. Detection alone decides who should act; it cannot stop the other party from acting, and designs that forget this are correct exactly until the first long pause.

**Related patterns:** Leases · Gossip Membership · Leader Election · Circuit Breaker

---

### Gossip Membership

*Each node exchanges state with a few random peers, so membership and failure information spreads through the cluster without any central registry.*

> **When you hear…** Hundreds or thousands of nodes · no central coordinator wanted · membership must survive partitions and keep working

**Flow:** `Pick random peers` → `Exchange state` → `Merge by version` → `Spread exponentially` → `Whole cluster converges`

**The problem**

Every node needs to know which other nodes exist and which are alive. Having each one heartbeat every other produces traffic proportional to the square of the cluster size — at a thousand nodes that is a million connections, which is infrastructure spent entirely on knowing who is present.

A central registry removes the traffic and adds a dependency that must be more available than the cluster it describes. When it is unreachable, no node can learn about any change, including the failure of the node it is currently trying to talk to.

> **Information spreads fastest when everyone repeats it to a few others**  
> If each node tells a small random sample of peers what it knows, and they do the same, knowledge reaches the whole cluster in a number of rounds that grows only logarithmically with size. No node needs a complete picture of who to inform, no node is essential to the process, and the traffic each one generates is constant regardless of how large the cluster becomes.

**Mental model**

Periodic pairwise exchanges with random peers, where each side merges the other's view using version numbers to decide what is newer.

1. **Select** — Every interval, each node picks a few peers at random.
2. **Exchange** — They send each other their view of cluster membership.
3. **Merge** — Each entry is kept or replaced according to its version.
4. **Spread** — Because every recipient repeats it, information reaches everyone exponentially fast.
5. **Converge** — All nodes eventually agree, without anyone coordinating.

> **Convergence is eventual, so nodes act on stale membership**  
> During the seconds it takes information to propagate, different nodes hold different views: one still routes to a node that another has already marked dead. Gossip does not provide a consistent membership snapshot, and any decision requiring all nodes to agree at the same instant — such as who owns a partition exclusively — needs consensus rather than gossip.

**How it works**

**Why gossip scales, and how fast it spreads**

```text
TRAFFIC
  all-to-all:  each node talks to n-1 others
    n = 1000 -> ~1,000,000 pairings
  gossip:      each node talks to k peers (k = 3)
    n = 1000 -> 3000 messages per round
  -> per-node cost is CONSTANT as the cluster grows

SPREAD
  each round, the number of informed nodes roughly
  multiplies by (k + 1)
    round 0:     1
    round 1:     4
    round 2:    16
    round 3:    64
    round 4:   256
    round 5:  1024
  -> ~log(n) rounds; at 1 s intervals, a 1000-node
     cluster learns anything in about 5 s

MERGING
  each entry carries { node, state, version }
  on exchange, keep the higher version
    alive(v7) vs suspect(v9) -> suspect wins
  -> versions, not timestamps; clocks disagree

SWIM REFINEMENT
  direct probe fails
  -> ask k other nodes to probe indirectly
  -> only if all fail, mark suspect
  -> suspect, then dead after a timeout
  -> cuts false positives from single bad links
```

1. **Gossip with a small constant number of peers** — Three or four is enough; more raises traffic without materially faster spread.
2. **Version every entry** — Merging needs a total order that does not depend on synchronised clocks.
3. **Use indirect probes before suspecting** — One broken link should not convict a node the rest of the cluster can reach.
4. **Introduce a suspect state before dead** — It gives the node a window to refute the rumour, which sharply cuts false positives.
5. **Let a node refute its own death** — A returning node increments its version and the correction spreads the same way.
6. **Do not build exclusive ownership on gossip** — Eventual agreement cannot support decisions that require simultaneous agreement.

**What gossip is good for and what it is not**

```text
GOOD FOR
  membership       who exists, who seems alive
  failure rumours  spread without a central detector
  configuration    values that tolerate brief
                   disagreement
  load information which nodes are busy

NOT GOOD FOR
  leader election      needs agreement at an instant
  exclusive ownership  two nodes may both believe
                       they own a partition
  ordered operations   gossip has no global order
  strong consistency   convergence is eventual by
                       construction

THE COMMON COMBINATION
  gossip for membership and liveness
  + consensus (Raft) for the few decisions that must
    be exclusive
  -> the cheap mechanism handles the high-volume,
     tolerant information; the expensive one handles
     the small, critical set

PARTITION BEHAVIOUR
  network splits into A and B
  each side gossips internally and converges
  each side believes the other side is dead
  on heal, versions reconcile and views merge
  -> gossip keeps working on both sides, which is the
     point, and is also why it cannot arbitrate
```

| Metric | Value | Note |
|---|---|---|
| Traffic per node | constant | **independent of n** |
| Spread time | log(n) rounds | ~5 s at n=1000 |
| Consistency | eventual | not a snapshot |
| Pair with | consensus | for exclusive decisions |

> **A partition produces two confident, incompatible views**  
> Each side of a network split gossips internally, converges, and concludes that the other side has died. Both views are internally consistent and both are wrong about half the cluster. This is not a flaw to be fixed — continuing to operate on both sides is precisely the availability gossip is chosen for — but it means anything requiring a single answer must be decided by a mechanism that refuses to proceed without a quorum.

**Technologies**

| System | Use | Note |
|---|---|---|
| SWIM protocol | Membership with indirect probing | The reference design most implementations follow |
| Consul and Serf | Cluster membership and events | SWIM in production form |
| Cassandra | Node state and token ownership | Gossip plus versioned state |
| Redis Cluster | Node and slot membership | Gossip between nodes on a separate port |
| Kubernetes | Central API server instead | A deliberate alternative choice |
| Consensus systems | The exclusive decisions | Paired with gossip, not replaced by it |

Kubernetes is the instructive counterexample: it uses a central, consensus-backed store for membership rather than gossip, accepting a dependency in exchange for a consistent view. At the cluster sizes it targets that is the better trade, which shows the choice is about scale and consistency needs rather than about one approach being superior.

**Trade-offs**

**Membership approaches**

| Approach | Traffic | Consistency | Central dependency |
|---|---|---|---|
| All-to-all heartbeats | O(n²) | Fast within a node's view | None |
| Central registry | O(n) | Consistent | Yes, must outlive the cluster |
| Consensus-backed store | O(n) | Strongly consistent | Yes, quorum required |
| Gossip | O(n) total, O(1) per node | Eventual | None |
| Gossip plus consensus | O(n) | Eventual plus strong where needed | Only for critical decisions |

The final row is what large systems actually run: gossip carries the high-volume, failure-tolerant information, and a small consensus group decides the handful of things that genuinely cannot be decided twice.

> **Ask before choosing it**  
> How many nodes, and does anything depend on every node agreeing at the same moment? Below a few dozen nodes a central registry is simpler and gives a consistent view; above a few hundred the quadratic and central options both strain, and gossip becomes the only comfortable choice.

**How it fails**

**How gossip membership goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Healthy node marked dead | Single failed probe path | Indirect probes and a suspect state |
| Slow convergence | Too few peers or too long an interval | Raise fan-out or frequency modestly |
| Traffic grows with cluster size | Fan-out proportional to n | Keep the peer count constant |
| Both sides of a partition act as the cluster | Gossip used for ownership decisions | Use consensus for exclusivity |
| Stale state resurrected | Merging on timestamps across skewed clocks | Version numbers, not wall clocks |
| Rumours never settle | No refutation mechanism | Let a node increment its version to refute |
| Split-brain data ownership | Ownership derived from membership view | Fence writes; require quorum for ownership |

> **Deriving exclusive ownership from a gossip view is a split-brain generator**  
> Because each node's membership picture is eventually consistent, two nodes can simultaneously conclude that they own a partition — one having heard that the previous owner died, the other never having heard it. Both act, both write, and the data diverges with no record of which was authoritative. Gossip is the right way to learn who probably exists; it is never sufficient for deciding who exclusively owns something.

**Where it is used**

- **Cassandra and Dynamo-style stores**, using gossip for node state and token ownership across large rings.
- **Consul and Serf**, providing SWIM-based membership and event propagation as a service.
- **Redis Cluster**, where nodes gossip slot ownership and liveness on a dedicated bus.
- **Large service meshes**, propagating endpoint health without every proxy polling a central registry.
- **Peer-to-peer and blockchain networks**, where there is no central authority available even in principle.

**In the interview**

Show the scaling argument with numbers and be explicit about what gossip cannot decide.

- **Contrast the traffic directly**: a million pairings for all-to-all at a thousand nodes against a constant few per node.
- **Give the logarithmic spread** — informed nodes multiply each round, so a thousand-node cluster converges in about five.
- **Merge on versions, not timestamps**, because clock skew makes wall-clock merging incorrect.
- **Describe suspect plus indirect probing** as the mechanism that keeps false positives low without slowing detection much.
- **State the limit plainly**: eventual membership cannot support exclusive ownership, so pair gossip with consensus for that.

**Practice drill**

Design membership for a 1,000-node cluster. Compare the message volume of all-to-all heartbeats with gossip at a fan-out of three, and estimate how many rounds a failure rumour needs to reach every node. Describe the merge rule and why it uses versions. Then trace a network partition splitting the cluster 600/400: what each side believes, what happens if partition ownership is decided from that view, and what you would use instead for ownership.

**Go deeper**

Gossip membership propagates cluster state through randomised pairwise exchanges, so every node learns who exists and who appears alive without a central authority.

**Its scaling property is the reason it exists.** All-to-all heartbeating costs traffic quadratic in cluster size, which is untenable past a few hundred nodes, while a central registry replaces that cost with a dependency that must be more available than everything depending on it. Gossip gives each node a constant workload — a few exchanges per interval — regardless of how large the cluster grows, and no node is required for the mechanism to function.

**Information spreads exponentially, which makes the logarithmic round count possible.** Each informed node informs several more, so the informed population multiplies every round and even very large clusters converge in a handful of intervals. This is why increasing fan-out has rapidly diminishing returns: going from three peers to six roughly doubles traffic to save one round, which is almost never the right trade.

**Merging must be version-based, because clocks disagree.** Deciding which of two views is newer by wall-clock time means a node with a fast clock can resurrect stale state and a node with a slow one can have its updates ignored. Monotonic version numbers per entry give a total order that depends on nothing external, and they also give a node a way to refute a rumour of its own death by incrementing past the version that declared it.

**Suspicion and indirect probing are what keep false positives manageable.** A single failed probe indicates only that one path failed, so asking several other nodes to probe before concluding anything separates a dead node from a broken link. Adding an intermediate suspect state gives the accused node a window to object, and that window converts most transient network trouble into a rumour that dies quietly rather than an eviction.

**Eventual convergence is the defining limitation.** For seconds after any change, different nodes hold different views, so one may route to a node another has already buried. That is acceptable for load balancing, health awareness and configuration, and it is unacceptable for anything requiring every participant to agree at the same instant — which is why deriving exclusive partition ownership from a gossip view produces two confident owners and divergent data.

**The mature pattern is gossip plus consensus, each doing what it is good at.** Gossip carries the high-volume, failure-tolerant information cheaply and keeps working on both sides of a partition, which is exactly the availability large clusters need. A small consensus group handles the few decisions that must have a single answer, refusing to proceed without a quorum. Systems that try to make gossip do the second job inherit split-brain; systems that try to make consensus do the first inherit a scaling ceiling and a hard dependency.

**Related patterns:** Heartbeat Failure Detection · Leader Election · Consistent Hashing · Anti Entropy

---

## Partitioning

### Consistent Hashing

*Map keys and nodes onto the same ring so adding or removing a node moves only its share of keys instead of remapping everything.*

> **When you hear…** A node pool that changes size · modulo hashing causing mass remapping · distributed caches and partitioned stores

**Flow:** `Hash keys to ring` → `Hash nodes to ring` → `Key goes clockwise` → `Node leaves` → `Only its keys move`

**The problem**

A cache is spread across ten servers by hashing the key modulo ten. Adding an eleventh server changes the divisor, so nearly every key now maps somewhere different — roughly 91% of the cache is instantly invalid, and the database receives the resulting miss storm.

The same arithmetic applies when a server fails. A pool that changes size for any reason experiences near-total remapping, which makes routine capacity changes and ordinary hardware failures equally catastrophic.

> **Stop mapping keys to positions in a list**  
> Modulo hashing makes every key's destination depend on the total number of nodes, so any change to that number changes almost every answer. If instead both keys and nodes are placed on a fixed circular space, a key belongs to whichever node it meets going clockwise — and removing a node only affects the keys that were pointing at it, because no other key's path changes.

**Mental model**

A circle of hash values. Nodes occupy positions on it; a key is owned by the first node found clockwise from the key's own position.

1. **Place nodes** — Hash each node identifier to a position on the ring, many times over.
2. **Place keys** — Hash each key to a position on the same ring.
3. **Walk clockwise** — The first node encountered owns the key.
4. **Remove a node** — Its keys pass to the next node clockwise; nothing else moves.
5. **Add a node** — It takes a contiguous arc from its clockwise successor; nothing else moves.

> **Without virtual nodes the distribution is badly uneven**  
> Ten randomly placed points on a circle do not divide it into ten equal arcs — some nodes receive several times the share of others, and removing a node hands its entire range to one unlucky successor. Assigning each physical node a hundred or more positions is what makes the distribution acceptably even, and it is not an optimisation but a requirement.

**How it works**

**The ring, and what each change costs**

```text
MODULO HASHING, 10 -> 11 nodes
  hash(key) % 10  vs  hash(key) % 11
  -> ~91% of keys change owner
  -> a near-total cache flush

CONSISTENT HASHING, 10 -> 11 nodes
  the new node claims one arc
  -> ~1/11 of keys move (about 9%)
  -> and ONLY from its clockwise neighbour

VIRTUAL NODES
  without: 10 nodes = 10 arcs, wildly uneven
    worst arc can be 3-5x the average
  with 150 virtual nodes each:
    1500 points around the ring
    -> arcs average out; typical imbalance < 5%
    -> a departing node's load spreads across many
       successors rather than landing on one

LOOKUP
  sorted array of virtual node positions
  binary search for the first position >= hash(key)
  wrap to index 0 if past the end
  -> O(log n), trivially fast

REPLICATION ON THE RING
  walk clockwise past the owner to the next N
  DISTINCT physical nodes
  -> distinct matters; three virtual nodes of the
     same machine is one copy, not three
```

1. **Use 100 to 200 virtual nodes per physical node** — Fewer produces uneven arcs; many more costs memory for little gain.
2. **Weight nodes by giving larger machines more positions** — It is the natural way to handle a heterogeneous fleet.
3. **Walk to distinct physical nodes for replicas** — Otherwise replicas can land on the same machine and provide no redundancy.
4. **Keep the ring definition identical everywhere** — Clients disagreeing about node positions route the same key to different places.
5. **Move data deliberately when the ring changes** — The ring says where a key belongs; it does not move anything.
6. **Consider bounded loads or rendezvous hashing** — They address the imbalance that virtual nodes only reduce.

**Where it still does not help**

```text
WHAT IT FIXES
  pool size changes no longer remap everything
  failures affect one node's share
  scaling up is incremental, not catastrophic

WHAT IT DOES NOT FIX
  HOT KEYS
    one key is 30% of traffic
    -> it hashes to exactly one node, always
    -> that node saturates; the ring is irrelevant
    -> needs key splitting or caching, not hashing

  LARGE PARTITIONS
    one tenant has 100x the data of others
    -> its keys still land wherever they hash
    -> uneven storage despite even key count

  COORDINATED MOVEMENT
    the ring decides ownership instantly
    the DATA has to be copied, which takes time
    -> during the transfer, who serves the key?
    -> needs explicit handoff or read-repair

RENDEZVOUS HASHING (HRW)
  for each key, compute hash(key, node) for every
  node, pick the highest
  -> no ring, no virtual nodes, naturally balanced
  -> O(n) per lookup instead of O(log n)
  -> often the better choice for small node counts
```

| Metric | Value | Note |
|---|---|---|
| Modulo, 10→11 | 91% move | cache flush |
| Consistent, 10→11 | ~9% move | **only one arc** |
| Virtual nodes | 100-200 each | for evenness |
| Not solved | hot keys | a different problem |

> **The ring assigns ownership instantly and moves no data**  
> The moment a node joins, the ring says it owns an arc — but the data for those keys is still on the previous owner and may take minutes to copy. Without an explicit handoff protocol, reads for that arc go to a node that has nothing, and the system either serves misses or returns empty results. Caches tolerate this because a miss is merely slow; stores do not, and need the new owner to serve from the old one until the transfer completes.

**Technologies**

| System | Use | Note |
|---|---|---|
| Memcached clients | Distributing keys across a cache pool | The original popular application |
| Cassandra and Dynamo-style stores | Token ring ownership | Virtual nodes are standard |
| Riak | Partition assignment | Fixed partition count mapped onto nodes |
| Load balancers | Sticky routing by key | Ring hashing as a balancing policy |
| Rendezvous hashing | The same goal, no ring | Better balance; O(n) lookups |
| Bounded-load consistent hashing | Caps per-node load | Overflows to the next node when full |

Bounded-load variants are worth knowing because they address the residual imbalance directly: each node has a capacity limit, and keys that would exceed it spill to the next node clockwise. This keeps the movement properties while preventing any single node from being handed a disproportionate share.

**Trade-offs**

**Key-to-node mapping schemes**

| Scheme | Keys moved on change | Balance | Lookup cost |
|---|---|---|---|
| Modulo hashing | Nearly all | Excellent | O(1) |
| Consistent hashing, no virtual nodes | 1/n | Poor | O(log n) |
| Consistent hashing with virtual nodes | 1/n | Good | O(log n) |
| Rendezvous hashing | 1/n | Excellent | O(n) |
| Fixed partitions mapped to nodes | Whole partitions | Good, controllable | O(1) |
| Central lookup table | Whatever you choose | Perfect | A lookup dependency |

Fixed partitions — creating, say, 1024 partitions once and assigning them to nodes — is what many production systems actually do. It keeps movement bounded, makes rebalancing explicit and inspectable, and avoids the ring entirely at the cost of choosing the partition count up front.

> **Ask before choosing it**  
> How often does the node set actually change, and what happens during the transfer? If the pool is stable and changes are planned, fixed partitions with an explicit assignment map are easier to reason about; the ring earns its place where membership changes frequently or automatically.

**How it fails**

**How consistent hashing goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Severely uneven load | Too few virtual nodes | 100-200 positions per physical node |
| One node overwhelmed after a failure | Departing node's arc landing on one successor | Virtual nodes spread it across many |
| Replicas on the same machine | Walking to positions, not distinct nodes | Skip virtual nodes of the same physical host |
| Keys routed inconsistently | Clients with different ring definitions | Single source of truth for membership |
| Empty reads after scaling | Ownership moved before the data did | Handoff protocol or read from the old owner |
| A hot key still saturates one node | Hashing cannot split a single key | Split the key or cache it locally |
| Storage imbalance despite even keys | Key sizes vary enormously | Weight by data volume, not key count |

> **Even distribution of keys is not even distribution of load**  
> The ring balances how many keys each node owns, which is only a proxy for what actually matters — request volume and data size. One key receiving a third of all traffic, or one tenant holding a hundred times more data than others, produces a saturated node while the ring reports a perfect split. Diagnosing imbalance by key count when the real skew is in access or size is a persistent source of confusion.

**Where it is used**

- **Distributed caches**, the original motivation, where adding capacity must not invalidate everything.
- **Dynamo-style databases**, using a token ring with virtual nodes for both ownership and replica placement.
- **Load balancers with session affinity**, routing a user consistently to the same backend while the pool changes.
- **Content delivery networks**, choosing which edge cache holds which object.
- **Sharded application tiers**, where a request must reach the instance holding a particular in-memory state.

**In the interview**

Lead with the modulo arithmetic; it makes the motivation concrete in one line.

- **Quantify the problem first**: going from ten to eleven nodes remaps about 91% of keys under modulo hashing.
- **Give the consistent-hashing figure** — roughly one node's share moves — and say it comes from only one neighbour.
- **Insist on virtual nodes** with a number, and explain both effects: even arcs and spread-out failure impact.
- **Note that replicas must be distinct physical nodes**, since walking the ring naively can place copies on one machine.
- **Say what it does not solve.** Hot keys and uneven data sizes are separate problems the ring cannot address.

**Practice drill**

A cache pool grows from 10 to 11 nodes. Compute the fraction of keys remapped under modulo hashing and under consistent hashing, and say where the moved keys come from in each case. Choose a virtual node count and justify it by describing what happens with too few. Then explain what a client should do for keys in a newly assigned arc while the data is still being copied, and what changes if this is a persistent store rather than a cache.

**Go deeper**

Consistent hashing maps keys and nodes into the same circular space so that changing the node set relocates only a small, contiguous share of keys.

**It removes a coupling that modulo hashing creates.** When a key's destination is computed from the total node count, every key depends on that count, and changing it invalidates nearly every mapping at once. Placing nodes on a ring makes each key depend only on its own position and the nearest node clockwise, so a change affects the keys in one arc and leaves everything else untouched — which is what turns adding capacity from an incident into a routine operation.

**Virtual nodes are not a refinement; without them the scheme barely works.** A handful of randomly placed points divides a circle very unevenly, so some nodes receive multiples of the average load, and a departing node dumps its whole range onto a single successor. Giving each machine a hundred or more positions averages the arcs out and spreads a failure's load across many nodes, which is the behaviour people assume the ring provides by itself.

**Ownership and data movement are separate events, and conflating them causes outages.** The ring declares a new node the owner the instant it joins, while the data for that arc still lives on the previous owner and takes time to copy. Caches absorb this as extra misses; persistent stores cannot, and need an explicit handoff in which the new owner serves from the old one, or reads fall back, until the transfer completes.

**Balanced key counts do not imply balanced load.** The ring distributes keys, but traffic concentrates on popular keys and storage concentrates on large ones, so a node can be saturated while the distribution looks perfect. A single key receiving a large share of requests always lands on one node no matter how the ring is arranged, which means hot keys require splitting or local caching — a different pattern entirely, not a hashing parameter to tune.

**Replica placement needs an extra rule.** Walking clockwise for additional copies naturally encounters more virtual nodes, and several of them may belong to the same physical machine, producing replicas that all die together. Skipping to distinct hosts — and, in rack- or zone-aware systems, to distinct failure domains — is what makes the replication factor mean what it claims.

**Alternatives are often the better fit, and knowing why matters.** Rendezvous hashing achieves the same movement properties with better balance and no virtual nodes, at the cost of evaluating every node per lookup, which is fine for modest fleets. Fixed partitions assigned to nodes through an explicit map give bounded movement with an inspectable, controllable assignment, which many production systems prefer precisely because rebalancing becomes a deliberate operation rather than an emergent consequence of a hash function.

**Related patterns:** Sharding · Hot Key Mitigation · Gossip Membership · Quorum Replication

---

### Sharding

*Split data across independent databases by a chosen key, so capacity scales horizontally at the cost of cross-shard operations.*

> **When you hear…** One database cannot hold the data or serve the writes · vertical scaling exhausted · a natural partition key exists

**Flow:** `Choose shard key` → `Route by key` → `Independent shards` → `Cross-shard queries hard` → `Rebalance deliberately`

**The problem**

A single database has reached the largest instance available and is still saturating on writes. Read replicas absorb reads but every write still goes to one machine, and the working set no longer fits in memory, so latency has become unpredictable.

Splitting is the only remaining direction, and it is irreversible in practice. Once data lives in several places, transactions, joins and unique constraints that spanned the whole dataset no longer work, and the application has to change to match.

> **Sharding buys capacity by giving up the single-database guarantees**  
> Each shard is an ordinary database with ordinary performance; putting ten of them together gives roughly ten times the capacity. What is lost is everything that depended on all the data being in one place — cross-entity transactions, arbitrary joins, global uniqueness and global ordering. The shard key determines which queries stay easy and which become distributed problems, which makes it the most consequential decision in the design.

**Mental model**

Independent databases holding disjoint subsets of the data, with a routing rule mapping every request to exactly one of them.

1. **Choose the key** — Pick the attribute that partitions the data along the access pattern.
2. **Map** — Define how key values become shard assignments.
3. **Route** — Every query resolves its shard before executing.
4. **Isolate** — Each shard runs independently, with its own resources and its own failures.
5. **Rebalance** — Adding capacity moves ranges of keys, deliberately and slowly.

> **The shard key is chosen once and is extremely expensive to change**  
> Changing it means rewriting every row into a new arrangement while the system serves traffic, typically through a dual-write migration lasting weeks. Teams therefore live with an unsuitable key far longer than they should, working around it with cross-shard queries and application-level joins — which is why the time spent choosing it is the best-spent time in the project.

**How it works**

**Choosing the shard key**

```text
THE TEST
  which single value appears in almost every query?
  that is your candidate

  e-commerce   -> customer_id
  multi-tenant -> tenant_id
  messaging    -> conversation_id
  social       -> user_id

A GOOD KEY
  high cardinality       many distinct values
  even distribution      no value dominates
  present in most queries so routing is possible
  stable                 it never changes for a row

A BAD KEY
  status / country / type   low cardinality, skewed
  timestamp                 all writes hit one shard
  auto-increment id         same, and no locality

THE CONSEQUENCE OF THE CHOICE
  shard by customer_id:
    "orders for customer X"      one shard, fast
    "all orders today"           every shard, slow
  shard by order_date:
    "all orders today"           one shard, fast
    "orders for customer X"      every shard, slow
  -> you are choosing which query is cheap; you
     cannot have both

CO-LOCATION
  put related entities on the SAME shard by using
  the same key
    customer, their orders, their addresses
  -> keeps the common transaction local
```

1. **Choose the key from the dominant access pattern** — Everything else follows from it, and it is nearly immutable.
2. **Co-locate related entities under the same key** — It keeps the transactions you actually need inside one shard.
3. **Use hash or range mapping deliberately** — Hash distributes evenly; range keeps scans local and creates hot spots.
4. **Route through a layer, not through hardcoded connections** — Rebalancing requires the mapping to be changeable at runtime.
5. **Plan for cross-shard queries** — Some will exist; decide whether they scatter, or are served from a separate store.
6. **Choose the shard count with headroom** — Resharding is a migration; over-provisioning logical shards avoids most of them.

**Cross-shard operations, and the logical shard trick**

```text
WHAT BECOMES HARD
  joins across shards      application-side, or a
                           denormalised copy
  transactions across      saga or two-phase commit
  global uniqueness        a separate service, or a
                           key that includes the shard
  global ordering          per-shard only
  aggregate queries        scatter-gather, or a
                           warehouse

SCATTER-GATHER COST
  query 16 shards, wait for all
  latency = the SLOWEST shard, not the average
  -> component tail becomes system median
  -> fine occasionally, unusable as the main pattern

LOGICAL SHARDS  (the standard mitigation)
  create 1024 logical shards on day one
  map them to 4 physical databases (256 each)
  growth: move logical shards, not rows
    4 -> 8 databases: move 512 logical shards
  -> rebalancing becomes a file copy plus a mapping
     change, not a row-by-row migration
  -> the expensive resharding never happens

READ PATH FOR CROSS-SHARD NEEDS
  keep a denormalised, differently-keyed copy in a
  search index or warehouse
  -> the shards serve the operational path; the copy
     serves the analytical one
```

| Metric | Value | Note |
|---|---|---|
| Capacity | n× per shard | **linear** |
| Lost | joins, transactions | across shards |
| Key choice | near-permanent | choose carefully |
| Mitigation | logical shards | 1024 from day one |

> **Scatter-gather latency is governed by the slowest shard**  
> A query touching sixteen shards completes when the last one answers, so the 99th percentile of a single shard becomes roughly the median of the composed query. This is the same tail-at-scale arithmetic that makes fan-out expensive everywhere, and it means a query pattern that requires touching every shard is not merely slower — it is systematically unpredictable, and no amount of per-shard optimisation fixes it.

**Technologies**

| Approach | Examples | Note |
|---|---|---|
| Application-level sharding | Routing logic in the service | Full control; full responsibility |
| Sharding middleware | Vitess, Citus | Routing and rebalancing handled for you |
| Natively sharded databases | MongoDB, Cassandra, CockroachDB | Sharding is built in, with its own constraints |
| Managed distributed SQL | Spanner-style systems | Cross-shard transactions, at a cost |
| Functional partitioning | Split by service, not by key | Often the better first step |
| Do not shard | Bigger instance, read replicas, archiving | Frequently sufficient for years |

The last row is genuine advice. Modern single instances handle very large datasets and high write rates, and archiving cold data or moving one heavy table out often buys several years — during which the access patterns become clearer and the eventual shard key choice is better informed.

**Trade-offs**

**Scaling a database**

| Approach | Write capacity | Query flexibility | Operational cost |
|---|---|---|---|
| Bigger instance | Limited by hardware | Unchanged | Lowest |
| Read replicas | Unchanged | Unchanged, with replica lag | Low |
| Functional partitioning | Per service | Lost across services | Medium |
| Sharding by key | Linear with shards | Lost across shards | High |
| Distributed SQL | Linear | Mostly preserved | High, plus latency |
| Archiving cold data | Unchanged | Unchanged | Low |

Functional partitioning deserves attention before key-based sharding: moving one heavy domain into its own database often removes the pressure entirely, and it preserves query flexibility within each domain rather than fragmenting everything by a single key.

> **Ask before choosing it**  
> Which queries must remain fast, and does one key appear in all of them? If the important queries have no key in common, sharding will make half of them scatter-gather regardless of the choice — and that usually means the data should be split functionally, or served from a second, differently-keyed store.

**How it fails**

**How sharding goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| One shard saturated | Skewed shard key | Choose a high-cardinality even key; split hot values |
| All writes hitting one shard | Time-based or sequential key | Hash the key, or shard on something else |
| Most queries touch every shard | Key does not match access patterns | Re-evaluate the key; add a differently-keyed read store |
| Latency dominated by one slow shard | Scatter-gather as the main pattern | Denormalise so the common query is single-shard |
| Resharding takes months | Physical shards with no logical layer | Many logical shards mapped to few physical ones |
| Cross-shard transactions needed constantly | Related entities on different shards | Co-locate by using the same key |
| Unique constraints violated | Uniqueness enforced per shard only | Separate uniqueness service, or include the shard in the key |
| Rebalancing causes an outage | Moving data without a handoff | Dual-write, verify, then cut over |

> **A skewed shard key reproduces the original problem with more machines**  
> Sharding by a low-cardinality or unevenly distributed attribute concentrates a large fraction of traffic on one shard, which then saturates exactly as the single database did — except now there is routing, rebalancing and cross-shard complexity on top. The capacity gain is proportional to how evenly the key distributes, and a bad key delivers none of it while charging the full operational price.

**Where it is used**

- **Multi-tenant SaaS platforms**, sharding by tenant so each customer's data and transactions stay local.
- **Large e-commerce systems**, sharding by customer with orders and addresses co-located.
- **Messaging systems**, sharding by conversation so a thread's history and writes stay on one shard.
- **Vitess-style deployments**, where logical shards and a routing layer make rebalancing routine.
- **Time-series stores**, sharding by time range where recent data is hot and old data is archived.

**In the interview**

Spend the time on the key and on what becomes hard; the mechanics of routing are the easy part.

- **Derive the key from the dominant query**, and state explicitly which queries become expensive as a result.
- **Say the key is effectively permanent** and that changing it is a multi-week dual-write migration.
- **Propose logical shards from day one** — 1024 mapped onto a few databases — so rebalancing is a mapping change.
- **Do the scatter-gather arithmetic.** The slowest shard sets the latency, so fan-out queries are systematically unpredictable.
- **Suggest cheaper options first**: functional partitioning, archiving, a bigger instance. Sharding before those is usually premature.

**Practice drill**

An e-commerce database has outgrown its largest instance. Choose a shard key and justify it against the three most common queries, stating which becomes expensive. Design the logical-to-physical mapping and say how you would go from 4 databases to 8. Then handle two specific problems: a query for all orders placed today, and a transaction spanning two customers. Finally, state what you would try before sharding at all.

**Go deeper**

Sharding splits data across independent databases by a key, trading the guarantees of a single database for horizontal capacity.

**The shard key is the design, and everything else is consequence.** It determines which queries resolve to one shard and which must touch all of them, which entities can participate in a local transaction, and how evenly load distributes. Because changing it requires rewriting the entire dataset while serving traffic, the choice is effectively permanent, and teams routinely spend years working around a key that was picked in an afternoon.

**What is lost is specific and worth enumerating before committing.** Joins across shards become application-level work or require a denormalised copy; transactions spanning shards need sagas or a distributed protocol; global uniqueness and global ordering stop being available from the database. None of these are insurmountable, but each one is code that did not exist before, and the total is the real price of the capacity gained.

**Scatter-gather is the failure mode that makes a bad key visible.** A query touching every shard finishes when the slowest one answers, so the tail latency of a single shard becomes the typical latency of the query. Occasional fan-out is tolerable; fan-out as the dominant pattern produces a system whose response times are unpredictable in a way no per-shard tuning addresses, and it is the clearest signal that the key does not match how the data is actually read.

**Logical shards convert resharding from a migration into a configuration change.** Creating far more logical partitions than physical databases — a thousand or so mapped onto a handful — means growth moves whole partitions rather than individual rows, which can be done by copying files and updating a mapping. Systems built without this layer face a genuine data migration every time they add capacity, which is why that migration keeps being postponed past the point where it was needed.

**Even distribution is what the capacity gain is proportional to.** A key with low cardinality or heavy skew concentrates traffic on one shard, which saturates exactly as the original single database did, while the system now also carries routing, rebalancing and cross-shard complexity. Verifying the distribution of the proposed key against real data, before committing, is the step that separates a successful shard from an expensive one.

**Sharding should be a late decision, not an early one.** A single modern instance holds far more data and sustains far higher write rates than most teams assume, and archiving cold rows, moving one heavy table out, or splitting the database functionally by domain often buys several years. That time is valuable in itself: access patterns become clear, the dominant queries stabilise, and the shard key chosen afterwards is chosen with evidence rather than with a guess about how the product will be used.

**Related patterns:** Consistent Hashing · Hot Key Mitigation · Scatter Gather · Saga

---

### Hot Key Mitigation

*Spread or absorb traffic for a single dominant key, since partitioning by hash always sends one key to exactly one place.*

> **When you hear…** One partition saturated while others idle · a celebrity account or viral item · a counter every request touches

**Flow:** `One key dominates` → `Hashes to one node` → `Node saturates` → `Split or replicate key` → `Load spreads`

**The problem**

A social platform shards by user. One account with eighty million followers receives a thousand times the traffic of a typical user, and every one of those requests lands on the single shard that owns it. That shard is saturated while the other fifteen sit at ten percent.

Nothing about the partitioning is wrong. Hashing sends a key to one place by definition, which is precisely the property that makes routing work — and it means a single key's load is fundamentally unsplittable by the partitioning scheme itself.

> **If one key is too big, stop treating it as one key**  
> The load cannot be divided while the key is atomic, so the remedy is to make it non-atomic: split the key into variants that hash differently, replicate it so several nodes can serve it, or absorb the traffic before it reaches the partition at all. Each of these changes what the key means, which is why the right choice depends on whether the traffic is reads, writes, or both.

**Mental model**

Three distinct remedies chosen by traffic type: absorb reads in a cache, replicate reads across nodes, or split writes across sub-keys that are recombined on read.

1. **Detect** — Identify which keys are hot, continuously rather than once.
2. **Classify** — Determine whether the pressure is reads, writes, or storage.
3. **Absorb** — For reads, cache in front so most requests never reach the partition.
4. **Split** — For writes, shard the key itself into N sub-keys and aggregate on read.
5. **Adapt** — Hotness changes hourly; the mitigation must be applied and removed automatically.

> **Splitting a key destroys the atomicity that made it a single key**  
> A counter split into a hundred sub-counters can no longer be read atomically — the total is a sum of values read at slightly different moments, and it is never exactly right. For a view count this is irrelevant; for an inventory level that must never oversell it is unacceptable. The split is a trade of correctness properties, not a free optimisation.

**How it works**

**Remedies by traffic type**

```text
HOT READS
  1  local cache in every application instance
       TTL 1-5 s, no coordination
       -> 16 instances = 16x reduction, almost free
       -> serves slightly stale data
  2  replicate the key to every partition
       read from any, write to all
       -> reads scale perfectly, writes get expensive
  3  CDN or edge cache for anything public
       -> the request never reaches your system

HOT WRITES  (counters, rate limits, likes)
  split the key:
    counter:post:123        -> saturated
    counter:post:123:0..99  -> 100 sub-counters
    write: pick a random suffix, increment
    read:  sum all 100
  -> writes spread 100x
  -> reads cost 100 lookups and are approximate

HOT PARTITION FROM MANY KEYS
  not one key, but a tenant whose keys all land in
  one place
  -> include a spreading component in the key
  -> or give that tenant dedicated capacity

DETECTION
  sample request keys continuously
  heavy-hitter sketch (count-min) per partition
  -> alert when one key exceeds ~5% of a partition
```

1. **Cache hot reads locally before anything more elaborate** — A few seconds of in-process TTL removes most read pressure with no coordination.
2. **Split hot write keys into a fixed number of sub-keys** — Spreads writes by that factor; reads become an aggregation.
3. **Choose the split factor from the actual overload** — Too small does not help; too large makes every read expensive.
4. **Detect hotness continuously and automatically** — Yesterday's hot key is not today's, and manual lists are always stale.
5. **Apply and remove mitigation dynamically** — A permanent split penalises reads for a key that is no longer hot.
6. **Consider dedicated capacity for a permanently hot tenant** — Sometimes the honest answer is that one customer needs their own shard.

**What each remedy costs**

```text
SCENARIO
  post:123 receives 50,000 reads/s and 5,000
  writes/s; a shard handles 10,000 ops/s

LOCAL CACHE, 1 s TTL, 20 app instances
  reads reaching the shard: 20/s (one per instance
  per second)
  writes unchanged: 5,000/s -> still overloaded
  -> solves reads completely, writes not at all
  -> data up to 1 s stale

SPLIT WRITES INTO 10 SUB-KEYS
  each sub-key: 500 writes/s
  spread across 10 shards -> comfortable
  reads must sum 10 keys
  -> combine with the local cache: 20 reads/s x 10
     lookups = trivial
  -> total is eventually consistent

REPLICATE TO ALL 16 SHARDS
  reads: 50,000 / 16 = 3,125 per shard, fine
  writes: 5,000 x 16 = 80,000 total, far worse
  -> only sensible for read-heavy keys

THE COMBINATION THAT USUALLY WINS
  local cache for reads
  + key splitting for writes
  -> each mechanism applied to the traffic it suits
```

| Metric | Value | Note |
|---|---|---|
| Local cache | reads only | **seconds stale** |
| Key splitting | writes | approximate reads |
| Replication | read-heavy | writes multiply |
| Detection | continuous | hotness moves |

> **A static list of hot keys is obsolete before it is deployed**  
> Hotness is created by events — a post goes viral, a product is featured, a celebrity joins — and it shifts within hours. Hard-coding known hot keys means the mitigation protects yesterday's traffic while today's hot key saturates a shard unprotected. Detection has to be a running measurement feeding an automatic response, and the response must expire when the key cools or reads pay the aggregation cost forever.

**Technologies**

| Remedy | Mechanism | Note |
|---|---|---|
| Local in-process cache | Short TTL per instance | Cheapest and most effective for reads |
| Distributed cache in front | Shared cache layer | Still one key, so it can move the hot spot |
| Key splitting | Suffixed sub-keys aggregated on read | The standard remedy for hot writes |
| Full replication of hot items | Copy to every partition | Reads scale; writes multiply |
| Heavy-hitter detection | Count-min sketch, sampling | Cheap, continuous identification |
| Dedicated capacity | A shard per large tenant | Honest answer for permanent skew |

A shared distributed cache is a partial remedy worth flagging: the hot key remains a single key in that cache, so it can simply relocate the hot spot from the database to one cache node. Local caching avoids this because the key exists independently in every instance.

**Trade-offs**

**Hot key remedies**

| Remedy | Helps reads | Helps writes | Cost |
|---|---|---|---|
| Local cache with short TTL | Enormously | No | Staleness of a few seconds |
| Distributed cache | Yes | No | May move the hot spot |
| Key splitting | No | Yes, by the split factor | Approximate, more expensive reads |
| Replicate to all partitions | Yes | Makes worse | Write amplification |
| Dedicated shard for the key or tenant | Yes | Yes | Operational and routing complexity |
| Rate limit the hot key | Protects others | Protects others | Degrades that key deliberately |

Rate limiting the hot key is a legitimate last resort and is used in practice: deliberately degrading the one item that is overwhelming a shard preserves service for everything else, which is a better outcome than a shard that fails for all its keys.

> **Ask before choosing it**  
> Is the hot key read-heavy or write-heavy? The remedies are almost disjoint — caching does nothing for writes, splitting does nothing for reads — so answering this first prevents applying an elaborate mitigation to the wrong half of the traffic.

**How it fails**

**How hot key mitigation goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Mitigation protects the wrong key | Static hot-key list | Continuous detection driving automatic response |
| Reads become expensive everywhere | Key splitting left in place after cooling | Expire the split when traffic normalises |
| Hot spot moved, not removed | Distributed cache with the same single key | Local caches, or replicate the cached entry |
| Counter drifts or double counts | Split sub-counters summed inconsistently | Accept approximation, or do not split |
| Writes get worse | Replicated a write-heavy key | Split instead of replicating |
| Oversold inventory | Split a key that required atomicity | Do not split keys with correctness constraints |
| Shard still saturated | Skew from many keys, not one | Re-key the tenant, or give it dedicated capacity |

> **Splitting a key that carries a correctness constraint is how inventory oversells**  
> A stock level maintained as a hundred sub-counters cannot be checked atomically, so two requests can each read a total that says one item remains and both proceed. For view counts and likes the approximation is harmless and the split is the right answer; for anything that must never go negative, the atomicity is the feature and the correct remedies are caching, rate limiting or dedicated capacity — never division.

**Where it is used**

- **Social platforms with celebrity accounts**, where one profile's reads dwarf the median by orders of magnitude.
- **E-commerce flash sales**, where a single product becomes the entire write load for a shard.
- **View and like counters**, the canonical use of key splitting with approximate reads.
- **Rate limiters**, where a single API consumer's counter concentrates on one node.
- **Multi-tenant platforms**, where a large customer's keys collectively overload one partition.

**In the interview**

Say early that partitioning cannot help here; that framing is what makes the rest of the answer coherent.

- **State the constraint**: a single key hashes to one node, so no partitioning scheme can split its load.
- **Separate read remedies from write remedies** and note they are almost disjoint.
- **Offer local caching first for reads** — a one-second TTL across twenty instances removes nearly all of it.
- **Describe key splitting for writes** with a concrete factor, and name what it costs: approximate, more expensive reads.
- **Require continuous detection** and automatic expiry, since hotness shifts within hours and static lists protect the wrong thing.

**Practice drill**

A post receives 50,000 reads and 5,000 writes per second on a shard that handles 10,000 operations per second. Choose remedies for each half of the traffic, with concrete parameters, and compute the resulting load on the shard. Say what correctness property you gave up and whether it matters for a view counter. Then redo the analysis for an inventory count that must never oversell, and explain which remedies are no longer available.

**Go deeper**

Hot key mitigation addresses load concentrated on a single key, which partitioning cannot spread because a key maps to one place by construction.

**The constraint is structural, not a tuning failure.** Hashing exists to send a given key consistently to one node, and that property is what makes routing possible at all. A key receiving a disproportionate share of traffic therefore saturates its node regardless of how many nodes exist or how evenly the ring is arranged, which is why adding capacity does nothing and why the remedy has to change the key rather than the partitioning.

**Read and write remedies are almost entirely separate.** Caching removes read pressure and has no effect on writes; splitting a key spreads writes and makes reads more expensive; replicating to every partition scales reads and multiplies writes. Establishing which half of the traffic is the problem before choosing is what prevents deploying an elaborate mechanism against the wrong bottleneck, and systems with both hot reads and hot writes usually need one of each.

**Short-lived local caching is the highest-leverage read remedy.** A one-second TTL in each application instance reduces the shard's read load by the instance count, requires no coordination, invalidation or shared infrastructure, and costs only a second of staleness on a value that is changing constantly anyway. A shared distributed cache is a weaker answer for this specific problem, because the hot key remains one key in that cache and can simply relocate the hot spot.

**Key splitting trades atomicity for write throughput, and the trade is not always acceptable.** A counter divided into sub-counters spreads writes by the split factor while making the total a sum of values read at different instants — never precisely correct. That is entirely fine for views and likes, where nobody can tell and nothing depends on exactness, and it is wrong for a stock level, where the atomic check is the mechanism preventing overselling. Recognising which constraint the key carries decides whether splitting is available.

**Hotness is transient, so the mitigation must be too.** Keys become hot because of events that pass within hours, and a hard-coded list protects last week's traffic while today's viral item saturates a shard unprotected. Detection needs to be a continuous measurement — sampling with a heavy-hitter sketch is cheap enough to run always — feeding an automatic response, and that response has to expire, or reads keep paying the aggregation cost for keys that cooled long ago.

**Some skew is permanent and deserves an honest answer.** A tenant genuinely a hundred times larger than the median, or an account that will always be extraordinary, is not a transient anomaly to be smoothed; dedicated capacity or a shard of their own is simpler and more predictable than an ever more elaborate mitigation. Deliberately rate limiting the hot key is also legitimate at the extreme, because degrading one item preserves service for every other key sharing that partition, which is the better of two bad outcomes.

**Related patterns:** Consistent Hashing · Sharding · Cache Aside · Token Bucket

---

## Replication

### Quorum Replication

*Write to a majority of replicas and read from a majority, so any read is guaranteed to see the most recent write without contacting them all.*

> **When you hear…** Replicated data that must not serve stale reads · availability required despite replica failures · tunable consistency needed per operation

**Flow:** `N replicas` → `Write waits for W` → `Read waits for R` → `R plus W over N` → `Overlap guarantees freshness`

**The problem**

Data is stored on five replicas. Waiting for all five on every write means one slow or failed replica blocks all writes; waiting for just one means a subsequent read may reach a replica that never received it and return stale data.

Neither extreme is acceptable. Full agreement gives consistency and no availability; single-replica acknowledgement gives availability and no guarantee about what a read will see.

> **Overlapping majorities make agreement unnecessary**  
> If a write is acknowledged by more than half the replicas and a read consults more than half, the two sets must share at least one replica — and that shared replica has the latest write. Freshness comes from set intersection rather than from contacting everyone, which means some replicas can be down or slow without affecting correctness at all.

**Mental model**

Three numbers: N replicas, W acknowledgements required for a write, R responses required for a read. When R plus W exceeds N, the read and write sets must overlap.

1. **Write** — Send to all N, return once W have acknowledged.
2. **Read** — Query enough replicas to collect R responses.
3. **Compare** — Use version information to determine which response is newest.
4. **Return** — Serve the newest value found among the R.
5. **Repair** — Update the replicas that were behind, so the divergence does not persist.

> **The overlap guarantees the latest value is present, not that it is chosen**  
> A quorum read reaches at least one replica holding the newest write — but it also reaches replicas holding older versions, and something must decide which is newest. That requires version vectors or timestamps carried with the data. A system that returns the first response, or resolves conflicts by wall-clock time across skewed machines, has the overlap and still serves stale or wrong values.

**How it works**

**Choosing N, W and R**

```text
THE RULE
  R + W > N   guarantees the read set overlaps the
              write set

N = 3
  W=2, R=2   R+W=4 > 3   balanced, the usual default
             tolerates 1 failure for both
  W=3, R=1   R+W=4 > 3   fast reads, writes fail if
             any replica is down
  W=1, R=3   R+W=4 > 3   fast writes, reads need all
  W=1, R=1   R+W=2 < 3   NO guarantee; eventual only

N = 5
  W=3, R=3   tolerates 2 failures on each side
  W=4, R=2   read-optimised
  W=2, R=4   write-optimised

LATENCY
  a quorum operation is as slow as the Wth fastest
  replica, not the slowest
  -> N=5, W=3 waits for the 3rd of 5 responses
  -> this is why quorums beat full replication on
     tail latency

AVAILABILITY
  writes survive N - W failures
  reads survive N - R failures
  -> W=2, R=2, N=3 survives exactly one failure
```

1. **Default to N=3 with W=2 and R=2** — It tolerates one failure on both paths and is the balanced choice.
2. **Tune W and R per operation where the store allows it** — Some reads need freshness; many can accept a faster, weaker read.
3. **Carry version information with every value** — Overlap only helps if the newest version can be identified.
4. **Repair stale replicas on read** — Otherwise divergence persists and accumulates.
5. **Spread replicas across failure domains** — Three replicas in one rack tolerate one disk failure, not one rack failure.
6. **Understand that quorum is not a transaction** — Concurrent writes can both succeed and produce a conflict to resolve.

**What a quorum does not give you**

```text
SLOPPY QUORUM
  the designated replicas are unreachable
  -> write to ANY W reachable nodes instead
  -> the write succeeds, availability preserved
  -> BUT it did not go to the real replicas, so
     R + W > N no longer holds
  -> hinted handoff later moves it home
  -> during that window, reads can be stale

CONCURRENT WRITES
  two clients write different values at the same
  time, both reach quorums
  -> both succeed
  -> the versions conflict
  -> resolution: last-write-wins (loses data), or
     version vectors plus application merge

LAST-WRITE-WINS HAZARD
  decided by wall-clock timestamps
  clock skew of 50 ms between nodes
  -> the objectively later write can lose
  -> silently, with no error anywhere

WHAT QUORUMS DO NOT DO
  no atomic multi-key operations
  no read-your-writes across different clients
  no ordering between keys
  -> for those, consensus (Raft/Paxos) is required,
     which is a stronger and more expensive tool
```

| Metric | Value | Note |
|---|---|---|
| Rule | R + W > N | **overlap** |
| Default | 3 / 2 / 2 | one failure tolerated |
| Latency | Wth fastest | not the slowest |
| Not provided | atomicity | use consensus |

> **Sloppy quorums trade the guarantee for availability, quietly**  
> When the proper replicas are unreachable, some systems accept the write on whichever nodes are available so the operation succeeds. This keeps writes flowing during a partition and means the arithmetic no longer holds — the write is not on the replicas a future read will consult, so a quorum read can miss it entirely. It is a deliberate and often correct choice, but a system configured for strong reads that silently degrades to sloppy quorums is not providing what its configuration claims.

**Technologies**

| System | Approach | Note |
|---|---|---|
| Cassandra | Tunable per query | ONE, QUORUM, ALL selectable per statement |
| DynamoDB | Eventually or strongly consistent reads | The quorum is hidden behind two options |
| Riak | Explicit N, R, W | Dynamo's model exposed directly |
| MongoDB | Write and read concerns | Majority is the quorum equivalent |
| etcd and ZooKeeper | Consensus, not quorum voting | Stronger: ordering and atomicity too |
| Single primary with replicas | Primary decides | Simpler; failover is the hard part |

The distinction between the last two rows matters in discussion: consensus systems also use majorities, but they order operations through a leader, which gives atomicity and linearizability that plain quorum voting does not — at the cost of the leader being a bottleneck and a failover point.

**Trade-offs**

**Replication acknowledgement strategies**

| Strategy | Consistency | Write availability | Latency |
|---|---|---|---|
| Write all, read one | Strong | Poor; any failure blocks | Slowest replica |
| Write one, read all | Strong | Excellent | Reads slowest |
| Quorum W=R=majority | Strong enough | Tolerates a minority down | Middle replica |
| Write one, read one | Eventual | Excellent | Fastest |
| Sloppy quorum | Weakened | Highest | Fast |
| Consensus | Linearizable, ordered | Tolerates a minority down | Leader round trip |

Write-one-read-one is a legitimate choice for data where staleness is harmless — metrics, activity feeds, caches — and it is substantially faster and more available than any quorum. The mistake is applying it uniformly to data where a stale read has consequences.

> **Ask before choosing it**  
> Does every read need the latest value, or only some of them? Tuning per operation is usually available and frequently better than one setting for the whole store, since most systems have a small set of reads that require freshness and a large set that does not.

**How it fails**

**How quorum replication goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Stale reads despite a quorum | R + W not greater than N | Check the arithmetic against the configuration |
| Stale reads with correct arithmetic | Sloppy quorum during a partition | Know when the store degrades; account for it |
| Writes lost silently | Last-write-wins with skewed clocks | Version vectors, or an application merge |
| Replicas diverge permanently | No read repair or anti-entropy | Repair on read; run background reconciliation |
| All replicas lost together | Replicas in one failure domain | Spread across racks and zones |
| Latency worse than expected | W or R set equal to N | Use a majority, not everything |
| Concurrent writes produce surprises | Expecting transactional behaviour | Quorums are not transactions; use consensus |

> **Last-write-wins discards data according to clock skew**  
> When conflicting versions are resolved by comparing wall-clock timestamps, the winner is decided by whose machine had the later clock rather than by what actually happened last. A few tens of milliseconds of skew is enough for a genuinely later write to lose, and the loss is silent — no error, no conflict surfaced, just a value that quietly reverts. Version vectors detect the conflict instead, which lets the application decide rather than the clocks.

**Where it is used**

- **Dynamo-style databases**, where tunable N, R and W are the core of the consistency model.
- **Cassandra deployments**, choosing consistency per query so critical reads are strong and bulk reads are fast.
- **Object stores**, acknowledging writes once a durable majority of copies exist.
- **Distributed caches with replication**, trading read consistency for latency deliberately.
- **Consensus-backed configuration stores**, which use majorities plus ordering for stronger guarantees.

**In the interview**

The arithmetic is quick to state; spend the time on what overlap does not give you.

- **State R + W > N and why it works** — the sets must intersect, so a read touches a replica with the newest write.
- **Give a default and its failure tolerance**: N=3, W=2, R=2 survives one replica being down on both paths.
- **Note the latency property.** A quorum waits for the Wth fastest replica, which is why it beats waiting for all.
- **Raise conflict resolution.** Overlap finds the newest version only if versions are comparable; last-write-wins loses data under skew.
- **Distinguish quorum from consensus**: no atomicity, no ordering across keys, and sloppy quorums can void the guarantee entirely.

**Practice drill**

Design replication for a user profile store with five replicas. Choose W and R, verify the overlap condition, and state how many replica failures each path tolerates. Then work through two scenarios: two clients writing different values concurrently, and a partition in which the store falls back to a sloppy quorum. Say what a reader can observe in each case, and what mechanism you would add so the divergence does not persist.

**Go deeper**

Quorum replication guarantees fresh reads by requiring the read and write sets to overlap, rather than by contacting every replica.

**The guarantee comes from set intersection, which is why it tolerates failures.** If more than half the replicas acknowledged the write and more than half are consulted on read, at least one replica is in both sets and holds the newest value. Nothing needs to coordinate, no replica is essential, and the operation proceeds while a minority is down — properties that waiting for all replicas cannot provide.

**It also improves tail latency, which is an underrated benefit.** A write that waits for every replica is as slow as the slowest, so one degraded machine sets the latency for all operations. A quorum waits for the Wth fastest response and simply leaves the stragglers behind, which means transient slowness on a minority of replicas never reaches the client at all.

**Overlap locates the newest value; it does not identify it.** Among the R responses are several versions, and something must determine which is current. That requires version information carried with the data — logical versions or version vectors — because deciding by wall-clock timestamp means clock skew picks the winner. Last-write-wins configured over skewed clocks discards genuinely newer writes silently, which is among the most confusing data-loss modes a system can have.

**Sloppy quorums preserve availability by abandoning the arithmetic.** When the designated replicas are unreachable, accepting the write on whatever nodes are available keeps the system writable during a partition, and the write is then not where a future quorum read will look. This is frequently the right operational choice, and it means the configured consistency is a best case rather than a guarantee — so a design claiming strong reads must also know when its store silently degrades.

**Tuning per operation is usually the right granularity.** Most systems have a small number of reads that genuinely need the latest value and a large number that do not, so a single store-wide setting either pays for consistency everywhere or provides it nowhere. Choosing per query costs a little care at each call site and avoids both the latency of unnecessary strong reads and the surprises of accidentally weak ones.

**A quorum is weaker than consensus, and the gap is where designs go wrong.** Overlapping majorities give per-key freshness and nothing more: no atomic multi-key operations, no ordering between keys, and concurrent writes that both succeed and leave a conflict behind. Systems needing a single agreed sequence of operations — leader election, configuration, anything with invariants spanning keys — require a consensus protocol, which uses majorities too but adds the ordering that makes those guarantees possible, at the cost of a leader and its failover.

**Related patterns:** Read Repair · Anti Entropy · Geographic Replication · Leader Election

---

### Geographic Replication

*Keep copies of data in several regions so reads are local and a region failure is survivable, accepting that writes cannot be both fast and globally consistent.*

> **When you hear…** Users spread across continents · a region outage must not be an outage · data residency requirements

**Flow:** `Regions worldwide` → `Local reads fast` → `Writes cross oceans` → `Choose consistency` → `Handle conflicts`

**The problem**

A service hosted in one region serves users in Sydney with a round trip of roughly 250 milliseconds before any work is done. Several dependent requests turn that into seconds of waiting caused entirely by the speed of light.

A single region is also a single failure domain. When it becomes unavailable — and every region eventually does — the whole service is unavailable, regardless of how well it is engineered internally.

> **Distance is a fixed cost that can be paid on writes or on reads, but not avoided**  
> Data in one place means somebody is always far away. Copying it to several regions makes reads local everywhere, and moves the problem to writes: either a write waits for distant regions to acknowledge, which makes it slow but consistent, or it returns locally and propagates afterwards, which makes it fast and allows conflicting concurrent writes. Every geo-replication design is a choice between those two costs.

**Mental model**

Full or partial copies of the data in each region, with a replication topology deciding where writes are accepted and how they reach everywhere else.

1. **Place** — Decide which regions hold copies, and of what data.
2. **Route** — Send each user to the nearest region that can serve them.
3. **Accept writes** — In one region only, or in any region.
4. **Propagate** — Ship changes across regions, synchronously or asynchronously.
5. **Reconcile** — Detect and resolve conflicts when more than one region accepts writes.

> **Cross-region latency is physics, not a tuning parameter**  
> London to Sydney is about 250 milliseconds round trip at realistic fibre speeds, and no amount of engineering reduces it. A synchronous write requiring acknowledgement from both adds that to every write, permanently. Designs that assume cross-region calls can be optimised into insignificance are budgeting for a latency that cannot be achieved.

**How it works**

**Topologies and what each costs**

```text
SINGLE WRITE REGION  (primary-secondary)
  writes -> one region, replicated out async
  reads  -> local everywhere
  + no conflicts, ever
  + simple mental model
  - distant users see slow writes (250 ms+)
  - primary region failure means no writes until
    failover
  -> the right default for most systems

MULTI-REGION WRITES, ASYNCHRONOUS
  every region accepts writes locally
  + fast writes everywhere
  + survives any region failure
  - CONCURRENT CONFLICTING WRITES are now possible
  - needs CRDTs, version vectors, or an
    application merge rule

SYNCHRONOUS ACROSS REGIONS
  write waits for a quorum spanning regions
  + strongly consistent globally
  - every write pays inter-region latency
  - a region partition blocks writes
  -> reserve for data where consistency is worth
     hundreds of milliseconds

PARTITIONED BY GEOGRAPHY
  European users' data lives in Europe
  each region is authoritative for its own users
  + local reads AND local writes
  + satisfies data residency rules
  - cross-region access is slow but rare
  -> often the best answer available
```

1. **Start with one write region and local read replicas** — It removes conflicts entirely and solves the read latency, which is usually most of the problem.
2. **Partition by user geography where the data allows** — Local reads and local writes, with cross-region access as the rare case.
3. **Reserve synchronous cross-region writes for data that requires it** — The latency is permanent; pay it only where consistency is worth it.
4. **Design conflict resolution before enabling multi-region writes** — Conflicts are certain, and deciding afterwards means data has already been lost.
5. **Measure replication lag continuously** — It determines how much data a region failure loses.
6. **Practise regional failover** — An untested failover path fails at the moment it is needed.

**Failover, lag and what is lost**

```text
ASYNC REPLICATION LAG
  typical 50-500 ms, seconds under load
  region fails -> everything not yet replicated is
  LOST
  -> RPO (recovery point objective) = the lag
  -> "we lose up to 2 seconds of writes" is a
     product decision, not just a technical one

SYNC REPLICATION
  RPO = 0, nothing is lost
  cost: every write pays cross-region latency
  -> and a partition means writes stop

FAILOVER MECHANICS
  detect        30-60 s (and false positives are
                catastrophic here)
  promote       seconds
  redirect DNS  60-300 s depending on TTL
  -> realistic RTO is minutes, not seconds

THE SPLIT-BRAIN RISK
  network partition between regions
  both promote themselves to primary
  both accept writes
  -> divergent histories that must be merged by hand
  -> prevention: promotion requires a quorum of
     regions, or a witness in a third region

DATA RESIDENCY
  some data legally may not leave a jurisdiction
  -> geographic partitioning is not an optimisation
     but a requirement
```

| Metric | Value | Note |
|---|---|---|
| Cross-region RTT | ~250 ms | **unavoidable** |
| Async RPO | lag, seconds | data lost on failure |
| Sync RPO | zero | latency on every write |
| Realistic RTO | minutes | DNS dominates |

> **Two regions both promoting themselves is the worst outcome available**  
> A partition between regions leaves each side unable to tell whether the other is down or merely unreachable, and a failover policy that promotes on local judgement will promote both. Both then accept writes to the same logical dataset, producing two divergent histories that must eventually be merged manually with real data loss. Promotion must require agreement — a quorum of regions, or an independent witness — rather than being a decision either region can take alone.

**Technologies**

| Approach | Examples | Note |
|---|---|---|
| Async cross-region replicas | Most managed databases | Simple; non-zero data loss on failover |
| Globally distributed SQL | Spanner-style systems | Strong consistency, at the cost of write latency |
| Multi-region active-active stores | DynamoDB global tables, Cassandra | Fast local writes; conflict resolution required |
| CRDTs | Collaborative and offline-first systems | Conflicts merge automatically by construction |
| CDN and edge caching | Read-only content | Removes most of the latency problem where applicable |
| Geographic partitioning | Per-region authoritative data | Best latency and residency compliance |

Edge caching is the cheapest large win and the most overlooked in this conversation: a substantial fraction of what makes distant users wait is static or cacheable, and serving it from an edge location removes that latency without touching the database topology at all.

**Trade-offs**

**Geographic strategies**

| Strategy | Read latency | Write latency | Conflicts | Data loss on failure |
|---|---|---|---|---|
| Single region | Poor for distant users | Poor for distant users | None | Total outage |
| Single write region, read replicas | Good | Poor for distant users | None | Replication lag |
| Multi-region async writes | Good | Good | Certain | Replication lag |
| Multi-region sync writes | Good | Poor everywhere | None | None |
| Geographic partitioning | Good | Good | Rare | Per-region |

Geographic partitioning is the strongest row and is available more often than people assume: most user data is accessed overwhelmingly by users in one region, so making each region authoritative for its own users gives local reads and writes with conflicts only in the rare cross-region case.

> **Ask before choosing it**  
> How much data may be lost if a region disappears right now? That single number chooses between asynchronous and synchronous replication, and it is a product decision rather than a technical one — which means it should be asked of the business, not assumed by the engineer.

**How it fails**

**How geographic replication goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Split brain across regions | Local promotion decisions | Require quorum or a third-region witness |
| Writes lost on failover | Asynchronous replication lag | Accept a defined RPO, or replicate synchronously |
| Failover takes far too long | DNS TTL and manual steps | Short TTLs, automated and rehearsed failover |
| Conflicting writes silently resolved | Last-write-wins across regions | Version vectors or CRDTs |
| Every write pays ocean latency | Synchronous replication applied to everything | Reserve it for data that needs it |
| Compliance violation | Data replicated outside its jurisdiction | Geographic partitioning with enforced boundaries |
| Failover never worked | Path untested | Regular drills, including full region loss |

> **Asynchronous replication means a defined amount of data is lost when a region fails**  
> The replication lag at the moment of failure is exactly the window of writes that no longer exist anywhere. Users received success responses for them. Whether that is acceptable — two seconds of orders, a minute of profile edits — is a business decision that must be made explicitly, because the alternative costs cross-region latency on every write, and choosing by default means choosing without anyone having agreed to the loss.

**Where it is used**

- **Global consumer applications**, replicating read paths to every region while writes go home.
- **Financial systems with residency rules**, where geographic partitioning is a legal requirement rather than a performance choice.
- **Globally distributed SQL deployments**, paying write latency in exchange for consistent global reads.
- **Collaborative editors**, using CRDTs so concurrent regional writes merge without a coordinator.
- **Content delivery networks**, which solve the read half of the problem for everything cacheable.

**In the interview**

Anchor the discussion in the latency number and in the data-loss question; both make the trade concrete.

- **State the physical cost** — roughly 250 milliseconds across the world — and note it cannot be engineered away.
- **Ask how much data may be lost on a region failure**, because that chooses between async and sync replication.
- **Propose geographic partitioning** where access is regional, since it gives local reads and writes with rare conflicts.
- **Raise split brain unprompted** and require quorum-based promotion rather than a local decision.
- **Give realistic failover numbers.** Detection plus promotion plus DNS means minutes, and saying seconds signals inexperience.

**Practice drill**

Users are in North America, Europe and Asia. Design the replication topology, naming where writes are accepted and how reads are served in each region. State the recovery point and recovery time objectives your design achieves, with the arithmetic behind them. Then handle a partition between Europe and North America: say what each region does, what a user in each sees, and what mechanism prevents both from promoting themselves.

**Go deeper**

Geographic replication places copies of data in several regions so that reads are local and a regional failure is survivable.

**The read half is straightforward and the write half is the whole design.** Local replicas remove hundreds of milliseconds from every read for distant users, at no cost to consistency if writes still go to one place. Everything difficult follows from deciding where writes are accepted: one region means distant users wait, several regions mean concurrent conflicting writes, and requiring all regions to agree means every write pays the distance.

**Inter-region latency is a physical constant and budgets must respect it.** A round trip across the world is roughly a quarter of a second at realistic speeds, so a synchronous cross-region write cannot be fast, ever. Designs that treat this as an optimisation problem waste effort; designs that accept it choose deliberately where to spend it, and usually conclude that only a small subset of data justifies synchronous global agreement.

**Asynchronous replication converts a region failure into a defined quantity of lost data.** Whatever had not yet replicated when the region disappeared is gone, despite users having received success responses for it. The size of that window is the replication lag, and whether it is acceptable is a product question — losing two seconds of orders is a business decision, not a technical detail — which means it should be stated explicitly rather than inherited from a default configuration.

**Geographic partitioning is frequently the best answer and is underused.** Most user data is accessed almost entirely from one region, so making each region authoritative for its own users delivers local reads and local writes with conflicts confined to genuinely cross-region access. It also satisfies data residency requirements structurally rather than through policy, which matters wherever data legally may not leave a jurisdiction.

**Split brain across regions is the most damaging failure in the space.** A partition leaves each side unable to distinguish the other's failure from its own isolation, and a promotion policy decided locally will promote both. Two regions then accept writes to the same dataset and produce divergent histories that must be reconciled by hand with real loss. Requiring a quorum of regions, or an arbitrating witness in a third location, is what makes promotion a decision no single region can take alone.

**Failover is a practised operation or it is a theoretical one.** Detection, promotion, and DNS propagation compose into minutes rather than seconds, and every untested step is a place where the real event diverges from the plan. Regular drills that actually remove a region are what turn a documented recovery objective into an achievable one — and they routinely reveal that the recovery time everyone assumed was several times optimistic.

**Related patterns:** Quorum Replication · Read Repair · Anti Entropy · CDN

---

### Read Repair

*Detect stale replicas during ordinary reads and update them, so the act of reading gradually repairs divergence at no extra cost.*

> **When you hear…** Replicas drift after failures or dropped writes · eventual consistency that must actually converge · repair without a separate process

**Flow:** `Read several replicas` → `Compare versions` → `Return newest` → `Push to stale ones` → `Divergence shrinks`

**The problem**

A write reaches two of three replicas because the third was briefly unavailable. The quorum succeeded and the client was told the write was durable, but one replica now holds an older value indefinitely — nothing in the write path will ever revisit it.

Divergence accumulates. Every dropped message, brief partition and restart leaves another replica slightly behind, and without a mechanism to converge, the probability that a read encounters stale data grows over time.

> **Reads already fetch several copies, so they already know who is wrong**  
> A quorum read contacts multiple replicas and compares their versions to decide what to return. That comparison identifies exactly which replicas are behind, at no additional cost — the information is a by-product of work the read was doing anyway. Sending the newest value back to the stale ones turns every read into an opportunity for repair.

**Mental model**

Repair as a side effect of reading. The read collects versions, determines the newest, serves it, and pushes it to whichever replicas were behind.

1. **Fetch** — Read from several replicas as the consistency level requires.
2. **Compare** — Use version information to find the newest value.
3. **Serve** — Return the newest to the client immediately.
4. **Repair** — Send it to the replicas that had older versions.
5. **Converge** — Frequently read keys stay consistent without any background work.

> **It only repairs what is read, which is the opposite of what needs repairing**  
> Popular keys are read constantly and repair themselves within seconds. Rarely read keys — old records, cold data, the long tail — can stay divergent indefinitely, and those are precisely the ones where a replica failure is most likely to lose the only current copy. Read repair is a necessary mechanism and never a sufficient one.

**How it works**

**Blocking and background repair**

```text
BLOCKING READ REPAIR
  read R replicas
  versions differ
  -> push the newest to the stale ones
  -> WAIT for acknowledgement
  -> then return to the client
  + guarantees monotonic reads afterwards
  - adds latency to the read that found the problem

BACKGROUND READ REPAIR
  return to the client immediately
  repair asynchronously
  + no added latency
  - a concurrent read may still see the stale value
  -> the common default

PROBABILISTIC READ REPAIR
  even when the consistency level does not require
  reading all replicas, read the extras on a small
  percentage of requests (e.g. 10%)
  -> catches divergence on replicas the read would
     not otherwise have consulted
  -> cost is proportional to the sampling rate

WHAT IT NEEDS
  versions that can be compared reliably
    logical versions, or version vectors
  NOT wall-clock timestamps across skewed machines
  -> otherwise repair can push an OLDER value over a
     newer one, actively corrupting data
```

1. **Repair in the background by default** — Blocking repair adds latency to exactly the reads that were already slow.
2. **Use blocking repair where monotonic reads matter** — It guarantees a subsequent read will not see the older value again.
3. **Sample extra replicas occasionally** — Divergence on replicas the read never consults is otherwise invisible.
4. **Compare logical versions, not wall clocks** — Skew can make repair overwrite a newer value with an older one.
5. **Pair with anti-entropy for cold data** — Unread keys are never repaired by reads, by definition.
6. **Monitor how often repairs occur** — A rising repair rate is an early signal of replication trouble.

**What it covers and what it misses**

```text
COVERAGE BY ACCESS FREQUENCY
  read 1000x/day  -> repaired within seconds of any
                     divergence
  read 1x/week    -> divergent for up to a week
  read never      -> divergent forever

-> the keys most likely to be lost are the ones
   least likely to be repaired

WHY THAT MATTERS
  replica A holds the only current copy of a cold key
  replica A dies
  -> the value silently reverts to the older version
     on B and C
  -> nobody notices, because nobody reads it
  -> until someone does

THE PAIRING
  read repair      fast, free, covers hot data
  anti-entropy     slow, costly, covers everything
  hinted handoff   covers writes missed during a
                   known outage
  -> all three together give convergence with
     reasonable cost; any one alone leaves a gap

COST OF PROBABILISTIC REPAIR
  10% sampling on a read-heavy system
  -> 10% more replica reads
  -> catches cold-ish keys that get some traffic
  -> still misses truly cold data
```

| Metric | Value | Note |
|---|---|---|
| Cost | near zero | reads already fetch |
| Coverage | read keys only | **cold data missed** |
| Default mode | background | no added latency |
| Requires | comparable versions | not wall clocks |

> **Repairing on wall-clock timestamps can overwrite newer data with older**  
> If versions are ordered by the clock of whichever node wrote them, a node running fast can stamp an older value with a later time, and read repair will then dutifully propagate that older value over the newer one on every other replica. The mechanism designed to converge on the correct value converges on the wrong one, and it does so permanently and silently.

**Technologies**

| System | Behaviour | Note |
|---|---|---|
| Cassandra | Blocking and background read repair | Repair chance is configurable per table |
| Dynamo-style stores | Read repair as described | Introduced alongside anti-entropy in the original design |
| Riak | Read repair on divergence | Version vectors make comparison safe |
| Object stores | Repair on access | Often paired with background scrubbing |
| Anti-entropy | Background comparison | Covers what reads never touch |
| Hinted handoff | Replay of writes missed during an outage | Covers a known window rather than arbitrary drift |

Hinted handoff is the third member of this family and is worth distinguishing: when a replica is known to be down, the coordinator stores the writes it missed and replays them on recovery. That handles planned and detected outages, while read repair and anti-entropy handle drift nobody noticed.

**Trade-offs**

**Convergence mechanisms**

| Mechanism | Cost | Coverage | Latency impact |
|---|---|---|---|
| Background read repair | Negligible | Read keys only | None |
| Blocking read repair | Low | Read keys only | Adds to the read |
| Probabilistic repair | Proportional to sampling | Slightly wider | Small |
| Hinted handoff | Storage for hints | Writes during known outages | None |
| Anti-entropy | High; scans and compares | Everything | None, runs separately |
| No repair | None | None | None |

The final row is not a straw man: systems where all data is rewritten frequently, or where replicas are rebuilt wholesale rather than repaired, genuinely do not need these mechanisms. Knowing why you do not need one is as useful as knowing how it works.

> **Ask before choosing it**  
> What fraction of the keyspace is read regularly? If most data is cold, read repair covers almost nothing and the real convergence work has to be done by anti-entropy — which is a much more expensive commitment and should be planned rather than discovered.

**How it fails**

**How read repair goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Cold data stays divergent | Repair only triggers on reads | Run anti-entropy for full coverage |
| Newer values overwritten | Ordering by skewed wall clocks | Logical versions or version vectors |
| Read latency degraded | Blocking repair on every divergence | Repair in the background by default |
| Divergence found too late | Never reading extra replicas | Probabilistic sampling of additional replicas |
| Repair traffic overwhelms replicas | Large-scale divergence repaired all at once | Throttle repair writes |
| Divergence growing unnoticed | Repair rate not monitored | Alert on the rate of repairs performed |
| Deleted data resurrected | Repairing over a delete with no tombstone | Tombstones with a defined lifetime |

> **Repairing over a deletion resurrects data unless deletes leave a marker**  
> If a delete simply removes the value on two replicas and the third still holds it, a read sees a present value and an absent one — and without a tombstone recording that the deletion is the newer event, repair concludes the value is the newer state and restores it everywhere. Deleted records reappear, which in systems handling personal data is a compliance failure rather than an inconvenience.

**Where it is used**

- **Dynamo-style databases**, where read repair is a core part of the eventual consistency story.
- **Cassandra**, exposing repair chance per table so hot tables converge without background work.
- **Distributed object stores**, repairing on access and scrubbing in the background for the rest.
- **Replicated caches**, where refreshing stale replicas on read is effectively the same mechanism.
- **Geo-replicated systems**, repairing regional divergence found during cross-region reads.

**In the interview**

The insight worth stating is that repair is free because the read already had the information.

- **Say the comparison is a by-product** of a quorum read, which is why the mechanism costs almost nothing.
- **Name the coverage gap immediately**: only read keys are repaired, so cold data diverges indefinitely.
- **Distinguish blocking from background** and explain when monotonic reads justify the added latency.
- **Require comparable versions**, because wall-clock ordering lets repair propagate older values over newer ones.
- **Complete the family** with hinted handoff and anti-entropy, and say what each one covers that the others do not.

**Practice drill**

Three replicas hold a key; one missed the last write. Trace a quorum read: which replicas answer, how the newest value is identified, what the client receives, and what repair does. Then explain what happens to a key that is written once and never read again, and design the mechanism that covers it. Finally, describe what could go wrong if versions were ordered by wall-clock time and one node's clock ran fast.

**Go deeper**

Read repair converges replicas by detecting stale copies during ordinary reads and updating them with the newest value.

**Its efficiency comes from reusing work already being done.** A quorum read fetches several copies and compares their versions to decide what to return, which simultaneously identifies exactly which replicas are behind. The repair then costs only the write to those replicas — there is no scanning, no coordination and no background job, which is why it is enabled by default in systems that use it.

**Its coverage is inversely related to where the risk is.** Frequently read keys repair themselves within seconds of any divergence, while cold keys are never read and therefore never repaired. That is precisely backwards from a durability perspective: the value held correctly on only one replica is most likely to be lost, and the less it is read the longer that vulnerable state persists — which is why read repair is always paired with something that scans.

**Blocking and background modes answer different questions.** Background repair returns to the client immediately and fixes the replicas afterwards, so a concurrent read can still see the old value; blocking repair waits for the stale replicas to acknowledge before responding, guaranteeing that a subsequent read cannot go backwards. The first is the right default because it adds no latency; the second is worth its cost only where monotonic reads matter to the application.

**It depends entirely on versions being comparable, and clocks are not.** Deciding which value is newer by the wall-clock time of the writing node means a node with a fast clock can label an older value as later, and read repair will then propagate that older value across every replica. The mechanism built to converge on the truth converges on the error instead, permanently and without any signal — which is why logical versions or version vectors are a prerequisite rather than a refinement.

**Deletions need explicit markers or repair undoes them.** A value absent on two replicas and present on a third looks like divergence in which the present value is the data, so without a tombstone recording that the deletion happened later, repair restores the record everywhere. Deleted data reappearing is an operational surprise in general and a compliance failure where personal data is concerned, and tombstone lifetime becomes a real constraint — they must outlive any replica's possible staleness.

**It is one of three complementary mechanisms, and the set matters more than any member.** Hinted handoff replays writes missed during a detected outage, read repair fixes drift on data people actually read, and anti-entropy scans everything on a slow cycle to catch what the other two miss. Each is cheap where the others are expensive, and a system running only one of them has a predictable gap — usually the cold, rarely-read data that nobody discovers is wrong until they need it.

**Related patterns:** Anti Entropy · Quorum Replication · Geographic Replication · Cache Invalidation

---

### Anti Entropy

*Periodically compare replicas in the background and reconcile differences, so divergence is bounded even for data nobody reads.*

> **When you hear…** Cold data diverging unnoticed · replicas rebuilt or restored from backup · a guarantee that divergence is bounded

**Flow:** `Build digests` → `Exchange summaries` → `Find differing ranges` → `Drill down` → `Transfer only diffs`

**The problem**

Read repair converges data that is read, which leaves the rarely accessed majority of most datasets drifting indefinitely. A replica that missed writes during an hour-long outage stays wrong until something happens to read each affected key.

Comparing replicas naively is prohibitive. A billion keys per replica cannot be enumerated and compared over the network on any useful schedule — the comparison itself would cost more than the data it protects.

> **Compare summaries, not data, and only descend where they differ**  
> A hash covering a range of keys is tiny, and two replicas whose hashes match over a range are provably identical there and need no further work. Arranging those hashes in a tree lets a comparison start with a single value covering everything and descend only into subtrees that disagree — so the cost scales with the amount of divergence rather than with the size of the dataset.

**Mental model**

A tree of hashes over the keyspace, exchanged between replicas. Matching nodes prune entire subtrees; mismatches are followed downward until the differing keys are identified.

1. **Build** — Each replica computes hashes over ranges of its data, combined upward into a tree.
2. **Exchange roots** — If the top hashes match, the replicas are identical and the comparison is over.
3. **Descend** — Where hashes differ, compare the children of that node only.
4. **Locate** — Continue until the specific differing keys are found.
5. **Reconcile** — Transfer only those keys, resolving versions as they are applied.

> **Anti-entropy is expensive and will compete with production traffic**  
> Building the tree reads the data, comparison generates network traffic, and reconciliation writes. Run unthrottled it can saturate disks and links, and because it typically runs on a schedule it tends to start at an inconvenient moment. Every production deployment needs it throttled and scheduled, and the throttle needs to be low enough that nobody notices it is running.

**How it works**

**Merkle trees and the cost of comparison**

```text
NAIVE COMPARISON
  1,000,000,000 keys x 16 B of hash = 16 GB
  transferred per replica pair, per run
  -> impossible on any useful schedule

MERKLE TREE
               root
           /          \
        h(A)          h(B)
       /    \        /    \
    h(A1)  h(A2)  h(B1)  h(B2)
     ...    ...    ...    ...
  leaves cover ranges of keys

COMPARISON
  exchange roots
    match    -> identical, DONE, one hash exchanged
    differ   -> exchange children of the root
  recurse only into differing subtrees

COST
  depth 20 tree over 1e9 keys
  one differing key
  -> ~20 hash comparisons to locate it
  -> a few kilobytes instead of 16 GB

  1000 differing keys, scattered
  -> more subtrees differ, cost rises
  -> still proportional to DIVERGENCE, not data size

RESOLUTION AT THE LEAVES
  transfer the differing keys with their versions
  apply the newer version, as decided by version
  vectors or logical clocks
  -> same version comparison problem as read repair
```

1. **Use a hash tree so cost scales with divergence** — Full comparison is impossible at any real data size.
2. **Throttle reconciliation aggressively** — Unthrottled repair competes directly with production traffic.
3. **Schedule runs for quiet periods** — The work is not urgent; it only needs to complete within the divergence window.
4. **Keep trees incrementally updated where possible** — Rebuilding from scratch every run reads the entire dataset.
5. **Resolve with versions, not timestamps** — The same clock-skew hazard as read repair, with wider reach.
6. **Run it after any known outage or restore** — A replica rebuilt from backup is divergent by definition.

**Scheduling and what it guarantees**

```text
THE GUARANTEE
  run every T -> divergence is bounded by T
  run weekly  -> a replica can be wrong for a week
  run daily   -> wrong for at most a day
  -> choose T from how long you can tolerate an
     undetected inconsistency

TOMBSTONE INTERACTION  (the sharp edge)
  deletes leave tombstones that expire after G
  if anti-entropy runs LESS often than G:
    replica missed the delete
    tombstone expires everywhere else
    anti-entropy compares: value present vs absent
    -> the value is treated as newer
    -> DELETED DATA IS RESURRECTED
  -> the iron rule: repair interval < tombstone
     lifetime, always

WHEN TO RUN IT IMMEDIATELY
  a replica returns after a long outage
  a node is rebuilt or restored from backup
  after any incident involving dropped writes
  -> scheduled runs handle drift; these are known
     divergences and should not wait

COST CONTROL
  compare a subset of ranges per run
  -> full coverage over several runs
  -> bounded cost per run, longer T
```

| Metric | Value | Note |
|---|---|---|
| Comparison cost | log-scale | **by divergence** |
| Guarantee | bounded by interval | choose deliberately |
| Iron rule | interval < tombstone TTL | or deletes revert |
| Must be | throttled | competes with traffic |

> **Repair intervals longer than tombstone lifetimes resurrect deleted data**  
> A replica that missed a deletion holds the old value. Once the tombstones recording that deletion have expired on the other replicas, a comparison sees a value on one side and nothing on the other, with no evidence that the absence is the newer state — so the value is restored everywhere. This is a well-documented operational hazard, it is entirely preventable by keeping the repair interval below the tombstone lifetime, and it is still one of the more common ways deleted records come back.

**Technologies**

| System | Mechanism | Note |
|---|---|---|
| Cassandra repair | Merkle tree comparison between replicas | Must complete within the tombstone window |
| Dynamo-style stores | Merkle trees as originally described | Paired with read repair and hinted handoff |
| Object stores | Background scrubbing and verification | Also detects bit rot, not only divergence |
| Distributed filesystems | Periodic checksum verification | Repairs from parity or other replicas |
| Read repair | Convergence on access | Covers hot data cheaply |
| Full rebuild | Copy a replica wholesale | Simpler when divergence is extensive |

Full rebuild is the pragmatic alternative when a replica has been out for a long time: comparing two datasets that differ substantially costs more than streaming a fresh copy, and most operators have a threshold beyond which they replace rather than repair.

**Trade-offs**

**Convergence strategies compared**

| Strategy | Covers cold data | Cost | Latency to converge |
|---|---|---|---|
| Read repair only | No | Negligible | Immediate for hot keys, never for cold |
| Hinted handoff | Only known outages | Low | On replica return |
| Anti-entropy on a schedule | Yes | High per run | Up to the interval |
| Continuous background repair | Yes | Sustained | Shorter |
| Full replica rebuild | Yes | Very high | Immediate once complete |
| Nothing | No | None | Never |

Continuous low-rate repair is increasingly preferred over periodic large runs: the same work spread evenly avoids the operational spike, keeps the divergence window shorter, and removes the temptation to skip a scheduled run because the cluster is busy.

> **Ask before choosing it**  
> How long may a replica hold wrong data before it matters? That interval sets the schedule, and it must also sit below the tombstone lifetime — if the two constraints conflict, the tombstone lifetime has to grow, because the alternative is deleted data returning.

**How it fails**

**How anti-entropy goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Deleted data returns | Repair interval exceeds tombstone lifetime | Repair more often, or extend tombstone retention |
| Production latency spikes during repair | Unthrottled comparison and transfer | Throttle; schedule for quiet periods |
| Repair never completes | Dataset too large for the window | Subset ranges per run; continuous repair |
| Divergence persists despite repair | Version resolution choosing wrongly | Version vectors rather than wall clocks |
| Repair costs more than a rebuild | Extensive divergence after a long outage | Replace the replica instead |
| Trees rebuilt from scratch each run | No incremental maintenance | Update tree nodes as data changes |
| Nobody notices it stopped running | No monitoring of completion | Alert on time since last successful run |

> **An anti-entropy process that quietly stopped is invisible until data is lost**  
> Nothing fails when repair stops running: reads succeed, writes succeed, and divergence simply grows unbounded in the background. The first symptom is typically a replica failure that exposes how far the remaining copies had drifted, or deleted records reappearing once tombstones expire. Time since the last successful full pass is the metric that has to be alerted on, because no other signal exists.

**Where it is used**

- **Cassandra clusters**, where scheduled repair is a documented operational requirement rather than an option.
- **Dynamo-style stores**, using Merkle trees exactly as the original design described.
- **Object storage systems**, scrubbing continuously to detect both divergence and silent data corruption.
- **Distributed filesystems**, verifying checksums and repairing from replicas or parity.
- **Backup verification**, comparing restored data against the source using the same tree technique.

**In the interview**

Show the comparison arithmetic and then raise the tombstone interaction; the second is what marks operational experience.

- **Explain why naive comparison is impossible** with a data-size figure, then introduce hash trees as the fix.
- **Say the cost scales with divergence, not dataset size**, which is the property that makes the technique viable.
- **State the guarantee as a bound**: running every interval means divergence lasts at most that long.
- **Raise the tombstone rule unprompted** — repair must run more often than tombstones expire, or deletes revert.
- **Insist on throttling and monitoring**, since repair competes with production traffic and fails silently when it stops.

**Practice drill**

Three replicas hold a billion keys each, and one missed an hour of writes during an outage. Explain why a direct comparison is infeasible, then design the hash-tree comparison and estimate the traffic needed to locate a thousand divergent keys. Choose a repair interval, state the guarantee it provides, and verify it against a tombstone lifetime you also choose. Finally, say how you would know the process had silently stopped running.

**Go deeper**

Anti-entropy compares replicas in the background and reconciles differences, providing convergence for data that reads never touch.

**It exists because read repair has a structural gap.** Convergence driven by access covers whatever is read and nothing else, so the cold majority of most datasets drifts indefinitely — and cold data held correctly on a single replica is exactly what is lost when that replica fails. A mechanism that scans independently of access is the only way to bound divergence across the whole keyspace.

**Hash trees are what make the comparison affordable.** Enumerating and exchanging every key is impossible at real data sizes, but a single hash covering a range proves equality over that range, so matching subtrees are pruned entirely. The cost then scales with how much divergence exists rather than with how much data exists, which turns an impossible comparison into one that usually finishes having transferred a few kilobytes.

**Running it on a schedule converts divergence from unbounded to bounded.** The interval is the guarantee: run daily and a replica can be wrong for at most a day. Choosing that number is a durability decision — how long an undetected inconsistency is tolerable — and it should be made explicitly, because the default of not running it at all provides no bound whatsoever.

**The tombstone interaction is the sharpest operational edge in replicated stores.** Deletions are recorded as markers that eventually expire, and a replica that missed a deletion still holds the value. If repair does not run before those markers expire elsewhere, the comparison sees a value on one side and an absence on the other with no evidence about which is newer, and restores the deleted record everywhere. Keeping the repair interval below the tombstone lifetime is an absolute constraint, not a tuning preference.

**It is expensive enough to need deliberate operational handling.** Building trees reads data, comparison uses the network, and reconciliation writes — all competing with production traffic on the same disks and links. Throttling is mandatory, and spreading the work continuously at a low rate is increasingly preferred over periodic large runs, because it shortens the divergence window and removes the temptation to skip a run during a busy period.

**Its failure mode is silence, which makes monitoring part of the design.** When repair stops running nothing breaks: reads and writes continue normally while divergence grows in the background, and the problem surfaces only when a replica is lost or deleted data reappears. Time since the last successful complete pass is the metric that reveals this, and a system that runs anti-entropy without alerting on that metric has a durability guarantee it cannot actually verify.

**Related patterns:** Read Repair · Quorum Replication · Gossip Membership · Geographic Replication

---

## Feeds

### Fanout On Write

*Push each new item into every follower's precomputed feed at write time, so reading a feed is a single cheap lookup.*

> **When you hear…** Feed reads vastly outnumber writes · feed latency must be minimal · followers per author are bounded

**Flow:** `User posts` → `Look up followers` → `Write to each inbox` → `Feed is prebuilt` → `Read is one query`

**The problem**

A feed assembled at read time must query every account the viewer follows, merge the results by time, and rank them. For someone following two thousand accounts that is two thousand lookups per feed open, and feeds are opened constantly.

The read path is the hot path. Most social products see reads outnumber writes by two or three orders of magnitude, so any work that can be moved off the read is worth a great deal of work on the write.

> **Do the work once per post rather than once per view**  
> A post is written once and read by all of its author's followers, many of them repeatedly. Computing each follower's feed at write time — inserting the post into a per-user list — means every subsequent read is a single sequential lookup of an already-assembled list. The total work is lower because the expensive assembly happens once instead of once per view.

**Mental model**

A materialised inbox per user. Writing a post appends its identifier to every follower's inbox; reading a feed is a range scan of one inbox.

1. **Post** — The author writes the item to a store of posts.
2. **Resolve** — The system fetches the author's follower list.
3. **Fan out** — The post identifier is appended to each follower's feed list.
4. **Read** — Opening a feed reads one list, already ordered.
5. **Hydrate** — Post identifiers are turned into full content, usually from a cache.

> **The write cost is proportional to follower count, without limit**  
> A post from an account with fifty million followers requires fifty million inserts. At any realistic write rate that is not a background task but a sustained load spike, and it arrives exactly when the account is most active. Pure fanout-on-write does not scale to accounts of that size, which is why no large platform uses it alone.

**How it works**

**The write amplification**

```text
TYPICAL USER, 200 followers
  1 post -> 200 inserts
  cheap, done in milliseconds

POPULAR USER, 1,000,000 followers
  1 post -> 1,000,000 inserts
  at 10,000 inserts/s -> 100 seconds
  -> the post appears in feeds over 100 seconds
  -> and the fanout job holds capacity the whole time

CELEBRITY, 50,000,000 followers
  1 post -> 50,000,000 inserts
  -> not a job, an incident

STORAGE
  average 500 followers, 1M posts/day
  -> 500,000,000 feed entries per day
  -> entries are small (post id + timestamp) but the
     volume is the cost, not the size

WHAT MAKES IT BEARABLE
  cap the stored feed length (e.g. 800 entries)
    older entries are dropped; deep scrolling falls
    back to a read-time query
  write asynchronously through a queue
    the post returns immediately; fanout catches up
  only fan out to ACTIVE followers
    an account dormant for a year does not need a
    prebuilt feed
```

1. **Fan out asynchronously through a queue** — The author's write must not wait for millions of inserts.
2. **Cap each stored feed at a few hundred entries** — Unbounded feeds cost storage for pages nobody scrolls to.
3. **Fan out only to active followers** — Dormant accounts are usually most of the follower list and none of the readers.
4. **Store identifiers, not content** — Copying full posts multiplies storage and makes edits impossible to propagate.
5. **Handle the backlog explicitly** — A large fanout must be throttled so it does not starve ordinary posts.
6. **Do not use it for very large accounts** — Above a threshold the cost is unbounded; those accounts need the read-time path.

**Where the design breaks**

```text
NEW FOLLOW
  a user follows someone
  -> their existing posts are not in the follower's
     feed
  -> backfill (expensive) or accept the gap

UNFOLLOW / BLOCK / DELETE
  the post is already in millions of feeds
  -> remove from all of them (expensive), or
  -> filter at read time (cheap, but every read pays
     a check)

EDIT
  storing ids rather than content means the edit is
  picked up on hydration automatically
  -> this alone justifies storing ids

RANKING
  a chronological inbox is easy
  a ranked feed needs scores that change over time
  -> the prebuilt list becomes a candidate set, and
     ranking happens at read time anyway

THE HONEST POSITION
  fanout-on-write optimises retrieval, not ranking
  modern feeds are ranked, so the pattern supplies
  candidates rather than the finished feed
```

| Metric | Value | Note |
|---|---|---|
| Read cost | one lookup | **very fast** |
| Write cost | × followers | unbounded |
| Storage | entries per follower | capped list |
| Breaks at | large accounts | needs a hybrid |

> **Deletions and blocks are expensive once a post is in millions of inboxes**  
> Removing a post that has already been fanned out means touching every feed that contains it, which costs as much as the original fanout. Most systems therefore filter at read time instead — checking blocks and deletions during hydration — which makes every read slightly more expensive in exchange for making deletion cheap. That trade has to be made deliberately, because discovering it after launch means choosing between slow deletes and stale feeds.

**Technologies**

| Piece | Options | Note |
|---|---|---|
| Feed storage | Redis lists or sorted sets | Fast, capped naturally, memory-bound |
| Feed storage | Wide-column stores | Cheaper per entry at very large scale |
| Fanout execution | Work queue with many workers | Must be throttled for large accounts |
| Follower lists | Graph or relational store | Read on every post; heavily cached |
| Hydration | Post cache keyed by id | Where the actual content is assembled |
| Alternative | Fanout on read | For accounts with very many followers |

Redis sorted sets are the common choice because capping a feed is a single trim operation and reads are already ordered, but memory becomes the binding constraint at large user counts — which is where teams typically move the long tail of inactive users to a cheaper store.

**Trade-offs**

**Feed construction strategies**

| Strategy | Read cost | Write cost | Fits |
|---|---|---|---|
| Fanout on write | One lookup | Follower count | Most users, bounded followings |
| Fanout on read | Following count | One insert | Celebrities, rarely read feeds |
| Hybrid | One lookup plus a few | Bounded | Real systems |
| No feed, search on demand | Expensive | None | Low-traffic products |

The hybrid row is where every large platform ends up, and arriving at it directly in a design discussion — rather than defending pure fanout-on-write against the celebrity case — is what demonstrates that the trade-off is understood.

> **Ask before choosing it**  
> What is the follower distribution? A product where the maximum following is a few hundred can use pure fanout-on-write happily; one with a long tail of accounts followed by millions cannot, and the threshold at which you switch strategies is the actual design decision.

**How it fails**

**How fanout on write goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Celebrity post stalls the fanout queue | Millions of inserts as one job | Threshold to the read-time path; throttle large fanouts |
| Posts appear minutes late | Fanout backlog behind a large job | Priority queues; cap per-job work |
| Storage grows faster than users | Uncapped feed length | Trim to a few hundred entries |
| Feeds full of content from dormant accounts | Fanning out to every follower | Only fan out to active users |
| Deleting a post takes minutes | Removal from every inbox | Filter at read time instead |
| Edits not reflected | Content copied into feeds | Store identifiers and hydrate |
| New follows show an empty feed | No backfill on follow | Backfill a page, or merge at read time |

> **A single large account can monopolise the fanout infrastructure**  
> One post to fifty million followers is fifty million writes that will occupy the fanout workers for a long time, delaying every ordinary user's post behind it. Without per-job limits and separate handling for large accounts, the platform's feed latency becomes a function of what one celebrity happened to do — which is both an availability problem and an unpleasant surprise during exactly the moments the product is busiest.

**Where it is used**

- **Early Twitter**, the canonical description of fanout-on-write and of the celebrity problem that broke it.
- **Instagram-style feeds**, where prebuilt inboxes serve the common case and large accounts are special-cased.
- **Activity and notification feeds**, which have the same shape and usually smaller fanouts.
- **Messaging group delivery**, fanning a message into per-member inboxes.
- **Email-style inbox models**, the original materialised-per-recipient design.

**In the interview**

Do the arithmetic for a celebrity early; it is what forces the hybrid conclusion naturally.

- **Justify it with the read/write ratio** — reads outnumber writes by orders of magnitude, so moving work to writes is profitable.
- **Compute the celebrity case out loud**: fifty million followers means fifty million inserts per post.
- **Cap the stored feed** and say what happens when a user scrolls beyond it.
- **Store identifiers, not content**, so edits propagate and storage stays small.
- **Raise deletion and blocking** as read-time filters, since removing a post from millions of inboxes is as expensive as writing it.

**Practice drill**

Design a feed for a platform where the median user has 200 followers and the largest account has 50 million. Compute the write amplification for both. Choose a feed length cap and justify it from scrolling behaviour. Then handle three operations: a user following someone new, a post being deleted, and a post being edited — saying for each whether you handle it at write time or read time and why.

**Go deeper**

Fanout on write precomputes each user's feed by inserting every new post into the inbox of every follower, so that reading a feed is a single lookup.

**It is an application of doing expensive work where it happens least often.** Feeds are read far more than they are written, so assembling a feed once per post rather than once per view reduces total work substantially, and it moves the remaining cost off the latency-critical path. This is the same reasoning behind any materialised view, applied to a workload where the read-to-write ratio is extreme.

**Its cost is proportional to follower count and therefore unbounded.** A typical user's post costs a few hundred inserts; a large account's post costs millions, arriving as a sustained burst that occupies the fanout infrastructure and delays everyone else's posts behind it. No tuning removes this, because the work is inherent to the approach, which is why the pattern must be bounded by a threshold rather than optimised.

**Storing identifiers rather than content is what keeps it viable.** Copying post bodies into millions of inboxes multiplies storage and makes editing impossible to propagate, whereas storing an identifier keeps entries tiny and lets hydration fetch the current version at read time. It also means a deleted post can be filtered during hydration instead of being chased through every inbox that contains it.

**Mutations after fanout are the awkward part of the design.** A new follow leaves the follower's feed missing history, deletions and blocks require either expensive removal or read-time filtering, and each of these decisions shifts cost between the two paths. The usual resolution is to filter at read time and accept a slightly more expensive hydration, because it keeps the write path bounded — but it is a choice that should be made explicitly rather than discovered when the first deletion takes minutes.

**Capping the stored feed matters more than it appears.** Users scroll a page or two, so storing thousands of entries per user buys almost no experience and multiplies storage across the entire user base. Trimming to a few hundred, with deeper scrolling falling back to a read-time query, keeps the storage bill proportional to what people actually read rather than to everything that was ever posted.

**Modern ranked feeds change what the pattern delivers.** A chronological inbox is a finished feed, but a ranked feed requires scores that depend on time, engagement and viewer context, none of which is known at write time. The prebuilt list therefore becomes a candidate set that ranking consumes at read time, which is still valuable — retrieval was the expensive part — but it means the pattern supplies input to the feed rather than the feed itself, and describing it as the latter overstates what it does.

**Related patterns:** Fanout On Read · Hybrid Fanout · Materialized View · Hot Key Mitigation

---

### Fanout On Read

*Store each post once and assemble a feed by querying the accounts a viewer follows at read time, keeping writes trivial at the cost of expensive reads.*

> **When you hear…** Authors with enormous follower counts · feeds read rarely relative to posting · write amplification unacceptable

**Flow:** `Post stored once` → `User opens feed` → `Query each followed` → `Merge by time` → `Return page`

**The problem**

Precomputing feeds means a post from an account with fifty million followers costs fifty million writes. The work is enormous, most of it lands in feeds nobody opens, and it arrives as a burst that delays everyone else's posts.

The wasted proportion is the striking part. Of those fifty million inboxes, perhaps a few million belong to people who will open the app today — the rest of the work is discarded without ever being read.

> **Do the work only for feeds someone actually opens**  
> A post read by nobody should cost nothing beyond storing it once. Assembling the feed when a viewer asks for it means the cost is paid exactly in proportion to consumption rather than to follower counts, which makes very large accounts free to post and makes inactive followers cost nothing at all.

**Mental model**

Posts stored once per author. A feed request queries the recent posts of every followed account, merges them and returns a page.

1. **Write** — The post is stored once against its author.
2. **Request** — A viewer opens their feed.
3. **Gather** — Recent posts are fetched from each followed account.
4. **Merge** — Results are combined and ordered.
5. **Return** — One page is served; the rest is discarded.

> **Read cost is proportional to how many accounts the viewer follows**  
> Someone following two thousand accounts triggers two thousand queries per feed open, and the response cannot be returned until the slowest of them completes. Because feeds are opened constantly, this cost is paid over and over for the same data — the exact inverse of the fanout-on-write problem, and equally unbounded.

**How it works**

**The read cost and how it is contained**

```text
NAIVE
  viewer follows 500 accounts
  -> 500 queries per feed open
  -> latency = the SLOWEST of 500
  -> at 10,000 feed opens/s that is 5,000,000
     queries/s

CONTAINMENT

1  PER-AUTHOR RECENT-POST CACHE
     each author's last ~50 posts cached
     -> the 500 lookups become cache hits
     -> this is the single most important measure

2  BATCH THE QUERIES
     one multi-get instead of 500 round trips
     -> latency becomes one slow shard, not 500

3  LIMIT WHAT IS FETCHED
     only posts newer than the viewer's last visit
     -> most followed accounts contribute nothing

4  CACHE THE ASSEMBLED FEED BRIEFLY
     30-60 s TTL on the merged result
     -> repeated opens and pagination are free
     -> this quietly converts the pattern into
        something close to fanout-on-write for
        active users

MERGE
  each source is already time-ordered
  -> k-way merge, stop once a page is filled
  -> do not fetch everything and sort
```

1. **Cache each author's recent posts** — It converts the fan-out from database queries into cache reads.
2. **Batch the lookups rather than iterating** — Latency should be one round trip, not the number of followed accounts.
3. **Merge lazily and stop at a page** — Fetching everything and sorting does far more work than is needed.
4. **Cache the assembled feed for a short period** — Pagination and repeated opens dominate real traffic.
5. **Bound how many accounts are queried** — Very large followings need sampling or prioritisation.
6. **Use it selectively for large accounts** — It is a complement to precomputation, not a replacement.

**Where it wins and where it fails**

```text
WINS
  celebrity posts       one write regardless of
                        follower count
  inactive users        cost nothing at all
  deletions and edits   nothing to clean up; the
                        feed is built fresh
  new follows           appear immediately, with
                        history, no backfill
  ranking changes       apply instantly, since
                        nothing is precomputed

FAILS
  large followings      read cost scales with them
  constant feed opens   the same work repeated
  deep pagination       each page costs another
                        assembly
  latency               bounded by the slowest source

THE SYMMETRY
  fanout-on-write  cost ∝ followers   (write side)
  fanout-on-read   cost ∝ following   (read side)
  -> neither distribution is bounded in a real social
     graph
  -> which is why production systems use both

WHAT MAKES IT PRACTICAL
  the per-author cache is doing most of the work; a
  pure database-backed fanout-on-read is rarely
  viable at scale
```

| Metric | Value | Note |
|---|---|---|
| Write cost | one insert | **regardless of reach** |
| Read cost | × following | unbounded |
| Deletes/edits | free | nothing precomputed |
| Depends on | author caches | not optional |

> **Deep pagination multiplies the cost of an already expensive read**  
> Each page requires another merge across the followed accounts, so a user scrolling through five pages pays the assembly cost five times. Cursor-based pagination over a short-lived cached merge is what makes scrolling affordable; recomputing from scratch per page turns an engaged user into the most expensive user on the platform.

**Technologies**

| Piece | Options | Note |
|---|---|---|
| Per-author post cache | Redis lists per author | The mechanism that makes this viable |
| Batched fetching | Multi-get or scatter-gather | Latency is one round trip, not many |
| Merge | K-way merge over ordered sources | Stop once the page is full |
| Short-lived feed cache | Cached merged result | Covers pagination and repeat opens |
| Following lists | Graph store | Read on every feed open; heavily cached |
| Complement | Fanout on write | For ordinary accounts |

The per-author cache is the component that decides whether this pattern is practical. With it, a five-hundred-source fanout is five hundred cache reads batched into a couple of round trips; without it, the same request is five hundred database queries and will not survive contact with real traffic.

**Trade-offs**

**Read-time against write-time assembly**

| Property | Fanout on read | Fanout on write |
|---|---|---|
| Write cost | Constant | Proportional to followers |
| Read cost | Proportional to following | Constant |
| Storage | One copy per post | One entry per follower |
| Deletes and edits | Free | Expensive or filtered |
| New follows | Immediate with history | Need backfill |
| Ranking changes | Apply instantly | Require recomputation |
| Scales badly when | Users follow many accounts | Accounts have many followers |

The final row states the symmetry plainly: each pattern is unbounded in the dimension the other handles, and real social graphs are unbounded in both directions — which is the argument for combining them rather than choosing.

> **Ask before choosing it**  
> How many accounts does a typical user follow, and how often do they open the feed? A product where people follow a handful of sources and check occasionally can use read-time assembly alone; one where users follow thousands and refresh constantly cannot.

**How it fails**

**How fanout on read goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Feed latency unacceptable | Querying sources serially | Batch the fetches into one round trip |
| Database overwhelmed by feed opens | No per-author caching | Cache each author's recent posts |
| Slowest source sets the latency | Waiting for every source | Timeout slow sources; serve what arrived |
| Scrolling is extremely expensive | Recomputing the merge per page | Cursor pagination over a cached merge |
| Users following thousands time out | Unbounded fan-out | Prioritise or sample sources |
| Memory exhausted during merge | Fetching everything before sorting | Lazy k-way merge, stop at a page |
| Cost dominated by repeat opens | No short-lived feed cache | Cache the assembled result briefly |

> **Waiting for every source makes the feed as slow as the worst one**  
> A merge across five hundred sources completes when the last responds, so a single slow shard determines the latency of the whole feed — the same tail-at-scale arithmetic that makes any large fan-out unpredictable. Timing out slow sources and serving a slightly incomplete feed is almost always better for the user than a complete feed that takes two seconds, and deciding that policy in advance is part of the design.

**Where it is used**

- **Twitter's celebrity path**, where posts from very large accounts are merged at read time rather than fanned out.
- **Feeds with small followings**, such as professional or interest-based products where read-time assembly is sufficient alone.
- **Search-driven and topic feeds**, which are assembled per request by construction.
- **Chronological timelines in smaller products**, where the simplicity outweighs the read cost.
- **Any feed where posts are frequently edited or removed**, since nothing precomputed has to be cleaned up.

**In the interview**

Present it as the mirror image of fanout-on-write, then show why neither alone is sufficient.

- **State the symmetry**: write-time cost scales with followers, read-time cost scales with following.
- **Say the per-author cache is what makes it viable**, since raw database fan-out at read time does not survive real traffic.
- **Batch the fetches** so latency is one round trip rather than the number of followed accounts.
- **Merge lazily and stop at a page**, rather than fetching everything and sorting.
- **Give the operational advantages** — free deletes and edits, immediate new follows, instant ranking changes — because they are real and often forgotten.

**Practice drill**

A user follows 500 accounts and opens their feed ten times a day; the platform serves 10,000 feed opens per second. Compute the naive query load and design the containment: caching, batching, merging and pagination. State the resulting per-request latency budget and where it is spent. Then explain what this design does well that fanout-on-write does badly, with three specific operations.

**Go deeper**

Fanout on read stores each post once and assembles a feed at request time by merging the recent posts of every followed account.

**It makes cost proportional to consumption rather than to reach.** A post costs one write regardless of whether its author has ten followers or fifty million, and feeds that are never opened cost nothing at all. In a social graph where a large fraction of accounts are dormant, that eliminates an enormous amount of work that precomputation performs and discards.

**Its cost reappears on the read side, scaled by following count.** A viewer following two thousand accounts triggers a two-thousand-way gather on every feed open, and because feeds are opened constantly, the same assembly is repeated endlessly for data that has not changed. This is the exact mirror of the write-side problem, and it is equally unbounded — which is why the two patterns are complements rather than alternatives.

**The per-author cache is what makes it practical rather than theoretical.** Caching each author's recent posts converts the gather from hundreds of database queries into hundreds of cache reads that batch into a couple of round trips. Without that layer the pattern is not viable at scale, and discussions that present read-time assembly as inherently cheap are usually assuming this component without saying so.

**Latency is governed by the slowest source, not the average.** Waiting for every followed account means one degraded shard sets the response time for the whole feed, so a policy of timing out stragglers and serving a marginally incomplete feed is almost always the better user experience. Making that decision explicitly, with a defined timeout and a defined degradation, is what separates a fan-out that performs from one that is merely correct.

**Mutations are free, and this is a real advantage.** Because nothing is precomputed, a deleted post disappears, an edit takes effect, a new follow brings history, and a ranking change applies — all immediately, with no cleanup and no backfill. Systems that precompute pay ongoing complexity for each of these, which is worth remembering when the read cost is being weighed.

**Short-lived caching of the assembled feed quietly blurs the distinction.** Caching a merged result for thirty seconds makes pagination and repeat opens free, and for an active user it means the feed is effectively precomputed — just triggered by their first request rather than by every author's post. That middle ground is where the hybrid approaches live, and recognising that the two patterns are ends of a spectrum rather than opposites is what makes choosing between them a matter of thresholds rather than doctrine.

**Related patterns:** Fanout On Write · Hybrid Fanout · Cache Aside · Scatter Gather

---

### Hybrid Fanout

*Precompute feeds for ordinary accounts and merge large accounts in at read time, so neither followers nor followings can make an operation unbounded.*

> **When you hear…** A follower distribution with a long tail · pure fanout failing on celebrities or on heavy followers · a production feed at scale

**Flow:** `Classify authors` → `Normal accounts fan out` → `Large accounts do not` → `Read merges both` → `Costs stay bounded`

**The problem**

Fanout on write costs one insert per follower, so an account with fifty million followers produces fifty million writes per post. Fanout on read costs one query per followed account, so a user following two thousand accounts pays two thousand lookups per feed open.

Both costs are unbounded in a real social graph, and they are unbounded in different directions. Choosing either one alone means accepting that some portion of the user base will have an experience that cannot be made acceptable.

> **Apply each strategy where it is bounded and avoid it where it is not**  
> The two approaches fail on opposite ends of the same distribution. Precomputing for accounts with ordinary follower counts is cheap and keeps reads fast; leaving the handful of very large accounts out of precomputation and merging them at read time keeps writes cheap, and because those accounts are few, the read-time merge stays small. Each mechanism is used only in the region where its cost is bounded.

**Mental model**

A threshold on follower count divides authors into two classes. Feeds are the union of a precomputed inbox and a small read-time merge over the large accounts the viewer follows.

1. **Classify** — Authors above a follower threshold are marked as non-fanout.
2. **Write** — Ordinary posts are fanned out to followers; large-account posts are stored only.
3. **Read** — The precomputed inbox is fetched.
4. **Merge** — Recent posts from the viewer's followed large accounts are merged in.
5. **Serve** — The combined, ordered result is returned.

> **The threshold creates two code paths that must produce identical results**  
> An account crossing the threshold changes strategy, and the transition must not duplicate posts, lose them, or reorder them relative to the other path. Getting this wrong produces feeds with missing or repeated items that appear only for users following an account that recently crossed the line — a bug that is difficult to reproduce and easy to dismiss as a client issue.

**How it works**

**The threshold and the read path**

```text
CLASSIFICATION
  followers < 100,000   -> fan out on write
  followers >= 100,000  -> do not fan out

WHY ~100,000
  fanout of 100,000 inserts is a few seconds of work
  -> acceptable as a background job
  and the number of accounts above it is small
  -> so the read-time merge stays small

WRITE PATH
  ordinary author  -> insert into every follower's
                      inbox (async, capped, active
                      followers only)
  large author     -> store the post; append to the
                      author's own recent-post cache

READ PATH
  1  fetch the viewer's precomputed inbox  (1 read)
  2  find which large accounts they follow (cached
     list, usually 0-20)
  3  fetch those authors' recent posts    (batched)
  4  merge by time or rank, take a page

TYPICAL COST
  1 inbox read + 1 batched multi-get
  -> two round trips regardless of following size
  -> because the number of LARGE accounts a person
     follows is small even when their total following
     is huge
```

1. **Set the threshold from what a fanout job can absorb** — It should be the largest fanout that runs comfortably in the background.
2. **Cache each viewer's list of followed large accounts** — Computing it per feed open reintroduces the read-time cost.
3. **Deduplicate carefully at the merge** — A post can appear in both paths while an account is crossing the threshold.
4. **Handle threshold crossings explicitly** — Backfill or exclude deliberately, rather than leaving it to chance.
5. **Keep both paths producing the same ordering** — Feeds that reorder depending on which path supplied an item look broken.
6. **Monitor the two paths separately** — The failure modes are different and an aggregate metric hides both.

**Why the read-time merge stays small**

```text
THE FOLLOWER DISTRIBUTION
  99.9% of accounts   < 10,000 followers
  ~0.1%               100,000 to millions
  -> almost every post is fanned out cheaply

THE FOLLOWING DISTRIBUTION
  a user follows 2,000 accounts
  of those, how many have > 100,000 followers?
  typically 5-30
  -> the read-time merge is over tens of sources,
     not thousands

THIS IS THE WHOLE TRICK
  large accounts are rare, so few of them appear in
  any one following list
  -> the expensive path is applied to a small,
     naturally bounded set

COST COMPARISON
  pure write fanout   50,000,000 inserts per
                      celebrity post
  pure read fanout    2,000 queries per feed open
  hybrid              a few hundred inserts per
                      ordinary post
                      + ~20 cached lookups per read

TUNING THE THRESHOLD
  lower  -> cheaper writes, larger read merges
  higher -> cheaper reads, heavier fanout jobs
  -> measure both and move it; it is the main dial
```

| Metric | Value | Note |
|---|---|---|
| Write path | bounded | by threshold |
| Read path | bounded | **few large accounts** |
| Round trips | two | regardless of following |
| Main dial | the threshold | measure and adjust |

> **Duplicates appear when an account crosses the threshold**  
> An account that grows past the limit has posts already sitting in followers' inboxes, and its new posts will arrive through the read-time path. If the merge does not deduplicate by post identifier, followers see items twice during the transition; if the transition is handled by removing the old entries, that removal is itself a large fanout. Deduplicating at the merge is the cheaper and more robust choice.

**Technologies**

| Piece | Options | Note |
|---|---|---|
| Precomputed inboxes | Redis sorted sets or wide-column store | Capped per user |
| Large-account posts | Per-author recent-post cache | The read-time source |
| Classification | Follower count with hysteresis | Avoid flapping at the boundary |
| Followed-large-accounts list | Cached per viewer | Must not be computed per request |
| Merge and ranking | Application layer | Where the two paths become one feed |
| Ranking service | Scores the merged candidate set | Modern feeds rank after merging |

Hysteresis on the classification matters more than it sounds: an account hovering at the threshold that flips strategy repeatedly produces inconsistent feeds and repeated backfills, so the promote and demote thresholds should be different values.

**Trade-offs**

**The three strategies**

| Strategy | Write cost | Read cost | Complexity |
|---|---|---|---|
| Fanout on write | Unbounded by followers | One lookup | Low |
| Fanout on read | One insert | Unbounded by following | Low |
| Hybrid | Bounded by threshold | Bounded by rare large accounts | Two paths to keep consistent |
| Hybrid with ranking | Same | Same plus scoring | Highest |

The complexity column is the honest cost. A hybrid has two write paths, two read sources and a merge that must reconcile them, which is meaningfully more code and more failure modes than either pure approach — justified only once one of those approaches has actually stopped working.

> **Ask before choosing it**  
> Does the follower distribution actually have a long tail? A product where no account exceeds a few thousand followers gains nothing from a hybrid and pays its complexity, so the pattern should be introduced when the distribution demands it rather than by default.

**How it fails**

**How hybrid fanout goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Duplicate posts in a feed | Both paths supplying the same post | Deduplicate by post identifier at the merge |
| Posts missing during a transition | Account crossed the threshold with no handling | Explicit promote and demote procedures |
| Read cost creeping upward | Threshold too low, many accounts on the read path | Raise it; measure the merge size |
| Fanout jobs backing up | Threshold too high for the worker pool | Lower it, or add capacity |
| Inconsistent ordering between paths | Different sort keys or precision | Single ordering rule applied after merging |
| Accounts flapping across the boundary | Single threshold with no hysteresis | Separate promote and demote values |
| Followed-large-account list computed per read | Not cached | Cache it and invalidate on follow changes |

> **Two paths diverging is the characteristic failure of this pattern**  
> Everything that makes a hybrid valuable comes from having two mechanisms, and everything that makes it fragile comes from needing them to agree. Different ordering, different filtering rules for blocks, different handling of deletions, or a post appearing in both — each produces feeds that are subtly wrong for a subset of users, and each is invisible in aggregate metrics. The merge must be the single place where ordering, filtering and deduplication are decided, for items from both sources.

**Where it is used**

- **Twitter's timeline**, the best-documented hybrid and the origin of the celebrity threshold idea.
- **Instagram and comparable feeds**, precomputing for ordinary accounts and merging large ones at read time.
- **Professional networks**, where following distributions are skewed but less extreme.
- **Notification systems**, which apply the same split for high-volume sources.
- **Any feed with a long-tailed follower distribution**, which is nearly all of them at scale.

**In the interview**

Arrive here by showing both pure approaches failing, then explain why the read-time merge stays small.

- **Show both failures with numbers** before proposing the hybrid; the arithmetic is the argument.
- **Give a concrete threshold** and justify it from what a fanout job can absorb comfortably.
- **Explain why the merge is small**: large accounts are rare, so a viewer follows only a few of them.
- **Raise deduplication and threshold crossings**, which are the specific bugs this design introduces.
- **Acknowledge the complexity cost** and say it should be adopted when a pure approach breaks, not by default.

**Practice drill**

Design a feed for a platform whose median account has 200 followers and whose largest has 50 million, with users following up to 2,000 accounts. Choose a threshold and justify it from both directions. Compute the write cost for an ordinary post and a celebrity post, and the read cost for a heavy follower. Then handle an account crossing the threshold in both directions, saying exactly how you avoid duplicates and gaps.

**Go deeper**

Hybrid fanout precomputes feeds for ordinary accounts and merges large accounts at read time, bounding both the write and read paths.

**It exists because the two pure strategies fail on opposite ends of the same distribution.** Precomputation is unbounded in followers, read-time assembly is unbounded in followings, and a real social graph is unbounded in both. Choosing either alone means a segment of users — celebrities or heavy followers — has an experience that cannot be fixed by tuning, which is why every large platform converges on a combination.

**The reason it works is a property of the graph rather than of the code.** Accounts with enormous follower counts are rare, so even a user following thousands of accounts follows only a few dozen of them. The expensive read-time merge is therefore over tens of sources rather than thousands, and stays small automatically as the platform grows — the bound comes from the distribution, not from a limit anyone has to enforce.

**The threshold is the primary control and should be measured, not guessed.** Lowering it makes writes cheaper and read merges larger; raising it does the reverse. The right value is roughly the largest fanout that the background workers absorb comfortably, and it moves as the platform's capacity and follower distribution change — which means it should be a configurable number with dashboards on both sides, not a constant chosen once.

**Its characteristic fragility is the two paths disagreeing.** Ordering, block filtering, deletion handling and deduplication must produce identical results whichever path an item came from, or feeds become subtly wrong for the users who happen to follow an account near the boundary. Concentrating all of those decisions in the merge — one place where items from both sources are ordered, filtered and deduplicated — is what keeps the divergence from accumulating.

**Threshold crossings need explicit handling in both directions.** An account growing past the limit has posts already in inboxes and new posts arriving through the read path, which produces duplicates unless the merge deduplicates by identifier. An account shrinking below it stops appearing in the read path and has no history in inboxes, which produces a gap unless something backfills. Hysteresis — different values for promotion and demotion — prevents an account at the boundary from oscillating between the two and triggering this repeatedly.

**It is a pattern to adopt under pressure rather than by default.** Two write paths, two read sources and a reconciliation layer are genuinely more code and more failure modes than either pure approach, and a product whose accounts all have comparable follower counts gains nothing from the split. The signal to introduce it is a measured failure — fanout jobs backing up behind large accounts, or feed latency scaling with following size — and arriving at it from that evidence produces a better threshold than adopting it because large platforms do.

**Related patterns:** Fanout On Write · Fanout On Read · Hot Key Mitigation · Materialized View

---

## Traffic Control

### Token Bucket

*Accumulate tokens at a fixed rate up to a cap, spending one per request, so a steady rate is enforced while short bursts are still allowed.*

> **When you hear…** A sustained rate limit that must still permit bursts · API quotas · protecting a backend from spikes

**Flow:** `Tokens refill steadily` → `Bucket has a cap` → `Request takes a token` → `Empty means reject` → `Burst absorbed`

**The problem**

A rate limit of one hundred requests per second, enforced strictly, rejects a client that sends five requests in the same millisecond and then nothing for a second — even though their average is far below the limit. Real clients batch, retry and page, so strict smoothing rejects legitimate traffic constantly.

Allowing bursts without any bound is equally wrong: a client that saves up an hour of quota and spends it in one second delivers a spike the backend was never sized for.

> **Separate the sustained rate from the permitted burst**  
> A bucket that fills at the allowed rate and holds a maximum number of tokens encodes both properties in two numbers. The refill rate bounds long-run throughput exactly; the capacity bounds how much unused allowance can be saved and spent at once. A client that has been idle can burst up to the capacity and no further, and one that is continuously active is limited precisely to the refill rate.

**Mental model**

A bucket refilled continuously at a fixed rate, capped at a maximum. Each request removes a token; a request arriving at an empty bucket is refused.

1. **Refill** — Tokens accrue at the rate limit, continuously rather than in steps.
2. **Cap** — The bucket never holds more than its capacity.
3. **Spend** — Each request consumes one token, or more for expensive operations.
4. **Refuse** — An empty bucket means rejection, usually with a retry hint.
5. **Recover** — Idle time refills the bucket, restoring burst capacity.

> **Capacity is the burst the backend must survive**  
> The bucket size is not a generosity setting; it is a promise that the system can absorb that many requests arriving simultaneously. A capacity of a thousand with ten thousand clients means ten million requests can arrive in the same instant while every client remains within its limit. Capacity must be chosen from what the backend can take, not from what feels fair.

**How it works**

**The algorithm and why it needs no timer**

```text
STATE PER KEY
  tokens        current count
  last_refill   timestamp

ON REQUEST
  now = current time
  elapsed = now - last_refill
  tokens = min(capacity, tokens + elapsed * rate)
  last_refill = now
  if tokens >= 1:
    tokens -= 1
    ALLOW
  else:
    retry_after = (1 - tokens) / rate
    REJECT with that hint

-> refill is COMPUTED on access, not by a background
   job
-> so idle keys cost nothing and there is no timer
   per client

TWO NUMBERS, TWO PROPERTIES
  rate = 100/s        long-run throughput
  capacity = 500      maximum instantaneous burst
  -> idle for 5 s, then 500 requests at once: allowed
  -> then 100/s sustained thereafter

VARIABLE COST
  cheap read      1 token
  expensive query 10 tokens
  -> the limit becomes about work, not request count
  -> far more useful than counting requests equally
```

1. **Compute refill lazily on access** — A background timer per client does not scale and is unnecessary.
2. **Choose capacity from backend burst tolerance** — It is the size of the spike you are promising to absorb.
3. **Charge expensive operations more tokens** — Counting all requests equally limits the wrong thing.
4. **Return a retry hint on rejection** — Clients that know when to return stop hammering.
5. **Decide the scope deliberately** — Per user, per key, per IP and global limits solve different problems.
6. **Account for multiple instances** — Per-instance buckets multiply the effective limit by the fleet size.

**Distributed enforcement**

```text
PER-INSTANCE BUCKETS
  limit 100/s, 10 instances
  -> each enforces 100/s
  -> the client can actually do 1000/s
  -> divide by instance count? then a client routed
     unevenly is throttled below their limit

SHARED COUNTER  (accurate)
  bucket state in Redis
  atomic script: refill, check, decrement
  -> exact enforcement
  -> a network round trip on EVERY request
  -> the limiter becomes a dependency and a
     bottleneck

LOCAL WITH PERIODIC SYNC  (practical)
  each instance holds a local allowance
  syncs with the shared view every second
  -> allows brief overshoot
  -> no per-request round trip
  -> the usual production choice

WHEN EXACTNESS MATTERS
  billing quotas      -> shared counter
  abuse prevention    -> local is fine; approximate
                         limits still stop abuse
  backend protection  -> local is fine; the goal is
                         a bound, not a number
```

| Metric | Value | Note |
|---|---|---|
| Rate | sustained limit | long-run exact |
| Capacity | burst allowed | **backend must absorb** |
| Refill | lazy on access | no timers |
| Distributed | sync periodically | slight overshoot |

> **Per-instance limiters multiply the limit by the fleet size**  
> Ten instances each enforcing a hundred per second permit a thousand per second from one client, and the limit written in the documentation is simply not the limit in effect. Dividing by the instance count fixes the arithmetic and breaks fairness when routing is uneven, which is why production systems usually keep a shared view synchronised periodically and accept brief overshoot as the price of not adding a round trip to every request.

**Technologies**

| Layer | Options | Note |
|---|---|---|
| API gateways | Built-in token bucket limiting | Usually the right place; no application code |
| Redis with an atomic script | Shared, accurate buckets | A round trip per request |
| In-process libraries | Per-instance buckets | Fast; multiplies by fleet size |
| Service meshes | Per-route limits | Applied without touching the application |
| Cloud provider quotas | Managed, per-account | The model most public APIs expose |
| Leaky bucket | Smooths instead of bursting | Different shape, different purpose |

The gateway is usually the correct layer: it sees every request before any backend work happens, its state is already shared across the fleet, and rejecting there costs almost nothing compared with rejecting after a service has already begun processing.

**Trade-offs**

**Rate limiting algorithms**

| Algorithm | Allows bursts | Memory per key | Accuracy |
|---|---|---|---|
| Token bucket | Yes, up to capacity | Two values | Exact over the long run |
| Leaky bucket | No, smooths output | Queue or counter | Exact output rate |
| Fixed window | Yes, at boundaries | One counter | Poor; 2× at edges |
| Sliding window log | No | One entry per request | Exact |
| Sliding window counter | Slightly | Two counters | Good approximation |

Token bucket is the usual default because it matches how clients actually behave — idle periods followed by short bursts — and because its two parameters map directly onto the two things the operator cares about: sustained load and peak instantaneous load.

> **Ask before choosing it**  
> What is the limit protecting? A backend that cannot absorb bursts needs smoothing rather than a large capacity; a billing quota needs exactness and a shared counter; abuse prevention needs neither and can happily run locally with approximate numbers.

**How it fails**

**How token buckets go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Effective limit is far higher than documented | Per-instance buckets | Shared state or periodic synchronisation |
| Backend overwhelmed despite limits | Capacity larger than burst tolerance | Size capacity from what the backend absorbs |
| Clients retry immediately on rejection | No retry hint returned | Return the wait time with the rejection |
| Expensive queries exhaust the backend | All requests cost one token | Variable token cost by operation |
| The limiter becomes the bottleneck | Shared counter round trip per request | Local allowance with periodic sync |
| Legitimate bursts rejected | Capacity too small for real client behaviour | Measure actual burst patterns |
| One client starves others | Global limit with no per-client scope | Per-client buckets beneath a global cap |

> **A rate limiter that fails closed takes the service down with it**  
> If the shared store holding bucket state becomes unavailable and the limiter rejects everything it cannot verify, an outage in the limiter becomes a total outage of the service it was protecting. Failing open risks a brief period of unlimited traffic; failing closed guarantees zero traffic. For most systems the first is clearly preferable, and the decision must be made explicitly rather than inherited from whatever the client library does on error.

**Where it is used**

- **Public API quotas**, where burst allowance plus sustained rate is the standard published model.
- **API gateways**, enforcing per-key limits before any backend work is done.
- **Network traffic shaping**, the origin of both bucket algorithms.
- **Internal service protection**, bounding what one caller can demand of a shared backend.
- **Cost control on expensive operations**, where variable token costs limit work rather than requests.

**In the interview**

The two parameters and what each one means to the backend is the substance here.

- **Name the two properties**: refill rate bounds sustained throughput, capacity bounds the instantaneous burst.
- **Say capacity is a promise about burst absorption**, not a fairness setting, and size it from the backend.
- **Describe lazy refill** — computed on access, no timers — which is what makes it cheap at scale.
- **Raise distributed enforcement** and the fleet-multiplication problem, then choose local with sync and justify the overshoot.
- **Decide fail-open versus fail-closed explicitly**, because a limiter that fails closed converts its own outage into a total one.

**Practice drill**

An API offers 1,000 requests per minute per key. Choose the refill rate and capacity, and state what burst behaviour each permits. Then deploy it across 20 instances: describe three enforcement options, pick one, and quantify the overshoot it allows. Finally, decide what happens when the shared state is unavailable, and justify your choice in terms of what the limit is protecting.

**Go deeper**

A token bucket enforces a sustained rate while permitting bounded bursts, by accruing tokens at the limit rate up to a maximum.

**Its two parameters map cleanly onto the two things that matter.** The refill rate determines long-run throughput exactly, and the capacity determines how much unused allowance can be spent at once. That separation is what makes the algorithm fit real clients, which are rarely smooth — they batch, retry, paginate and idle — and which a strictly smoothed limiter rejects while they remain well under their average allowance.

**Capacity is a statement about the backend, not about generosity.** Whatever the bucket size permits can arrive simultaneously, multiplied by the number of clients, so a large capacity chosen to be accommodating is a commitment to absorb a proportionally large spike. Sizing it from measured burst tolerance rather than from what seems reasonable is what keeps the limiter protective rather than merely nominal.

**Lazy refill is what makes it cheap.** Computing accrued tokens from elapsed time on each access means there are no timers, no background jobs and no cost for idle keys — state is two numbers per client, updated only when that client is active. This is why token buckets scale to millions of keys where implementations based on scheduled refills do not.

**Distributed enforcement forces a choice between accuracy and latency.** Per-instance buckets multiply the effective limit by the fleet size, a shared counter adds a round trip to every request and makes the limiter a dependency of everything it protects, and local allowances synchronised periodically permit brief overshoot while costing nothing per request. The third is the usual answer, and it is correct because most limits exist to bound load rather than to be exact — billing quotas being the notable exception.

**Charging by cost rather than by request makes the limit meaningful.** Treating a cheap lookup and an expensive aggregation as one unit each limits the wrong quantity: a client within its request limit can still exhaust the backend by choosing expensive operations. Variable token costs turn the bucket into a budget for work, which is what the backend actually has a finite amount of.

**Its failure behaviour deserves an explicit decision.** When the state store is unreachable, a limiter that rejects what it cannot verify converts its own outage into a total outage of the protected service, while one that allows traffic through risks a window of unlimited load. For nearly all systems the second risk is smaller, but the choice is frequently left to a client library's default — which means the behaviour under the limiter's own failure has never actually been chosen.

**Related patterns:** Leaky Bucket · Sliding Window Rate Limit · Load Shedding · Backpressure

---

### Leaky Bucket

*Queue incoming requests and release them at a constant rate, converting bursty arrivals into a perfectly smooth output stream.*

> **When you hear…** A downstream system that cannot absorb bursts · a strict output rate required · smoothing matters more than latency

**Flow:** `Requests arrive bursty` → `Enter a bounded queue` → `Drain at fixed rate` → `Output is smooth` → `Overflow rejected`

**The problem**

A backend can process exactly two hundred operations per second and degrades badly above that. A token bucket with any meaningful capacity permits a burst that exceeds it, and the backend's failure under overload is worse than the delay that queueing would cause.

Some downstreams have hard rate constraints that are not about capacity at all — a third-party API with a contractual limit, a device with a fixed processing rate, a system whose behaviour beyond a threshold is undefined.

> **Smooth the output rather than policing the input**  
> If requests are placed in a queue that drains at a constant rate, the arrival pattern stops mattering: whatever shape the traffic has going in, the output is a steady stream at exactly the configured rate. The cost is latency for requests that wait, and rejection only when the queue itself fills — which converts burstiness into delay rather than into failure.

**Mental model**

A bounded queue with a constant-rate drain. Arrivals join the queue if there is room; the drain releases work at a fixed interval regardless of how much is waiting.

1. **Arrive** — A request joins the queue if space remains.
2. **Overflow** — A full queue means immediate rejection.
3. **Drain** — Work is released at the configured rate, no faster.
4. **Smooth** — Output is constant regardless of the arrival pattern.
5. **Wait** — Queued requests experience latency proportional to their position.

> **Queued requests may have already been abandoned**  
> A request waiting three seconds in a smoothing queue behind a burst may face a client that timed out after two. Processing it does work for nobody, and in a sustained burst the queue can fill entirely with requests whose callers have gone. Bounding the queue by time — dropping anything older than the client's timeout — is what prevents the smoother from spending all its capacity on abandoned work.

**How it works**

**Smoothing versus bursting**

```text
ARRIVALS  (bursty)
  t=0.0  50 requests
  t=0.5   0
  t=1.0  50 requests
  t=1.5   0

TOKEN BUCKET  (rate 100/s, capacity 100)
  t=0.0  all 50 pass immediately
  t=1.0  all 50 pass immediately
  -> output is as bursty as the input
  -> the backend sees 50 at once, twice

LEAKY BUCKET  (rate 100/s, queue 100)
  t=0.0  50 queued
  drain releases one every 10 ms
  -> output: a steady 100/s
  -> the backend never sees more than one at a time
  -> the 50th request waits 500 ms

THE TRADE
  token bucket   low latency, bursty output
  leaky bucket   smooth output, added latency

QUEUE SIZING
  queue = rate x acceptable_wait
  100/s, 1 s acceptable wait -> queue of 100
  -> a larger queue does not add throughput; it only
     adds waiting
```

1. **Size the queue from acceptable latency, not from memory** — Queue depth converts directly into waiting time at the drain rate.
2. **Drop requests older than the client timeout** — Processing abandoned work consumes the capacity live requests need.
3. **Reject immediately when the queue is full** — Fast rejection is better than an unbounded wait.
4. **Use it only where smoothing is the requirement** — Where bursts are harmless, the added latency buys nothing.
5. **Consider priority within the queue** — Not all waiting work is equally worth the remaining capacity.
6. **Monitor queue depth and wait time** — Depth alone does not say whether anything is still being served usefully.

**Where each bucket belongs**

```text
USE LEAKY BUCKET WHEN
  the downstream has a HARD rate limit
    third-party API with a contractual cap
  bursts cause disproportionate damage
    a backend that collapses above its rate
  the output rate itself is the requirement
    device control, media streaming, egress shaping
  delay is preferable to rejection
    background jobs, uploads, notifications

USE TOKEN BUCKET WHEN
  clients are interactive and latency matters
  bursts are normal and absorbable
  rejecting is better than delaying
  you are limiting a caller rather than protecting a
  fragile downstream

COMBINE THEM
  token bucket at the edge   (limit each client)
  leaky bucket before a
  fragile dependency         (smooth the aggregate)
  -> clients get burst tolerance; the dependency
     gets a steady stream

THE QUEUE IS THE DIFFERENCE
  token bucket has no queue: allowed or rejected
  leaky bucket has a queue: delayed or rejected
  -> which is acceptable depends entirely on whether
     the caller is waiting
```

| Metric | Value | Note |
|---|---|---|
| Output | perfectly smooth | **constant rate** |
| Cost | added latency | proportional to depth |
| Queue size | rate × wait | not memory-driven |
| Best for | fragile downstream | or hard caps |

> **A larger queue adds waiting, not throughput**  
> The drain rate is fixed, so doubling the queue does not process more work — it only allows twice as many requests to wait twice as long. Teams that respond to overflow rejections by increasing the queue convert a fast, honest failure into slow, invisible timeouts, and the arrival rate that exceeded the drain rate still exceeds it. The only real remedies are raising the drain rate or reducing arrivals.

**Technologies**

| Context | Implementation | Note |
|---|---|---|
| Network shaping | Traffic shapers and queue disciplines | The original application |
| Message consumers | Fixed-rate polling from a queue | The queue is already there |
| Outbound API calls | Scheduler releasing at the allowed rate | Respects a third party's contractual limit |
| Job processing | Worker pool with a fixed dispatch rate | Smooths load on shared databases |
| Token bucket | The bursty alternative | Different shape, different purpose |
| Backpressure | Slow the producer instead | Better where the producer can be slowed |

Backpressure deserves the comparison: where the producer is under your control, telling it to slow down is superior to queueing its output, because it moves the problem to a place that can actually reduce demand rather than storing it.

**Trade-offs**

**Shaping against limiting**

| Property | Leaky bucket | Token bucket |
|---|---|---|
| Output shape | Constant rate | As bursty as the input |
| Excess handling | Queued, then rejected | Rejected immediately |
| Latency | Increases with queue depth | Unaffected |
| Memory | Queue of pending work | Two numbers |
| Best for | Protecting fragile downstreams | Limiting client behaviour |
| Client experience | Slow but served | Fast success or fast failure |

The last row is usually the deciding one. An interactive user prefers a fast rejection they can retry over a request that eventually succeeds after four seconds; a background job prefers the opposite, which is why the two algorithms tend to appear at different layers of the same system.

> **Ask before choosing it**  
> Is anyone waiting for this request? Smoothing suits work whose latency nobody observes — uploads, notifications, batch jobs — and suits interactive requests badly, because the queue converts their burst into a wait they experience directly.

**How it fails**

**How leaky buckets go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Capacity spent on abandoned requests | No expiry on queued work | Drop entries older than the client timeout |
| Timeouts instead of rejections | Queue enlarged to avoid overflow | Size from acceptable wait; reject beyond it |
| Memory exhausted | Unbounded queue | A bound is mandatory |
| Interactive requests feel broken | Smoothing applied to user-facing traffic | Token bucket at the user-facing edge |
| Throughput lower than expected | Drain rate below actual capacity | Measure the downstream's real rate |
| Important work stuck behind bulk | Single undifferentiated queue | Priority classes within the queue |
| Problem hidden until collapse | Only depth monitored | Track wait time and drop rate as well |

> **Enlarging the queue to stop rejections converts failure into latency**  
> Overflow rejections are a signal that arrivals exceed the drain rate, and a bigger queue does not change that arithmetic — it only lets more requests accumulate before the same rejection happens, with every one of them waiting longer first. The visible symptom shifts from honest errors to client timeouts, which are harder to diagnose and worse for the user, while the underlying imbalance is untouched.

**Where it is used**

- **Network traffic shaping**, where constant-rate output is the defining requirement.
- **Outbound calls to third-party APIs**, releasing requests at exactly the contractual rate.
- **Job dispatch to shared databases**, smoothing bursts that would otherwise spike load.
- **Media streaming and egress control**, where the output rate is the product requirement.
- **Notification delivery**, spreading a large batch over time rather than sending it all at once.

**In the interview**

Define it against the token bucket; the contrast is what makes each one's purpose clear.

- **Give the distinguishing property**: output is smooth regardless of input, which a token bucket does not provide.
- **Say the queue is the difference** — delayed rather than rejected — and that this is only acceptable when nobody is waiting.
- **Size the queue from acceptable wait time**, and state that a bigger queue adds latency rather than throughput.
- **Raise abandoned requests.** Work queued past the client's timeout consumes capacity for nobody.
- **Place both in one system**: token bucket at the client edge, leaky bucket in front of a fragile dependency.

**Practice drill**

A third-party API allows exactly 100 calls per second and your system generates bursts of 500. Design the smoothing layer: drain rate, queue size, overflow behaviour and expiry policy. Compute how long the last request in a 500-item burst waits. Then decide whether this design suits a user-facing request path, and if not, say what you would use there instead and why.

**Go deeper**

A leaky bucket converts bursty arrivals into a constant-rate output stream by queueing work and releasing it at a fixed interval.

**Its defining property is output shape, which is what distinguishes it from a token bucket.** A token bucket decides whether each request is allowed and passes it through immediately, so the downstream sees whatever burst pattern the client produced. A leaky bucket decides when each request happens, so the downstream sees a steady stream no matter what arrived. Where a dependency's problem is burstiness rather than volume, only the second addresses it.

**It trades rejection for delay, which is only acceptable when nobody is waiting.** Queued work is eventually served rather than refused, which suits uploads, notifications and background jobs where latency is unobserved. Applied to an interactive path the same behaviour is worse than rejecting: the user experiences the queue directly as an unresponsive application, and a fast error they can retry would have been preferable.

**Queue depth is a latency parameter disguised as a capacity parameter.** At a fixed drain rate, depth converts directly into waiting time, so sizing it should start from how long a request may acceptably wait and work backwards. Enlarging the queue in response to overflow rejections is the characteristic mistake: it does not increase throughput, it converts honest fast failures into client timeouts, and the imbalance between arrival and drain rates remains exactly as it was.

**Abandoned work is a real and underestimated cost.** During a sustained burst the queue fills with requests whose clients have already timed out, and the drain spends its entire fixed capacity producing results nobody will receive — while newly arriving live requests are rejected for lack of space. Expiring entries older than the client timeout keeps capacity pointed at work that still matters, and it is the difference between a smoother that degrades gracefully and one that stops being useful precisely under load.

**It complements rather than competes with the token bucket, and mature systems use both.** A token bucket at the client edge gives each caller burst tolerance and rejects excess quickly, which is right for interactive traffic. A leaky bucket in front of a fragile dependency smooths the aggregate of all callers into the steady stream that dependency requires. They sit at different layers because they solve different halves of the problem: one bounds what a caller may demand, the other bounds what the dependency must survive.

**Where the producer can be slowed, backpressure is better than smoothing.** A queue stores excess demand; backpressure removes it, by making the producer wait rather than accumulating its output. Smoothing is the right answer when the producer is outside your control — an external client, a third-party integration — and the second-best answer when it is not, because storing demand always risks storing more of it than the system can eventually serve.

**Related patterns:** Token Bucket · Backpressure · Work Queue · Load Shedding

---

### Fixed Window Rate Limit

*Count requests within aligned time windows and reject beyond the limit, the simplest possible limiter and the one with a known boundary flaw.*

> **When you hear…** A rate limit needed quickly · minimal state acceptable · approximate enforcement sufficient

**Flow:** `Window starts` → `Counter increments` → `Limit reached` → `Reject until reset` → `Counter clears`

**The problem**

A limit has to be enforced with as little state and complexity as possible. Tracking individual request times per client costs memory proportional to traffic, and maintaining continuously refilling state is more machinery than a simple quota seems to warrant.

The straightforward answer — count requests in each minute and reset the count when the minute changes — is trivially cheap and has a specific, well-known defect that must be understood before it is chosen.

> **One counter per window is the cheapest limiter that works at all**  
> A single integer and an expiry per client is the minimum viable state for rate limiting, and it enforces the limit exactly within each window. The flaw is at the boundaries: because the counter resets instantly, a client can spend a full window's quota just before the reset and another immediately after, achieving twice the limit across a span shorter than one window.

**Mental model**

Time divided into aligned windows. Each client has one counter per window, incremented per request and discarded when the window ends.

1. **Bucket** — Requests are attributed to the current aligned window.
2. **Count** — Each request increments that window's counter.
3. **Compare** — Beyond the limit, requests are rejected.
4. **Reset** — The counter disappears when the window rolls over.
5. **Repeat** — The next window starts from zero, with no memory of the last.

> **Twice the limit can pass across a window boundary**  
> A client sending its full quota in the last moments of one window and again in the first moments of the next has sent double the limit within a span far shorter than the window. The limiter records two compliant windows and the backend experiences one spike of twice the intended size — which is the entire reason more elaborate algorithms exist.

**How it works**

**The counter and the boundary problem**

```text
IMPLEMENTATION
  key = client_id + ":" + floor(now / window)
  count = INCR key
  if count == 1: EXPIRE key window
  if count > limit: REJECT
  -> two Redis operations, one integer of state
  -> keys expire themselves; no cleanup needed

THE BOUNDARY
  limit 100 per minute

  10:00:59  client sends 100 requests  -> allowed
            (window 10:00 count = 100)
  10:01:00  window rolls over, count = 0
  10:01:00  client sends 100 requests  -> allowed
            (window 10:01 count = 100)

  -> 200 requests within ~1 second
  -> both windows appear compliant
  -> the backend saw 2x the limit instantaneously

HOW BAD IS IT
  worst case is exactly 2x the limit, for a duration
  approaching zero
  -> if the backend can absorb 2x briefly, the flaw
     is acceptable
  -> if it cannot, use a sliding window

WHY IT PERSISTS
  one integer per client per window
  self-expiring keys
  two operations per request
  -> nothing else is this cheap
```

1. **Use aligned windows derived from the timestamp** — It avoids storing a window start time per client.
2. **Let the key expire itself** — Self-expiring counters remove all cleanup work.
3. **Account for the doubling in capacity planning** — Size the backend for twice the limit, or choose a different algorithm.
4. **Keep windows short where the doubling matters** — A shorter window reduces the absolute size of the boundary burst.
5. **Return the time until reset** — Clients can then wait correctly rather than retrying blindly.
6. **Reserve it for cases where approximation is fine** — Billing and hard caps need something exact.

**When it is good enough**

```text
ACCEPTABLE
  abuse prevention
    a scraper limited to 2x instead of 1x is still
    effectively stopped
  coarse API quotas
    documented as approximate
  internal services with generous headroom
  very high volume where per-request state matters
    millions of keys: memory per key is the
    binding constraint

NOT ACCEPTABLE
  billing quotas
    charging for 200 when the plan allows 100
  hard third-party limits
    the provider will reject or ban regardless of
    your accounting
  fragile backends
    a 2x spike causes real damage

MITIGATION WITHOUT CHANGING ALGORITHM
  window 1 minute, limit 100   -> burst up to 200
  window 1 second, limit ~2    -> burst up to 4
  -> shorter windows shrink the absolute burst
  -> but reject legitimate bursty clients more often

THE UPGRADE PATH
  sliding window counter costs one extra counter and
  removes most of the flaw
  -> usually the right move once the flaw matters
```

| Metric | Value | Note |
|---|---|---|
| State | one integer | **per key per window** |
| Worst case | 2× the limit | at a boundary |
| Cost | two operations | cheapest available |
| Upgrade | sliding counter | one extra counter |

> **Aligned windows synchronise every client's reset**  
> When all windows begin on the minute, every client's quota refreshes simultaneously, and clients that were being throttled all resume at once. The result is a traffic spike at each window boundary across the whole user base — the boundary problem multiplied by client count. Offsetting each client's window by a hash of their identifier spreads the resets and removes the synchronised surge.

**Technologies**

| Layer | Implementation | Note |
|---|---|---|
| Redis | INCR with EXPIRE | Two commands; the standard approach |
| API gateways | Built-in fixed-window limiting | Often the default offering |
| In-memory per instance | A map of counters | Multiplies the limit by the fleet size |
| Sliding window counter | Two windows weighted | The natural upgrade |
| Token bucket | Continuous refill | No boundary flaw; slightly more state |
| CDN and edge limits | Applied before origin | Cheapest place to reject |

The sliding window counter is worth knowing as the upgrade because it costs almost nothing extra — one additional counter and a weighted calculation — and removes the doubling, which makes the fixed window's simplicity advantage much narrower than it first appears.

**Trade-offs**

**Limiter algorithms by cost and accuracy**

| Algorithm | State per key | Boundary flaw | Operations per request |
|---|---|---|---|
| Fixed window | One counter | Up to 2× | Two |
| Sliding window counter | Two counters | Small | Two to three |
| Sliding window log | One entry per request | None | Several |
| Token bucket | Two values | None | Two to three |
| Leaky bucket | A queue | None; smooths output | Queue operations |

Ranked by what they cost and what they give, the fixed window is only the right choice when either the boundary flaw genuinely does not matter or the per-key memory difference is decisive — which usually means tens of millions of keys.

> **Ask before choosing it**  
> Can the protected system absorb twice the limit briefly? If yes, the simplicity is free and the flaw is theoretical. If no, the sliding window counter costs one more integer and removes the problem, which is a trade almost always worth making.

**How it fails**

**How fixed window limiting goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Backend sees twice the limit | Boundary burst | Sliding window, or plan for 2× |
| Fleet-wide spike every minute | Aligned windows across all clients | Offset each client's window by a hash |
| Billing disputes | Approximate counting used for charging | Exact algorithm for anything billable |
| Third party rejects traffic | Their limit enforced differently from yours | Match their algorithm, or stay well below |
| Effective limit multiplied | Per-instance counters | Shared counter store |
| Clients retry immediately | No reset time returned | Return seconds until the window rolls over |
| Legitimate clients throttled | Window too short for real usage patterns | Longer window, or token bucket for burst tolerance |

> **Two systems counting differently will disagree about compliance**  
> If a provider enforces a sliding window and you account with a fixed one, your records will show compliance at moments when theirs show a violation — and their view is the one that results in rejected traffic or a suspended key. Matching the upstream's algorithm, or keeping a margin large enough that the discrepancy cannot matter, is the only way to make local accounting mean anything.

**Where it is used**

- **Simple API rate limits**, where a documented approximate quota is sufficient.
- **Abuse and scraping prevention**, where twice the limit is still an effective bound.
- **Gateway default configurations**, which very often implement exactly this algorithm.
- **Very high-cardinality limiting**, where per-key memory is the binding constraint.
- **Internal service quotas**, where headroom makes the boundary flaw immaterial.

**In the interview**

Name the boundary flaw immediately and quantify it; describing the algorithm without it reads as unfamiliarity.

- **State the implementation in one line** — a counter keyed by client and window, self-expiring — to show why it is chosen.
- **Give the boundary example with times and numbers**, and say the worst case is exactly twice the limit.
- **Judge whether that matters** by asking what the backend can absorb, rather than dismissing the algorithm outright.
- **Mention synchronised resets** across clients, which multiplies the boundary spike fleet-wide.
- **Offer the sliding window counter as a cheap upgrade**, since it removes the flaw for one additional integer.

**Practice drill**

Implement a limit of 100 requests per minute using fixed windows, writing out the key structure and the operations per request. Construct the worst-case boundary scenario with explicit timestamps and say how many requests reach the backend and over what interval. Then decide whether to keep it for an endpoint that can absorb 300 requests per second, and describe the smallest change that would remove the flaw.

**Go deeper**

A fixed window rate limiter counts requests per aligned time window and rejects once the count exceeds the limit.

**It is the minimum viable limiter, and that is its entire argument.** One integer per client per window, two operations per request, and keys that expire themselves with no cleanup — nothing else provides enforcement this cheaply. At very high cardinality, where per-key memory across millions of clients is the binding constraint, that economy can be decisive in a way it is not at smaller scale.

**Its flaw is precise and bounded, which makes it possible to reason about.** Because the counter resets instantly at the boundary, a client can spend a full quota immediately before and another immediately after, delivering twice the limit in an arbitrarily short span. The worst case is exactly double, never more, so the question becomes concrete: can the protected system absorb twice the configured rate briefly? If it can, the flaw is theoretical.

**Aligned windows compound the problem across clients.** When every window begins on the same second, every throttled client resumes simultaneously and the boundary spike is multiplied by the number of clients waiting. Offsetting each client's window by a hash of their identifier spreads the resets across the window and removes the synchronised surge at no cost, and it is omitted far more often than it should be.

**Approximate counting is unsuitable wherever the number has consequences.** Billing against a quota that can be exceeded by half is indefensible, and accounting locally with a fixed window against a provider enforcing a sliding one produces records that show compliance at the moments they are rejecting your traffic. Approximation is acceptable for bounding load and preventing abuse, and not for anything where the count is the product.

**The upgrade is cheap enough to narrow its advantage considerably.** A sliding window counter keeps one extra integer and weights the previous window by how much of it remains in view, which removes nearly all of the boundary error for one more value and one more arithmetic step. Once the doubling matters at all, that trade is almost always worth making — which confines the fixed window to cases where the flaw is genuinely irrelevant.

**Choosing it should be a decision, not a default.** It is frequently what a gateway implements out of the box, so systems end up with it without anyone having considered the boundary behaviour or the synchronised resets. Knowing that both exist, and having concluded that the protected system tolerates them, is the difference between an appropriate choice and an accidental one that will be discovered during a traffic spike.

**Related patterns:** Sliding Window Rate Limit · Token Bucket · Leaky Bucket · Load Shedding

---

### Sliding Window Rate Limit

*Evaluate the limit over a window that moves with the present moment, removing the boundary burst that fixed windows permit.*

> **When you hear…** Fixed-window boundary bursts causing real load · limits that must be accurate at every instant · third-party limits enforced this way

**Flow:** `Window ends now` → `Count recent requests` → `Old ones age out` → `No reset boundary` → `Smooth enforcement`

**The problem**

A fixed window lets a client send its whole quota just before the reset and again just after, producing twice the limit in a fraction of a second. For a backend that cannot absorb that spike, the limiter is not providing the bound it claims.

The underlying reason is that a fixed window asks the wrong question. It asks how many requests occurred since an arbitrary aligned instant, when the meaningful question is how many occurred in the last minute counted from now.

> **Let the window end at the present moment, not at a clock boundary**  
> A window anchored to now has no reset, so there is no instant at which allowance suddenly reappears. Requests age out continuously as they pass beyond the window's trailing edge, which means the limit holds over every possible interval of that length rather than only over the aligned ones — and the boundary burst becomes impossible by construction.

**Mental model**

Two implementations of one idea: an exact log of recent request times, or an approximation that weights the previous fixed window by how much of it remains in view.

1. **Anchor** — The window always ends at the current instant.
2. **Include** — Requests within the trailing window count toward the limit.
3. **Expire** — Requests older than the window stop counting, continuously.
4. **Decide** — The count over that moving window determines allow or reject.
5. **Approximate** — Where exactness is too costly, weight the previous window instead.

> **The exact version stores one entry per request**  
> Keeping a timestamp for every request gives perfect accuracy and memory proportional to the limit multiplied by the number of clients. A limit of a thousand per minute across a million clients is a billion timestamps — accuracy that is unaffordable at scale, which is why the weighted approximation exists and why it is what most systems actually run.

**How it works**

**Exact log and weighted counter**

```text
SLIDING WINDOW LOG  (exact)
  store a sorted set of request timestamps per client
  on request:
    remove entries older than now - window
    count remaining
    if count < limit: add now, ALLOW
    else: REJECT
  + exact at every instant
  - memory = limit x clients
  - several operations per request

SLIDING WINDOW COUNTER  (approximate, common)
  keep counts for the current and previous fixed
  windows
  weight the previous one by how much of it is still
  within view:

    now is 30% into the current minute
    estimate = current + previous x 0.7

    previous minute: 90 requests
    current minute:  20 requests
    estimate = 20 + 90 x 0.7 = 83
    limit 100 -> ALLOW

  + two counters per client
  + error is small and bounded
  - assumes requests were evenly spread in the
    previous window

THE ERROR
  if the previous window's requests were all at its
  very start, the estimate overcounts
  if all at its end, it undercounts
  -> in practice within a few percent
  -> vastly better than 2x
```

1. **Use the weighted counter unless exactness is required** — Two integers per client instead of one entry per request.
2. **Reserve the exact log for low-cardinality, high-value limits** — Billing and hard external caps justify the memory.
3. **Understand the approximation's assumption** — It presumes even distribution within the previous window.
4. **Match the algorithm a third party uses** — Accounting differently from the enforcer produces surprises.
5. **Return a precise retry time** — The sliding window can compute exactly when capacity returns.
6. **Keep the state shared across instances** — Per-instance windows multiply the limit as with any limiter.

**Cost comparison at scale**

```text
1,000,000 CLIENTS, limit 1,000/minute

SLIDING WINDOW LOG
  worst case 1,000 timestamps x 1,000,000 clients
  = 1,000,000,000 entries
  at ~16 B with overhead -> tens of GB
  -> and several sorted-set operations per request

SLIDING WINDOW COUNTER
  2 integers x 1,000,000 clients
  = 2,000,000 values, a few hundred MB
  -> two reads and one write per request

FIXED WINDOW
  1 integer x 1,000,000 = 1,000,000 values
  -> cheapest, with the 2x flaw

-> the counter is close to the fixed window in cost
   and close to the log in accuracy
-> which is why it is the usual production choice

WHEN THE LOG EARNS ITS COST
  few clients, expensive requests
  billing where every unit is charged
  regulatory or contractual exactness
```

| Metric | Value | Note |
|---|---|---|
| Accuracy | exact or near | **no 2× burst** |
| Log cost | entry per request | tens of GB |
| Counter cost | two integers | few hundred MB |
| Usual choice | weighted counter | best trade |

> **The weighted estimate assumes the previous window was evenly distributed**  
> If a client sent all of the previous window's requests in its final second, the weighting underestimates their recent activity and allows more than intended; if they arrived at the very start, it overestimates and throttles too early. The error is bounded and small in practice, but it exists — and describing the counter as a sliding window without mentioning that it is an approximation misrepresents what it guarantees.

**Technologies**

| Approach | Implementation | Note |
|---|---|---|
| Sliding window log | Redis sorted set of timestamps | Exact; memory proportional to the limit |
| Sliding window counter | Two Redis counters plus weighting | The common production choice |
| Gateway limiters | Often sliding by default | Check which variant is in use |
| Token bucket | Continuous refill | Also has no boundary flaw; allows bursts by design |
| Fixed window | Single counter | Cheapest; 2× burst at boundaries |
| Cloud API limits | Usually sliding | Match their model in your own accounting |

The token bucket row is worth pausing on: it also avoids the boundary problem, but it deliberately permits bursts up to its capacity, whereas a sliding window enforces the limit uniformly over every interval. Which behaviour is wanted depends on whether client bursts are welcome or hazardous.

**Trade-offs**

**Window strategies**

| Approach | Accuracy | Memory | Burst behaviour |
|---|---|---|---|
| Fixed window | Poor at boundaries | One counter | Up to 2× the limit |
| Sliding window log | Exact | One entry per request | None |
| Sliding window counter | Within a few percent | Two counters | Very small |
| Token bucket | Exact over the long run | Two values | Up to capacity, by design |
| Leaky bucket | Exact output rate | A queue | None; smoothed |

Between the sliding counter and the token bucket the question is intent rather than accuracy: the sliding window says no interval of this length may exceed the limit, while the token bucket says a client may save allowance and spend it in a burst. Both are defensible; they encode different policies.

> **Ask before choosing it**  
> How is the limit you are mirroring actually enforced? When an upstream provider uses a sliding window, accounting locally with a fixed one guarantees that your records and their enforcement disagree — and theirs is the one that rejects your traffic.

**How it fails**

**How sliding window limiting goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Memory exhausted | Exact log at high cardinality | Use the weighted counter |
| Latency added to every request | Several sorted-set operations per call | Counter variant, or limit at the gateway |
| Slightly over the limit | Uneven distribution in the previous window | Accept it, or use the exact log |
| Limit multiplied by instance count | Per-instance state | Shared store for window state |
| Clients retry too early | Generic retry advice | Compute the exact time capacity returns |
| Disagreement with an upstream limiter | Different algorithm from the provider | Match theirs, or keep a margin |
| Old entries never removed | Log not pruned on access | Prune before counting, every time |

> **Exact sliding windows scale with traffic, not with clients**  
> The log holds an entry per request within the window, so its memory grows with how much traffic the system receives rather than with how many clients exist. A busy period therefore increases the limiter's memory consumption at precisely the moment the system is under load — which is a failure mode that does not exist in any counter-based algorithm and is the practical reason the exact variant is rarely deployed at scale.

**Where it is used**

- **Cloud provider API limits**, which are commonly enforced with sliding windows.
- **Payment and financial APIs**, where exactness justifies the cost of a log.
- **Gateway rate limiting**, where the counter variant is a standard offering.
- **Abuse prevention at high cardinality**, using the weighted counter for millions of keys.
- **Any system mirroring a third party's limit**, where matching their algorithm avoids disagreement.

**In the interview**

Present both variants and choose between them with numbers; naming only one suggests a partial picture.

- **Explain why the window must end at now** — there is no reset, so no instant when allowance reappears.
- **Give both implementations** and their costs, then choose the weighted counter for scale.
- **Work the weighting arithmetic aloud** with a concrete example; it is short and demonstrates real familiarity.
- **State the approximation's assumption** about even distribution, and that the error is a few percent rather than 2×.
- **Contrast with the token bucket** on policy: uniform enforcement against deliberate burst allowance.

**Practice drill**

Enforce 100 requests per minute with a sliding window. Write out the weighted counter calculation for a client 30 seconds into the current window with 90 requests in the previous one and 20 so far. Then compute the memory for the exact log and for the counter across a million clients, and choose one. Finally, construct the traffic pattern that produces the largest error in the approximation, and say how far off it is.

**Go deeper**

A sliding window rate limiter evaluates the limit over an interval ending at the present moment, eliminating the reset boundary that fixed windows create.

**Moving the window's end to now changes what the limit means.** A fixed window bounds requests since an arbitrary aligned instant; a sliding window bounds requests over every interval of that length. Because allowance is never restored all at once, the doubling that fixed windows permit becomes impossible, and the limit describes the traffic the backend actually experiences rather than an accounting artefact.

**The exact implementation is correct and rarely affordable.** Storing a timestamp per request gives perfect enforcement at every instant, at a memory cost proportional to the limit times the client count — and, importantly, proportional to traffic rather than to clients, so consumption rises exactly when the system is busiest. That combination confines it to low-cardinality, high-value limits where every unit matters.

**The weighted counter captures most of the benefit for almost none of the cost.** Keeping the current and previous window counts and weighting the previous by how much of it remains in view costs two integers per client and one extra arithmetic step. The error is a few percent rather than a factor of two, which is close enough to exact for every purpose except billing — and that is why it, rather than either pure form, is what production systems typically run.

**The approximation makes an assumption worth stating.** Weighting presumes the previous window's requests were spread evenly through it, so a client that concentrated them at one end is measured slightly wrongly in one direction or the other. Describing the counter as a sliding window without that caveat overstates the guarantee, and knowing which direction the error runs under a given traffic shape is what lets an operator judge whether it matters.

**Matching the enforcing party's algorithm matters more than the algorithm's abstract quality.** When a provider enforces a sliding window and a client accounts with a fixed one, the two views disagree precisely at the boundaries, and the client's records will show compliance while their traffic is being rejected. Local accounting is only useful if it models the same thing the enforcer models, or keeps a margin wide enough that the difference cannot surface.

**It encodes a different policy from the token bucket, and the choice should follow intent.** Both avoid boundary bursts, but a sliding window enforces the limit uniformly across every interval, while a token bucket deliberately lets an idle client accumulate allowance and spend it at once. A backend that cannot survive bursts wants the first; an API designed to accommodate clients that batch and page wants the second — and picking on accuracy grounds alone misses that they are answering different questions.

**Related patterns:** Fixed Window Rate Limit · Token Bucket · Leaky Bucket · Load Shedding

---

### Backpressure

*Signal upstream to slow down when a system cannot keep up, so demand is reduced at the source instead of accumulating in queues.*

> **When you hear…** Queues growing without bound · memory pressure from buffered work · a producer faster than its consumer

**Flow:** `Consumer saturated` → `Signal upstream` → `Producer slows` → `Queues stay bounded` → `System stays stable`

**The problem**

A producer generates work faster than the consumer can process it. The difference accumulates in a buffer, and if nothing intervenes the buffer grows until memory is exhausted — at which point the process dies and everything buffered is lost.

Adding a bigger buffer postpones this and makes it worse. Latency grows with queue depth, so by the time the buffer is large enough to matter, every item in it has been waiting longer than anyone is prepared to accept, and much of it is work whose requester has already given up.

> **Storing excess demand is not the same as handling it**  
> A queue converts a rate mismatch into a growing backlog, not into extra capacity. The only stable responses are to increase throughput or reduce arrivals, and where the producer can be influenced, reducing arrivals is available immediately. Backpressure is the mechanism that carries that information upstream, so the mismatch is resolved at the source rather than stored in the middle.

**Mental model**

A feedback signal travelling opposite to the data flow. The consumer communicates its capacity, and the producer's rate is bounded by what has been granted.

1. **Observe** — The consumer tracks its own saturation — queue depth, latency, or outstanding work.
2. **Signal** — That state is communicated upstream, explicitly or by blocking.
3. **Slow** — The producer reduces its rate or stops.
4. **Stabilise** — Arrivals match capacity and the queue stops growing.
5. **Resume** — Capacity is granted again as the consumer catches up.

> **Backpressure propagates until it reaches something that cannot be slowed**  
> Slowing a consumer slows its producer, which slows that producer's producer, all the way to the origin. If the origin is an internal job it simply waits, which is the desired outcome; if it is a user clicking a button, the pressure surfaces as a slow application, and if it is an external system that cannot be slowed, the chain ends in rejection. Knowing where the chain terminates is what decides whether backpressure is the right mechanism at all.

**How it works**

**Mechanisms, from explicit to implicit**

```text
EXPLICIT CREDIT  (strongest)
  consumer grants the producer N items of credit
  producer sends at most N, waits for more
  -> arrivals can never exceed granted capacity
  -> used by TCP windows, HTTP/2 flow control,
     reactive streams

BLOCKING
  producer calls a bounded queue that blocks when
  full
  -> simple, effective in-process
  -> the producer's thread is held, which may be
     unacceptable

REJECTION AS A SIGNAL
  return 429 or 503 with a retry hint
  -> works across service boundaries where blocking
     does not
  -> depends on clients honouring it

PULL INSTEAD OF PUSH
  the consumer fetches when ready, rather than being
  pushed to
  -> backpressure is inherent; nothing arrives
     unbidden
  -> message queue consumers work this way

LAG AS A SIGNAL
  consumer lag rising -> reduce production
  -> indirect but works across systems that share
     nothing else

THE BEST OPTION IS USUALLY PULL
  it requires no protocol, no signalling and no
  cooperation: the consumer simply does not ask for
  more than it can handle
```

1. **Prefer pull-based consumption** — It provides backpressure structurally, with nothing to implement or honour.
2. **Bound every buffer** — An unbounded queue is backpressure that was designed out of the system.
3. **Signal before saturation, not at it** — By the time the buffer is full, latency is already unacceptable.
4. **Know where the chain terminates** — Pressure reaching a user is a slow application; reaching an uncontrollable source means rejection.
5. **Never block a request thread on a backpressured path** — Held threads convert one slow dependency into total exhaustion.
6. **Shed load where backpressure cannot reach** — External producers must be refused rather than slowed.

**Why buffering is not a solution**

```text
PRODUCER 1,000/s   CONSUMER 800/s

WITH A LARGE BUFFER
  surplus 200/s accumulates
  after 60 s   12,000 queued -> 15 s of latency
  after 300 s  60,000 queued -> 75 s of latency
  -> every request has timed out long before it is
     processed
  -> the system is doing work for nobody
  -> eventually memory is exhausted and everything
     buffered is lost

WITH BACKPRESSURE
  buffer reaches its bound
  producer is slowed to 800/s
  -> latency stays at the buffer's depth, constant
  -> throughput is 800/s either way
  -> but now the surplus is visible at the source,
     where it can be handled deliberately

THE POINT
  buffering NEVER increases throughput
  it only decides where the surplus waits and how
  long each item sits before being served

SURPLUS MUST GO SOMEWHERE
  slow the producer     (backpressure)
  reject the excess     (load shedding)
  add capacity          (scaling)
  -> there is no fourth option; storing it is just
     deferring the choice
```

| Metric | Value | Note |
|---|---|---|
| Buffering | adds latency | not throughput |
| Backpressure | bounds latency | **at the source** |
| Best form | pull-based | structural |
| Terminates at | user or rejection | know which |

> **Blocking a request-handling thread to apply backpressure exhausts the pool**  
> Backpressure implemented by making the caller wait works within a process and fails badly across a service boundary, because the waiting caller is holding a thread that other requests need. One slow consumer then consumes the entire thread pool of everything upstream of it, which is how a local rate mismatch becomes a fleet-wide outage. Across service boundaries the signal has to be a response, not a delay.

**Technologies**

| Mechanism | Where | Note |
|---|---|---|
| TCP flow control | Network transport | The original credit-based scheme |
| HTTP/2 flow control | Per-stream windows | Backpressure inside a connection |
| Reactive streams | In-process pipelines | Explicit demand signalling |
| Bounded queues | Within a process | Blocking provides the signal |
| Message consumers | Pull-based by design | The consumer sets the pace |
| 429 and 503 responses | Across service boundaries | Requires clients to honour them |

Pull-based message consumption is worth emphasising as the practical default: a consumer that fetches only what it can process needs no protocol, no cooperation and no signalling, because it never receives work it did not ask for.

**Trade-offs**

**Responses to a rate mismatch**

| Response | Latency | Work lost | Requires |
|---|---|---|---|
| Unbounded buffering | Grows without limit | Everything, on crash | Nothing |
| Bounded buffer, then reject | Bounded | The excess | A bound |
| Backpressure | Bounded | None | A controllable producer |
| Load shedding | Bounded | The shed requests | A rejection policy |
| Scale the consumer | Bounded | None | Capacity and time |

Backpressure and load shedding are the two coherent answers and they apply in different circumstances: slow the producer where you can reach it, refuse the work where you cannot. Systems generally need both, because some producers are internal and some are the open internet.

> **Ask before choosing it**  
> Can the producer actually be slowed, and what happens when it is? Backpressure applied to an internal pipeline is ideal; applied to user-facing traffic it manifests as an unresponsive product, and applied to an external system that ignores the signal it does nothing at all.

**How it fails**

**How backpressure goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Memory exhausted | Unbounded buffer with no signal | Bound every queue |
| Thread pools exhausted upstream | Blocking across a service boundary | Respond with rejection instead of waiting |
| Pressure reaches users as unresponsiveness | Chain terminates at the request path | Shed load at the edge instead |
| Signal ignored | External clients not honouring 429 | Enforce with rate limits, not only signals |
| Oscillation between full speed and stopped | On/off signalling with no gradation | Gradual rate adjustment with hysteresis |
| Latency already unacceptable when the signal fires | Threshold set at full saturation | Signal earlier, at partial saturation |
| Work processed after clients gave up | No expiry on queued items | Drop entries older than the client timeout |

> **An unbounded queue is a decision to fail later and worse**  
> Every buffer without a limit is a promise that the surplus will be stored indefinitely, which ends in memory exhaustion and the loss of everything accumulated. Before that, latency has already grown past every client's timeout, so the system spends its final minutes doing work whose results nobody will receive. Bounding the queue converts that into a visible, survivable rejection at the moment the mismatch begins.

**Where it is used**

- **TCP**, whose receive window is credit-based backpressure and the model everything else borrows from.
- **Stream processing frameworks**, propagating demand upstream through the topology.
- **Message queue consumers**, which pull at their own pace and therefore have backpressure inherently.
- **Reactive libraries**, where explicit demand signalling is the core abstraction.
- **Ingestion pipelines**, slowing collection when downstream storage cannot keep up.

**In the interview**

The sentence worth saying is that buffering never adds throughput; everything else follows from it.

- **State that surplus demand must be slowed, rejected or absorbed by capacity** — storing it only defers the choice.
- **Do the arithmetic** of a 200-per-second mismatch to show how quickly latency exceeds every timeout.
- **Prefer pull-based consumption**, since it provides backpressure with nothing to implement or honour.
- **Warn against blocking across service boundaries**, which converts one slow consumer into upstream thread exhaustion.
- **Trace where the chain terminates** and say what happens there, because that determines whether backpressure or shedding is appropriate.

**Practice drill**

A pipeline stage produces 1,000 items per second and the next stage processes 800. Compute the queue depth and added latency after one minute and after five, assuming an unbounded buffer. Then design backpressure for it: the mechanism, the threshold and the signal. Say what the producer does when slowed, and follow the chain upstream to identify where the pressure ultimately lands and what a user would observe.

**Go deeper**

Backpressure is a signal travelling against the flow of data, telling a producer to slow down when the consumer cannot keep up.

**It rests on the fact that buffering never creates capacity.** A queue between a fast producer and a slow consumer stores the difference, and the difference accumulates without limit until memory runs out. Throughput is identical with or without the buffer; all the buffer decides is how long each item waits and how much is lost when the process finally fails. Recognising this makes the response set small and clear: slow the producer, reject the excess, or add capacity.

**Latency degrades long before memory does.** A surplus of a couple of hundred items per second produces a queue deep enough to exceed every client timeout within a minute or two, so the system spends the remainder of the incident processing work nobody is waiting for any more. The failure is therefore not a sudden crash but a long period of expensive uselessness, which is why bounded buffers with early signalling are better than large ones.

**Pull-based consumption provides the mechanism structurally.** A consumer that fetches work when it is ready never receives more than it asked for, so no protocol, signal or cooperation is required — the absence of a request is the backpressure. Where a pipeline can be arranged this way, it is strictly better than any push-based design with a signalling layer bolted on, because there is nothing to implement incorrectly and nothing for a producer to ignore.

**Across service boundaries the signal must be a response, not a delay.** Applying backpressure by making the caller wait works inside a process and is destructive between services, because the waiting caller holds a thread that other work needs. One saturated consumer then drains the thread pools of everything upstream, turning a local rate mismatch into a cascading outage — which is why rejection with a retry hint, rather than a slow response, is the correct cross-service form.

**Pressure propagates until it reaches something that cannot be slowed, and that endpoint determines the design.** An internal batch job simply waits, which is exactly right. A user pressing a button experiences an unresponsive product, which is usually worse than a clear refusal. An external system that ignores the signal absorbs nothing at all. Tracing the chain to its origin before choosing backpressure is what reveals whether it will work or whether load shedding is the mechanism actually needed.

**Signalling should begin before saturation and change gradually.** A threshold set at a full buffer fires when latency is already unacceptable, and an on/off signal produces oscillation between full speed and a complete stop. Signalling at partial saturation, and adjusting the rate rather than toggling it, keeps the system near its sustainable throughput instead of swinging around it — which is the same reason congestion control algorithms adjust continuously rather than in steps.

**Related patterns:** Load Shedding · Leaky Bucket · Work Queue · Bulkhead

---

### Load Shedding

*Deliberately reject a portion of traffic when overloaded, so the remainder is served properly instead of everything failing together.*

> **When you hear…** Demand exceeding capacity with no way to slow the source · degradation affecting all requests · protecting a service from collapse

**Flow:** `Load exceeds capacity` → `Classify requests` → `Reject low priority` → `Serve the rest well` → `Recover`

**The problem**

Demand reaches twice what a service can handle. Without intervention every request is accepted, each one takes twice as long, queues grow, timeouts fire, and clients retry — which adds more load. The system ends up serving almost nothing while consuming all of its capacity.

Attempting to serve everyone is what produces the total failure. Capacity is finite, and admitting work beyond it does not create more; it simply degrades every request equally until none of them completes usefully.

> **Refusing some work is how the rest gets served**  
> If a service admits only what it can handle and rejects the remainder immediately, the admitted requests receive normal service and the rejected ones receive a fast, honest error they can act on. Total useful throughput is higher than in the case where everything is accepted and everything times out — deliberate partial failure is strictly better than accidental total failure.

**Mental model**

An admission decision at the entrance. When a saturation signal exceeds a threshold, a portion of incoming work is refused before any resources are committed to it.

1. **Measure** — Track a signal that reflects real saturation — latency, queue depth, concurrency.
2. **Decide** — Determine how much must be rejected to stay within capacity.
3. **Classify** — Rank requests by importance so the right ones are dropped.
4. **Reject** — Refuse immediately, before doing any work.
5. **Recover** — Admit more as the signal improves, gradually.

> **Rejecting must be far cheaper than serving, or shedding cannot help**  
> If a refusal costs a database lookup to check a quota, a policy evaluation and a log write, then at twice capacity the rejections alone can consume everything the service has. The decision must happen at the edge, on information already available in memory, and cost orders of magnitude less than processing — otherwise the protective mechanism becomes part of the overload.

**How it works**

**Choosing the signal and what to drop**

```text
SIGNALS, WORST TO BEST
  CPU utilisation     lags; already saturated when
                      it looks high
  request rate        says nothing about cost per
                      request
  queue depth         better; shows accumulation
  concurrency         good; work in flight now
  LATENCY vs target   best; directly measures whether
                      the service is meeting its
                      obligations

PRIORITY CLASSES
  1  payments, checkout       never shed
  2  logged-in interactive    shed last
  3  anonymous browsing       shed next
  4  prefetch, analytics,
     background refresh       shed first
  -> at 2x load, dropping classes 3 and 4 may fully
     restore classes 1 and 2

HOW MUCH TO SHED
  target: keep latency at the objective
  if p99 > target: increase the shed fraction
  if p99 < target: decrease it
  -> a control loop, not a fixed threshold

COST OF REJECTION
  must be microseconds
  -> in-memory decision at the edge
  -> no database, no downstream call, no per-request
     policy fetch
```

1. **Shed at the edge, before resources are committed** — Rejecting after a service has begun work wastes the capacity shedding was meant to protect.
2. **Use latency against the objective as the signal** — Utilisation lags; latency states directly whether obligations are being met.
3. **Classify requests by importance in advance** — Undifferentiated shedding drops checkout as readily as a prefetch.
4. **Adjust the shed fraction continuously** — A fixed threshold either over-rejects or under-protects as load moves.
5. **Return a retry hint with the rejection** — Rejections that trigger immediate retries add to the load they relieve.
6. **Recover gradually** — Admitting everything the moment the signal improves re-triggers the overload.

**What shedding buys**

```text
CAPACITY 1,000/s     DEMAND 2,000/s

NO SHEDDING
  all 2,000 accepted
  each takes ~2x as long, queues grow
  clients time out at 1 s and retry
  retries add load -> effective demand 3,000/s
  -> useful throughput approaches ZERO
  -> everyone fails

WITH SHEDDING
  admit 1,000/s, reject 1,000/s immediately
  admitted: normal latency, all succeed
  rejected: fast 503 with Retry-After
  -> useful throughput 1,000/s
  -> 50% of users served perfectly, 50% told to wait

WITH PRIORITY SHEDDING
  classes 1 and 2 total 900/s -> all admitted
  classes 3 and 4 total 1,100/s -> 900 rejected
  -> every checkout succeeds
  -> analytics and prefetch absorb the loss
  -> the business impact is far smaller than 50%

THE RETRY AMPLIFIER
  rejections without a retry hint produce immediate
  retries
  -> shedding 1,000/s creates 1,000/s of retries
  -> always return Retry-After, and honour it
```

| Metric | Value | Note |
|---|---|---|
| No shedding | ≈0 useful | **everyone fails** |
| Flat shedding | 50% served | randomly chosen |
| Priority shedding | all critical served | loss absorbed elsewhere |
| Rejection cost | microseconds | or it fails too |

> **Rejections without a retry hint manufacture their own load**  
> A client refused with no guidance retries immediately, so shedding a thousand requests per second creates a thousand retries per second — the mechanism relieving the overload is simultaneously feeding it. Returning a concrete wait time, and having clients honour it with jitter, is what makes the rejection actually reduce demand rather than merely redistributing it in time.

**Technologies**

| Layer | Mechanism | Note |
|---|---|---|
| Load balancers | Queue limits and rejection | Cheapest place; never reaches the application |
| Service meshes | Adaptive concurrency limits | Applied without application changes |
| Application admission control | Priority-aware shedding | The only layer that knows request importance |
| CDN and edge | Reject before origin | Stops traffic furthest from the backend |
| Rate limiters | Per-client bounds | Prevents one client causing the overload |
| Backpressure | Slow the producer | Better where the producer is controllable |

Adaptive concurrency limiting is the most practical modern form: rather than configuring a fixed threshold, the system continuously probes for the concurrency at which latency begins to degrade and admits up to that level, which tracks real capacity as it changes with deploys and dependencies.

**Trade-offs**

**Overload responses**

| Response | Useful throughput | Who suffers | Complexity |
|---|---|---|---|
| Accept everything | Approaches zero | Everyone | None |
| Random shedding | Near capacity | A random half | Low |
| Priority shedding | Near capacity | Low-value traffic | Classification needed |
| Backpressure | Near capacity | The producer waits | Controllable producer |
| Autoscaling | Eventually higher | Everyone, during the delay | Provisioning time |
| Queue everything | Approaches zero | Everyone, slowly | Low |

Autoscaling belongs in the table but not as a substitute: new capacity takes minutes to arrive and a spike does damage in seconds, so shedding is what keeps the service alive during the interval in which scaling is still happening.

> **Ask before choosing it**  
> Which requests can be dropped with the least harm? If every request is equally important the mechanism degrades to random rejection, which is still far better than total failure but leaves most of its value unclaimed — and the classification is usually a product conversation rather than a technical one.

**How it fails**

**How load shedding goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Shedding consumes the remaining capacity | Expensive rejection path | Decide in memory at the edge |
| Critical requests dropped | No prioritisation | Classify traffic in advance |
| Rejections cause more load | No retry hint, clients retry instantly | Return Retry-After; require jitter |
| Overload not detected until collapse | Utilisation used as the signal | Use latency against the objective |
| Oscillation between shedding and overload | Immediate full recovery | Ramp admission gradually |
| Wasted work | Requests rejected after processing started | Shed at admission, not mid-request |
| Retry storms after recovery | All clients returning at once | Jittered retry advice |

> **Accepting everything during overload serves nobody at all**  
> Admitting twice the capacity does not halve the service quality — it collapses it, because queues grow, every request exceeds its timeout, clients retry, and the added load compounds until useful throughput approaches zero while the machines run at full utilisation. Deliberately refusing half the traffic feels worse and produces a dramatically better outcome, which is why shedding has to be designed in before the incident rather than argued about during it.

**Where it is used**

- **Large-scale web services**, which shed at the edge as standard practice during traffic spikes.
- **Service meshes with adaptive concurrency limits**, discovering the admission threshold automatically.
- **Payment and checkout paths**, protected by shedding everything of lower priority first.
- **Search and recommendation backends**, dropping optional enrichment when saturated.
- **Electrical grids**, the origin of the term and the same reasoning applied to a physical network.

**In the interview**

Make the throughput argument explicitly; it converts an uncomfortable idea into an obviously correct one.

- **Show that accepting everything yields near-zero useful throughput**, with the retry amplification included.
- **Choose latency against the objective as the signal** and explain why utilisation is too slow to be useful.
- **Introduce priority classes** with concrete examples, since that is where most of the value comes from.
- **Insist the rejection path is cheap**, because an expensive refusal at twice capacity is part of the overload.
- **Return a retry hint and recover gradually**, otherwise the mechanism generates the load it is relieving.

**Practice drill**

A service handles 1,000 requests per second and receives 2,000. Compute what happens with no shedding, including the effect of client retries after a one-second timeout. Then design shedding: the signal, the threshold, the priority classes and the rejection response. State the useful throughput your design achieves and who is refused. Finally, describe the recovery behaviour and why admitting everything as soon as latency improves would be wrong.

**Go deeper**

Load shedding rejects a portion of incoming work during overload so that the remainder receives proper service.

**Its justification is arithmetic rather than philosophical.** Admitting more work than capacity does not distribute the shortfall evenly; it collapses throughput, because queues grow past every timeout, clients retry, and the retries add load. Refusing the excess immediately keeps admitted requests within their latency objective and produces far more completed work than the alternative — deliberate partial failure genuinely beats accidental total failure.

**The choice of signal decides whether shedding engages in time.** Utilisation lags badly: by the time it looks high the queues are already deep, and it says nothing about whether the service is meeting its obligations. Latency measured against the objective states directly whether the system is keeping its promises, and concurrency or queue depth are reasonable proxies. Feeding that signal into a continuous control loop, rather than a fixed threshold, keeps admission tracking real capacity as it changes.

**Prioritisation is where most of the value lies.** Random shedding at twice capacity serves half of everyone; priority shedding can serve every checkout and every logged-in session while dropping prefetches, analytics and background refreshes that nobody will notice. The classification is a product decision about what matters, and doing it in advance is what converts a blunt survival mechanism into one whose business impact is small.

**The rejection path must be dramatically cheaper than the serving path.** A refusal requiring a database lookup, a policy fetch or a downstream call costs a meaningful fraction of a real request, and at twice capacity the rejections alone can saturate the service. Deciding in memory, at the edge, before any resources are committed, is what makes the mechanism protective — and shedding after work has already begun wastes precisely the capacity it was protecting.

**Rejections without guidance recreate the load they relieve.** A client refused with no retry information returns immediately, so shed traffic reappears within milliseconds and the system is refusing the same requests repeatedly. Returning a concrete wait time, and clients honouring it with jitter, is what turns a rejection into an actual reduction in demand rather than a redistribution of it across the next second.

**Recovery has to be gradual or the system oscillates.** Admitting everything the instant latency improves sends the full suppressed demand back at once, which re-triggers the overload and produces a cycle of collapse and recovery. Ramping admission upward while watching the signal keeps the system near its sustainable point, and it is the same reasoning that makes congestion control increase slowly and back off sharply — the asymmetry exists because overload is much more expensive than under-utilisation.

**Related patterns:** Backpressure · Circuit Breaker · Token Bucket · Bulkhead

---

## Service Architecture

### API Gateway

*Put one entry point in front of many services to handle authentication, routing, limiting and observability once instead of everywhere.*

> **When you hear…** Many services exposed directly to clients · cross-cutting concerns duplicated in each · clients coupled to internal topology

**Flow:** `Clients call one host` → `Gateway authenticates` → `Applies limits` → `Routes to services` → `Services stay simple`

**The problem**

Forty services are exposed directly to clients. Each implements its own authentication, rate limiting, request logging and CORS handling, and each implements them slightly differently — so a security fix must be applied forty times and verified forty times.

Clients are also bound to the internal structure. Splitting a service, renaming it or moving an endpoint requires every client, including mobile applications that take weeks to update, to change with it.

> **Concerns that are identical everywhere belong in one place**  
> Authentication, rate limiting, request logging and TLS termination do not vary by service; only their configuration does. Handling them once at a single entry point removes forty implementations and forty opportunities to get it wrong, and simultaneously decouples clients from internal topology — because the address they call is the gateway rather than whichever service currently answers.

**Mental model**

A reverse proxy with policy. Every external request passes through it, is validated and shaped, and is then forwarded to whichever internal service handles it.

1. **Terminate** — TLS ends here; internal traffic is simpler.
2. **Authenticate** — Identity is established once and passed onward as a trusted claim.
3. **Enforce** — Rate limits, quotas and request validation are applied before any service is involved.
4. **Route** — The request is directed to the appropriate service.
5. **Observe** — Every request is logged and traced from one consistent place.

> **The gateway is in the path of every request, so its failure is total**  
> A component every request must traverse is by definition a single point of failure, and a deployment mistake, a bad configuration or a memory leak there takes down services that are themselves perfectly healthy. It must be redundant, deployed with more care than the services behind it, and kept simple enough that changes to it are low risk.

**How it works**

**What belongs in the gateway and what does not**

```text
BELONGS
  TLS termination
  authentication (verify the token, reject invalid)
  rate limiting and quotas
  request and response logging, tracing headers
  routing by path or host
  request size limits, timeouts
  CORS, compression
  -> identical for every service, configuration only

DOES NOT BELONG
  business logic of any kind
  response transformation specific to one service
  authorisation decisions needing domain knowledge
  data aggregation across services
  -> these make the gateway a shared service that
     every team must change, and it becomes the
     bottleneck for all delivery

THE TEST
  if adding a feature to a service requires changing
  the gateway, the boundary is wrong

AUTHENTICATION VERSUS AUTHORISATION
  gateway: is this a valid token? who is it?
  service: may this user perform this action on this
           resource?
  -> the second needs domain knowledge the gateway
     must not have
```

1. **Keep cross-cutting concerns only** — Business logic in the gateway turns it into a shared bottleneck for every team.
2. **Authenticate at the gateway, authorise in the service** — Identity is generic; permission decisions need domain context.
3. **Run it redundantly across zones** — Every request passes through it, so its availability is the system's availability.
4. **Pass a trusted identity downstream** — Services should not re-verify tokens on every internal call.
5. **Keep configuration declarative and reviewed** — Most gateway outages are caused by configuration, not code.
6. **Do not let it become a deployment bottleneck** — If every team must queue for gateway changes, it has absorbed too much.

**Operational shape**

```text
CAPACITY
  it handles 100% of traffic
  -> size it for peak, with headroom
  -> a single instance is never acceptable

LATENCY
  adds one network hop plus policy evaluation
  typical 1-5 ms
  -> negligible against most backend work
  -> becomes significant only for very fast endpoints

FAILURE MODES
  configuration error       most common by far
  certificate expiry        total outage, entirely
                            preventable
  connection pool exhausted downstream slowness
                            consuming gateway
                            resources
  -> bulkhead per upstream so one slow service does
     not consume the gateway

TRUST BOUNDARY
  internal services must not accept external traffic
  directly, or the gateway can be bypassed and every
  policy with it
  -> network policy, not just convention

WHAT IT IS NOT
  a service mesh: that handles service-to-service
  traffic; the gateway handles north-south traffic
  from outside
```

| Metric | Value | Note |
|---|---|---|
| Scope | cross-cutting | **not business logic** |
| Availability | must be redundant | total failure otherwise |
| Latency | 1-5 ms | usually negligible |
| Common outage | configuration | not code |

> **A gateway that accumulates business logic becomes every team's bottleneck**  
> Each small transformation added for one service is reasonable in isolation, and the accumulation turns the gateway into a shared codebase that every team must modify and every deployment must coordinate. Changes to it become high-risk because it is in the path of everything, so they queue and slow down — and the component introduced to decouple teams becomes the thing that couples them.

**Technologies**

| Option | Strength | Watch for |
|---|---|---|
| Managed cloud gateways | Operated, integrated, scalable | Provider-specific features and limits |
| Kong, Envoy, Traefik | Flexible, plugin ecosystems | You operate and scale them |
| NGINX | Ubiquitous, fast, simple | Policy features are more limited |
| Service mesh ingress | Consistent with internal routing | Couples ingress to mesh adoption |
| Backend for frontend | Per-client aggregation | A complement, not a replacement |
| No gateway | One less hop | Cross-cutting concerns duplicated everywhere |

The distinction between gateway and mesh is worth keeping clear: the gateway governs traffic entering from outside, while the mesh governs traffic between internal services. They overlap in mechanism and differ in purpose, and conflating them produces designs where internal calls traverse an external entry point unnecessarily.

**Trade-offs**

**Handling cross-cutting concerns**

| Approach | Duplication | Coupling | Failure impact |
|---|---|---|---|
| Each service implements its own | High | None | Isolated |
| Shared library in each service | Low | Version coupling | Isolated |
| API gateway | None | Clients to the gateway | Total if it fails |
| Service mesh sidecars | None | Platform coupling | Per-instance |
| Gateway plus mesh | None | Both | Split by direction |

Shared libraries are a reasonable middle ground for smaller systems and have one persistent weakness: upgrading them requires every service to redeploy, so a security fix propagates at the speed of the slowest team rather than at the speed of a configuration change.

> **Ask before choosing it**  
> How many services are exposed and how much genuinely varies between them? For three services with similar needs, a shared library is simpler and has no single point of failure; the gateway earns its place when the number of services and the number of policies both grow.

**How it fails**

**How API gateways go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Total outage from a configuration change | Untested routing or policy update | Staged rollout with automated validation |
| Everything down on a certificate expiry | Manual renewal | Automated rotation with alerting |
| Gateway becomes a delivery bottleneck | Business logic accumulating in it | Keep it to cross-cutting concerns |
| One slow service degrades all traffic | Shared connection pools | Bulkhead per upstream |
| Policies bypassed | Services reachable directly | Enforce with network policy |
| Latency on fast endpoints | Heavy policy evaluation per request | Simplify the hot path; cache decisions |
| Identity re-verified everywhere | Downstream services not trusting the gateway | Signed internal identity headers |

> **Services reachable without passing through the gateway make every policy optional**  
> If internal services accept traffic directly, then authentication, rate limiting and logging apply only to callers who happen to use the front door. An attacker who finds an internal address bypasses all of it, and so does any internal caller taking a shortcut. The gateway's guarantees are only as strong as the network policy that forces traffic through it — convention is not enforcement.

**Where it is used**

- **Public API platforms**, where quotas, keys and versioning are enforced centrally.
- **Microservice architectures**, presenting one coherent surface over many internal services.
- **Mobile backends**, where clients cannot be updated quickly and must be decoupled from internal changes.
- **Partner and third-party integrations**, needing distinct authentication and quota policies.
- **Migration and strangler patterns**, routing individual paths from a legacy system to new services.

**In the interview**

Define the boundary precisely; the interesting content is what the gateway must not do.

- **List the cross-cutting concerns** and say they are identical everywhere, which is the argument for centralising them.
- **Draw the authentication/authorisation line** — identity at the gateway, permission in the service — and explain why.
- **Name it as a single point of failure** and describe redundancy and staged configuration rollout as the mitigations.
- **Warn against business logic accumulating**, since that converts it into a shared bottleneck for every team.
- **Require network enforcement** so services cannot be reached directly and the policies bypassed.

**Practice drill**

Forty services are exposed directly to web and mobile clients, each implementing its own authentication and rate limiting. Design the gateway: what it handles, what stays in the services, and where the authorisation boundary sits. State how you make it highly available and how a configuration change is rolled out safely. Then identify two requests that would tempt you to add service-specific logic to it, and say what you would do instead.

**Go deeper**

An API gateway is a single entry point that applies cross-cutting concerns once for many services and decouples clients from internal structure.

**Its value comes from the concerns being genuinely identical.** Token verification, rate limiting, request logging and TLS termination do not vary by domain — only their configuration does — so implementing them once removes both the duplication and the divergence that accumulates when many teams implement the same thing separately. A security fix becomes one change rather than forty, applied at the speed of a deployment rather than the speed of the slowest team.

**The decoupling it provides is as valuable as the deduplication.** Clients address the gateway rather than individual services, so services can be split, merged, renamed or moved without any client changing. That matters most for clients that cannot be updated on demand — mobile applications in particular — where internal refactoring would otherwise be constrained by release cycles outside the team's control.

**Drawing the authorisation line correctly is what keeps it maintainable.** Establishing who the caller is requires no domain knowledge and belongs at the entrance; deciding whether that caller may perform a specific action on a specific resource requires knowing what those things mean. Pushing the second into the gateway forces domain logic into shared infrastructure, which is the first step towards the component becoming everyone's bottleneck.

**Accumulated business logic is the characteristic decay of this pattern.** Each small service-specific transformation is defensible individually, and together they turn the gateway into a codebase every team must change and every deployment must coordinate around. Because it sits in the path of all traffic, those changes are high-risk and slow — so the component adopted to decouple teams ends up serialising their delivery, and the test for whether a change belongs is whether a service feature should require touching it at all.

**It is a single point of failure, which is acceptable only if treated as one.** Redundancy across zones, staged configuration rollout with automated validation, automated certificate rotation and per-upstream bulkheads are not refinements but the minimum for a component whose failure is total. In practice the overwhelming majority of gateway outages come from configuration rather than code, which means the review and rollout process for configuration deserves the same rigour as for software.

**Its guarantees depend on traffic having no alternative route.** If internal services accept direct connections, every policy the gateway enforces becomes optional for anyone who knows an internal address, including internal callers taking shortcuts. Network policy that makes the gateway the only reachable path is what turns its rules into actual constraints, and a system relying on convention has documentation rather than enforcement.

**Related patterns:** Backend For Frontend · Service Discovery · Sidecar · Token Bucket

---

### Backend For Frontend

*Give each client type its own backend that shapes data specifically for it, so no single API has to compromise between very different consumers.*

> **When you hear…** One API serving clients with conflicting needs · mobile making many round trips · over-fetching on constrained connections

**Flow:** `Mobile BFF` → `Web BFF` → `Each shapes data` → `Same services behind` → `Clients get exact payloads`

**The problem**

A mobile application needs a compact payload containing exactly eight fields, assembled from four services. A web application on the same endpoint wants forty fields and does not mind the size. One shared API either sends everything, wasting bandwidth on the connection least able to afford it, or sends the minimum and forces the web client to make several more calls.

Mobile suffers most from the compromise. Each additional round trip on a mobile network costs hundreds of milliseconds, so an API shaped for a browser turns one screen into six sequential requests.

> **Let the client's constraints shape its own backend**  
> The conflict exists because one interface is serving consumers with genuinely different requirements — payload size, round-trip cost, update cadence, and what the screen actually displays. Giving each client type a backend owned by the team that builds that client removes the negotiation entirely: each API can be exactly what its consumer needs, because it has only one consumer.

**Mental model**

A thin, client-specific layer that calls the same downstream services and assembles responses tailored to one client's screens.

1. **Own** — The client team owns its backend and can change it freely.
2. **Call** — It fans out to the same shared services everyone uses.
3. **Shape** — Responses match what the client actually renders.
4. **Aggregate** — Several service calls collapse into one client request.
5. **Evolve** — It changes at the pace of that client, not of every client.

> **Business logic duplicated across backends will drift**  
> Once there are three of these, a pricing rule or an eligibility check implemented in each will eventually be implemented differently, and the clients will disagree about what a user is allowed to do. These layers must aggregate and shape, never decide — any rule that matters belongs in a shared service that all of them call.

**How it works**

**What it does and what it must not**

```text
WITHOUT  (one shared API)
  mobile home screen
    GET /user         120 fields, needs 3
    GET /orders       full objects, needs id + status
    GET /recommended  needs 5 items, gets 50
    GET /notifications
  -> 4 sequential round trips, ~800 ms on mobile
  -> payload ~180 KB, of which ~8 KB is used

WITH A MOBILE BFF
  GET /mobile/home
    -> BFF calls 4 services in parallel
    -> returns exactly what the screen renders
  -> 1 round trip, ~200 ms
  -> payload ~8 KB

RESPONSIBILITIES
  aggregate calls        yes
  reshape and rename     yes
  filter fields          yes
  client-specific
  caching and defaults   yes
  business rules         NO
  authorisation logic    NO
  data ownership         NO

OWNERSHIP
  the mobile team owns the mobile backend
  -> they change it with their release
  -> no cross-team coordination for a screen change
  -> this is the main organisational benefit
```

1. **Give each client type its own backend** — Web, mobile and partner integrations have genuinely different constraints.
2. **Let the client team own it** — The coordination saving is most of the value.
3. **Keep business rules in shared services** — Duplicated logic across backends will diverge.
4. **Call downstream services in parallel** — Aggregation only helps if the fan-out is concurrent.
5. **Keep it thin enough to be disposable** — A backend that accumulates state stops being a shaping layer.
6. **Consider GraphQL as an alternative** — One flexible endpoint can serve the same purpose with one deployment.

**Cost, and when GraphQL is the better answer**

```text
COST OF THE PATTERN
  3 client types = 3 services to build, deploy,
  monitor and secure
  + shared aggregation code duplicated or extracted
  + one more hop in every request path
  -> justified when client needs genuinely differ
  -> not justified for cosmetic differences

GRAPHQL INSTEAD
  one endpoint, clients request the fields they need
  + no per-client backend
  + client-driven shaping without a deployment
  - query complexity and cost control become real
    problems
  - caching is harder than with fixed endpoints
  -> often the right answer when the difference is
     mostly about field selection

BFF IS BETTER WHEN
  clients differ in more than field selection
    different auth, different caching, different
    protocols, different update cadence
  teams want independent deployment
  payload shaping requires real logic

FAILURE ISOLATION BONUS
  a broken mobile backend does not affect web
  -> an underrated benefit of the split
```

| Metric | Value | Note |
|---|---|---|
| Round trips | 4 → 1 | **~600 ms saved** |
| Payload | 180 KB → 8 KB | on mobile |
| Cost | one service per client | plus a hop |
| Rule | shape, never decide | logic stays shared |

> **Aggregation is only a win if the downstream calls run concurrently**  
> A backend that calls four services one after another has replaced four client round trips with four server round trips plus one more hop, which on a fast internal network is still an improvement but a much smaller one than expected. The saving comes from issuing the calls in parallel and from the internal network being faster than the client's — and sequential implementation quietly discards most of it.

**Technologies**

| Approach | Strength | Watch for |
|---|---|---|
| Per-client backend services | Full control per client | More services to operate |
| GraphQL | Client-driven field selection | Query cost control and caching |
| API gateway aggregation | No new services | Business logic creeping into the gateway |
| Edge compute functions | Backends close to users | Cold starts and limited runtimes |
| Shared API with field selection | Simple | Compromises on payload shape and round trips |
| No separation | Least infrastructure | Every client gets the same compromise |

GraphQL and the per-client backend are not really competitors: several organisations run client-specific backends that each expose a tailored schema, using GraphQL as the shaping mechanism while keeping the ownership and isolation benefits of separate deployments.

**Trade-offs**

**Serving multiple client types**

| Approach | Payload fit | Team autonomy | Services to run |
|---|---|---|---|
| One shared API | Compromised | Low | One |
| Shared API with field selection | Better | Low | One |
| GraphQL | Excellent | Medium | One |
| Backend per client | Excellent | High | One per client |
| Gateway aggregation | Good | Low | One |

Team autonomy is the column that usually decides it. The technical shaping can be achieved several ways, but only separate deployments let the mobile team change their backend alongside their app release without coordinating with anyone.

> **Ask before choosing it**  
> Do the clients differ in more than which fields they need? If the difference is purely field selection, GraphQL or a field parameter achieves it with one service; the pattern earns its extra deployments when caching, protocols, authentication or release cadence differ as well.

**How it fails**

**How backends for frontends go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Clients disagree about business rules | Logic duplicated across backends | Rules live in shared services only |
| Aggregation saves little | Downstream calls made sequentially | Fan out concurrently |
| Backends drift into full services | Accumulating state and ownership | Keep them thin and disposable |
| More deployments than the benefit warrants | Split by cosmetic differences | Consolidate; consider GraphQL |
| One slow service breaks a screen | No timeouts or fallbacks in the aggregation | Per-call timeouts with partial responses |
| Duplicated aggregation code | No shared client libraries | Extract common access code, not logic |
| Security policies inconsistent | Each backend implementing its own | Authenticate at the gateway, uniformly |

> **Rules implemented in several backends will eventually disagree**  
> A discount calculation or an eligibility check written once per client starts identical and diverges with the first bug fix applied in one place. The result is a user who can complete an action on the web and not on mobile, which is confusing to report and difficult to diagnose because both implementations look correct in isolation. These layers must shape data and never decide anything.

**Where it is used**

- **Mobile-heavy consumer products**, where round trips and payload size dominate perceived performance.
- **Organisations with separate web and mobile teams**, where independent deployment is the main benefit.
- **Partner-facing APIs**, shaped and versioned differently from internal clients.
- **Streaming and media platforms**, where device classes have very different capabilities.
- **GraphQL deployments**, achieving the shaping benefit through a schema rather than separate services.

**In the interview**

Give the round-trip and payload numbers; they make the case far better than an abstract description.

- **Quantify the mobile cost** — four sequential round trips against one, and a payload mostly discarded.
- **Emphasise team ownership**, since independent deployment is usually the larger benefit in practice.
- **State the hard boundary**: shape and aggregate, never decide, because duplicated rules drift.
- **Insist the fan-out is parallel**, or the aggregation saves far less than claimed.
- **Offer GraphQL as the alternative** and say when it is preferable — when the difference is only field selection.

**Practice drill**

A mobile home screen currently makes four sequential calls returning 180 KB, of which 8 KB is displayed. Design the backend for it: the endpoint, the fan-out, and the response shape. Estimate the latency improvement and state what assumption it depends on. Then decide what happens when one of the four downstream services is slow, and explain why a pricing rule needed by both web and mobile must not live in this layer.

**Go deeper**

A backend for frontend is a client-specific layer that aggregates and shapes data for exactly one kind of consumer.

**It exists because a shared API must compromise between consumers with incompatible constraints.** A mobile client on a high-latency connection needs few round trips and small payloads; a web client can afford more of both and wants richer data. One interface serving both either over-fetches for mobile or under-serves the browser, and no amount of parameterisation removes the tension when the differences extend beyond field selection.

**The organisational benefit usually exceeds the technical one.** When the team building a client also owns its backend, a screen change ships with the client release and requires no negotiation with a platform team. That removes a coordination point from every feature, which compounds over time in a way that the latency improvement does not — and it is why the pattern persists in organisations where the raw payload savings could have been achieved another way.

**The boundary must be shaping, never deciding.** Business rules implemented separately in each backend begin identical and diverge at the first divergent fix, producing clients that disagree about what a user may do. Because each implementation looks correct on its own, the resulting inconsistency is genuinely difficult to diagnose. Keeping every rule in a shared service that all the backends call is what makes multiple layers safe to have.

**Aggregation only pays when the fan-out is concurrent.** Replacing four client round trips with four sequential server calls plus an extra hop captures very little of the available saving; issuing them in parallel and returning one assembled response is where the improvement comes from. The pattern is frequently described in terms of fewer requests, when the actual mechanism is parallelism plus a faster internal network.

**Partial failure has to be handled deliberately at this layer.** A screen assembled from four services will sometimes have one of them slow or unavailable, and the choice between failing the screen and returning it with a section missing is a product decision the shaping layer is well positioned to make. Per-call timeouts with defined fallbacks turn one degraded dependency into a slightly reduced screen rather than a blank one.

**GraphQL addresses the same problem differently, and the choice depends on where the differences lie.** Where clients want different fields from the same data, a flexible schema serves them all from one deployment and avoids the additional services. Where they differ in caching, protocols, authentication or release cadence, separate backends are the better fit — and the two combine happily, with each client's backend exposing a tailored schema while retaining independent ownership and failure isolation.

**Related patterns:** API Gateway · Aggregator · Scatter Gather · Service Discovery

---

### Service Discovery

*Let services find each other through a registry of currently healthy instances, so addresses can change without reconfiguring callers.*

> **When you hear…** Instances that come and go · autoscaling and rolling deploys · hardcoded addresses breaking on every change

**Flow:** `Instance registers` → `Health checked` → `Client queries registry` → `Gets healthy addresses` → `Unhealthy removed`

**The problem**

Service addresses are listed in configuration files. Every deploy replaces instances with new addresses, every autoscaling event adds or removes them, and every one of those changes requires updating and redeploying the services that call them.

Between the change and the reconfiguration, callers are sending traffic to addresses that no longer answer. With instances rotating several times a day, the window in which the configuration is accurate becomes shorter than the time it takes to update it.

> **Membership is the thing that changes; addresses should be looked up, not configured**  
> Instances know when they start and stop, and health checks know which of them are actually working. If that information is kept in one place that callers consult, the configuration stops needing to describe a constantly changing set — callers ask for whatever is healthy now, and the answer is correct at the moment they need it.

**Mental model**

A registry of service instances, kept current by registration and health checking, consulted by callers at connection time.

1. **Register** — An instance announces itself on startup, or the platform does it.
2. **Check** — Health checks continuously confirm it is serving.
3. **Discover** — A caller asks for the healthy instances of a service.
4. **Balance** — The caller, or a proxy, chooses among them.
5. **Deregister** — An instance leaves cleanly, or is removed when checks fail.

> **The registry can be wrong in both directions, and both are harmful**  
> An instance that has died but has not yet failed enough health checks continues receiving traffic, which fails; an instance that is healthy but briefly unreachable from the checker is removed, reducing capacity. The registry always describes the recent past rather than the present, so callers must handle connecting to an instance that is no longer there.

**How it works**

**Client-side and server-side discovery**

```text
CLIENT-SIDE
  caller queries the registry
  caller picks an instance and connects directly
  + no extra hop
  + the caller can balance intelligently (least
    loaded, zone-aware)
  - discovery logic in every service and language

SERVER-SIDE
  caller connects to a stable address (a load
  balancer or virtual IP)
  that component consults the registry
  + callers need no discovery logic at all
  - one more hop
  - the balancer is in the request path

SIDECAR / MESH
  a local proxy handles discovery and balancing
  caller connects to localhost
  + no discovery logic in the application
  + rich balancing and retries
  - a proxy per instance to run

DNS-BASED
  service name resolves to healthy instances
  + works with everything
  - TTL caching means stale answers
  - no load or health awareness beyond membership
  -> simple and widely used; the TTL is the weakness
```

1. **Prefer platform-provided discovery where one exists** — A container platform already tracks instance lifecycle accurately.
2. **Health check what the service actually does** — A process that is running but cannot reach its database is not healthy.
3. **Deregister on shutdown before stopping** — Otherwise traffic arrives at an instance that has already closed.
4. **Cache discovery results briefly** — Querying per request makes the registry a hot dependency.
5. **Handle connections to dead instances** — The registry is always slightly behind reality.
6. **Keep the registry highly available** — It is consulted by everything; its failure prevents all new connections.

**Health checks and graceful shutdown**

```text
SHALLOW CHECK
  GET /health -> 200 if the process is up
  -> misses a service that cannot reach its database
  -> instance stays registered while failing every
     request

DEEP CHECK
  verifies critical dependencies
  -> accurate, but a shared dependency failing marks
     EVERY instance unhealthy at once
  -> the whole service disappears from the registry
  -> and then nothing can serve, including the
     requests that did not need that dependency

THE BALANCE
  readiness: can this instance serve requests now?
             (used for routing)
  liveness:  should this instance be restarted?
             (used for supervision)
  -> keep dependency checks shallow enough that a
     shared outage does not deregister everything

GRACEFUL SHUTDOWN SEQUENCE
  1  deregister
  2  wait for propagation (caches, DNS TTL)
  3  stop accepting new connections
  4  finish in-flight requests
  5  exit
  -> skipping step 2 causes errors on every deploy,
     and it is the step most often skipped
```

| Metric | Value | Note |
|---|---|---|
| Registry | current membership | **always slightly stale** |
| Check type | readiness vs liveness | different purposes |
| Shutdown | deregister first | then drain |
| Caller | must handle failures | registry is not truth |

> **A deep health check can deregister an entire service at once**  
> If every instance verifies a shared database and that database becomes briefly unreachable, every instance fails its check simultaneously and the service vanishes from the registry. Recovery then requires instances to pass checks again before any traffic flows, extending a short dependency blip into a full outage — and requests that did not need that dependency fail alongside the ones that did.

**Technologies**

| Option | Model | Note |
|---|---|---|
| Kubernetes services and endpoints | Platform-managed | Discovery is built in; usually the right answer |
| Consul | Registry with health checking | Works outside container platforms |
| etcd or ZooKeeper | Generic coordination store | Registry is one use among several |
| DNS-based discovery | Names resolve to instances | Universal; TTL caching is the weakness |
| Service mesh | Sidecar-based discovery and balancing | Rich features, platform commitment |
| Cloud load balancers | Server-side with health checks | Simple, managed, one extra hop |

When the workload already runs on a container platform, its built-in discovery is almost always preferable to an additional registry: the platform knows the instance lifecycle authoritatively because it controls it, rather than inferring it from registration and probes.

**Trade-offs**

**Discovery approaches**

| Approach | Extra hop | Client complexity | Balancing quality |
|---|---|---|---|
| Static configuration | No | None | None |
| DNS | No | None | Membership only |
| Client-side discovery | No | High | Excellent |
| Server-side load balancer | Yes | None | Good |
| Sidecar proxy | Local only | None | Excellent |
| Platform-managed | Varies | None | Good |

The sidecar row is why service meshes gained ground: it delivers client-side discovery quality without putting discovery logic into every application and every language, at the cost of running a proxy alongside every instance.

> **Ask before choosing it**  
> Does the platform already provide this? Adding a separate registry to an environment whose orchestrator already tracks instances creates two sources of truth that will disagree during exactly the events — deploys and scaling — that discovery exists to handle.

**How it fails**

**How service discovery goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Traffic to dead instances | Deregistration slower than shutdown | Deregister first, then drain |
| Errors on every deploy | No propagation delay before stopping | Wait for caches and TTLs to expire |
| Entire service deregistered | Deep health checks on a shared dependency | Keep readiness checks shallow |
| Stale routing for minutes | Long DNS TTLs | Short TTLs, or a non-DNS mechanism |
| Registry becomes a bottleneck | Queried on every request | Cache results with a short lifetime |
| All new connections fail | Registry unavailable | Highly available registry; cached fallback |
| Two sources of truth disagree | Separate registry alongside a platform | Use one mechanism |
| Traffic sent across zones | Discovery ignoring topology | Zone-aware balancing |

> **Deregistering after shutdown begins produces errors on every single deploy**  
> If an instance stops accepting connections before the registry has propagated its removal, callers holding the old address connect and fail — and because this happens during every rolling deploy, it becomes a steady background of errors that teams learn to ignore. The correct order is to deregister, wait long enough for every cache and TTL to expire, and only then stop serving, which turns deploys into non-events.

**Where it is used**

- **Container orchestration platforms**, where discovery is provided as a primitive.
- **Service meshes**, handling discovery and balancing through per-instance proxies.
- **Microservice estates on virtual machines**, using a dedicated registry with health checking.
- **Autoscaling groups behind load balancers**, the server-side form most systems start with.
- **Multi-region deployments**, where zone-aware discovery keeps traffic local.

**In the interview**

The health check and shutdown details are what separate a real answer from a description of a registry.

- **Contrast client-side and server-side discovery** with their respective costs, and mention the sidecar as the modern resolution.
- **Distinguish readiness from liveness**, and warn that deep dependency checks can deregister a whole service at once.
- **Give the graceful shutdown sequence** and stress the propagation wait, which is the step usually missed.
- **Say the registry is always slightly stale**, so callers must handle connecting to instances that have gone.
- **Use the platform's mechanism where one exists**, rather than adding a second source of truth.

**Practice drill**

A service scales between 5 and 50 instances and deploys several times a day. Design discovery for it: the registration mechanism, the health check, and how callers find instances. Write the exact shutdown sequence with timings, and explain which step prevents errors during rolling deploys. Then describe what happens if the health check verifies the database and the database has a 30-second outage.

**Go deeper**

Service discovery maintains a current view of which instances of a service are healthy, so callers can find them without configuration.

**It exists because instance membership changes faster than configuration can be updated.** Deploys, autoscaling and failures rotate instances continuously, and any static list is wrong within hours of being written. Moving membership into a registry that instances update and health checks verify means callers ask a question whose answer is current, rather than reading an answer that was current when someone last edited a file.

**The registry always describes the recent past.** Health checks take time to notice a failure and propagation takes time to reach callers, so there is always a window in which an instance is listed and not serving, or serving and not listed. Callers therefore cannot treat discovery as authoritative — they must handle connection failures and retry elsewhere — and designs that assume the registry is correct fail during precisely the events it exists to manage.

**Health check depth is a genuine trade rather than a detail.** A shallow check misses an instance that is running but unable to do useful work, so traffic continues to a failing instance; a deep check that verifies shared dependencies marks every instance unhealthy when that dependency blips, removing the entire service from rotation and turning a brief outage into a complete one. Keeping readiness checks shallow enough to remain independent, and using liveness separately for restart decisions, is what keeps both failure modes bounded.

**Graceful shutdown ordering determines whether deploys are visible to users.** Deregistering first, waiting long enough for every cache and DNS TTL to expire, and only then refusing connections means no caller ever holds an address that has stopped answering. Omitting the wait produces a small burst of errors during every rolling deploy — frequent enough to become background noise that teams stop investigating, and entirely avoidable.

**Where discovery logic lives has shifted over time for good reasons.** Client-side discovery balances best because the caller knows its own context, and it requires implementing and maintaining that logic in every service and language. Server-side load balancers remove the burden and add a hop. The sidecar proxy resolves the tension by putting client-side discovery in a process next to the application rather than inside it, which is much of why service meshes became attractive despite the operational weight of running a proxy everywhere.

**Two sources of truth are worse than either alone.** Adding a separate registry to a platform that already tracks instance lifecycle creates two views that disagree during deploys and scaling events, and callers then receive different answers depending on which they consult. Because the platform controls the lifecycle it observes it authoritatively, while an external registry infers it from registrations and probes — so where the platform provides discovery, using anything else introduces a disagreement that surfaces at the worst moment.

**Related patterns:** API Gateway · Sidecar · Heartbeat Failure Detection · Gossip Membership

---

### Sidecar

*Run shared infrastructure concerns in a separate process alongside each service instance, so every language gets the same behaviour without a library.*

> **When you hear…** Several languages needing the same infrastructure behaviour · libraries requiring redeploys to upgrade · a service mesh being adopted

**Flow:** `Service container` → `Sidecar alongside` → `Shares network and lifecycle` → `Handles cross-cutting work` → `Service stays focused`

**The problem**

Retries, circuit breaking, mutual TLS, tracing and metrics have to behave identically in services written in four languages. A shared library means four implementations that drift apart, and upgrading any of them requires every service to redeploy.

A security fix in that library therefore propagates at the speed of the slowest team's release cycle, which for infrequently changed services can be months.

> **Put shared behaviour in a process, not in a dependency**  
> A separate process beside the application, intercepting its network traffic, implements the behaviour once for every language — because it operates on connections rather than on function calls. It also upgrades independently: replacing the sidecar changes the behaviour without rebuilding or redeploying the application it serves.

**Mental model**

A companion process deployed with each instance, sharing its lifecycle and network namespace, handling concerns the application should not implement.

1. **Deploy** — The sidecar is scheduled alongside the instance and starts with it.
2. **Intercept** — Inbound and outbound traffic is routed through it.
3. **Handle** — Encryption, retries, discovery, limiting and telemetry happen there.
4. **Forward** — The application sees plain local traffic and implements none of it.
5. **Upgrade** — The sidecar is replaced without touching the application.

> **Every instance now has two processes that can fail**  
> A sidecar that crashes, is misconfigured or exhausts its memory takes the instance with it, and it doubles the resource footprint of the deployment. The failure modes are also less familiar — a proxy rejecting traffic due to a certificate problem looks to the application like the network being broken — which makes debugging harder until teams learn where to look.

**How it works**

**What it takes over, and what it costs**

```text
TYPICAL RESPONSIBILITIES
  mutual TLS between services
  service discovery and load balancing
  retries, timeouts, circuit breaking
  rate limiting
  metrics, distributed tracing, access logs
  traffic shifting for canaries
  -> all identical regardless of language

APPLICATION AFTER ADOPTION
  connects to localhost
  no TLS code, no retry code, no discovery code
  -> the application is genuinely simpler

RESOURCE COST
  ~50-100 MB memory and a fraction of a core per
  sidecar
  1,000 instances -> 50-100 GB of memory and
  substantial CPU just for proxies
  -> a real cost that must be justified by the
     consistency gained

LATENCY COST
  two extra hops (out through one proxy, in through
  another)
  typically 0.5-2 ms total
  -> negligible for most calls
  -> significant for very fast internal calls made
     in a tight loop

WHEN IT IS NOT WORTH IT
  one language, few services
  -> a library gives the same behaviour for far less
```

1. **Adopt it when several languages need identical behaviour** — That is the problem it solves better than any alternative.
2. **Share lifecycle and network namespace** — Otherwise the coupling is unreliable and the interception fragile.
3. **Keep the application unaware of it** — Configuration leaking into the application defeats the separation.
4. **Budget the resource overhead explicitly** — At a thousand instances the proxies are a substantial line item.
5. **Ensure it starts before and stops after the application** — Startup ordering causes failures that are difficult to diagnose.
6. **Consider a library for single-language estates** — The sidecar's advantage is polyglot consistency, which a uniform estate does not need.

**Startup ordering and debugging**

```text
STARTUP RACE
  application starts, makes a call
  sidecar not ready yet
  -> connection refused
  -> looks like a downstream outage, is not
  -> fix: the sidecar must be ready before the
     application starts

SHUTDOWN RACE
  sidecar exits first
  application still finishing in-flight requests
  -> those requests fail
  -> fix: the sidecar drains last

DEBUGGING SHIFT
  before: a failed call is the application's problem
  after:  it may be the application, the local
          sidecar, the remote sidecar, or the remote
          application
  -> four places instead of two
  -> mesh observability is not a luxury; it is what
     makes this tractable

CONFIGURATION SURFACE
  mesh policy is powerful and easy to get wrong
  a misapplied retry policy can amplify load
  a wrong mTLS setting blocks all traffic
  -> treat mesh configuration with the same care as
     production code
```

| Metric | Value | Note |
|---|---|---|
| Consistency | all languages | **one implementation** |
| Overhead | 50-100 MB | per instance |
| Latency | 0.5-2 ms | two hops |
| Debug surface | four components | instead of two |

> **Startup and shutdown ordering produces failures that look like network problems**  
> An application that starts before its sidecar is ready gets connection refusals that appear to be downstream outages, and a sidecar that exits before the application drains causes in-flight requests to fail during every deploy. Both are ordering bugs in the deployment definition rather than faults in either process, and both are commonly misdiagnosed for weeks because the symptoms point elsewhere.

**Technologies**

| Use | Examples | Note |
|---|---|---|
| Service mesh data plane | Envoy in Istio, Linkerd proxy | The dominant use of the pattern |
| Log and metric shipping | Collector sidecars | Simple, low-risk adoption |
| Secret management | Agents fetching and refreshing credentials | Keeps secret handling out of the application |
| Protocol adaptation | Translating between protocols | Legacy integration without changing the service |
| Shared library | In-process equivalent | Cheaper for a single-language estate |
| Node agent | One agent per host instead of per pod | Lower overhead, weaker isolation |

The per-host agent is a meaningful middle ground: one proxy serving every instance on a machine costs far less memory than one per instance, at the price of weaker isolation between workloads and a larger blast radius when it fails.

**Trade-offs**

**Delivering cross-cutting behaviour**

| Approach | Language independence | Upgrade cost | Overhead |
|---|---|---|---|
| Per-service implementation | None | Very high | None |
| Shared library | Per language | Every service redeploys | None |
| Sidecar | Complete | Replace the sidecar | Memory and CPU per instance |
| Node agent | Complete | Replace the agent | Per host |
| Gateway only | Complete for north-south | One component | One hop |

The upgrade column is the argument that usually decides it: a library fix reaches production at the pace of the slowest team's release, while a sidecar fix is a rollout the platform team performs directly.

> **Ask before choosing it**  
> How many languages are actually in use, and how often does the shared behaviour change? A single-language estate with stable requirements gets the same result from a library with none of the overhead, and adopting a mesh for consistency you already have is a large cost for no gain.

**How it fails**

**How sidecars go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Connection refused at startup | Application starting before the sidecar is ready | Enforce startup ordering |
| Requests failing during deploys | Sidecar exiting before the application drains | Sidecar drains last |
| Memory cost unexpectedly large | Overhead multiplied across instances | Budget it; consider a node agent |
| Failures hard to attribute | Four components in every call path | Invest in mesh observability |
| Retry amplification | Retries configured in both mesh and application | Retry in one layer only |
| All traffic blocked | Misapplied mutual TLS policy | Stage policy changes; treat config as code |
| Latency on hot paths | Proxy hops on very fast calls | Bypass the mesh for latency-critical paths |

> **Retries configured in both the mesh and the application multiply silently**  
> A mesh retry policy applied to a service whose client library already retries produces the product of the two, so three attempts each becomes nine — and because neither configuration is visibly wrong, the amplification is only discovered during an incident when a struggling dependency receives many times the expected load. Adopting a mesh requires auditing and removing application-level retries, not merely adding a policy.

**Where it is used**

- **Service meshes**, where a proxy per instance is the data plane and the pattern's dominant use.
- **Kubernetes deployments**, using containers in the same pod for logging, secrets and proxying.
- **Polyglot estates**, obtaining identical mutual TLS and telemetry across four or five languages.
- **Legacy integration**, adapting protocols beside a service that cannot be modified.
- **Observability pipelines**, shipping logs and metrics without instrumenting every application.

**In the interview**

Make the polyglot argument explicitly, then be honest about the overhead and the debugging cost.

- **Frame it as library versus process**: a sidecar works on connections, so it serves every language identically.
- **Emphasise independent upgrades**, since a library fix propagates only as fast as the slowest team redeploys.
- **Quantify the overhead** — tens of megabytes per instance, multiplied across the fleet — rather than calling it small.
- **Raise startup and shutdown ordering**, which produce failures that masquerade as network problems.
- **Warn about duplicated retries** between the mesh and the application, because the multiplication is invisible until an incident.

**Practice drill**

Four services in three languages must all use mutual TLS, consistent retries and distributed tracing. Compare implementing this as a shared library against a sidecar, including the upgrade path for a security fix. Compute the memory overhead of the sidecar across 300 instances. Then describe the startup and shutdown ordering you require, and explain what a developer would see if either were wrong.

**Go deeper**

A sidecar runs infrastructure concerns in a process deployed alongside each service instance, so they are implemented once for every language.

**It converts a dependency problem into a deployment problem, which is easier to manage.** A library must be implemented per language and upgraded by redeploying every service, so a fix reaches production at the pace of the slowest team. A sidecar operates on network traffic rather than on function calls, so one implementation serves everything, and replacing it is a rollout the platform team performs without any application changing.

**Its advantage is specifically about heterogeneity.** In a single-language estate a library provides the same behaviour with no extra processes, no additional hops and no resource overhead, so adopting a sidecar there buys consistency that already existed. The pattern earns its cost when several languages must behave identically, or when the behaviour changes often enough that redeploying everything to update it is genuinely painful.

**The overhead is real and multiplies with fleet size.** Tens of megabytes and a fraction of a core per instance is unremarkable for one service and becomes a substantial infrastructure line item across a thousand, which is why per-host agents exist as a middle ground — lower total overhead, weaker isolation, larger blast radius. Treating the cost as negligible is the mistake; treating it as a considered trade against consistency is the correct framing.

**It widens the debugging surface in a way teams underestimate.** A failed call now involves the application, its local proxy, the remote proxy and the remote application, and a proxy rejecting traffic because of a certificate problem presents to the application as an unreachable network. Without strong mesh observability the additional indirection makes ordinary incidents considerably harder to diagnose, which is why the telemetry the sidecar provides is part of what makes it tolerable.

**Lifecycle ordering is the most common source of confusing failures.** An application starting before its proxy is ready receives connection refusals that look like downstream outages; a proxy exiting before the application drains fails in-flight requests during every deploy. Both are properties of the deployment definition rather than of either process, and both typically persist for a long time because the symptoms point at something else entirely.

**Policy in the mesh must be reconciled with policy in the application.** Retries configured in both layers multiply, so an application retrying three times behind a mesh retrying three times sends nine requests to a struggling dependency — an amplification invisible in either configuration and discovered during an incident. Adopting a mesh therefore includes auditing and removing the application-level behaviour it replaces, which is a migration rather than an addition.

**Related patterns:** Service Discovery · API Gateway · Circuit Breaker · Retry With Jitter

---

### Aggregator

*Have one service call several others and combine their responses, so a client makes one request instead of coordinating many.*

> **When you hear…** A screen needing data from several services · clients orchestrating service calls · chatty client-server interaction

**Flow:** `One client request` → `Fan out in parallel` → `Collect responses` → `Merge and shape` → `Single response`

**The problem**

An order details page needs the order, the customer, the shipment status and the payment record, each owned by a different service. If the client fetches them itself, it makes four calls, handles four failures, and needs to know which services exist and how they relate.

That knowledge belongs in the system, not the client. It also means every change to the service topology requires a client change, and clients that cannot be updated quickly — mobile applications in particular — hold the architecture still.

> **Composition is a server-side responsibility**  
> The services are close to one another and far from the client, so combining their responses server-side turns four slow round trips into one, plus four fast internal ones issued in parallel. The client is then insulated from the topology, and the decisions about partial failure — what to do when one of the four is unavailable — are made in a place that has the context to make them well.

**Mental model**

A service whose job is calling other services concurrently, merging their results, and returning one coherent response.

1. **Receive** — One request arrives describing what the client needs.
2. **Fan out** — Calls to each required service are issued in parallel.
3. **Collect** — Responses are gathered within a deadline.
4. **Handle gaps** — Missing pieces are filled with defaults or omitted.
5. **Merge** — One shaped response is returned.

> **Availability is the product of every dependency unless partial failure is handled**  
> Four services at 99.9% each give 99.6% if all must succeed — roughly three hours of monthly failure created purely by composition. Returning something useful when one dependency is missing is not a refinement; it is what stops the aggregator being less reliable than any of the services behind it.

**How it works**

**Parallel fan-out with partial results**

```text
SEQUENTIAL  (the common mistake)
  order    80 ms
  customer 60 ms
  shipment 90 ms
  payment  70 ms
  total = 300 ms

PARALLEL
  all four issued at once
  total = max(80, 60, 90, 70) = 90 ms
  -> the whole benefit of the pattern lives here

WITH DEPENDENCIES
  order first (needed for the others' ids)
  then customer, shipment, payment in parallel
  total = 80 + 90 = 170 ms
  -> parallelise every stage that can be

DEADLINE AND PARTIAL RESULTS
  overall budget 200 ms
  each call gets a timeout inside it
  at the deadline, return what arrived

  order    ok      required -> without it, fail
  customer ok      required
  shipment TIMEOUT optional -> omit the section
  payment  ok      optional

  -> return the page with shipment status missing
  -> mark it as unavailable rather than absent

REQUIRED VERSUS OPTIONAL
  classify every dependency before writing the code
  -> it determines both the failure behaviour and
     the real availability
```

1. **Issue independent calls concurrently** — Sequential fan-out captures almost none of the benefit.
2. **Classify each dependency as required or optional** — It decides what a missing response does to the response.
3. **Give the whole request a deadline and each call a share** — Without one, the slowest dependency defines the latency.
4. **Return partial results rather than failing** — Composed availability is otherwise the product of every dependency.
5. **Mark missing sections explicitly** — A silently absent section is indistinguishable from empty data.
6. **Keep it stateless and free of business rules** — An aggregator that decides things becomes a service with hidden ownership.

**Availability arithmetic**

```text
ALL REQUIRED
  4 dependencies at 99.9%
  0.999^4 = 99.60%
  -> ~3 hours unavailable per month from composition
     alone

TWO REQUIRED, TWO OPTIONAL
  0.999^2 = 99.80% for a complete response
  but a USEFUL response requires only the two
  -> the page renders through most failures
  -> availability as experienced is much higher than
     the arithmetic suggests

CACHING OPTIONAL DEPENDENCIES
  serve the last known value when a service is down
  -> the section is stale rather than missing
  -> usually better for the user, and must be
     labelled

WHAT THE AGGREGATOR MUST NOT BECOME
  a place where business rules live
  -> it has no data of its own and no clear owner
  -> rules there are invisible to the teams who own
     the domains
  -> keep it to calling, waiting, merging, shaping
```

| Metric | Value | Note |
|---|---|---|
| Sequential | 300 ms | avoidable |
| Parallel | 90 ms | **the point** |
| All required | 99.6% | from 99.9% parts |
| Partial results | much higher | as experienced |

> **An aggregator without partial-failure handling is less available than anything it calls**  
> Requiring every dependency multiplies their failure rates, so a composition of four highly available services is measurably less available than each of them. The component added to simplify the client has then become the least reliable thing in the path, and the only remedy is deciding, per dependency, what the response looks like without it.

**Technologies**

| Approach | Strength | Watch for |
|---|---|---|
| A dedicated aggregation service | Full control over merging and failure | Another service to operate |
| Backend for frontend | Aggregation shaped per client | One per client type |
| GraphQL server | Client-driven composition | Query cost and caching |
| Gateway aggregation | No new service | Logic accumulating in the gateway |
| Client-side composition | No server component | Round trips and topology coupling |
| Materialised view | Precomputed joins | Staleness; a write-side commitment |

A materialised view is the alternative worth remembering: where the same combination is requested constantly and tolerates slight staleness, precomputing the joined result removes the fan-out entirely and replaces it with a single read.

**Trade-offs**

**Where composition happens**

| Location | Client round trips | Topology coupling | Failure handling |
|---|---|---|---|
| Client-side | One per service | High | In the client |
| Aggregator service | One | None | Server-side, with context |
| Backend for frontend | One | None | Per client type |
| GraphQL | One | None | Per field resolution |
| Materialised view | One | None | Precomputed; staleness instead |

Client-side composition is not always wrong: a web application on a fast connection making three parallel calls is perfectly reasonable, and the pattern earns its place where round trips are expensive or where failure handling needs context the client does not have.

> **Ask before choosing it**  
> Which of these dependencies must succeed for the response to be useful? Answering that first produces both the failure design and an honest availability figure, and teams that skip it usually discover the composed availability during an incident rather than during design.

**How it fails**

**How aggregators go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Latency equals the sum of dependencies | Sequential calls | Fan out concurrently |
| Less available than any dependency | All dependencies treated as required | Classify and return partial results |
| One slow service delays everything | No per-call timeout | Deadline budget split across calls |
| Missing data looks like empty data | Sections omitted silently | Explicitly mark unavailable sections |
| Thread pool exhausted | Blocking calls to slow dependencies | Asynchronous calls with bounded concurrency |
| Business rules hidden in the aggregator | Logic accumulating in a layer with no owner | Rules belong in domain services |
| Cascading retries | Aggregator retrying while clients also retry | Retry at one layer |

> **Sequential fan-out discards the entire benefit while keeping all the costs**  
> An aggregator calling four services one after another produces the same total latency the client would have experienced, plus an additional hop, plus a new service to operate. It looks correct in code review and in a single-service trace, and it is only visible as a problem when someone compares the response time against the sum of the dependency latencies — at which point the pattern has been paying for itself in nothing but complexity.

**Where it is used**

- **Product and order detail pages**, combining data from several domain services.
- **Mobile backends**, where one aggregated call replaces several expensive round trips.
- **Search result enrichment**, merging results with inventory, pricing and personalisation.
- **Dashboards**, gathering metrics from many sources into one response.
- **GraphQL servers**, which are aggregators driven by a client-supplied query.

**In the interview**

Two things carry the answer: parallel fan-out and the availability arithmetic.

- **Show the latency difference** between sequential and parallel fan-out with concrete numbers.
- **Do the availability multiplication** — four services at three nines give 99.6% — to motivate partial results.
- **Classify dependencies as required or optional** and describe the response in each failure case.
- **Give the whole request a deadline** and split it across calls, so one slow service cannot define the latency.
- **Keep business rules out of it**, because a layer owning no data and no domain is the wrong place for decisions.

**Practice drill**

An order page needs data from four services with latencies of 80, 60, 90 and 70 milliseconds, each 99.9% available. Compute the response time for sequential and parallel fan-out, and the availability if all are required. Classify which are genuinely required, design the partial-response behaviour, and state the resulting user-visible availability. Then describe what happens when one service is slow rather than down, and what the client receives.

**Go deeper**

An aggregator composes responses from several services into one, so clients make a single request and remain unaware of the topology behind it.

**Its benefit comes from where the network is fast.** Services sit close to each other and far from the client, so moving composition to the server replaces several expensive round trips with one, plus several cheap internal ones. That advantage is entirely dependent on issuing the internal calls concurrently — a sequential implementation produces the same latency as the client would have had, with an extra hop and an extra service to run.

**Composition multiplies failure unless partial results are designed in.** Four dependencies at three nines yield 99.6% when all must succeed, which is worse than any individual component and amounts to hours of monthly unavailability produced by the composition itself. Classifying each dependency as required or optional, and defining what the response looks like without each optional one, is what keeps the aggregate at least as available as its most critical part.

**A deadline budget is what makes slow dependencies survivable.** Without one, the response time is dictated by whichever service is slowest at that moment, and a single degraded dependency makes every request slow rather than making one section missing. Allocating an overall budget and giving each call a share means the aggregator returns on time with whatever arrived, which is almost always more useful than a complete response that came too late to matter.

**Missing data must be distinguishable from empty data.** A section omitted because a service timed out looks identical to a section that is genuinely empty, so a client cannot tell whether to show nothing or to show an error. Marking unavailable portions explicitly lets the interface say that shipment status could not be loaded rather than implying that no shipment exists, which is a materially different statement to the user.

**It must remain a composition layer with no opinions.** An aggregator owns no data and belongs to no domain, so business rules placed there are invisible to the teams responsible for the concepts they govern, and they drift from the rules those teams maintain. Keeping the layer to calling, waiting, merging and shaping is what prevents it from becoming a service with hidden ownership and unclear authority.

**Where the same composition is requested constantly, precomputing it is often better.** A materialised view of the joined result turns a fan-out into a single read and removes the availability multiplication entirely, at the cost of staleness and a write-side pipeline to maintain. That trade is frequently worthwhile for high-traffic read paths, and considering it before building an aggregator is what distinguishes choosing the pattern from reaching for it.

**Related patterns:** Scatter Gather · Backend For Frontend · API Gateway · Materialized View

---

### Scatter Gather

*Send the same request to many nodes in parallel and combine their partial answers, so work that spans a dataset completes in the time of one node.*

> **When you hear…** A query that must consult every shard · search across a partitioned index · parallel computation over distributed data

**Flow:** `Query arrives` → `Broadcast to shards` → `Each searches locally` → `Partial results returned` → `Merged and ranked`

**The problem**

A search index is spread across sixty-four shards because it does not fit on one machine. A query has no way to know which shard holds the best matches, so consulting one is not an option — the answer depends on all of them.

Consulting them one after another takes sixty-four times as long as consulting one, which for an interactive search is hopeless. The data was partitioned to make it fit, and the query must now be made to work across that partitioning.

> **Broadcast the query, and let the fan-out cost be latency rather than throughput**  
> Because each shard holds a disjoint portion, all of them can search simultaneously and the elapsed time is that of a single shard rather than their sum. The query is duplicated across the fleet — total work is unchanged — but the user waits only as long as the slowest response, which turns an impossible sequential cost into a tractable parallel one.

**Mental model**

One coordinator broadcasting an identical request to every partition, then merging the partial results into a final answer.

1. **Broadcast** — The same query is sent to every shard.
2. **Search locally** — Each shard produces its best partial answer.
3. **Return** — Partial results, usually with scores, come back.
4. **Merge** — The coordinator combines and ranks them.
5. **Bound** — A deadline decides what to do about shards that have not replied.

> **Latency is set by the slowest shard, so component tails become the system median**  
> With sixty-four shards, a 1% chance of any one being slow means roughly a 47% chance that some shard is slow on every query. The 99th percentile of a shard therefore becomes something close to the typical experience of the query, which is why large fan-out systems must attack tail latency directly rather than average latency.

**How it works**

**Tail latency and the merge**

```text
THE ARITHMETIC
  P(a shard is slow) = 1%
  64 shards
  P(all fast) = 0.99^64 = 0.526
  -> ~47% of queries hit at least one slow shard
  -> the shard p99 becomes near the query median

MITIGATIONS
  hedged requests   send a duplicate to a replica
                    after the usual latency elapses
  deadline          return at the budget with what
                    arrived
  partial results   report coverage: "62 of 64 shards"
  -> most search systems use all three

THE MERGE
  each shard returns its local top K with scores
  coordinator merges and takes the global top K
  -> scores must be COMPARABLE across shards
  -> local relevance normalisation breaks this
     silently, producing subtly wrong rankings

HOW MANY TO REQUEST FROM EACH SHARD
  want top 10 globally
  request top 10 from each shard, not top 1
  -> all 10 could be on one shard
  -> requesting top 1 each gives a wrong answer

DEEP PAGINATION
  page 100 of 10 results
  each shard must return its top 1000 to guarantee
  correctness
  -> cost grows linearly with the page number
  -> which is why deep pagination is usually capped
```

1. **Request the full result count from every shard** — Asking each for a fraction produces incorrect global rankings.
2. **Ensure scores are comparable across shards** — Locally normalised scores make the merge quietly wrong.
3. **Set a deadline and return partial results** — Waiting for every shard means the slowest always wins.
4. **Report coverage with partial responses** — The consumer needs to know the answer is incomplete.
5. **Hedge against slow shards** — Duplicating to a replica after the usual latency cuts the tail sharply.
6. **Cap pagination depth** — Cost per page grows linearly and deep pages are rarely used.

**Where the cost lands**

```text
THROUGHPUT COST
  one query = 64 shard queries
  1,000 queries/s = 64,000 shard queries/s
  -> the cluster must be sized for the fan-out, not
     for the query rate

LATENCY BENEFIT
  sequential: 64 x 20 ms = 1,280 ms
  parallel:   max over 64 ≈ 20-60 ms
  -> the entire reason the pattern exists

REDUCING THE FAN-OUT
  route by a partition key where the query has one
    "orders for customer X" -> one shard
  filter shards by metadata
    date-partitioned index, query has a date range
    -> touch only the relevant shards
  -> the best optimisation is not broadcasting

WHEN NOT TO USE IT
  the query has a natural partition key
    -> route directly, do not broadcast
  results needed from only one partition
  the fan-out exceeds what the cluster can sustain
    at the query rate
```

| Metric | Value | Note |
|---|---|---|
| Latency | one shard | **not the sum** |
| Throughput | × shard count | size for fan-out |
| 64 shards | 47% hit a straggler | tail dominates |
| Best fix | do not broadcast | route by key |

> **Requesting fewer results per shard than the global count returns the wrong answer**  
> If ten results are wanted overall and each shard is asked for one, the correct answer is lost whenever several of the best matches sit on the same shard — which is common, because relevance and partitioning are unrelated. Each shard must return the full count so the merge has enough candidates, and this is the reason deep pagination becomes expensive rather than merely slow.

**Technologies**

| System | Use | Note |
|---|---|---|
| Elasticsearch and OpenSearch | Distributed search across shards | The canonical implementation |
| Distributed SQL engines | Parallel scans across partitions | Aggregation pushed down where possible |
| MapReduce-style frameworks | Batch scatter-gather | The same idea at a much coarser grain |
| Sharded databases | Queries without a shard key | Works, and should be the exception |
| Hedged requests | Tail latency control | Essential at large fan-out |
| Routing by key | Avoiding the fan-out entirely | The best optimisation available |

Pushing aggregation down to the shards matters more than it appears: having each shard compute a local count or sum and returning only those values, rather than raw rows, reduces the data crossing the network by orders of magnitude and moves the work to where the data already is.

**Trade-offs**

**Query routing strategies**

| Strategy | Shards touched | Latency | Correctness |
|---|---|---|---|
| Route by partition key | One | One shard | Exact |
| Filter shards by metadata | A subset | Slowest of the subset | Exact |
| Scatter-gather all shards | All | Slowest shard | Exact |
| Scatter-gather with a deadline | All attempted | Bounded | Partial |
| Sample a subset of shards | Some | Fast | Approximate |

Sampling is a legitimate strategy for analytics and trend queries where an approximate answer computed quickly is more valuable than an exact one computed slowly — and it is completely wrong for anything a user expects to be complete.

> **Ask before choosing it**  
> Does the query contain something that identifies a partition? If it does, routing directly is faster, cheaper and more reliable than broadcasting, and the fan-out should be reserved for the queries that genuinely cannot be narrowed.

**How it fails**

**How scatter-gather goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Latency dominated by stragglers | Waiting for every shard | Deadlines, partial results, hedging |
| Wrong results returned | Too few results requested per shard | Request the full count from each |
| Rankings subtly incorrect | Scores not comparable across shards | Global scoring or normalised statistics |
| Cluster saturated | Fan-out multiplied by query rate | Route by key where possible; add replicas |
| Deep pagination times out | Cost growing with page depth | Cap depth; use cursors |
| Partial results presented as complete | Coverage not reported | Return which shards answered |
| Memory exhausted in the coordinator | All raw results merged in memory | Push aggregation down; stream the merge |

> **Fan-out multiplies every query against the entire cluster**  
> A thousand queries per second across sixty-four shards is sixty-four thousand shard queries per second, so the cluster must be provisioned for the product rather than for the query rate. This is why reducing the fan-out — routing by key, filtering shards by metadata, denormalising so the common query touches one partition — is worth far more than optimising the individual shard query, and why systems that broadcast everything hit a capacity wall that more shards do not fix.

**Where it is used**

- **Distributed search engines**, the defining use, broadcasting queries across an index.
- **Analytics engines**, scanning partitions in parallel and merging aggregates.
- **Sharded databases**, for queries that lack a shard key and must consult everything.
- **Batch processing frameworks**, applying the same idea with much longer time scales.
- **Federated queries**, fanning out across independent systems and combining their answers.

**In the interview**

The tail arithmetic is the centre of this answer; without it the pattern sounds straightforward and is not.

- **Compute the straggler probability** — a 1% shard tail across 64 shards affects nearly half of queries.
- **Say latency equals the slowest shard**, and propose deadlines, partial results and hedging as the mitigations.
- **Explain the top-K merge correctly**: each shard returns the full count, not a share, or the answer is wrong.
- **Size for the fan-out**, since one query becomes as many shard queries as there are shards.
- **Propose avoiding the broadcast** by routing on a partition key wherever the query permits it.

**Practice drill**

A search index has 64 shards, each answering in 20 ms at the median and 200 ms at the 99th percentile. Compute the probability that a query touches a slow shard and estimate the query's p99. Design the mitigations: deadline, hedging and partial-result reporting. Then explain why asking each shard for the top 1 when 10 results are wanted is incorrect, and what deep pagination does to the cost.

**Go deeper**

Scatter-gather broadcasts one request to every partition in parallel and merges the partial answers into a complete result.

**It converts a sequential cost into a parallel one.** Consulting sixty-four shards one after another takes sixty-four times a single shard's latency, which is unusable interactively; consulting them simultaneously takes as long as the slowest. Total work is unchanged — the cluster performs the same number of shard queries either way — so what the pattern buys is latency, paid for in throughput capacity.

**Tail latency dominates, and this is the defining characteristic at scale.** The chance that at least one shard is slow rises quickly with fan-out, so a component's 99th percentile becomes close to the query's typical experience. Improving each shard's median does almost nothing for this; only compressing tails helps, which is why deadlines with partial results and hedged requests to replicas are standard rather than optional in large fan-out systems.

**The merge has a correctness requirement that is easy to violate.** Each shard must return the full desired result count rather than a share of it, because the best matches may be concentrated on one partition — and the scores those shards return must be comparable, or the coordinator ranks apples against oranges. Locally normalised relevance scores break this quietly, producing rankings that look plausible and are wrong, which is among the harder bugs to notice in a search system.

**Deep pagination degrades in a way that surprises people.** Returning the hundredth page of ten results requires every shard to produce its top thousand so the merge can be correct, so cost grows linearly with page depth while the value of those pages approaches zero. Capping depth or switching to cursor-based navigation is the standard resolution, and it is a product decision as much as a technical one.

**Throughput must be provisioned for the product of query rate and fan-out.** A thousand queries per second across sixty-four shards is sixty-four thousand shard queries, so the cluster is sized by the broadcast rather than by the external rate. Adding shards to handle more data also multiplies this, which means fan-out systems have a capacity relationship that does not improve as they grow — a reason to take shard-count decisions seriously rather than treating them as a storage detail.

**The strongest optimisation is to avoid broadcasting at all.** A query carrying a partition key can be routed to one shard, and metadata filters — time ranges against a time-partitioned index, for instance — can exclude most shards before the request is sent. Denormalising so that the highest-volume queries have a routable key is frequently worth considerable effort, because it moves those queries from an expensive fan-out to a single lookup and leaves the broadcast for the queries that genuinely need it.

**Related patterns:** Aggregator · Hedged Requests · Sharding · Inverted Index

---

## Data Processing

### Batch Processing

*Process a bounded set of data as one job on a schedule, trading freshness for throughput, simplicity and the ability to reprocess.*

> **When you hear…** Large volumes to process efficiently · results that can be hours old · computations that must be reproducible

**Flow:** `Data accumulates` → `Job starts on schedule` → `Reads a bounded set` → `Computes results` → `Writes output atomically`

**The problem**

Billing needs to aggregate a month of usage across millions of accounts. Doing this incrementally as events arrive means maintaining running state everywhere, handling late and corrected events in that state, and having no straightforward way to recompute if a rule changes.

It also processes each record individually, which is the least efficient way to touch large volumes — losing every advantage of sequential reads, bulk writes and columnar formats.

> **A bounded input makes correctness tractable**  
> When a job knows exactly which records it is processing, it can read them efficiently, compute a complete answer, and write the result atomically. There is no partial state to maintain between runs, no ambiguity about what has been included, and rerunning the job with corrected logic produces a corrected answer — because the input is still there and the computation is a pure function of it.

**Mental model**

A scheduled job over a fixed window of data, producing an output that replaces rather than amends the previous one.

1. **Accumulate** — Data collects in storage until the job runs.
2. **Bound** — The job selects a definite input — a day, a partition, a range.
3. **Process** — The computation runs over that whole set, usually in parallel.
4. **Write** — Output is produced and published atomically.
5. **Repeat** — The next run takes the next window, independently.

> **Publishing output incrementally exposes a half-computed result**  
> A job writing directly into the table consumers read makes partially processed data visible for the duration of the run, and a failure mid-way leaves it permanently inconsistent. Writing to a new location and switching consumers to it atomically is what makes a batch job safe to fail — the previous output remains correct and complete until the new one is entirely ready.

**How it works**

**Idempotence, partitioning and late data**

```text
IDEMPOTENT BY PARTITION
  each run writes to a partition determined by its
  input window
    output/date=2026-09-14/
  rerunning replaces that partition entirely
  -> reruns are safe, which is the property that
     makes batch operable

  NOT idempotent: appending to a single output
  -> a rerun doubles the results, silently

ATOMIC PUBLISH
  write to output/_tmp/date=.../
  verify
  atomically move or swap the pointer
  -> consumers never observe a partial result

LATE DATA
  the job for 14 September runs at 02:00 on the 15th
  an event with timestamp 23:58 on the 14th arrives
  at 02:05
  -> it is not in the output

  options:
    wait longer before running (delays everything)
    reprocess affected days on a slower cadence
    accept the loss for low-value data
  -> this must be a decision, not a discovery

BACKFILL
  logic changes -> rerun historical partitions
  -> only possible because the input is retained and
     the computation is deterministic
  -> this is batch processing's defining advantage
```

1. **Make each run idempotent by partition** — Reruns are routine, and appending makes them destructive.
2. **Publish output atomically** — Consumers must never see a partially computed result.
3. **Keep the raw input** — Reprocessing is the main advantage, and it requires the source data.
4. **Decide the late-data policy explicitly** — Every batch boundary excludes something; the question is what happens to it.
5. **Partition input and output by time** — It makes both parallelism and selective reprocessing straightforward.
6. **Monitor duration against the schedule** — A job whose runtime approaches its interval is about to overlap with itself.

**Why batch is efficient, and where it stops working**

```text
EFFICIENCY
  sequential reads over columnar files
  bulk writes rather than per-record transactions
  compression across many rows
  parallelism across partitions
  -> often 10-100x cheaper per record than
     processing events individually

THE COST IS FRESHNESS
  hourly job  -> results up to 1 hour old
  daily job   -> results up to 24 hours old
  -> plus the job's own runtime

WHEN THE SCHEDULE BREAKS DOWN
  job takes 45 min, scheduled hourly
  volume grows 50%
  -> job takes 68 min
  -> runs overlap, resources contend, both slow
  -> a queue of delayed runs forms and never drains
  -> alert on duration / interval, not just failures

WHEN TO USE STREAMING INSTEAD
  results needed in seconds
  the value of an answer decays quickly
  -> but keep batch for correction and backfill;
     most mature systems run both
```

| Metric | Value | Note |
|---|---|---|
| Efficiency | 10-100× cheaper | **per record** |
| Freshness | interval + runtime | the trade |
| Reruns | safe, by partition | the key property |
| Watch | duration vs interval | overlap risk |

> **A job whose runtime approaches its interval will eventually overlap with itself**  
> Growing data makes jobs take longer, and a job scheduled hourly that now takes fifty minutes has almost no margin. When it crosses the boundary, two runs execute concurrently, compete for the same resources, and both slow down — producing a backlog that never drains without intervention. The ratio of runtime to interval is the metric to alert on, long before any job actually fails.

**Technologies**

| Layer | Options | Note |
|---|---|---|
| Processing engines | Spark and similar | Distributed computation over partitioned data |
| Warehouse SQL | Scheduled queries | Simplest option when the data is already there |
| Orchestration | Airflow, Dagster and similar | Dependencies, retries, backfills |
| Storage formats | Parquet, ORC | Columnar, compressed, partition-aware |
| Object storage | Partitioned by date | Cheap, durable, atomic swaps by prefix |
| Streaming | The low-latency alternative | Usually complementary rather than a replacement |

Orchestration is what turns a collection of jobs into a system: dependency ordering, retry policy, backfill tooling and alerting are most of the operational work, and they matter more than the choice of processing engine for all but the largest workloads.

**Trade-offs**

**Batch against streaming**

| Property | Batch | Streaming |
|---|---|---|
| Latency | Minutes to hours | Seconds or less |
| Cost per record | Low | Higher |
| Reprocessing | Natural; rerun the job | Requires replay and care |
| Late data | Handled by rerunning | Handled by windows and watermarks |
| Operational complexity | Lower | Higher; always running |
| Failure recovery | Rerun the partition | Resume from a checkpoint |

The reprocessing row is why batch persists even where streaming is available. A rule change or a bug fix applied to a batch job is a rerun over the affected partitions; the same change to a streaming pipeline requires replaying history through a system designed for the present.

> **Ask before choosing it**  
> How old may this answer be before it stops being useful? A dashboard refreshed hourly and a fraud decision needed in milliseconds are different problems, and the acceptable staleness — not the data volume — is what decides between batch and streaming.

**How it fails**

**How batch processing goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Duplicated results after a rerun | Output appended rather than replaced | Idempotent partition overwrite |
| Consumers read partial output | Non-atomic publish | Write elsewhere, then swap atomically |
| Overlapping runs | Runtime approaching the interval | Alert on the ratio; scale or lengthen the interval |
| Late events silently dropped | No late-data policy | Reprocess affected windows on a slower cadence |
| Backfill impossible | Raw input discarded | Retain source data for the reprocessing window |
| One failure blocks everything downstream | Rigid dependency chain | Isolate failures; allow partial progress |
| Results wrong after a logic change | No reprocessing of historical partitions | Backfill as part of the change |

> **A batch job that cannot be safely rerun is not really a batch job**  
> The entire operational model assumes that any run can be repeated — after a failure, after a bug fix, after a logic change. A job that appends rather than replaces, or that mutates state incrementally, turns every rerun into a corruption risk, which means failures must be resolved by hand and improvements to the logic cannot be applied to history. Idempotence by partition is what makes the rest of the model work.

**Where it is used**

- **Billing and usage aggregation**, where exactness matters more than immediacy.
- **Analytics and reporting pipelines**, computing daily aggregates over large volumes.
- **Machine learning feature generation**, producing training data from historical events.
- **Search index rebuilds**, regenerating an index from the source of record.
- **Data warehouse loading**, the archetypal scheduled transformation.

**In the interview**

Lead with idempotence and atomic publishing; they are what make batch operationally sound.

- **Say each run replaces a partition** rather than appending, which is what makes reruns safe.
- **Describe atomic publication** so consumers never observe a half-computed result.
- **State the late-data policy explicitly**, since every batch boundary excludes something.
- **Raise runtime against interval** as the metric that predicts overlapping runs before they happen.
- **Give reprocessing as the reason batch survives** alongside streaming — a logic change is a rerun, not a replay.

**Practice drill**

Design a nightly job aggregating a day of usage events for billing. Specify the input bounds, the output partitioning, the publication mechanism and the rerun behaviour. State your policy for events that arrive after the job has run. Then the job's runtime grows from 40 to 55 minutes against an hourly schedule — say what you would alert on, what happens if nothing is done, and what your options are.

**Go deeper**

Batch processing computes results over a bounded set of data on a schedule, trading freshness for efficiency, simplicity and reproducibility.

**Bounded input is what makes the model tractable.** A job that knows exactly which records it covers produces a complete answer that is a pure function of that input, with no running state to maintain between executions and no ambiguity about what has been counted. Correctness becomes a property of one computation rather than of an ongoing process, which is a substantially easier thing to reason about and to test.

**Idempotence by partition is the property everything else rests on.** Runs fail, logic changes, and bugs are found after the fact, so every execution must be repeatable — which means writing to a partition determined by the input window and replacing its contents entirely. A job that appends instead turns each rerun into a silent duplication, and the operational model collapses into manual intervention for every failure.

**Publication must be atomic or consumers see incoherent data.** Writing directly into the tables consumers read exposes partial results for the duration of the run and leaves permanent inconsistency if the job fails midway. Producing output in a separate location and switching over once it is verified keeps the previous, complete answer available until the new one is entirely ready, which is what allows a job to fail without an incident.

**Its efficiency comes from doing everything in bulk.** Sequential reads over columnar files, compression across many rows, bulk writes instead of per-record transactions, and parallelism across partitions combine to make batch processing an order of magnitude or more cheaper per record than handling events individually. That advantage grows with volume, which is why batch remains the economical choice for large-scale aggregation regardless of what streaming infrastructure exists alongside it.

**Every boundary excludes something, so the late-data policy is part of the design.** An event arriving after its window has been processed is either lost, or captured by a later reprocessing pass, or forces the job to wait longer before running. All three are legitimate and they have different costs, but the policy has to be chosen deliberately — systems that never decide discover it as a discrepancy in a report months later, at which point the affected data may no longer be reprocessable.

**Reprocessing is why batch survives the availability of streaming.** A changed rule or a corrected bug is applied to history by rerunning the affected partitions, which is routine precisely because the input is retained and the computation is deterministic. Achieving the same in a streaming system means replaying history through infrastructure built for the present, with all the state and ordering concerns that implies — so mature architectures typically run both, using streaming for immediacy and batch for correctness and correction.

**Related patterns:** Stream Processing · Materialized View · Partitioned Log · Durable Workflow

---

### Stream Processing

*Compute continuously over unbounded event streams, maintaining state and emitting results within seconds of the events that caused them.*

> **When you hear…** Results needed in seconds · continuously updated aggregates · detection that must happen while it still matters

**Flow:** `Events arrive` → `Continuous operators` → `State per key` → `Windows close` → `Results emitted`

**The problem**

Fraud detection based on an hourly batch job identifies a fraudulent pattern up to an hour after it began, by which time the transactions have completed. The value of the answer decayed long before it was computed.

Running the batch job more frequently does not fix it: at minute-level intervals the fixed cost of starting jobs dominates, the same data is re-read repeatedly, and the latency floor remains the interval plus the runtime.

> **Keep the computation running and feed events through it**  
> If the operators are always live and hold their state in memory, each event updates the result as it arrives rather than waiting for a job to start. Latency becomes the processing time of one event instead of the interval between runs — and the price is that the system must maintain durable state continuously, handle events that arrive out of order, and decide when an answer is final.

**Mental model**

A standing dataflow of operators over an unbounded input, with keyed state, windows that bound aggregation, and checkpoints that make the state recoverable.

1. **Consume** — Events are read from a log, in partition order.
2. **Transform** — Operators filter, enrich and key the stream.
3. **Accumulate** — State per key is updated as events arrive.
4. **Window** — Aggregations are bounded by time or count.
5. **Emit** — Results are produced when a window closes or on every update.

> **Event time and arrival time are not the same, and the gap is where correctness lives**  
> A mobile device offline for ten minutes sends events timestamped ten minutes ago. Grouping by arrival time puts them in the wrong window and quietly produces wrong answers; grouping by event time requires deciding how long to wait for stragglers, which is a trade between latency and completeness that has no universally correct setting.

**How it works**

**Windows, watermarks and late events**

```text
WINDOW TYPES
  tumbling  fixed, non-overlapping
            [0-60) [60-120)
  sliding   fixed, overlapping
            every 10 s, covering the last 60 s
  session   grouped by inactivity gaps
            closes after 30 min of silence

WATERMARKS
  a claim: "no events older than T will arrive"
  -> windows ending before T can be closed
  -> derived from observed event times minus an
     allowance

  watermark lag 5 minutes
    -> results are 5 minutes behind real time
    -> and events later than 5 minutes are late

THE LATENCY / COMPLETENESS TRADE
  short watermark  fast results, more late events
  long watermark   slower results, fewer missed

LATE EVENT POLICIES
  drop            simplest, silently wrong
  side output     collect and handle separately
  update the
  emitted result  correct, but downstream must
                  handle retractions
  -> pick one deliberately; the default is usually
     drop

STATE
  keyed state lives in the operator
  checkpointed periodically to durable storage
  -> recovery restores state and resumes from the
     checkpointed offsets
  -> checkpoint interval bounds the replay on
     failure
```

1. **Use event time, not arrival time** — Arrival-time windows are wrong whenever events are delayed, which is always.
2. **Choose the watermark from observed lateness** — It is a direct trade between result latency and completeness.
3. **Define the late-event policy explicitly** — Dropping silently is a decision, and usually an unexamined one.
4. **Checkpoint state at an interval matched to acceptable replay** — Recovery replays from the last checkpoint.
5. **Keep state bounded** — Unbounded keyed state grows forever; expire keys deliberately.
6. **Retain the source log for replay** — Reprocessing after a logic change depends on it entirely.

**Operations and delivery semantics**

```text
EXACTLY-ONCE, PRECISELY
  exactly-once STATE: checkpointed offsets and state
  advance together, so recovery does not double-count
  internally
  exactly-once OUTPUT: only if the sink is
  transactional or idempotent
  -> "exactly-once" without a qualifying sink means
     at-least-once at the boundary

BACKPRESSURE
  a slow operator slows its upstream
  -> the source reads more slowly
  -> consumer lag grows
  -> lag is the primary health metric

RESTARTS ARE EXPENSIVE
  state must be restored from the checkpoint
  large state -> minutes of recovery
  -> deployments are not free, unlike a batch job
     that simply runs again

STATE SIZE
  1M keys x 1 KB = 1 GB per instance
  -> memory or embedded storage becomes the limit
  -> expiry policy is a capacity decision

WHAT MAKES IT HARDER THAN BATCH
  always running, so failures are incidents
  state to manage, restore and size
  out-of-order events
  no natural point at which an answer is final
```

| Metric | Value | Note |
|---|---|---|
| Latency | seconds | **per event** |
| Trade | watermark lag | completeness |
| State | checkpointed | recovery cost |
| Health metric | consumer lag | primary signal |

> **Exactly-once is a property of the pipeline, not of the output, unless the sink cooperates**  
> Frameworks provide exactly-once by advancing state and input offsets together, so internal recovery does not double-count. The moment results are written to an external system, the guarantee holds only if that write is transactional or idempotent — otherwise a recovery re-emits results the sink has already applied. Describing a pipeline as exactly-once without qualifying the sink overstates what it delivers.

**Technologies**

| Layer | Options | Note |
|---|---|---|
| Processing frameworks | Flink, Kafka Streams, Spark Structured Streaming | Differ in state and windowing models |
| Source | Partitioned log | Replay is what makes reprocessing possible |
| State backends | Embedded key-value stores with checkpoints | State size drives instance sizing |
| Sinks | Transactional or idempotent writes | Required for end-to-end guarantees |
| Streaming SQL | Declarative continuous queries | Lower barrier; less control |
| Batch | The complementary approach | Correction, backfill and reprocessing |

Streaming SQL has changed who can build these pipelines: windowed aggregations expressed declaratively remove most of the boilerplate, while leaving the hard decisions — watermarks, late data, state expiry — exactly where they were.

**Trade-offs**

**Streaming against batch**

| Property | Streaming | Batch |
|---|---|---|
| Latency | Seconds | Minutes to hours |
| Cost per record | Higher | Lower |
| State | Continuous, checkpointed | None between runs |
| Out-of-order handling | Watermarks and late policies | Naturally handled by rerunning |
| Reprocessing | Replay the log | Rerun the job |
| Operational burden | Always running | Fails between runs, not during |

The operational row understates the difference in practice: a failed batch job is rerun in the morning, while a failed streaming job is an incident with growing lag, which changes how the team must be organised around it.

> **Ask before choosing it**  
> Does the value of this answer decay in seconds? Fraud detection and live alerting genuinely need streaming; a dashboard someone opens twice a day does not, and choosing streaming for it buys continuous operational burden in exchange for freshness nobody observes.

**How it fails**

**How stream processing goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Aggregates silently wrong | Windows keyed on arrival time | Use event time with watermarks |
| Late events discarded unnoticed | Default drop policy | Side outputs and explicit handling |
| State grows without limit | No expiry on keyed state | TTL on keys; bound the keyspace |
| Slow recovery after a restart | Very large checkpointed state | Smaller state; incremental checkpoints |
| Duplicate results downstream | Non-idempotent sink | Transactional or idempotent writes |
| Lag growing unnoticed | No monitoring of consumer lag | Alert on lag per partition |
| Deployment causes an outage | State restore time underestimated | Plan deployments around recovery cost |
| Results never final | No watermark, continuous updates | Define when a window is closed |

> **Unbounded keyed state fails slowly and then all at once**  
> A pipeline keyed on something with growing cardinality — session identifiers, user agents, URLs — accumulates state that nothing removes, and the job runs correctly for weeks while memory climbs. When it finally exceeds capacity the restart must restore an enormous checkpoint, which takes far longer than usual, extending a memory problem into a prolonged outage. Expiry has to be designed in from the start rather than added when it becomes urgent.

**Where it is used**

- **Fraud and anomaly detection**, where a decision is worthless once the transaction completes.
- **Real-time analytics dashboards**, updating continuously rather than on a schedule.
- **Materialised view maintenance**, keeping a derived store current as events arrive.
- **Monitoring and alerting pipelines**, aggregating metrics into conditions within seconds.
- **Personalisation**, updating recommendations from behaviour as it happens.

**In the interview**

Event time against processing time is the distinguishing topic; everything else follows from it.

- **Separate event time from arrival time** and say that arrival-time windows are wrong whenever events are delayed.
- **Explain watermarks as the latency/completeness dial**, not as a technical detail.
- **State the late-event policy** you would choose and why, since the default is silent dropping.
- **Qualify exactly-once**: it covers pipeline state, and reaches the output only with a transactional or idempotent sink.
- **Raise state size and restart cost**, because they make deployments materially different from batch.

**Practice drill**

Detect more than five transactions per card in one minute, from a stream where mobile events can be delayed by up to ten minutes. Choose the window type, the time semantics and the watermark, and state the resulting alert latency. Define what happens to an event arriving twelve minutes late. Then estimate the state size for ten million cards and describe what a restart costs at that size.

**Go deeper**

Stream processing computes continuously over unbounded input, maintaining state so that results are available within seconds of the events that produced them.

**It exists for answers whose value decays quickly.** Fraud detection, live alerting and real-time personalisation are worth much less an hour later, and shortening a batch interval does not reach that latency — the floor is the interval plus the runtime, and per-run overheads dominate as the interval shrinks. Keeping the computation running removes the interval entirely, at the cost of a system that must be operated continuously.

**The distinction between event time and processing time is the core difficulty.** Events arrive late, out of order, and in bursts when devices reconnect, so grouping by arrival time produces aggregates that are quietly wrong. Event-time processing is correct and requires deciding how long to wait for stragglers — a watermark — which is a direct trade between how fresh results are and how complete they are, with no setting that optimises both.

**Late events need an explicit policy because the default is silent loss.** An event arriving after its window has closed is dropped unless something else is specified, which produces under-counts nobody notices. Collecting late events to a side output, or emitting corrections that downstream consumers must handle as retractions, are the alternatives — each with real cost, and each better than an unexamined default.

**State is what makes streaming powerful and what makes it operationally heavy.** Keyed state enables aggregation, joins and pattern detection, and it must be checkpointed durably so failures do not lose it. That makes restarts expensive in proportion to state size, so a deployment is not the trivial operation it is for a batch job, and a pipeline with very large state can take minutes to become healthy after any change.

**Unbounded state is the failure that arrives slowly.** Keying on something with growing cardinality accumulates entries that nothing removes, and the pipeline behaves perfectly while memory climbs over weeks. The eventual failure is worse than an ordinary one because recovery must restore an oversized checkpoint, turning a capacity problem into an extended outage — which is why expiry belongs in the initial design rather than in the incident review.

**Exactly-once is real but narrower than it sounds.** Frameworks advance state and input offsets together so that internal recovery does not double-count, which is a genuine and valuable guarantee. It extends to the outside world only when the sink is transactional or idempotent; otherwise a recovery re-emits results that were already applied. Stating the guarantee with its boundary is the difference between an accurate description and a marketing one, and it determines whether the consumer needs deduplication of its own.

**Related patterns:** Batch Processing · Partitioned Log · Materialized View · Change Data Capture

---
