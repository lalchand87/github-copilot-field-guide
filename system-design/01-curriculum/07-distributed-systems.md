# Curriculum · Distributed Systems

[← System Design index](../README.md)

> 8 lessons in **Distributed Systems**. Part of the Curriculum — Build the fundamentals.

## Contents

- **Distributed Systems** (8): [CAP during partitions](#cap-during-partitions) · [Consistency models](#consistency-models) · [Leader-based replication](#leader-based-replication) · [Quorum reads and writes](#quorum-reads-and-writes) · [Consensus and Raft](#consensus-and-raft) · [Leases and fencing tokens](#leases-and-fencing-tokens) · [Logical clocks and causal order](#logical-clocks-and-causal-order) · [CRDTs](#crdts)

## Distributed Systems

### CAP during partitions

*When the network splits, every operation must choose: reject and stay correct, or accept and risk divergence. CAP is a statement about that moment, not about your architecture.*

**Flow:** `Network split` → `Operation policy` → `Quorum path` → `Local path` → `Reconciliation`

> **The 30-second version**  
> During a partition, an operation either refuses without a quorum or proceeds locally and diverges. Choose per operation from the cost of divergence, enforce with quorums and fencing, and design the reconciliation before you need it.

**The problem**

CAP is the most cited and least usefully applied result in distributed systems. It is usually recited as “pick two of consistency, availability and partition tolerance,” which is misleading — partitions are not a design choice, they are a property of networks. You do not choose P; you experience it.

The useful statement is narrower and sharper: **during a network partition**, an operation may either refuse to proceed without contacting the other side (giving up availability) or proceed on local information (giving up consistency). There is no third option, and the choice is made per operation, not per system.

> **What the theorem actually constrains**  
> It says nothing about performance, nothing about normal operation, and nothing about your database's marketing label. It says that in the specific moment when two parts of your system cannot communicate, a request arriving at one side either waits or fails (CP) or answers from what it can see (AP). Everything interesting is in deciding which, for which operation, and what happens afterwards.

**Mental model**

Picture two data centres with a severed link. Both are alive. Both are receiving requests. Neither can tell whether the other is down or merely unreachable — which is the whole difficulty, because those two situations require opposite responses.

1. **Partition** — Some nodes cannot communicate with others, while all remain alive and serving. Not a crash — a split.
2. **CP choice** — Refuse operations that cannot be confirmed by a quorum. The minority side becomes unavailable; correctness is preserved.
3. **AP choice** — Accept operations locally on both sides. Everyone stays available; the two sides diverge and must be reconciled.
4. **Reconciliation** — The AP side's real work: merging divergent state after the partition heals, via last-writer-wins, CRDTs, or human intervention.
5. **Per-operation choice** — Reads and writes, and different endpoints, can make different choices in the same system. This is the practical unit.

> **The indistinguishability that makes it hard**  
> A node cannot distinguish “the other side is down” from “I cannot reach the other side.” If it assumes the other side is down and proceeds, and the other side is making the same assumption, you have split brain: two sides both believing they are authoritative, both accepting writes to the same entities. That is the failure CP designs exist to prevent, and it is why they are willing to be unavailable.

**How it works**

**The same partition, two policies**

```text
SETUP  5 replicas, network splits into {A,B,C} and {D,E}

CP POLICY (quorum required, majority = 3)
  writes to A/B/C: succeed (quorum of 3 available)
  writes to D/E:   REJECTED (cannot reach a majority)
  reads from D/E:  rejected, or served as explicitly stale
  -> the minority side is unavailable for writes
  -> no divergence; nothing to reconcile when it heals

AP POLICY (accept locally)
  writes to A/B/C: succeed
  writes to D/E:   succeed
  -> both sides available throughout
  -> the SAME key may now have two different values
  -> on heal: conflict resolution required

WHAT DECIDES IT
  not the database, not the architecture - the CONSEQUENCE
  of being wrong for this particular operation.
```

1. **Decide per operation, from the consequence of divergence** — Accepting a duplicate comment is harmless. Accepting two conflicting balance deductions is not. The same system should choose differently for each.
2. **Use quorums to make the CP choice concrete** — A majority is reachable from at most one side of any partition, which is exactly what prevents split brain. Requiring a quorum is how “refuse without confirmation” is implemented.
3. **Fence the minority explicitly** — A demoted or partitioned leader must be prevented from acting, not merely asked to stop. Leases with epochs and storage-level fencing tokens are how this is enforced.
4. **Design the AP reconciliation before you need it** — “We'll merge later” is not a design. Last-writer-wins silently discards data; CRDTs constrain the operations you may use; human reconciliation needs tooling and a queue.
5. **Degrade rather than fail wholesale** — A CP system need not return errors for everything. Serving reads as explicitly stale, or allowing a bounded overdraft, are legitimate partial-availability designs.
6. **Remember PACELC's second half** — Even with no partition, you still trade latency against consistency on every request. That trade is present all the time; CAP's is present only during failures.

**Quorums prevent split brain by arithmetic**

```text
N = 5 replicas, write quorum W = 3, read quorum R = 3
W + R > N  ->  3 + 3 > 5  ->  read and write sets OVERLAP
                               -> a read always sees the
                                  latest committed write

DURING A PARTITION {A,B,C} | {D,E}
  side 1 has 3 nodes -> can form a quorum -> writes succeed
  side 2 has 2 nodes -> cannot           -> writes rejected

ONLY ONE SIDE CAN EVER HAVE A MAJORITY.
That is the whole mechanism. It is not clever; it is counting.

W=1, R=1 (AP-style)
  both sides accept writes -> divergence -> reconciliation
  W + R = 2, not > 5 -> no overlap guarantee -> stale reads
```

> **Most real partitions are partial and asymmetric**  
> The textbook picture is a clean split into two groups. Reality includes one-way links where A can reach B but not the reverse, gray failures where packets are delayed rather than dropped, and a node partitioned from its peers but not from clients — which keeps serving requests while being unable to coordinate. These are harder than clean splits precisely because failure detectors behave inconsistently, and they are why fencing matters more than detection.

**Worked example**

A multi-region e-commerce system, deciding CAP policy per operation rather than per system.

**Per-operation CAP decisions**

```text
OPERATION            PARTITION BEHAVIOUR     WHY
-------------------  ----------------------  --------------------
browse catalogue     AP - serve stale        a stale price for
                                             60s costs little;
                                             unavailability costs
                                             the whole session

add to cart          AP - accept locally     carts are per-user,
                                             merge on heal is
                                             trivial (union)

check inventory      AP - stale, with a      approximate stock is
                     "may be unavailable"    acceptable to display
                     caveat

RESERVE inventory    CP - reject if no       two reservations of
                     quorum                  the last item is an
                                             oversell -> refund,
                                             apology, lost trust

take payment         CP - reject             double charge is a
                                             financial incident

write order          CP - reject             an order that exists
                                             in one region only
                                             is a support ticket

THE SHAPE THAT EMERGES
  95% of traffic (browse, cart) stays available
  the 5% that must be correct fails cleanly with a
  clear message
  -> "unavailable" for a narrow slice, not for the site
```

| Metric | Value | Note |
|---|---|---|
| Browse / cart | AP | 95% of traffic |
| Reserve / pay | CP | **5%, must be right** |
| User impact | partial | not an outage |
| Reconciliation | cart union | trivial by design |

> **The interesting design is making the CP surface small**  
> A system that is CP everywhere is unavailable during every partition. A system that is AP everywhere corrupts. The good design identifies the narrow set of operations where divergence is genuinely unacceptable — allocation, payment, identity — and makes everything else available. That usually means 95% of traffic keeps working while a specific, explainable slice fails, which is a vastly better outcome than either extreme.

**When to use it**

- **Choosing CP** for allocation, payment, identity, uniqueness and anything where divergence means money or data corruption.
- **Choosing AP** for browsing, search, recommendations, social content, telemetry and anything where staleness is cheaper than downtime.
- **Reasoning about multi-region designs**, where partitions are frequent enough to be a design input rather than an edge case.
- **Evaluating a datastore's behaviour**, by asking what it does to a minority partition rather than what its marketing says.
- **Writing degradation policy**, so on-call engineers know what is expected to fail and what is not.

**When to avoid it**

- **Do not treat CAP as a system-level label.** “We're an AP system” is almost always wrong; real systems make the choice per operation.
- **Do not choose AP without designing reconciliation**, which is where the actual work lives.
- **Do not assume a partition is a clean split**; partial and asymmetric failures are more common and harder.
- **Do not rely on failure detection alone** to prevent split brain — a paused node will wake up and act. Fence it.
- **Do not cite CAP to justify weak consistency in the absence of partitions**; that is PACELC's latency trade, a different argument.

**Advantages**

- **Per-operation choice keeps most traffic available** while protecting the operations that matter.
- **Quorums implement the CP choice with simple arithmetic** rather than complex agreement protocols at the application layer.
- **Explicit AP designs with CRDTs** give availability with automatic, correct convergence for suitable operations.
- **Stating the policy in advance** turns a partition from an improvisation into a documented, expected behaviour.

**Disadvantages**

- **CP means real unavailability** for the minority, which is a genuine cost to users and revenue.
- **AP means reconciliation**, which is engineering work plus, often, unresolvable conflicts requiring human judgement.
- **The choice is invisible until a partition occurs**, so systems ship with it undecided and discover it under pressure.
- **Partial partitions defeat simple reasoning**, and detection-based approaches misbehave under them.
- **Last-writer-wins looks like reconciliation** but is silent data loss with a timestamp attached.

**Trade-offs**

**CP versus AP, concretely**

|  | CP | AP |
|---|---|---|
| During a partition | Minority rejects operations | Both sides accept |
| Correctness | Preserved | Divergence, then reconciliation |
| User experience | Errors for some requests | Works, possibly on stale data |
| After healing | Nothing to do | Merge; possibly manual |
| Implementation | Quorums, leases, fencing | Conflict resolution, CRDTs, versioning |
| Fits | Allocation, payment, identity | Content, search, carts, telemetry |

> **The sentence that shows real understanding**  
> “CAP applies only during a partition, and it is an operation-level decision. I'd keep browse and cart available on stale data, and make inventory reservation and payment require a quorum — so a partition costs us a narrow slice of functionality rather than the whole site, and there's nothing to reconcile for the operations where reconciliation would be unacceptable.”

**How it fails**

**Partition-related failures**

| Failure | Mechanism | Prevention |
|---|---|---|
| Split brain | Both sides believe they are authoritative | Quorum requirement; only one side can have a majority |
| Zombie leader writes after demotion | Paused node resumes and acts on a stale belief | Leases with epochs; storage rejects stale fencing tokens |
| Silent data loss after reconciliation | Last-writer-wins discarded a concurrent update | Version vectors; CRDTs; surface conflicts rather than resolving blindly |
| Whole site down for a partial failure | System-wide CP policy | Per-operation policy; degrade rather than fail |
| Oversold inventory | AP policy on an allocation operation | CP for allocation; or a reserved buffer with explicit overbooking |
| Partition undetected | Asymmetric or gray failure | Fencing rather than detection; timeouts on both directions |
| Cascading failure on heal | Backlog of divergent writes merged at once | Rate-limited reconciliation; bounded divergence windows |

**Limits**

> **Facts worth carrying**
>
> - **Quorum**: with N replicas, a majority is ⌊N/2⌋+1, and at most one partition side can hold one.
> - **W + R > N** guarantees read/write overlap and therefore that a read sees the latest committed write.
> - **Partitions are common at multi-region scale** — plan for them as routine rather than exceptional.
> - **PACELC**: else, latency versus consistency — the trade that applies every day, not just during failures.
> - **Fencing tokens are required**, because no failure detector can prevent a paused node from waking and acting.

**Alternatives**

| Approach | Partition behaviour | Cost |
|---|---|---|
| Quorum-based CP | Minority unavailable | Real downtime for some users |
| Leader with lease and fencing | Minority rejects; failover after lease expiry | Unavailability window equal to lease |
| AP with last-writer-wins | Both available | Silent loss of concurrent updates |
| AP with CRDTs | Both available, converges correctly | Restricted operations; metadata growth |
| AP with manual reconciliation | Both available | Human queue; does not scale |
| Single region | No partition between regions | No geographic redundancy |

The last row deserves honesty: much of CAP's difficulty is self-inflicted by distributing data that did not need distributing. Keeping a strongly consistent core in one region, with read replicas elsewhere, removes the hardest partitions from the design entirely — at the cost of regional availability for writes.

**In real systems**

- **ZooKeeper, etcd and Consul** are deliberately CP: they refuse writes without a quorum, because their job is to be the authority that prevents split brain elsewhere.
- **Dynamo-style stores** default toward AP with tunable quorums, making the choice per-request rather than per-system.
- **Shopping carts** are the canonical AP example, because merging two divergent carts by union is nearly always acceptable to the user.
- **Payment and ledger systems** are uncompromisingly CP, accepting regional unavailability rather than risking double-spend.
- **Spanner** uses synchronised clocks and consensus to offer strong consistency across regions, paying for it in write latency — an explicit choice on the PACELC axis.

**Common mistakes**

- **Treating CAP as a system-wide label** rather than a per-operation decision.
- **Choosing AP without designing reconciliation**, then discovering conflicts have no resolution.
- **Relying on failure detection** to prevent split brain instead of fencing.
- **Using last-writer-wins** and calling the resulting data loss “eventual consistency”.
- **Assuming clean partitions**, when partial and asymmetric failures are more common.
- **Making the entire system CP**, so every partition is a full outage.
- **Citing CAP to justify weak consistency during normal operation**, which is a latency argument, not a partition one.

**The staff-level view**

CAP's practical value is not the theorem but the discipline of deciding, in advance and per operation, what the system does when it cannot agree.

- **Write the partition policy down per operation**, in the same document as the requirements. It is the degradation runbook, and improvising it during an incident goes badly.
- **Push the CP surface as small as it can be.** Most operations do not need agreement; identifying the few that do is the design work.
- **Insist on fencing, not detection.** Every ownership or leadership scheme needs epochs enforced at the storage layer, because a paused process will wake up and act.
- **Treat last-writer-wins as data loss**, and require an explicit decision to accept it rather than allowing it as a default.
- **Reframe vendor claims as behaviour questions**: not “is this AP or CP” but “what happens to a write on the minority side of a partition, and what happens when it heals”.

**Go deeper**

CAP is not “pick two” — partitions are a property of networks, not a design option. The real statement is that *during a partition*, an operation either requires confirmation from nodes it cannot reach and becomes unavailable, or proceeds locally and risks divergence. The choice is per operation, not per system, and it matters only during failures.

The CP choice is implemented by quorums: with N replicas a majority requires more than half, so at most one side of any partition can proceed, which prevents split brain by arithmetic rather than by detection. It also needs fencing — leases with monotonic epochs enforced at the storage layer — because a paused node will wake and act on a stale belief regardless of what any failure detector concluded.

The AP choice requires reconciliation, and that is where the work is. Last-writer-wins is silent data loss with a timestamp attached; CRDTs converge correctly but constrain the operations you may use; manual resolution needs tooling. The good design keeps the CP surface small — allocation, payment, identity — so a partition costs a narrow, explainable slice of functionality while browse, search and carts stay available.

CAP is cited more often than it is applied, largely because the popular framing obscures what it constrains. Partition tolerance is not a choice; networks partition. The theorem is about a specific moment, and the decision it forces is per-operation.

**The moment it describes.** When some nodes cannot reach others while all remain alive and serving, a request arriving at either side must either wait for confirmation it cannot obtain — becoming unavailable — or answer from local state, allowing the two sides to diverge. The difficulty underneath is indistinguishability: a node cannot tell “the other side is down” from “I cannot reach the other side,” and those situations demand opposite responses. If both sides assume the other is down and proceed, you have split brain.

**Quorums are the mechanism, and they are just counting.** With N replicas, a majority is more than half, and at most one side of any partition can contain one. Requiring a majority for writes therefore makes split brain structurally impossible, without any need to detect the partition. Setting read and write quorums so W + R > N additionally forces read and write sets to overlap, guaranteeing a read observes the latest committed write. The simplicity is the point — no protocol subtlety to get wrong.

**Fencing, not detection.** Quorums alone do not stop a node acting on stale belief. A leader paused by a long garbage collection or briefly partitioned loses its lease, ownership moves, and then it resumes and completes work it began — with no knowledge anything changed. Failure detectors cannot help, because the issue is not noticing the failure but preventing the effects of an actor who believes it is still valid. Every ownership scheme therefore needs monotonically increasing epochs carried with each write and enforced at the storage layer, which rejects anything older than the highest it has seen.

**AP's real cost is reconciliation.** Choosing availability means accepting that the same entity may hold different values on each side, and the merge strategy is the design. Last-writer-wins is the default in many systems and is silent data loss: one concurrent update is discarded, with a timestamp making it look principled. CRDTs converge correctly and automatically but restrict you to operations with the right algebraic properties. Manual reconciliation needs a queue, tooling and human judgement, and does not scale. Choosing AP without specifying which of these applies is deferring the decision to an incident.

**The design work is shrinking the CP surface.** A system that requires agreement everywhere is unavailable during every partition; one that requires it nowhere corrupts. The valuable exercise is identifying the narrow set of operations where divergence is genuinely unacceptable — allocation, payment, uniqueness, identity — and making everything else available on local or stale data. In a typical consumer system that leaves the large majority of traffic working during a partition while a specific, explainable slice fails, which is far better than either extreme. And that policy belongs in the requirements document, because it is the degradation runbook and improvising it under pressure reliably goes badly.

**PACELC is the half that applies daily.** Even with no partition, every request trades latency against consistency: a strongly consistent read across regions costs a cross-region round trip. That trade is present all the time, whereas CAP's is present only during failures — which means most teams citing CAP are in fact making PACELC decisions, and would reason better if they said so.

**Prove it — interview questions**

1. **[Basic] What does CAP actually say?**

   <details><summary>Model answer</summary>

   That during a network partition, an operation must choose between remaining available and remaining consistent. If it requires confirmation from nodes it cannot reach, it becomes unavailable; if it proceeds on local information, the two sides can diverge. Partition tolerance is not a choice — networks partition — so the meaningful decision is between the other two, and only during the partition. It says nothing about normal operation or performance.

   </details>

2. **[Basic] Why can't both sides of a partition just keep accepting writes?**

   <details><summary>Model answer</summary>

   They can, and that is the AP choice — but then the same entity can have two different values, and when the partition heals you must reconcile them. For a shopping cart that merge is trivial and acceptable. For an inventory reservation or a payment it is not: two sides each allocating the last item produces an oversell, and no merge strategy recovers the item. The cost of divergence for that specific operation is what decides whether AP is acceptable.

   </details>

3. **[Senior] How do quorums implement the CP choice?**

   <details><summary>Model answer</summary>

   By arithmetic: with N replicas, a majority requires more than half, and at most one side of any partition can hold a majority. So requiring a quorum for writes means only one side can proceed, which structurally prevents split brain without needing to detect anything. Setting read and write quorums so that W plus R exceeds N additionally guarantees that read and write sets overlap, meaning a successful read always observes the most recent committed write. It is not a clever protocol — it is counting — which is why it is so reliable.

   </details>

4. **[Senior] Why is fencing necessary if you already detect failures?**

   <details><summary>Model answer</summary>

   Because detection cannot prevent a node from acting on a stale belief. A leader paused by a long garbage collection or partitioned from its peers loses its lease, ownership moves, and then it wakes up and completes the write it had started — with no knowledge that anything changed. No failure detector helps, because the problem is not detecting the failure but stopping the effects of an actor that believes it is still valid. Fencing solves it by giving each ownership term a monotonically increasing epoch that travels with every write, and having the storage layer reject anything carrying an older epoch.

   </details>

5. **[Staff] How would you apply CAP to a multi-region e-commerce system?**

   <details><summary>Model answer</summary>

   Per operation, chosen from the consequence of divergence rather than as a system-wide label. Browsing, search and cart operations stay available on stale or local data, because a stale price for a minute costs far less than losing the session, and carts merge by union trivially. Inventory reservation, payment capture and order creation require a quorum and fail cleanly on the minority side, because two concurrent allocations of the last item, a double charge, or an order existing in only one region are all outcomes no reconciliation strategy recovers from. The shape that emerges is that around ninety-five per cent of traffic continues working during a partition while a narrow, explainable slice returns a clear error — which is a much better outcome than either being fully unavailable or silently diverging. I would also write that policy down alongside the requirements, because it is the degradation runbook.

   </details>

6. **[Principal] What is wrong with how CAP is usually taught, and what would you teach instead?**

   <details><summary>Model answer</summary>

   The “pick two” framing is actively misleading, because it implies partition tolerance is optional when it is simply a property of networks, and it implies the choice is architectural when it is per-operation and only relevant during a failure. Teams then label their system AP or CP, stop thinking, and are surprised by both the unavailability and the divergence. I would teach it as three questions instead. First: for this specific operation, what happens if two sides both proceed — is the outcome a merge, a refund, or a regulatory incident? That determines the choice. Second: how is the CP choice enforced — quorums plus leases plus fencing at the storage layer, not detection. Third: if AP, what exactly is the reconciliation, written down and implemented, because last-writer-wins is silent data loss wearing a respectable name. And I would add PACELC's second half explicitly, since the latency-versus-consistency trade applies on every request of every normal day, while CAP's applies only during failures — which means most teams are actually making PACELC decisions while citing CAP.

   </details>

---

### Consistency models

*A spectrum of promises about what a read may observe, from strict linearizability to eventual convergence — each removing an anomaly at a measurable cost.*

**Flow:** `Write` → `Replication` → `Read policy` → `Observed version` → `Client history`

> **The 30-second version**  
> Each consistency model forbids a specific set of anomalies at a specific cost. Choose per operation: linearizability for uniqueness and allocation, session guarantees for anything a user watches, eventual for everything else.

**The problem**

A user updates their profile and immediately reloads the page — and sees the old value. A counter shows a number that goes backwards. Two services reading the same data at the same moment disagree. Each of these is a different anomaly, permitted by a different consistency model, and “the database is eventually consistent” explains none of them precisely.

Without a vocabulary for exactly which anomalies are possible, teams either over-engineer — paying cross-region coordination for a profile picture — or under-engineer, discovering that their “eventually consistent” store permits a read to move backwards in time.

> **Consistency models are anomaly lists**  
> Each model is defined by what it *forbids*. Linearizability forbids observing a stale value after a newer one has been acknowledged anywhere. Monotonic reads forbid time going backwards for one client. Read-your-writes forbids a client failing to see its own update. Choosing a model is choosing which anomalies your users and your code must tolerate.

**Mental model**

Order the models by how much they constrain the set of possible histories. Stronger models permit fewer histories, cost more coordination, and require less defensive application code.

1. **Linearizability** — Every operation appears to take effect instantaneously at some point between its start and end, and all clients agree on that order. The strongest single-object guarantee, and the most expensive.
2. **Sequential consistency** — All clients see operations in the same order, but that order need not match real time. Cheaper; permits a read to miss a write that happened earlier by the clock.
3. **Causal consistency** — Operations causally related are seen in order by everyone; concurrent operations may be seen in different orders. Often the sweet spot.
4. **Read-your-writes / monotonic reads** — Per-client guarantees: you see your own writes; time does not move backwards for you. Cheap and disproportionately valuable for user experience.
5. **Eventual consistency** — Replicas converge if writes stop. Forbids almost nothing in the interim — including reads moving backwards.

> **Eventual consistency permits far more than people assume**  
> Read two replicas and see different values: allowed. Read the same replica twice and see a newer value then an older one: allowed, if you were routed to different nodes. See your own write disappear: allowed. Eventual consistency promises only convergence in the absence of new writes — every intermediate observation is unconstrained unless you add a session guarantee on top.

**How it works**

**What each model forbids**

```text
CLIENT A: write(x, 1) ... write(x, 2)
CLIENT B: read(x), read(x)

EVENTUAL          B may see: 2 then 1      (backwards!)
                             1 then 1
                             0 then 2
MONOTONIC READS   B never goes backwards: 1 then 2, or 2 then 2
READ-YOUR-WRITES  A, reading after its own write, sees >= 2
CAUSAL            if write(x,2) causally follows write(x,1),
                  nobody sees 2 before 1
SEQUENTIAL        all clients agree on one order of all ops
LINEARIZABLE      that order also respects real time: once
                  anyone reads 2, nobody may later read 1

COST ROUGHLY TRACKS THE LIST:
  per-client guarantees: routing/session token, near-free
  causal:  version metadata, modest
  sequential/linearizable: coordination per operation
```

1. **Start with the anomaly, not the model** — Ask what the user or the code cannot tolerate — seeing their own edit vanish, a count decreasing, two services disagreeing — and choose the weakest model forbidding it.
2. **Session guarantees are the highest-value cheap win** — Read-your-writes and monotonic reads are usually implemented by routing a client's reads to the replica that served its writes, or by carrying a version token. They eliminate the anomalies users actually notice.
3. **Causal consistency handles the “reply before the post” problem** — Where operations have real dependencies — a comment on a photo, a reply to a message — causal ordering prevents nonsensical observations without global coordination.
4. **Reserve linearizability for operations that need it** — Uniqueness checks, allocation, leader election, distributed locks. Applying it broadly means coordinating on every read.
5. **Quorums give tunable consistency per operation** — W + R > N gives read-your-writes across the replica set; W=1, R=1 gives speed and staleness. Deciding per request is usually better than per system.
6. **Make staleness bounded and visible** — If reads may be stale, bound it and expose it — “last updated 4s ago” — so users and downstream systems can reason about it rather than being silently misled.

**Session guarantees: cheap and disproportionately valuable**

```text
PROBLEM
  user edits their display name, page reloads,
  old name appears -> "the site is broken"
  (technically: read routed to a lagging replica)

FIX 1  STICKY ROUTING
  route this user's reads to the replica that took
  their writes, for a short window
  + trivial to implement
  - undermines load balancing; breaks on replica failure

FIX 2  VERSION TOKEN
  write returns a version/LSN; client sends it with reads
  replica either has caught up to that version, or
  forwards the read to one that has (or to the leader)
  + no sticky routing; survives failover
  + composable across services if the token propagates
  - requires replicas to expose and compare versions

FIX 3  READ FROM THE LEADER for a short window after a write
  + simplest correct option
  - leader load; does not compose across services

MOST USER-VISIBLE "CONSISTENCY BUGS" ARE THIS ONE PROBLEM.
```

> **Consistency is a per-read decision, not a database property**  
> The same store usually supports several points on the spectrum: read from the leader, read from a follower, read with a quorum, read with a version token. Treating consistency as something chosen per operation — expensive where it matters, cheap everywhere else — gives far better outcomes than picking one global setting.

**Worked example**

A social feed: which guarantee each operation needs, and what it costs.

**Assigning models by anomaly cost**

```text
OPERATION           MODEL              MECHANISM         COST
------------------  -----------------  ---------------- ------
view someone's      eventual           any replica       ~0
profile             (stale ok)

view YOUR OWN       read-your-writes   version token     ~0
profile after edit

feed timeline       monotonic reads    sticky replica    ~0
                    (must not jump
                     backwards)

comment appears     causal             version vector    low
after the post it
replies to

username claim      linearizable       consensus /       high
                                       unique index      (one op)

follower count      eventual           async aggregate   ~0
                    (approximate)

THE PATTERN
  exactly one operation needs strong consistency.
  everything else needs a session guarantee or nothing.
  -> total coordination cost is tiny, and every anomaly
     a user would actually notice is eliminated.
```

| Metric | Value | Note |
|---|---|---|
| Strong | 1 operation | username uniqueness |
| Session | 2 operations | **near-free** |
| Causal | 1 operation | version vectors |
| Eventual | the rest | no cost |

> **Users notice session anomalies, not global ones**  
> Nobody notices that two strangers' feeds are momentarily inconsistent with each other. Everybody notices when their own edit does not appear, or when a number they were watching goes backwards. That asymmetry means read-your-writes and monotonic reads — the cheapest guarantees on the spectrum — remove almost all of the perceived inconsistency, while expensive global guarantees remove anomalies no user was ever going to see.

**When to use it**

- **Linearizability** for uniqueness, allocation, locks, leader election and anything where two observers disagreeing causes a correctness failure.
- **Causal consistency** where operations have genuine dependencies — replies, edits, workflows — and nonsensical orderings would be visible.
- **Session guarantees** almost everywhere a human is looking at their own data. Cheap, and they remove the anomalies users notice.
- **Eventual consistency** for aggregates, counts, recommendations, search indexes and other people's data.
- **Per-operation choice**, using quorum settings or read routing rather than one global setting.

**When to avoid it**

- **Do not apply linearizability broadly**; it puts coordination on every read and its cost is most visible in the tail.
- **Do not assume eventual consistency includes session guarantees** — it does not, and reads can move backwards.
- **Do not describe a system as “eventually consistent” without stating the staleness bound**; unbounded staleness is not a design.
- **Do not rely on a single consistency setting** for a system with genuinely different operation classes.
- **Do not confuse consistency with isolation** — one is about what replicas show, the other about how concurrent transactions interleave.

**Advantages**

- **A precise vocabulary** turns vague arguments about staleness into decisions about specific anomalies.
- **Weak models are dramatically cheaper**, especially across regions where coordination costs a round trip.
- **Session guarantees are near-free** and eliminate most user-visible inconsistency.
- **Per-operation choice** lets a system pay for strength only where it is needed.
- **Bounded staleness can be exposed to users**, turning an invisible anomaly into understandable information.

**Disadvantages**

- **Weak models push complexity into application code**, which must tolerate anomalies the store permits.
- **Strong models cost latency**, and the cost is worst exactly when the network is degraded.
- **The vocabulary is genuinely subtle**, and terms are used inconsistently across vendor documentation.
- **Anomalies appear only under concurrency and failure**, so testing rarely surfaces them.
- **Mixing models across services** means the end-to-end guarantee is the weakest link, which is rarely analysed.

**Trade-offs**

**The spectrum**

| Model | Forbids | Typical cost |
|---|---|---|
| Linearizable | Any observation inconsistent with real-time order | Coordination per operation; cross-region round trip |
| Sequential | Clients disagreeing about order | Consensus, without real-time constraint |
| Causal | Seeing an effect before its cause | Version vectors; metadata growth |
| Read-your-writes | A client not seeing its own write | Routing or a version token |
| Monotonic reads | A client's view moving backwards | Sticky routing or a token |
| Eventual | Permanent divergence only | None |

> **Framing it well**  
> “Only the username claim needs linearizability — everything else needs read-your-writes for the user's own data and eventual for everyone else's. That means one coordinated operation and near-zero cost elsewhere, while still removing every anomaly a user would actually notice.”

**How it fails**

**Consistency-related failures**

| Symptom | Model gap | Fix |
|---|---|---|
| User's own edit does not appear | No read-your-writes | Version token on reads, or read from the leader briefly |
| Count goes backwards | No monotonic reads | Sticky routing or a session token |
| Reply visible before the post | No causal ordering | Version vectors; or fetch dependencies together |
| Two services disagree and both act | Eventual consistency on a decision path | Linearizable read, or a single owner for the decision |
| Duplicate usernames | Uniqueness enforced without coordination | Unique index or consensus-backed allocation |
| Stale data with no bound | Unbounded replication lag | Monitor and bound lag; fail reads exceeding the bound |
| End-to-end guarantee weaker than expected | Strong service reading from a weak one | Analyse the whole chain; the weakest link governs |

> **The weakest link governs the chain**  
> A service offering linearizable reads that internally reads from an eventually consistent cache or replica provides eventual consistency to its callers, regardless of its own protocol. End-to-end guarantees must be traced through every hop — including caches, read replicas and downstream services — because the guarantee a client experiences is the weakest one on the path, and it is almost never the one documented.

**Limits**

> **Practical numbers**
>
> - **Cross-region linearizable write**: 50–150 ms, dominated by the round trip; often unacceptable for interactive paths.
> - **Replication lag** on healthy asynchronous replicas: typically milliseconds to seconds; it is the staleness bound to monitor.
> - **W + R > N** gives read-your-writes within a replica set without leader reads.
> - **Session guarantees** cost essentially nothing when implemented with a version token.
> - **Causal metadata** grows with the number of participants; version vectors are not free at large scale.

**Alternatives**

| Mechanism | Gives | Cost |
|---|---|---|
| Read from leader | Read-your-writes, linearizable-ish | Leader load; no scaling |
| Quorum reads/writes | Overlap guarantee | Latency of the slowest quorum member |
| Version token in the request | Session guarantees across services | Token propagation plumbing |
| Sticky routing | Session guarantees, simply | Breaks load balancing and failover |
| Consensus (Raft/Paxos) | Linearizability | Coordination per operation |
| CRDTs | Convergence without coordination | Restricted operations; metadata |

Version tokens are the underrated option: because the token can be propagated across service boundaries, they give end-to-end session guarantees in a way sticky routing cannot, and they survive failover because any sufficiently caught-up replica can serve the read.

**In real systems**

- **Read replicas everywhere** provide eventual consistency by default, which is why read-your-writes anomalies are the most common user-visible consistency bug in ordinary web applications.
- **DynamoDB's eventually and strongly consistent reads** make the per-operation choice explicit, with a documented cost difference.
- **Spanner** provides external consistency — effectively linearizability across regions — using synchronised clocks, and pays for it in commit latency.
- **Session consistency in Cosmos DB and similar stores** is offered as a named tier precisely because it is the level most applications actually want.
- **CRDT-based collaborative editors** choose convergence over coordination, accepting restricted operations in exchange for offline editing and no round trips.

**Common mistakes**

- **Assuming eventual consistency prevents reads going backwards**; it does not.
- **Applying strong consistency globally**, paying coordination on every read.
- **Ignoring session guarantees**, then fielding bug reports about edits not appearing.
- **Describing staleness as “eventual” with no bound or monitoring.**
- **Confusing consistency with transaction isolation**, which are different axes.
- **Failing to trace the whole chain**, so a strong service silently offers weak guarantees.
- **Testing only sequentially**, where none of these anomalies can occur.

**The staff-level view**

The value of consistency models is precision: they convert “is this consistent enough?” into “which of these specific anomalies can occur, and which can we tolerate?”

- **Ask for the anomaly, not the model.** “What would a user see that they shouldn't?” produces a better decision than “should this be strongly consistent?”
- **Make session guarantees a platform default.** Read-your-writes and monotonic reads remove almost all perceived inconsistency and cost essentially nothing when done with version tokens.
- **Trace end-to-end guarantees through every hop.** A linearizable service reading from a cache offers cache consistency to its callers, and nobody notices until it matters.
- **Bound and expose staleness.** An unbounded “eventually” is not a design; a monitored lag with a documented ceiling is.
- **Keep the strongly consistent set small and named.** Uniqueness, allocation, locks, leadership — enumerate them, and let everything else be cheap.

**Go deeper**

Consistency models are best understood as lists of forbidden anomalies. Eventual consistency forbids only permanent divergence — reads can move backwards, replicas can disagree, and a client can fail to see its own write. Read-your-writes and monotonic reads forbid the per-client anomalies. Causal consistency forbids seeing an effect before its cause. Sequential and linearizable consistency force all clients to agree on one order, with linearizability additionally requiring that order to respect real time.

Cost rises steeply along that spectrum, but perceived benefit does not. Users only notice inconsistency relative to their own actions: nobody sees that two strangers' feeds differ, while everybody notices their own edit vanishing or a count decreasing. So the cheapest guarantees — session guarantees implemented with a version token returned by writes and carried on reads — remove almost all user-visible inconsistency, while expensive global coordination eliminates anomalies nobody was positioned to observe.

Assign models per operation rather than per system: linearizability for uniqueness, allocation, locks and leadership; causal where operations genuinely depend on each other; session guarantees for a user's own data; eventual for everything else. And trace the end-to-end path, because a linearizable service reading from a cache provides cache consistency to its callers — the weakest link governs, and it is rarely the one documented.

Consistency models give a precise vocabulary for what a read may observe, which converts vague arguments about staleness into decisions about specific, enumerable anomalies.

**The spectrum, by what each forbids.** Eventual consistency forbids only permanent divergence; every intermediate observation is permitted, including a single client reading a newer value and then an older one after being routed to a different replica. Monotonic reads forbid that backwards movement for one client. Read-your-writes forbids a client missing its own update. Causal consistency forbids observing an effect before its cause, while leaving concurrent operations unordered. Sequential consistency forces all clients to agree on one order. Linearizability additionally requires that order to respect real time — once any client has observed a new value, no client may subsequently observe the old one — which is what makes it expensive, because it demands coordination at the moment of the operation rather than agreement after the fact.

**Cost and perceived benefit diverge sharply.** Coordination cost climbs steeply toward the strong end, and across regions a linearizable operation costs a full round trip. But human perception of inconsistency is almost entirely relative to one's own actions and one's own continuous observation: nobody notices that two strangers' views differ, while everybody notices their own edit failing to appear or a number moving backwards. That asymmetry means the two cheapest guarantees on the spectrum eliminate most of the *felt* inconsistency, which is an unusually favourable trade.

**Session guarantees are the highest-value implementation detail.** Sticky routing — sending a client's reads to the replica that served its writes — is trivial but undermines load balancing and breaks on failover. A version token returned by the write and carried on subsequent reads is better: any replica caught up to that version may serve the read, it survives failover, and crucially it composes across service boundaries if the token is propagated. Most user-visible consistency bugs in ordinary applications are this single problem.

**Assign per operation.** In a typical social application, exactly one operation — claiming a unique username — requires linearizability, because two clients succeeding is a correctness failure with no reconciliation. Comment-after-post needs causal ordering. A user's own profile needs read-your-writes; a timeline needs monotonic reads. Everything else can be eventually consistent. The total coordination cost is therefore tiny, while every anomaly a user could notice is eliminated — an outcome unreachable by choosing a single global setting.

**The weakest link governs, and nobody owns it.** A service documented as linearizable that internally reads from a cache or an asynchronous replica delivers cache consistency to its callers. End-to-end guarantees must be traced through every hop, including caches, replicas and downstream services, because the guarantee that reaches the user is the weakest on the path and it is almost never the one written down. Organisationally, the failure is rarely a team choosing the wrong model in isolation; it is that no one owns the composed guarantee. Making consistency a stated part of each service's contract, propagating session tokens across boundaries, and maintaining a short named list of operations that genuinely require strong consistency is what turns this from folklore into design.

**Prove it — interview questions**

1. **[Basic] What does eventual consistency actually guarantee?**

   <details><summary>Model answer</summary>

   Only that replicas converge to the same value if writes stop. In the interim, almost nothing is forbidden: two replicas can show different values, a client reading twice can see a newer value and then an older one if routed to different nodes, and a client can fail to see its own write. People assume it includes session guarantees, and it does not — those have to be added deliberately on top.

   </details>

2. **[Basic] What is read-your-writes and why does it matter?**

   <details><summary>Model answer</summary>

   It is the guarantee that a client always observes its own previous writes. It matters because it eliminates the single most common user-visible consistency complaint: editing something, reloading, and seeing the old value because the read was routed to a lagging replica. It is also among the cheapest guarantees available — implemented by routing the client's reads to a replica that has caught up, or by returning a version token from the write and requiring reads to meet it.

   </details>

3. **[Senior] How does linearizability differ from sequential consistency?**

   <details><summary>Model answer</summary>

   Both require all clients to agree on a single order of operations. Linearizability additionally requires that order to respect real time: if a write completes and is acknowledged, no subsequent read anywhere may return the older value. Sequential consistency permits an agreed order that does not match wall-clock ordering, so a read that began after a write completed may still miss it as long as everyone is consistent about it. That real-time constraint is what makes linearizability expensive, because it requires coordination at the moment of the operation rather than merely agreement after the fact.

   </details>

4. **[Senior] Why do users notice some inconsistencies and not others?**

   <details><summary>Model answer</summary>

   Because people only perceive inconsistency against their own actions and their own continuous observation. Nobody notices that two strangers' feeds momentarily differ. Everybody notices when their own edit vanishes, or when a number they are watching decreases. That means the two cheapest guarantees on the spectrum — read-your-writes and monotonic reads, both per-client — remove nearly all perceived inconsistency, while expensive global guarantees eliminate anomalies no user was ever positioned to observe. It is a rare case where the cheap option is also the one that matters.

   </details>

5. **[Staff] How would you assign consistency models across a social application?**

   <details><summary>Model answer</summary>

   By anomaly cost, per operation. Viewing another user's profile can be eventually consistent, since staleness of a few seconds is invisible. Viewing your own profile after an edit needs read-your-writes, which I would implement with a version token rather than sticky routing so it survives failover and composes across services. A feed needs monotonic reads, because a timeline that jumps backwards is jarring. A comment appearing before the post it replies to needs causal ordering. And exactly one operation — claiming a username — needs linearizability, because two people succeeding is a correctness failure with no reconciliation. The result is one coordinated operation and near-zero cost everywhere else, while every anomaly a user would actually notice is gone.

   </details>

6. **[Principal] How do you reason about end-to-end consistency across many services?**

   <details><summary>Model answer</summary>

   By tracing the weakest link, because the guarantee a client experiences is the weakest one on the whole path and it is almost never the one documented on any individual service. A service offering linearizable reads that internally serves from a cache or a read replica is providing cache consistency to its callers, and that gap goes unnoticed until a correctness incident. So the practical discipline is to make consistency a stated property in each service's contract, including what it reads from, and to propagate session tokens across service boundaries so read-your-writes survives more than one hop — which sticky routing cannot do. At an organisational level I would also enumerate and name the small set of operations that genuinely require strong consistency, because that list should be short and stable, and anything not on it should default to cheap plus session guarantees. The failure I am guarding against is not a team choosing the wrong model in isolation, it is nobody owning the composed guarantee that actually reaches the user.

   </details>

---

### Leader-based replication

*One node accepts writes and streams them to followers, which gives simple ordering and read scaling — and makes failover the hard part.*

**Flow:** `Client` → `Leader log` → `Followers` → `Durability policy` → `Acknowledgment`

> **The 30-second version**  
> One node takes all writes and streams them to followers: no conflicts, free ordering, and cheap read scaling — with failover, replication lag and stale follower reads as the costs.

**The problem**

You need more than one copy of your data: to survive a node failure, to serve reads at scale, and to place data near users. But multiple copies that both accept writes must reconcile conflicts, which is a genuinely hard problem that most applications should not take on.

Leader-based replication sidesteps it by designating one node as the writer. All writes go there, are assigned an order, and stream to followers that apply them in the same order. There are no write conflicts because there is only one writer — and that single decision makes everything simple except failover.

> **The trade in one sentence**  
> A single writer gives you a total order for free and eliminates conflict resolution entirely. In exchange, that writer is a capacity ceiling, a single point of failure, and the thing whose replacement — failover — is the most dangerous operation your system performs.

**Mental model**

A leader appends to a log; followers replay it. Everything else — durability guarantees, read scaling, failover behaviour — is a consequence of how you handle acknowledgement and promotion.

1. **Leader** — Accepts all writes, orders them, appends to its log. The single source of ordering truth.
2. **Replication stream** — The leader's log, shipped to followers. This is the same mechanism as write-ahead logging, viewed from outside.
3. **Follower** — Applies the stream in order, serves reads. May lag arbitrarily under load.
4. **Acknowledgement policy** — Whether a commit waits for zero, some, or all followers. This is the durability/latency dial.
5. **Failover** — Promoting a follower when the leader fails. The hard part, and the source of nearly every replication incident.

> **Failover is where data is lost**  
> Under asynchronous replication, a leader that fails has usually acknowledged writes that no follower received. Promoting a follower discards them — silently, from the client's perspective, because those writes were reported as successful. That is not a bug in the implementation; it is the definition of asynchronous replication, and the only way to avoid it is to make the acknowledgement synchronous.

**How it works**

**Acknowledgement policies and what they cost**

```text
ASYNCHRONOUS
  leader commits locally, acknowledges, ships later
  latency: leader fsync only       (~1 ms)
  on leader loss: acknowledged writes not yet shipped are LOST
  use: read scaling, analytics replicas, cross-region DR

SYNCHRONOUS to one follower (semi-sync)
  leader waits for one follower to persist
  latency: + one network round trip   (~1-2 ms same region)
  on leader loss: that follower has everything -> no loss
  risk: if the sync follower is down, writes BLOCK
        (unless you allow fallback to async - a decision
         you must make in advance)

SYNCHRONOUS to a quorum
  leader waits for a majority
  latency: the slowest of the majority
  on leader loss: any majority contains a node with the
                  latest write -> no loss, and safe election
  this is what consensus protocols do

ALL followers
  any single follower's failure blocks all writes
  almost never the right choice
```

1. **Choose the acknowledgement policy per dataset** — Session data and payment records do not deserve the same guarantee. The policy is a recovery-point decision, and should be stated as one.
2. **Decide the sync-follower-unavailable policy in advance** — Block writes and be unavailable, or degrade to asynchronous and accept data-loss risk. This choice gets made either way; making it during an incident goes badly.
3. **Fence the old leader on failover** — A demoted leader that was merely partitioned will resume and accept writes unless prevented. Epochs plus storage-level rejection are the mechanism; asking it politely is not.
4. **Expect and monitor replication lag** — Followers lag under write bursts, long transactions and their own load. Lag is the staleness bound for follower reads, and it is what turns read scaling into a consistency problem.
5. **Route reads deliberately** — Follower reads are eventually consistent. A user reading their own write must be routed to the leader or use a version token, or they will report a bug.
6. **Practise failover regularly** — An untested failover path is a hypothesis. The time to discover that promotion takes eleven minutes is not during an outage.

**Why follower reads break read-your-writes**

```text
t0  client writes profile name -> leader, acknowledged
t1  client reloads page -> read routed to follower
t2  follower has not applied the write yet
-> user sees the OLD name and reports a bug

FIXES, cheapest first
  1  read from the leader for N seconds after a write
     simple; adds leader load; does not compose across services
  2  version token: write returns an LSN; read requires a
     follower at or beyond it, else falls back to the leader
     composes; survives failover; needs replica version exposure
  3  sticky routing to the follower that is caught up
     breaks on failover and undermines balancing

THE GENERAL RULE
  follower reads are fine for OTHER people's data and
  for anything a user is not actively editing.
```

> **Read scaling moves the bottleneck, it does not remove it**  
> Followers multiply read capacity but every write still executes on every replica, so write throughput is unchanged and total write work grows with replica count. A workload that is write-bound gains nothing from adding followers — and the extra replication traffic can make it worse.

**Worked example**

A service with a 20:1 read/write ratio, deciding replication topology and failover policy.

**Topology, durability and failover, decided together**

```text
WORKLOAD  8,000 reads/s, 400 writes/s, two availability zones

TOPOLOGY
  leader in AZ-A
  synchronous follower in AZ-B   (durability)
  two asynchronous followers      (read scaling)
  one asynchronous cross-region   (disaster recovery)

DURABILITY
  commit waits for leader fsync + AZ-B follower
  -> ~1-3 ms added latency
  -> surviving an AZ failure loses nothing
  cross-region replica is async -> region loss has a
  non-zero recovery point, stated explicitly as ~5 seconds

READ ROUTING
  user's own data after a write -> leader (or version token)
  everything else                -> async followers
  analytics and reporting        -> a dedicated follower,
                                    isolated so a heavy query
                                    cannot lag the serving ones

FAILOVER POLICY (written down BEFORE the incident)
  leader fails: promote the SYNCHRONOUS follower only.
    promoting an async follower risks losing acknowledged
    writes, so it requires explicit human approval.
  sync follower fails: block writes, page immediately.
    do NOT silently fall back to async - that converts a
    stated zero recovery point into an unstated one.
  old leader returns: it is fenced and must be rebuilt
    from the new leader, never reintroduced as-is.
```

| Metric | Value | Note |
|---|---|---|
| Write latency | +1–3 ms | sync to AZ-B |
| AZ failure | zero loss | **sync follower** |
| Region failure | ~5 s loss | stated explicitly |
| Sync follower down | block writes | decided in advance |

> **The failover policy is the design**  
> Topology diagrams are easy; what matters is the written answer to three questions: which replica may be promoted, what happens when the synchronous follower is unavailable, and what happens when the old leader comes back. Systems that leave these undecided make the choices by accident, under pressure, and usually wrongly — silently falling back to asynchronous replication is the most common and most damaging of them.

**When to use it**

- **Almost all transactional databases**, where a single writer removes conflict resolution entirely.
- **Read-heavy workloads**, where followers multiply read capacity cheaply.
- **High availability within a region**, using a synchronous follower for zero-loss failover.
- **Disaster recovery**, using an asynchronous cross-region replica with a stated recovery point.
- **Isolating analytical load**, by giving heavy queries a dedicated follower.

**When to avoid it**

- **Do not use follower reads for data a user just wrote** without a version token or leader routing.
- **Do not add followers to solve a write bottleneck** — every write still executes everywhere.
- **Do not promote an asynchronous follower automatically**, which silently discards acknowledged writes.
- **Do not allow silent fallback from synchronous to asynchronous** replication; make it an explicit, alerted decision.
- **Do not reintroduce a recovered old leader** without rebuilding it from the new one — it holds writes that no longer exist.

**Advantages**

- **No write conflicts**, because there is exactly one writer — which removes an entire class of hard problems.
- **A total order for free**, since the leader's log defines it.
- **Read scaling is nearly linear** for read-heavy workloads.
- **Tunable durability** by choosing how many followers must acknowledge.
- **Mature and universally implemented**, with well-understood operational tooling.

**Disadvantages**

- **Write capacity is bounded by one node**, so scaling writes requires partitioning.
- **Failover is dangerous**: data loss under asynchronous replication, split brain without fencing, and unavailability during promotion.
- **Replication lag makes follower reads eventually consistent**, which surfaces as user-visible bugs.
- **Synchronous replication couples availability** — a failed synchronous follower blocks writes unless a fallback is defined.
- **Replication traffic grows with replica count**, adding load to the leader.

**Trade-offs**

**Replication topologies**

| Topology | Write conflicts | Write scaling | Complexity |
|---|---|---|---|
| Single leader | None | One node's capacity | Low — failover is the hard part |
| Multi-leader | Yes — must be resolved | Per-leader | High — conflict resolution |
| Leaderless (quorum) | Yes — version reconciliation | Good | Moderate — tunable quorums |
| Partitioned single-leader | None within a partition | Scales with partitions | Moderate — routing and rebalancing |

The bottom row is the usual destination at scale: keep the simplicity of a single writer, but have many of them, each owning a partition. That preserves conflict-free writes while removing the single-node write ceiling, at the cost of a partition key and a routing layer.

**How it fails**

**Replication failures**

| Failure | Cause | Fix |
|---|---|---|
| Acknowledged writes lost on failover | Asynchronous replication; promoted a lagging follower | Synchronous follower; promote only from it |
| Split brain after failover | Old leader not fenced, resumed accepting writes | Epochs and storage-level fencing; never reintroduce the old leader |
| Users see stale data after their own writes | Follower reads without a session guarantee | Version token, or leader reads briefly after a write |
| Writes blocked by a failed replica | Synchronous replication with no fallback policy | Decide in advance; alert loudly; consider quorum instead of a named follower |
| Replication lag grows unbounded | Follower slower than the write rate, or a long-running query | Isolate analytical replicas; monitor lag as an SLI |
| Failover takes far longer than expected | Untested promotion path | Practise failover on a schedule |
| Disk fills on the leader | Replication slot retained for a dead follower | Bound slot retention; alert on log volume |

**Limits**

> **Operating numbers**
>
> - **Synchronous replication cost**: ~1–2 ms within a region, 50–150 ms across regions — which is why cross-region synchronous commit is rarely acceptable.
> - **Read scaling** is near-linear with followers; write capacity is unchanged.
> - **Replication lag** is the staleness bound for follower reads; monitor it as a service-level indicator.
> - **Failover time** is dominated by detection plus promotion plus client reconnection — measure it rather than assuming.
> - **Every write executes on every replica**, so replication multiplies total write work.

**Alternatives**

| Approach | Gives | Costs |
|---|---|---|
| Single-leader replication | Simple ordering, read scaling | Write ceiling; failover risk |
| Partitioned single-leader | Write scaling with no conflicts | Partition key; routing; rebalancing |
| Multi-leader | Local writes in multiple regions | Conflict resolution; convergence complexity |
| Leaderless quorum | Tunable consistency; no failover event | Read repair; reconciliation |
| Consensus replication (Raft) | Automatic safe failover | Coordination on every write |
| No replication | Simplicity | No redundancy, no read scaling |

Consensus-based replication is increasingly the default in modern systems precisely because it makes failover automatic and safe rather than an operational event — the leader is elected by the protocol, and a quorum acknowledgement means the new leader always has every committed write.

**In real systems**

- **PostgreSQL streaming replication** with synchronous and asynchronous standbys is the canonical implementation, and its `synchronous_commit` settings expose the durability dial directly.
- **MySQL semi-synchronous replication** waits for one replica, with a configurable timeout that silently falls back to asynchronous — a well-known source of surprise data loss.
- **Read replicas in managed cloud databases** make read scaling a checkbox, which is why read-your-writes anomalies are so common in applications using them.
- **Raft-based systems (etcd, CockroachDB, TiDB)** replace manual failover with protocol-driven election, eliminating the most dangerous operational step.
- **The GitHub 2018 incident** is a widely studied example of what leader failover across regions can cost when replication is asynchronous and promotion is automatic.

**Common mistakes**

- **Promoting an asynchronous follower automatically**, discarding acknowledged writes.
- **Not fencing the old leader**, allowing split brain when it returns.
- **Serving a user their own data from a follower** with no session guarantee.
- **Adding followers to fix write throughput.**
- **Allowing semi-synchronous replication to time out into asynchronous** without alerting.
- **Never testing failover**, so its duration and its failure modes are unknown.
- **Sharing one follower between serving and analytics**, so a heavy query lags user-facing reads.

**The staff-level view**

Replication topology is easy to draw and hard to operate; the value is in writing down the failover policy before it is needed.

- **Write the failover policy as a document**: which replica may be promoted, what happens if the synchronous follower is unavailable, and what happens when the old leader returns. All three get decided under pressure otherwise.
- **Forbid silent synchronous-to-asynchronous fallback.** It converts a stated zero recovery point into an unstated one, and nobody notices until a failover loses data.
- **Insist on fencing at the storage layer.** A partitioned leader will resume; the only reliable defence is rejecting its writes by epoch.
- **Make replication lag a service-level indicator**, since it is the staleness bound for every follower read and it grows quietly.
- **Practise failover on a schedule.** An untested promotion path is a hypothesis, and the measurement you need is how long it actually takes.

**Go deeper**

A single leader accepts every write, assigns it an order, and ships its log to followers that replay it. Because there is only one writer there are no conflicts and the order is free, which removes an entire class of hard problems. Followers then multiply read capacity nearly linearly, though write throughput is unchanged because every write still executes on every replica.

The acknowledgement policy is the durability dial. Asynchronous replication is fast and loses acknowledged writes when the leader fails. Synchronous replication to a follower in another availability zone costs one or two milliseconds and loses nothing — but it forces a decision that must be made in advance: when that follower is unavailable, do you block writes or silently continue asynchronously? The silent fallback is the common default and it converts a stated zero recovery point into an unstated one.

Failover is where the incidents are. Only a synchronous follower can be promoted without discarding acknowledged writes; the old leader must be fenced by epoch at the storage layer, because a merely partitioned leader will resume and accept writes; and a recovered old leader must be rebuilt rather than reintroduced. Meanwhile replication lag makes follower reads eventually consistent, so a user reading their own write needs a version token or a leader read.

Leader-based replication is the default in transactional databases because a single writer eliminates conflict resolution entirely. Everything else about it — durability, read scaling, failover — follows from that one decision and from how acknowledgement is handled.

**The acknowledgement dial.** Asynchronous replication commits locally and ships later, costing only a local fsync but losing every acknowledged write the followers had not yet received when the leader fails. Synchronous replication to one follower adds a network round trip — one to two milliseconds within a region — and guarantees that at least one surviving node has every committed write. Quorum acknowledgement generalises this: any majority contains the latest write, which is what makes consensus-based election safe. Cross-region synchronous replication costs fifty to a hundred and fifty milliseconds per commit and is therefore rarely acceptable on an interactive path, which is why region loss almost always carries a non-zero recovery point that should be stated explicitly rather than discovered.

**Failover is the dangerous operation.** Three distinct risks. Data loss occurs when an asynchronous follower is promoted, discarding writes clients were told had succeeded — silently, with no error anywhere. Split brain occurs when the old leader was partitioned rather than dead and resumes accepting writes; no failure detector prevents this, only fencing by monotonic epoch enforced at the storage layer. And unavailability spans detection, promotion and client reconnection, a duration that is almost always unmeasured because failover is rarely practised. A recovered old leader must be rebuilt from the new one rather than reintroduced, because it holds a divergent history.

**The policy is what matters.** Three questions need written answers before an incident: which replicas may be promoted automatically, what happens when the synchronous follower is unavailable, and what happens when the old leader returns. The middle one is where the most damage occurs, because semi-synchronous implementations commonly time out into asynchronous mode by default — writes keep succeeding at normal latency while the durability guarantee has quietly changed, and nobody discovers it until a failover loses data. Blocking writes instead is an availability cost that at least announces itself; either choice can be right, but it must carry an alert.

**Read scaling has a consistency price.** Followers make reads cheap, but they lag, so a user who writes and then reads may be routed to a replica that has not applied their change. Routing reads to the leader for a brief window after a write is the simplest fix; a version token returned by the write and required on the read is better, because it composes across services and survives failover. The general rule is that follower reads are fine for other people's data and for anything the user is not actively editing.

**Where it ends.** A single leader is a write-throughput ceiling, and the usual destination at scale is partitioned single-leader replication — many leaders, each owning a partition — which preserves conflict-free writes while removing the ceiling, at the cost of a partition key and a routing layer. Separately, consensus-based replication is increasingly the default because it makes election part of the protocol rather than an operational event: a majority acknowledgement means any elected leader has every committed write, which removes both data-loss-on-promotion and split brain structurally rather than procedurally. The trade is coordination on every write and latency bounded by the slowest majority member, which makes node placement a first-order design concern.

**Prove it — interview questions**

1. **[Basic] Why does single-leader replication avoid write conflicts?**

   <details><summary>Model answer</summary>

   Because there is exactly one node accepting writes, so every write is assigned a position in one order before it is replicated. Followers replay that order. Two clients writing the same row are serialised by the leader, so there is never a situation where two copies of the data disagree about what happened — which is the entire class of conflict-resolution problems that multi-leader and leaderless systems have to solve.

   </details>

2. **[Basic] What is replication lag and why does it matter?**

   <details><summary>Model answer</summary>

   It is the delay between a write committing on the leader and being applied on a follower. It matters because it is the staleness bound for any read served from a follower: a user who writes and then reads may be routed to a replica that has not caught up and will see the old value. Lag grows under write bursts, long-running queries on the follower, and follower resource contention, so it should be monitored as a service-level indicator rather than assumed to be small.

   </details>

3. **[Senior] What are the risks of failover?**

   <details><summary>Model answer</summary>

   Three. Data loss, if replication was asynchronous and the promoted follower had not received the leader's most recent acknowledged writes — those are gone, and the clients were told they succeeded. Split brain, if the old leader was merely partitioned rather than dead and resumes accepting writes when it recovers. And unavailability, since detection, promotion and client reconnection all take time, and that duration is usually unmeasured. Mitigation is a synchronous follower that is the only promotion candidate, fencing by epoch enforced at the storage layer, and regularly practised failover so the duration is a known number.

   </details>

4. **[Senior] Why is silent fallback from synchronous to asynchronous replication dangerous?**

   <details><summary>Model answer</summary>

   Because it changes your durability guarantee without telling anyone. A semi-synchronous configuration that times out and continues asynchronously looks healthy — writes keep succeeding at normal latency — but the system now has a non-zero recovery point that nobody knows about, and the next failover will lose acknowledged data. The alternative, blocking writes when the synchronous replica is unavailable, is an availability cost that at least announces itself. Either choice can be correct, but it has to be a decision with an alert attached, not a timeout default nobody reviewed.

   </details>

5. **[Staff] Design the replication topology and policy for a service that cannot lose acknowledged writes.**

   <details><summary>Model answer</summary>

   A leader with a synchronous follower in a different availability zone, so an AZ failure loses nothing, plus asynchronous followers for read scaling and one asynchronous cross-region replica for disaster recovery with an explicitly stated recovery point — cross-region synchronous would add fifty to a hundred and fifty milliseconds per commit, which is rarely acceptable. The policy matters more than the diagram: only the synchronous follower may be promoted automatically, promoting an asynchronous one requires human approval because it discards acknowledged writes, and if the synchronous follower is unavailable we block writes and page rather than silently degrading. The recovered old leader is fenced and rebuilt from the new leader rather than reintroduced, because it contains writes that no longer exist in the system's history. And I would schedule failover drills, because the number I actually need — how long promotion takes end to end including client reconnection — cannot be obtained any other way.

   </details>

6. **[Principal] When would you move from leader-based replication to consensus-based?**

   <details><summary>Model answer</summary>

   When manual or externally coordinated failover has become the dominant operational risk, which it usually does once the system is large enough that leader failures are routine rather than rare. Consensus replication makes election part of the protocol: a write is committed once a majority has it, so any elected leader necessarily has every committed write, and there is no promotion decision for a human or a script to get wrong. That removes the two worst failure modes — losing acknowledged writes and split brain — structurally rather than procedurally. The costs are real: coordination on every write rather than optional, a minimum of three nodes, and latency bounded by the slowest member of the majority, which makes geographic placement a first-order concern. So I would frame it as trading a small, constant write latency increase for the elimination of an operational event that is rare, high-stakes and impossible to fully rehearse — which for most organisations at scale is a clearly good trade, and for a small deployment with an experienced operator may not be.

   </details>

---

### Quorum reads and writes

*Require overlapping subsets of replicas for reads and writes, so any read sees any completed write — tunable consistency without a leader.*

**Flow:** `Replica set` → `Write quorum` → `Version metadata` → `Read quorum` → `Resolution`

> **The 30-second version**  
> Require W replicas to accept a write and R to answer a read, with W + R > N so the sets overlap. Tunable, leaderless consistency — but concurrent writes still conflict and must be merged.

**The problem**

Leader-based replication concentrates writes on one node, which makes failover an event and the leader a bottleneck. Leaderless systems avoid both by letting any replica accept a write — but then how do you know a read sees the latest value, when different replicas may have received different subsets of writes?

Quorums answer this with arithmetic rather than coordination. If a write must reach W replicas and a read must consult R replicas, and W + R exceeds the replica count N, then the read and write sets necessarily overlap: at least one replica in every read has seen every completed write.

> **The overlap guarantee, and what it does not give you**  
> W + R > N guarantees that a read *sees* the latest completed write among the responses it collects. It does not tell you which response is latest — that requires version metadata — and it says nothing about concurrent writes, which can still produce conflicting versions that must be reconciled. The arithmetic solves visibility, not ordering.

**Mental model**

Picture N copies. A write is durable once W of them confirm it. A read collects R responses and picks the newest. If W + R > N, the two sets cannot be disjoint, so the newest value among the read's responses is the latest committed value.

1. **N** — The replication factor — how many copies exist. Usually 3 or 5.
2. **W** — How many replicas must acknowledge a write before it is considered successful.
3. **R** — How many replicas must respond to a read before it returns.
4. **Version metadata** — A timestamp or version vector attached to each value, so the reader can identify the newest — or detect that two are concurrent.
5. **Read repair and anti-entropy** — Background mechanisms that propagate the newest value to replicas that missed it, so the system converges.

> **Quorums do not prevent concurrent conflicting writes**  
> Two clients writing different values to the same key simultaneously can both achieve a quorum, because their quorums overlap with each other but neither saw the other's write. The result is two concurrent versions, and the system must either pick one (last-writer-wins, which discards data) or return both for the application to merge. This is why leaderless systems need version vectors and a conflict-resolution strategy in a way leader-based systems do not.

**How it works**

**The arithmetic, and the configurations it produces**

```text
N = 3 replicas

W=2, R=2   W+R=4 > 3  -> overlap guaranteed
  tolerates 1 replica down for both reads and writes
  the standard balanced choice

W=3, R=1   W+R=4 > 3  -> overlap guaranteed
  fast reads; any replica down BLOCKS writes
  use: read-heavy, writes rare, configuration data

W=1, R=3   W+R=4 > 3  -> overlap guaranteed
  fast writes; any replica down blocks reads
  use: write-heavy ingest with rare, careful reads

W=1, R=1   W+R=2, NOT > 3  -> NO overlap guarantee
  fastest, most available, eventually consistent
  a read may miss a just-completed write entirely

W=2 with N=3 means a write survives losing 1 replica.
For durability, W must exceed the number of simultaneous
failures you intend to survive.
```

1. **Set W and R per operation, not per system** — A critical write can use a quorum while a telemetry write uses W=1, in the same cluster. The dial is per request in most leaderless stores.
2. **Use version vectors, not timestamps, where concurrency matters** — Wall-clock timestamps cannot distinguish “later” from “concurrent”, so last-writer-wins silently discards one of two concurrent updates. Version vectors detect the concurrency and let the application decide.
3. **Remember sloppy quorums change the guarantee** — Some systems accept a write on any W available nodes, not necessarily the N that own the key, storing hints to hand off later. That preserves availability but breaks the overlap guarantee until the hints are delivered.
4. **Rely on read repair plus anti-entropy for convergence** — Quorums make reads correct; they do not make replicas equal. Background repair is what eventually brings lagging replicas up to date.
5. **Understand that quorum latency is the slowest of W or R** — You wait for the Wth fastest response, so a single slow replica does not block you — but as W approaches N, tail latency approaches the slowest node's.
6. **Do not confuse quorum reads with linearizability** — Overlap gives you the latest *completed* write, but without additional mechanisms (such as writing back the value read) sequential reads can still go backwards.

**Why timestamps lose data and version vectors do not**

```text
TWO CONCURRENT WRITES, no communication between clients
  client A: set(cart, [apple])         at t=100
  client B: set(cart, [banana])        at t=101

LAST-WRITER-WINS (timestamp)
  keeps [banana]; [apple] is silently discarded
  the user added two items and one vanished
  worse: clock skew can make the EARLIER write win

VERSION VECTOR
  A writes with context {A:1}
  B writes with context {B:1}
  neither dominates the other -> CONCURRENT
  the read returns BOTH versions (siblings)
  the application merges: [apple, banana]

THE COST
  the application must implement merge logic
  version metadata grows with the number of writers
  unmerged siblings accumulate if nobody resolves them
```

> **Read repair is not a substitute for anti-entropy**  
> Read repair only fixes keys that are actually read, so rarely accessed keys can stay divergent indefinitely — and if a replica is restored from an old backup, the stale values it holds will never be corrected on keys nobody queries. A background anti-entropy process comparing replicas (typically via Merkle trees) is what guarantees eventual convergence for cold data.

**Worked example**

A shopping cart in a leaderless store: choosing quorum settings and a conflict strategy.

**From requirement to configuration**

```text
REQUIREMENTS
  cart must be available during a partition (AP choice)
  losing an item the user added is unacceptable
  N = 3 replicas across availability zones

QUORUM SETTINGS
  writes: W = 2  -> survives one replica loss without
                    losing the write
  reads:  R = 2  -> W+R = 4 > 3, so a read sees any
                    completed write
  during a partition isolating one AZ:
    2 replicas remain reachable -> W=2 and R=2 still met
    -> the cart stays fully available

CONFLICT STRATEGY
  version vectors, NOT last-writer-wins
  two devices adding items concurrently produce siblings
  merge rule: UNION of items
    -> add-add conflicts resolve perfectly
  removal is the hard case:
    a naive union resurrects removed items
    -> model removals explicitly (tombstones with versions)
       or use an OR-Set CRDT

WHY LWW WOULD BE WRONG HERE
  it would silently drop one of two concurrent additions,
  which is precisely the failure the requirement forbids.
```

| Metric | Value | Note |
|---|---|---|
| N / W / R | 3 / 2 / 2 | overlap guaranteed |
| One AZ lost | still available | 2 of 3 reachable |
| Concurrent adds | merged by union | **no loss** |
| Removals | need tombstones | the hard case |

> **The merge function is the design, not the quorum**  
> Choosing W and R is arithmetic that takes a minute. Deciding what happens when two concurrent versions exist is the actual engineering, and it is domain-specific: union for a cart, maximum for a high-water mark, application-defined for a document. Systems that pick quorum settings carefully and then default to last-writer-wins have solved the easy half and silently discarded data in the hard half.

**When to use it**

- **Leaderless replication**, where quorums are the mechanism that gives any consistency at all.
- **Tunable per-operation consistency**, where different requests genuinely need different guarantees.
- **High availability during partitions**, since a quorum can often still be formed on one side.
- **Avoiding failover as an event**, because there is no leader to promote.
- **Multi-datacentre deployments**, where quorum placement determines both latency and failure tolerance.

**When to avoid it**

- **Do not use W=1, R=1 and expect to read your writes** — the overlap guarantee does not hold.
- **Do not use last-writer-wins** for data where concurrent updates carry real information.
- **Do not assume quorum reads are linearizable**; they give visibility of the latest completed write, not a total order.
- **Do not ignore sloppy quorums**, which trade the overlap guarantee for availability without announcing it.
- **Do not rely on read repair alone**, which never touches keys nobody reads.

**Advantages**

- **No leader**, so there is no failover event and no single write bottleneck.
- **Tunable per operation**, letting one cluster serve both strict and relaxed requirements.
- **Availability during partitions**, as long as a quorum is reachable.
- **The guarantee is arithmetic**, making it easy to reason about and hard to implement incorrectly.
- **Graceful degradation**, since losing a replica reduces headroom rather than causing an outage.

**Disadvantages**

- **Concurrent writes produce conflicts** that the application must resolve.
- **Version metadata grows** with the number of concurrent writers.
- **Latency is bounded by the Wth or Rth fastest replica**, so tail latency worsens as quorums approach N.
- **Not linearizable** without additional mechanisms, so sequential reads can still move backwards.
- **Sloppy quorums silently weaken the guarantee** in exchange for availability.
- **Background repair is required** and consumes resources continuously.

**Trade-offs**

**Common configurations with N=3**

| W | R | Overlap | Tolerates | Best for |
|---|---|---|---|---|
| 1 | 1 | No | 2 replicas down | Telemetry, caches, best-effort |
| 2 | 2 | Yes | 1 replica down | The balanced default |
| 3 | 1 | Yes | 0 down for writes | Read-heavy config data |
| 1 | 3 | Yes | 0 down for reads | Write-heavy ingest |
| 2 | 1 | No | 1 down | Fast reads, accepting staleness |

> **Framing the configuration**  
> “N equals three with W and R both two gives overlap, so a read always sees any completed write, and we survive losing one replica for both operations. The harder decision is conflict resolution: concurrent writes both reach a quorum, so I'd use version vectors and merge by union rather than last-writer-wins, which would silently drop one of two items the user added.”

**How it fails**

**Quorum-related failures**

| Failure | Cause | Fix |
|---|---|---|
| Read misses a just-written value | W + R ≤ N | Increase W or R so they overlap |
| Concurrent update silently lost | Last-writer-wins on genuinely concurrent writes | Version vectors; application merge; CRDTs |
| Earlier write wins over a later one | Clock skew with timestamp-based resolution | Logical clocks rather than wall clocks |
| Writes rejected during a partition | Quorum unreachable on the minority side | Expected behaviour — or use sloppy quorums, accepting weaker guarantees |
| Stale data on cold keys | Read repair only fixes what is read | Run anti-entropy with Merkle-tree comparison |
| Tail latency worse than expected | Quorum waits for the Wth fastest replica | Lower W or R where acceptable; use hedged requests |
| Siblings accumulate | Nobody resolves conflicting versions | Resolve on read and write back; cap sibling count |

**Limits**

> **Working numbers**
>
> - **W + R > N** is the overlap condition; it is necessary but not sufficient for linearizability.
> - **W > N/2** is required for a write to survive the loss of a minority and for safe leader-free durability.
> - **Latency** is that of the Wth (or Rth) fastest replica — not the slowest, unless W = N.
> - **Version vector size** grows with distinct writers; prune carefully, since pruning can lose causality information.
> - **Anti-entropy** should run continuously, not only on demand, since read repair never touches unread keys.

**Alternatives**

| Approach | Consistency | Cost |
|---|---|---|
| Quorum reads/writes | Latest completed write visible | Conflicts; version metadata |
| Single leader | Total order, no conflicts | Write ceiling; failover event |
| Consensus (Raft/Paxos) | Linearizable | Coordination on every write |
| CRDTs | Convergence without coordination | Restricted operations |
| Last-writer-wins | Simple convergence | Silent data loss |
| W=1, R=1 | Eventual only | Fastest; no guarantees |

Consensus is the natural comparison: it gives linearizability where quorums give only visibility of the latest completed write, at the cost of a coordinated protocol on every operation. Quorum systems are cheaper and more available; consensus systems are correct in more cases. The choice follows from whether conflicting concurrent writes are acceptable.

**In real systems**

- **Dynamo and its descendants** introduced tunable N, W and R as a per-request setting, which is where this vocabulary comes from.
- **Cassandra's consistency levels** (`ONE`, `QUORUM`, `ALL`, `LOCAL_QUORUM`) are exactly this dial, with `LOCAL_QUORUM` restricting the quorum to one datacentre for latency.
- **Riak's version vectors and sibling resolution** made the conflict-merge problem explicit to application developers rather than hiding it behind last-writer-wins.
- **Read repair with Merkle-tree anti-entropy** is the standard convergence mechanism, and its cost is a continuous background load teams often overlook.
- **Sloppy quorums with hinted handoff** preserve write availability during partitions at the price of temporarily breaking the overlap guarantee — a trade worth knowing is being made.

**Common mistakes**

- **W=1, R=1 with an expectation of read-your-writes.**
- **Last-writer-wins on data where concurrent updates matter**, silently discarding one.
- **Wall-clock timestamps for conflict resolution**, making clock skew a data-loss mechanism.
- **Assuming `QUORUM` means linearizable**, then building allocation on it.
- **Relying on read repair alone**, leaving cold keys permanently divergent.
- **Ignoring sloppy quorum behaviour**, which weakens the guarantee without notice.
- **Letting siblings accumulate** with no resolution path.

**The staff-level view**

Quorum arithmetic is the easy part; what distinguishes a good design is the conflict strategy and an honest account of what the guarantee actually is.

- **Treat last-writer-wins as a data-loss decision** requiring explicit sign-off, not a default. Wall-clock resolution also makes clock skew a correctness issue.
- **Design the merge function per data type.** Union for sets, maximum for high-water marks, domain logic for documents — and removals are always the hard case.
- **Be precise that quorum reads are not linearizable.** Teams assume `QUORUM` means strong, and then build allocation logic on a guarantee that does not exist.
- **Budget for anti-entropy.** Continuous background repair is not optional for convergence and it consumes real resources.
- **Check whether sloppy quorums are enabled**, since they change the guarantee silently in exchange for availability.

**Go deeper**

In a leaderless system, quorums provide consistency by arithmetic: if writes must reach W replicas and reads consult R, and W + R exceeds N, the two sets must overlap, so a read always sees the latest completed write among its responses. With three replicas, W and R both at two is the balanced default, tolerating one replica loss for both operations.

What the arithmetic does not do is order concurrent writes. Two clients writing simultaneously can both achieve a quorum without seeing each other, producing conflicting versions. Resolving them with wall-clock timestamps — last-writer-wins — silently discards one update and makes clock skew a data-loss mechanism. Version vectors instead detect that neither version dominates, return both as siblings, and let the application merge, which is the actual engineering work.

Two further caveats. Quorum reads are not linearizable: they guarantee visibility of the latest completed write, not a total order, so sequential reads can still move backwards. And read repair only fixes keys that are read, so continuous anti-entropy is required for cold data to converge — a background cost teams frequently overlook when sizing.

Quorums are how leaderless replication obtains a consistency guarantee without coordination, and their appeal is that the guarantee is arithmetic rather than protocol.

**The overlap condition.** With N replicas, requiring W acknowledgements for a write and R responses for a read, W + R > N means the two sets cannot be disjoint — at least one replica in every read has seen every completed write. This is simple enough to be hard to implement wrongly, and it is tunable per request, so one cluster can serve strict and relaxed operations simultaneously. Separately, W > N/2 is what makes a write survive the loss of a minority, which matters for durability independently of read visibility.

**What it does not solve.** Two clients writing concurrently can each reach a quorum without observing the other, producing two versions of the same key. The arithmetic guarantees visibility, not ordering. Resolving with wall-clock timestamps discards one update silently and, worse, makes clock skew a correctness issue — an earlier write can win. Version vectors record what each writer had observed, allowing the system to distinguish “later than” from “concurrent with”, return both versions as siblings, and defer the decision to an application merge function.

**The merge function is the real design.** Choosing W and R takes a minute; deciding what two concurrent versions mean is domain work. Union works for a shopping cart's additions; maximum works for a high-water mark; documents need application semantics. Removal is always the hard case, because a naive union resurrects deleted items — which requires versioned tombstones or an operation-based CRDT where a removal is defined relative to the specific addition it removes. A team that configures quorums carefully and then defaults to last-writer-wins has solved the easy half and is quietly losing data in the hard half.

**Convergence needs two mechanisms.** Read repair propagates the newest value to lagging replicas when a key is read, which handles hot data well and cold data not at all — a rarely read key, or a replica restored from an old backup, can stay divergent indefinitely. Anti-entropy, typically comparing replicas via Merkle trees, is what guarantees eventual convergence, and it runs continuously at a real resource cost that should be included in capacity planning.

**Two things that silently change the guarantee.** Sloppy quorums accept a write on any W available nodes rather than the N that own the key, storing hints for later handoff — which preserves availability during a partition at the price of breaking the overlap guarantee until the hints are delivered. And quorum reads are commonly assumed to be linearizable when they are not: they show the latest completed write among the contacted replicas, but sequential reads can still move backwards, so building allocation or uniqueness logic on a `QUORUM` setting is a category error that consensus, not quorums, is required to fix.

**Choosing the model.** Quorums suit domains whose conflicts are mergeable — carts, sets, counters, presence, sensor data — and reward you with no failover event, no write bottleneck, and continued availability while a quorum is reachable. Domains without a sensible merge function — balances, inventory allocation, unique names — need a single writer or consensus, and paying that coordination cost is the price of correctness. The recurring failure is adopting quorums for the operational benefits and only then discovering the domain has no merge semantics, at which point last-writer-wins becomes the accidental default.

**Prove it — interview questions**

1. **[Basic] What does W + R > N guarantee?**

   <details><summary>Model answer</summary>

   That the set of replicas a read consults and the set that acknowledged any completed write must overlap by at least one node. So a read always sees the latest completed write among its responses. It does not tell you which response is the latest — that requires version metadata — and it does not prevent two concurrent writes from both succeeding, which produces conflicting versions the application must resolve.

   </details>

2. **[Basic] Why is W=1, R=1 not safe for read-your-writes?**

   <details><summary>Model answer</summary>

   Because with N equals three, W plus R is two, which does not exceed three, so the read and write sets can be disjoint. A write acknowledged by one replica and a read served by a different one means the read misses it entirely. That configuration is the fastest and most available but provides only eventual consistency — the value will propagate through replication and repair, but a read immediately afterwards may not see it.

   </details>

3. **[Senior] Why are version vectors better than timestamps for conflict resolution?**

   <details><summary>Model answer</summary>

   Because a timestamp can only say which write is later, not whether two writes were concurrent. Last-writer-wins therefore discards one of two genuinely concurrent updates — a user adds two items from two devices and one silently vanishes — and clock skew can even make the earlier write win. A version vector records what each writer had seen, so the system can determine that neither version dominates the other, return both as siblings, and let the application merge them. The cost is that the application must implement merge logic and the metadata grows with the number of distinct writers.

   </details>

4. **[Senior] Are quorum reads linearizable?**

   <details><summary>Model answer</summary>

   No. The overlap guarantee means a read sees the latest completed write among the replicas it contacts, but it does not establish a total order respecting real time. Two sequential reads can return different values if they contact different replica subsets and a repair is in flight, so a client's view can move backwards. Achieving linearizability on top of quorums requires additional mechanisms such as writing back the value that was read, or a coordinated protocol — which is precisely what consensus provides and what quorums alone do not.

   </details>

5. **[Staff] Design quorum settings and conflict handling for a shopping cart.**

   <details><summary>Model answer</summary>

   With three replicas across availability zones, W and R both at two gives the overlap guarantee while tolerating the loss of one replica for both reads and writes, so an AZ failure leaves the cart fully available — which matches the requirement that a cart stays usable during a partition. The harder and more important decision is conflict resolution. Concurrent writes from two devices both reach a quorum, so last-writer-wins would silently discard one of two items the user added, which is exactly the failure we cannot accept. So version vectors, returning siblings, with a merge rule of union for additions. Removals are the genuinely hard case, because a naive union resurrects deleted items; that needs explicit tombstones with versions, or an OR-Set CRDT where removal is defined in terms of the specific add it removes.

   </details>

6. **[Principal] When is a quorum system the right choice over consensus or a single leader?**

   <details><summary>Model answer</summary>

   When availability during partitions matters more than a total order, and when the data's conflicts are mergeable. Quorums give you no failover event, no single write bottleneck, per-operation tunability, and continued availability as long as a quorum is reachable — which is a genuinely better operational profile than leader-based replication for the right workload. What you give up is ordering: concurrent writes conflict and the application must resolve them, and the guarantee is visibility of the latest completed write rather than linearizability. So the deciding question is whether the domain has a sensible merge function. Carts, counters, sets, presence and sensor readings do. Account balances, inventory allocation and unique name registration do not, and for those the coordination cost of consensus or a single writer is the price of correctness. The failure I look for in review is a team choosing quorums for the operational benefits and then discovering their domain has no merge semantics, at which point last-writer-wins becomes the accidental answer and data starts disappearing.

   </details>

---

### Consensus and Raft

*A majority of nodes agree on an ordered log of commands, so any elected leader already holds every committed entry — making failover safe rather than an operational gamble.*

**Flow:** `Client command` → `Leader` → `Replicated log` → `Majority commit` → `State machine`

> **The 30-second version**  
> A majority agrees on an ordered log, and the election rules guarantee any new leader already has every committed entry — so failover is automatic and safe, at the cost of a majority round trip per write.

**The problem**

Leader-based replication makes failover a judgement call: promote a follower and hope it had the latest writes, or block and hope the leader comes back. Quorum systems avoid the failover event but permit concurrent conflicting writes. Neither gives you a single agreed order that survives node failure without human involvement.

Consensus does. A group of nodes agrees on an append-only sequence of commands such that every node applies the same commands in the same order, and the protocol guarantees that any node which can become leader already possesses every committed entry — so promotion cannot lose data, ever.

> **What consensus actually buys**  
> Not speed, and not availability — it costs a round trip to a majority and it stops entirely when a majority is unreachable. What it buys is that **failover stops being a decision**. There is no “which replica should we promote” and no “did that write make it”, because the protocol structurally cannot elect a leader missing committed data.

**Mental model**

Think of a group agreeing on a shared notebook. One member is the writer for a period; entries are only official once a majority has copied them down. If the writer disappears, the group elects a new one — but only from members whose notebook is at least as complete as anyone else's.

1. **Term** — A logical period with at most one leader. Terms increase monotonically and act as a fencing epoch.
2. **Log** — The ordered sequence of commands. Replication means making followers' logs identical to the leader's.
3. **Commit** — An entry is committed once a majority has stored it. Only then may it be applied and acknowledged.
4. **Election restriction** — A candidate can only win if its log is at least as up to date as the majority's. This is why promotion is always safe.
5. **State machine** — The application of committed entries in order. Every node reaches the same state because it applies the same sequence.

> **Majority overlap is the whole safety argument**  
> Any two majorities of the same group share at least one member. A committed entry is on a majority; a winning candidate has the votes of a majority; therefore some node voted for the candidate *and* holds the committed entry — and the election restriction means the candidate's log was at least as complete as that voter's. Everything else in the protocol is bookkeeping around this one overlap property.

**How it works**

**The three sub-problems Raft separates**

```text
1  LEADER ELECTION
   followers wait for a heartbeat; on timeout they become
   candidates, increment the term, and request votes
   a node grants at most one vote per term
   a candidate wins with a MAJORITY
   randomised timeouts make split votes rare and self-resolving

2  LOG REPLICATION
   client -> leader appends to its log
   leader sends AppendEntries to followers
   once a MAJORITY has stored it, the entry is COMMITTED
   leader applies it and responds to the client
   followers apply committed entries in order

3  SAFETY
   election restriction: a candidate must have a log at least
     as up-to-date as the voter's, or the vote is denied
   -> any leader has every committed entry
   -> committed entries are never lost or reordered

WHY THREE NODES AND NOT TWO
   majority of 2 is 2 -> losing either node stops everything
   majority of 3 is 2 -> survives one failure
   majority of 5 is 3 -> survives two failures
```

1. **Size the group for failures tolerated, not throughput** — 2f+1 nodes survive f failures. Adding members increases the majority size and therefore commit latency — consensus groups get slower as they grow, not faster.
2. **Keep the group small and put the state machine elsewhere** — Consensus is for the metadata that must be agreed — leadership, membership, configuration, shard assignment — not for bulk data. Most systems run a small consensus group and store the data in nodes it coordinates.
3. **Use terms as fencing tokens** — The term number is monotonically increasing and travels with every operation, which is exactly what downstream systems need to reject a stale leader's writes.
4. **Understand that reads are not automatically linearizable** — A leader that has been partitioned may still believe it is leader. Linearizable reads require either a round trip confirming leadership, a lease with a safety margin, or reading through the log.
5. **Change membership one node at a time** — Adding or removing several members at once can create two disjoint majorities from old and new configurations. Joint consensus or single-server changes avoid this.
6. **Snapshot to bound the log** — The log grows forever otherwise. Periodic snapshots of the state machine let old entries be discarded, and are what a lagging or new member catches up from.

**Why a naive leader read can be stale**

```text
t0  leader L is partitioned from the group
t1  the rest elect a new leader L2 (term+1)
t2  L2 commits new entries
t3  L, unaware, serves a read from its local state
    -> returns data from BEFORE t2 -> NOT linearizable

MAKING READS LINEARIZABLE
  a) read index + heartbeat round trip
     leader confirms it is still leader with a majority
     before serving -> correct, costs one round trip
  b) leader lease
     leader holds a time-bounded lease; within it, no other
     leader can be elected -> serves reads locally
     -> depends on bounded clock drift, which is an
        assumption you are choosing to make
  c) read through the log
     treat the read as a log entry -> fully safe, slowest

MOST SYSTEMS USE (a) BY DEFAULT AND OFFER (b) AS A
PERFORMANCE OPTION WITH A STATED CLOCK ASSUMPTION.
```

> **Consensus stops when a majority is unreachable**  
> This is a feature, not a bug: it is what prevents split brain. But it means a three-node group in three availability zones survives one zone loss, while a three-node group with two nodes in one zone does not survive losing that zone. Placement determines what you actually tolerate, and it is the part most often got wrong.

**Worked example**

Using consensus for cluster coordination rather than for bulk data — the usual and correct pattern.

**Where consensus belongs in a system**

```text
WRONG: run the primary data store on consensus
  every write costs a majority round trip
  write throughput bounded by the group
  the group cannot be scaled by adding members
  -> correct, but expensive for high-volume data

RIGHT: consensus for METADATA, data elsewhere
  consensus group (3 or 5 nodes) holds:
    - which node owns which shard
    - current leader of each shard's replica set
    - cluster membership and configuration
    - distributed locks and leases
  data nodes hold the actual data, replicated by
  simpler mechanisms, coordinated BY the consensus group

WHAT THIS GIVES
  shard ownership changes are atomic and agreed
  the term number is a fencing token the data layer
    uses to reject a demoted owner's writes
  failover of a data shard is a metadata change, not a
    judgement call
  consensus traffic is tiny (kilobytes/s) while data
    traffic is unconstrained

THIS IS THE ARCHITECTURE OF NEARLY EVERY MODERN
DISTRIBUTED DATABASE AND ORCHESTRATOR.
```

| Metric | Value | Note |
|---|---|---|
| Group size | 3 or 5 | survives 1 or 2 failures |
| Consensus traffic | tiny | metadata only |
| Fencing | term number | **data layer enforces** |
| Failover | automatic | not a decision |

> **Consensus is a coordination primitive, not a storage engine**  
> The instinct on learning consensus is to put data in it, and that is usually wrong: every write pays a majority round trip and the group cannot scale by adding members. The productive use is to agree on the small amount of metadata that everything else depends on — ownership, leadership, membership — and let that agreement coordinate cheaper replication for the bulk data. The term number then doubles as the fencing token that makes the whole arrangement safe.

**When to use it**

- **Cluster metadata**: shard ownership, leadership, membership, configuration.
- **Distributed locks and leases**, where a stale holder must be reliably fenced.
- **Automatic failover**, replacing a manual promotion decision with a protocol guarantee.
- **Small, critical state** that must be strongly consistent and survive node failure.
- **Anywhere split brain is unacceptable**, since majority requirement makes it structurally impossible.

**When to avoid it**

- **Do not use it for high-volume data writes** — every write costs a majority round trip.
- **Do not grow the group to scale**; more members means a larger majority and slower commits.
- **Do not place a majority in one failure domain**, which negates the fault tolerance you paid for.
- **Do not assume leader reads are linearizable** without a confirmation round trip or a lease.
- **Do not change membership by more than one node at a time** without joint consensus.

**Advantages**

- **Failover is automatic and safe**, because an elected leader necessarily has every committed entry.
- **Split brain is structurally impossible**, since only one side of a partition can hold a majority.
- **A total order** for all agreed operations, with every node reaching identical state.
- **Terms provide a natural fencing epoch** for downstream systems to reject stale actors.
- **Well understood and widely implemented**, with Raft in particular designed for comprehensibility.

**Disadvantages**

- **Every write costs a round trip to a majority**, which is expensive across regions.
- **Unavailable without a majority**, by design — a deliberate trade of availability for safety.
- **Throughput does not improve with group size**; it degrades.
- **Leader is a write bottleneck** within the group.
- **Membership changes and snapshotting are subtle** and a common source of implementation bugs.
- **Clock assumptions creep in** with lease-based read optimisations.

**Trade-offs**

**Group size trade-offs**

| Nodes | Majority | Survives | Commit latency |
|---|---|---|---|
| 3 | 2 | 1 failure | 2nd fastest node |
| 5 | 3 | 2 failures | 3rd fastest node |
| 7 | 4 | 3 failures | 4th fastest node |
| Even numbers | n/2 + 1 | Same as n−1 | No benefit — avoid |

Even-sized groups are strictly worse: four nodes need three for a majority, which tolerates only one failure — identical to three nodes but with higher latency. Consensus groups should always be odd, and five is usually the practical maximum before latency dominates.

> **A strong framing**  
> “I'd run a five-node Raft group across three availability zones for the metadata — shard ownership and leadership — and keep the bulk data out of it, replicated by simpler means and fenced by the Raft term. That gives automatic, safe failover for the thing that must be agreed, without paying a majority round trip on every data write.”

**How it fails**

**Consensus failure modes**

| Failure | Cause | Fix |
|---|---|---|
| Cluster unavailable | Majority unreachable | By design; check placement across failure domains |
| Repeated leader elections | Heartbeat timeout too short for real latency, or an overloaded leader | Tune timeouts to observed network conditions; reduce leader load |
| Stale read from a partitioned leader | Serving reads locally without confirming leadership | Read-index round trip, or a lease with a safety margin |
| Split majority during reconfiguration | Multiple membership changes at once | Single-node changes, or joint consensus |
| Log grows without bound | No snapshotting | Periodic snapshots; discard applied entries |
| Slow commits | Group too large, or a member in a distant region | Smaller group; place members to control the majority's latency |
| Whole group lost with one zone | Majority co-located in one failure domain | Distribute across at least three domains |

> **Placement determines your real fault tolerance**  
> A three-node group tolerates one node failure — but if two of those nodes are in the same availability zone, losing that zone loses the majority and the group stops. The nominal fault tolerance and the actual fault tolerance diverge entirely based on placement, and this is one of the most common ways a correctly implemented consensus system fails to deliver the availability it was chosen for.

**Limits**

> **Sizing and tuning**
>
> - **2f + 1 nodes tolerate f failures**; always use odd sizes, and rarely more than five.
> - **Commit latency** equals the round trip to the (f+1)th fastest member, so one distant node in a five-node group is tolerable while two are not.
> - **Election timeout** should be several times the observed round-trip time, randomised to avoid split votes.
> - **Cross-region consensus** costs 50–150 ms per commit — acceptable for metadata, rarely for data.
> - **Snapshot frequency** bounds both log size and the time for a new or lagging member to catch up.

**Alternatives**

| Approach | Failover | Write cost | Best for |
|---|---|---|---|
| Consensus (Raft/Paxos) | Automatic and safe | Majority round trip | Metadata, locks, leadership |
| Leader + manual failover | Human decision, risky | Local commit | Traditional databases |
| Leader + synchronous replica | Safe if promoting the sync replica | One replica round trip | Zero-loss HA with simpler operations |
| Quorum (leaderless) | No failover event | W replicas | Mergeable data; availability priority |
| External coordination service | Delegated to the service | Its cost | Reusing an existing consensus group |

The last row is the practical shortcut most systems take: rather than implementing consensus, use an existing coordination service to hold leadership and ownership, and get the safety guarantees without owning the protocol implementation — which is genuinely hard to get right.

**In real systems**

- **etcd and ZooKeeper** exist to be the consensus group that other systems delegate coordination to, which is why Kubernetes stores all cluster state in etcd.
- **Raft was explicitly designed for understandability** after years of Paxos implementations diverging from the paper in subtle, unsafe ways.
- **CockroachDB and TiDB run a Raft group per data range**, which scales consensus horizontally by having many small groups rather than one large one.
- **Kafka's KRaft mode** replaced ZooKeeper with an internal Raft group, illustrating the metadata-versus-data separation directly.
- **Leader leases for fast reads** appear in most production implementations, always with an explicit statement of the clock-drift assumption they rely on.

**Common mistakes**

- **Putting high-volume data in the consensus group**, paying a majority round trip per write.
- **Growing the group to increase throughput**, which increases the majority and slows commits.
- **Even-numbered groups**, which cost latency without improving fault tolerance.
- **Majority co-located in one failure domain**, negating the redundancy.
- **Assuming leader reads are linearizable** without a read-index or lease.
- **Changing multiple members at once**, risking disjoint majorities.
- **No snapshotting**, so the log grows forever and recovery takes hours.

**The staff-level view**

Consensus is the right tool for a narrow and important job, and the two most common errors are using it for too much and placing it carelessly.

- **Use it for metadata, not bulk data.** Agree on ownership and leadership, then let cheaper replication move the data, fenced by the consensus term.
- **Check failure-domain placement explicitly.** Nominal fault tolerance means nothing if the majority shares a rack, a zone, or a power supply.
- **Prefer an existing coordination service to implementing the protocol.** Correct consensus implementations are genuinely hard, and the subtle bugs appear only during partitions.
- **Be explicit about read semantics.** Leader-local reads are not linearizable without a confirmation round trip or a clock-based lease, and teams routinely assume otherwise.
- **Scale by adding groups, not members.** Many small Raft groups scale; one large group gets slower.

**Go deeper**

Consensus makes a group of nodes agree on an ordered sequence of commands. A leader appends entries and replicates them; an entry is committed once a majority stores it. The safety argument rests on majority overlap: any two majorities share a member, and a candidate can only win an election if its log is at least as complete as each voter's — so an elected leader necessarily holds every committed entry, and promotion can never lose data.

That is what consensus buys: failover stops being a decision. There is no judgement about which replica to promote and no possibility of split brain, because only one side of a partition can hold a majority. The costs are that every write pays a round trip to a majority, the group is unavailable without one, and throughput degrades as the group grows — so use odd sizes, typically three or five, and never grow the group to gain capacity.

The standard pattern is to use consensus for metadata rather than bulk data: shard ownership, leadership, membership and locks live in a small group, with the term number serving as the fencing token that lets the data layer reject a demoted owner's writes. And be explicit about reads — a partitioned leader may still believe it is leader, so linearizable reads need a confirmation round trip or a clock-based lease.

Consensus solves the problem that leader-based replication leaves open: how to replace a failed leader without a human deciding whether the replacement has the data.

**The safety argument is one sentence of set theory.** Any two majorities of a group intersect. A committed entry is stored on a majority; a winning candidate holds votes from a majority; so at least one node both voted for the candidate and stores the entry. The election restriction — a vote is only granted to a candidate whose log is at least as up to date as the voter's — then guarantees the candidate has that entry too. Everything else in Raft is bookkeeping around this property: terms, heartbeats, log matching, and the rules for when entries may be applied.

**Costs and shape.** Every committed write requires a round trip to a majority, so commit latency is that of the (f+1)th fastest member, and cross-region consensus costs fifty to a hundred and fifty milliseconds. Groups should be odd-sized — four nodes need three for a majority, tolerating one failure exactly as three does but more slowly — and rarely larger than five, because adding members enlarges the majority and slows commits. Consensus groups get slower as they grow, which is the opposite of most people's intuition about replication.

**Placement determines real fault tolerance.** A three-node group nominally survives one failure, but two members in the same availability zone means losing that zone loses the majority. Racks, power domains and hypervisors create the same hazard. Nominal and actual tolerance diverge entirely on placement, and scheduling systems will happily co-locate members that were intended to be spread, so the distribution must be enforced rather than assumed. This is among the most common ways a correct implementation fails to deliver the availability it was chosen for.

**Reads need explicit handling.** A partitioned leader may continue believing it is leader while the remaining group elects a successor and commits new entries. Serving reads from its local state is therefore not linearizable. The correct options are a read-index confirmation round trip with a majority, a time-bounded leader lease that relies on an explicit bounded-clock-drift assumption, or treating the read as a log entry. Most implementations default to the round trip and offer the lease as a documented performance trade — and teams routinely assume leader reads are strong without checking which they have.

**Use it for metadata, not data.** Putting bulk writes through consensus means paying a majority round trip each time in a group that cannot scale by growing. The productive architecture — used by essentially every modern distributed database and orchestrator — runs a small consensus group holding shard ownership, replica-set leadership, membership and configuration, while the data is replicated by cheaper mechanisms that the metadata coordinates. Crucially, the term number becomes the fencing epoch: the data layer rejects writes carrying an old term, which is what makes a demoted owner harmless. Scaling then comes from running many small groups, one per data range, rather than from enlarging one.

**Prefer delegating to implementing.** Correct consensus is hard in exactly the places ordinary testing does not reach — membership changes, log truncation interacting with snapshots, the precise conditions for granting a vote — and the bugs appear only during partitions. The history of Paxos implementations quietly diverging from the paper is why Raft was designed for comprehensibility. Unless you need per-range consensus at scale, use an existing coordination service and take its term as your fencing token; if you do need it, use a well-tested library and invest in deterministic simulation of partitions and reconfiguration, because conventional tests will not find these failures.

**Prove it — interview questions**

1. **[Basic] Why does consensus require a majority?**

   <details><summary>Model answer</summary>

   Because any two majorities of the same group must share at least one member. A committed entry is stored on a majority, and a winning leader has votes from a majority, so some node both voted for the leader and holds the committed entry. Combined with the rule that a candidate only wins if its log is at least as up to date as each voter's, this guarantees any elected leader has every committed entry. The majority requirement also means only one side of a partition can make progress, which is what makes split brain impossible.

   </details>

2. **[Basic] How many nodes should a consensus group have?**

   <details><summary>Model answer</summary>

   An odd number, usually three or five. With 2f+1 nodes you survive f failures, so three tolerates one and five tolerates two. Even sizes are strictly worse — four nodes need three for a majority, tolerating only one failure, identical to three but with higher latency. Beyond five, commit latency grows because you wait for a larger majority, and consensus groups get slower rather than faster as they grow.

   </details>

3. **[Senior] Why aren't reads from a Raft leader automatically linearizable?**

   <details><summary>Model answer</summary>

   Because a leader that has been partitioned may not know it. The rest of the group can elect a new leader and commit further entries while the old leader, still believing it is in charge, serves reads from stale local state. Making reads linearizable requires confirming leadership with a majority before answering — a read-index round trip — or holding a time-bounded lease during which no other leader can be elected, which trades a round trip for an assumption about bounded clock drift. Systems typically default to the round trip and offer the lease as an explicit performance option.

   </details>

4. **[Senior] Why do most systems keep bulk data out of the consensus group?**

   <details><summary>Model answer</summary>

   Because every write costs a round trip to a majority, and the group cannot be scaled by adding members — a larger group means a larger majority and slower commits. The productive pattern is to use consensus for the small, critical metadata everything else depends on: which node owns which shard, who the current leader of each replica set is, cluster membership. The data itself is replicated by cheaper mechanisms coordinated by that metadata, with the consensus term doubling as a fencing token so the data layer can reject writes from a demoted owner. That is the architecture of essentially every modern distributed database.

   </details>

5. **[Staff] What determines the actual fault tolerance of a consensus group?**

   <details><summary>Model answer</summary>

   Placement, not node count. A three-node group nominally tolerates one failure, but if two of its members share an availability zone, losing that zone loses the majority and the group halts — so the real tolerance is zero zone failures. The same applies to racks, power domains and hypervisors. I would insist on distributing members across at least three independent failure domains, and I would check that distribution is actually enforced rather than merely intended, because scheduling systems will happily co-locate pods that were meant to be spread. This is one of the most common ways a correctly implemented consensus system fails to deliver the availability it was chosen for, and it is invisible until the day it matters.

   </details>

6. **[Principal] When would you implement consensus yourself versus delegating to a coordination service?**

   <details><summary>Model answer</summary>

   Almost never implement it yourself. Correct consensus is genuinely difficult — the subtle bugs are in membership changes, snapshot and log truncation interactions, and the exact conditions under which a candidate may win, and they manifest only during partitions, which is precisely when testing is least likely to have covered them. The history of Paxos implementations diverging unsafely from the paper is why Raft was designed for understandability in the first place. So the default should be delegating leadership, ownership and locking to an existing coordination service, taking the term number as a fencing token, and keeping your own system free of the protocol. The exception is when you need consensus per data range at scale, as distributed databases do, where the coordination cost of an external service per range would be prohibitive — and in that case you use a well-tested library rather than writing the protocol, and you invest heavily in deterministic simulation testing of partitions and reconfiguration, because ordinary testing will not find these bugs.

   </details>

---

### Leases and fencing tokens

*A lease grants time-bounded exclusive access and a fencing token makes expiry enforceable — because a paused process will wake up and act on a lease it no longer holds.*

**Flow:** `Lease grant` → `Epoch token` → `Owner action` → `Resource check` → `Reject stale owner`

> **The 30-second version**  
> A lease bounds ownership in time; a fencing token makes expiry enforceable by having the resource reject writes from a stale epoch. Locks without fencing are unsafe whenever a process can pause.

**The problem**

A process acquires a distributed lock and begins writing. It is then paused — a long garbage collection, a hypervisor stall, a network partition, a debugger breakpoint — for longer than its lock's timeout. The lock expires, another process acquires it and begins writing. Then the first process resumes, finishes its write, and corrupts the data.

No failure detector prevents this, because there is no failure to detect. The first process was never dead; it was slow. And it has no way to know that time passed, because from its perspective the pause did not happen.

> **The core impossibility**  
> You cannot make a lock holder stop. You can expire its lease, revoke its permission and tell every other participant, but the holder itself may be unreachable and will resume believing it is valid. The only reliable defence is to make the **resource** reject its writes — which requires the resource to know which epoch is current.

**Mental model**

A lease is a lock with an expiry, so ownership cannot be lost forever to a crashed holder. A fencing token is a monotonically increasing number attached to each lease, carried on every operation, and checked by the resource — which refuses anything older than the highest it has seen.

1. **Lease** — Exclusive access for a bounded time. Must be renewed or it expires automatically, so a crashed holder cannot block forever.
2. **Epoch / fencing token** — A number that increases with every grant. Never reused, never decreases.
3. **Carried on every operation** — The token travels with the write, not just with the acquisition.
4. **Resource-side check** — The storage layer records the highest token it has seen and rejects anything lower. This is the enforcement point.
5. **Renewal** — The holder heartbeats to extend. Failure to renew means the lease lapses and someone else may take it.

> **The token turns an unenforceable rule into an enforced one**  
> Without fencing, “only the lease holder may write” is a convention that every participant must voluntarily honour — including one that has been asleep and does not know the rules changed. With fencing, the rule is checked at the only place that matters: the resource itself. A stale holder can try, and it simply fails.

**How it works**

**The failure, and the fix**

```text
WITHOUT FENCING
  t0  process A acquires lease (30s)
  t1  A begins a write
  t2  A pauses (GC / hypervisor / partition) for 45s
  t30 lease expires
  t31 process B acquires the lease and writes value=X
  t47 A resumes, completes its write, value=Y
  -> B's write is silently overwritten by a process that
     lost its lease 17 seconds earlier

WITH FENCING
  t0  A acquires lease, token = 33
  t31 B acquires lease, token = 34
  t31 B writes with token 34 -> storage records 34
  t47 A writes with token 33 -> 33 < 34 -> REJECTED
  -> A learns it is stale and aborts

REQUIREMENT: the STORAGE LAYER must check the token.
A check inside the application does not help, because
the stale process IS the application.
```

1. **Make the token monotonic and never reused** — Consensus systems give this naturally as the term or revision number; a database sequence works too. What matters is that it strictly increases with each grant.
2. **Attach it to every operation, not just acquisition** — The pause can occur at any point, so the token must be checked at the moment of the write, not at the moment the lock was taken.
3. **Enforce at the resource, not in the client** — The whole point is that the client's view is wrong. If the check is in client code, the stale client will happily pass its own check.
4. **Set lease duration from failure-detection needs, not from work duration** — A long lease means long unavailability after a genuine crash; a short one means more renewals and a higher chance of spurious expiry. Renew frequently and keep the lease short.
5. **Never assume clock synchronisation between holder and grantor** — The holder's sense of remaining lease time can be wrong. Safety must come from the token, with the lease duration being an availability mechanism rather than a safety one.
6. **Accept that some resources cannot be fenced** — A third-party API with no epoch parameter cannot reject a stale caller. There, the only options are idempotency, compensations, or accepting the risk explicitly.

**Lease duration: two different jobs**

```text
SAFETY comes from the TOKEN, not the duration.
DURATION controls AVAILABILITY.

SHORT LEASE (e.g. 10s, renewed every 3s)
  + a crashed holder blocks others for at most 10s
  - more renewal traffic
  - a brief network blip can cause spurious expiry and
    an unnecessary ownership change

LONG LEASE (e.g. 60s)
  + tolerant of transient network issues
  - a crashed holder blocks the resource for up to 60s

THE COMMON MISTAKE
  lengthening the lease to "prevent" the stale-writer
  problem. It does not - it only makes the window bigger
  and the unavailability longer. Fencing is the fix;
  duration is a separate, availability-only decision.
```

> **Lock services are not safe on their own**  
> Acquiring a lock from a coordination service tells you that you held it at that moment. It says nothing about whether you still hold it when your write reaches the database, because arbitrary time may have passed. A distributed lock without a fencing token enforced at the resource provides mutual exclusion only in the absence of pauses — which is to say, only when you did not need it.

**Worked example**

Fencing a partitioned shard owner in a storage system.

**End-to-end fencing through the stack**

```text
COORDINATION LAYER (Raft / etcd)
  grants ownership of shard 7 to node A, term 41
  node A renews its lease every 2s; lease is 8s

NODE A writes
  every write carries (shard=7, term=41)

STORAGE LAYER
  maintains highest_term_seen[shard 7] = 41
  accepts writes with term >= 41
  rejects anything lower

PARTITION
  t0   A is partitioned from the coordination layer
  t8   A's lease expires; coordination grants shard 7
       to node B with term 42
  t8   B writes with term 42 -> storage updates
       highest_term_seen[7] = 42
  t15  A's partition heals; A does not yet know it lost
       the lease and issues a write with term 41
  t15  storage: 41 < 42 -> REJECT, return "stale epoch"
  t15  A sees the rejection, stops, and re-registers

WHAT WOULD HAVE HAPPENED WITHOUT THE TERM CHECK
  A's write lands, silently overwriting B's data,
  and both nodes believe they own shard 7.
```

| Metric | Value | Note |
|---|---|---|
| Lease | 8 s | renewed every 2 s |
| Token | Raft term | monotonic, free |
| Enforcement | storage layer | **not the client** |
| Stale write | rejected | A learns and stops |

> **Reuse the consensus term as your fencing token**  
> Systems that already use a coordination service for ownership get a monotonic epoch for free — the Raft term or the key's revision number. There is no need to invent a separate token, and using the existing one guarantees it increases exactly when ownership changes. The work is not generating the token; it is threading it through every write and making the storage layer check it.

**When to use it**

- **Any distributed lock or leadership**, without exception — a lock without fencing is not safe.
- **Partition or shard ownership**, where a demoted owner must be prevented from writing.
- **Leader election**, where a partitioned old leader will resume and act.
- **Exclusive access to an external resource**, where the resource can be made to check an epoch.
- **Coordinating scheduled jobs**, so a paused runner cannot duplicate work after its lease lapsed.

**When to avoid it**

- **Do not use a distributed lock without fencing** and call it mutual exclusion.
- **Do not lengthen the lease to fix stale writers**; it makes the window longer, not safer.
- **Do not check the token in the client**, which is the very component whose view is wrong.
- **Do not rely on clock synchronisation** for safety — only for availability.
- **Do not assume every resource can be fenced**; some third-party APIs cannot, and that limitation must be acknowledged.

**Advantages**

- **Makes expiry enforceable**, converting a convention into a check.
- **Requires no clock synchronisation for safety**, only monotonicity.
- **Trivially cheap** — an integer comparison at the resource.
- **Composes with existing coordination**, since consensus terms and key revisions already provide monotonic values.
- **Fails loudly**: a stale owner learns immediately rather than corrupting data silently.

**Disadvantages**

- **The resource must cooperate**, which is impossible for some external systems.
- **Every operation must carry the token**, which is invasive across a codebase.
- **Leases still cause unavailability** for their duration after a genuine crash.
- **Spurious expiry** from transient network issues causes unnecessary ownership churn.
- **Partial fencing is misleading** — if some write paths carry the token and others do not, the guarantee does not hold.

**Trade-offs**

**Mutual exclusion approaches**

| Approach | Safe under pauses | Requires |
|---|---|---|
| Distributed lock alone | No | Nothing — and provides nothing |
| Lock + fencing token | Yes | Resource-side epoch check |
| Single-writer partition ownership | Yes, if fenced | Routing plus fencing |
| Optimistic concurrency (version check) | Yes | Version on the row; no lock at all |
| Database transaction | Yes | Single database; no distribution |

The fourth row deserves emphasis: for many cases where teams reach for a distributed lock, an optimistic conditional write on the target row provides the same safety with no lock service, no lease, and no fencing infrastructure — because the version check *is* the fence.

**How it fails**

**Lease and fencing failures**

| Failure | Cause | Fix |
|---|---|---|
| Stale owner overwrites new data | No fencing token | Monotonic token checked at the resource |
| Token checked in the application | Enforcement in the wrong place | Push the check into the storage layer |
| Some writes bypass the check | Token threaded through only part of the code | Audit every write path; fail closed on a missing token |
| Frequent unnecessary failovers | Lease too short for real network variance | Lengthen modestly; renew more often |
| Long unavailability after a crash | Lease too long | Shorten; safety does not depend on it |
| Ownership changes but the token does not | Token derived from something non-monotonic | Use the consensus term or a database sequence |
| External resource cannot be fenced | Third-party API with no epoch concept | Idempotency keys, compensations, or accept the risk explicitly |

**Limits**

> **Practical parameters**
>
> - **Lease duration**: typically 5–30 s, with renewal at roughly one third of that interval.
> - **Safety is independent of duration** — it comes from the token, which is why the duration is purely an availability choice.
> - **Token source**: a consensus term, a key revision, or a database sequence; anything strictly monotonic.
> - **Check cost**: one integer comparison plus storing the highest seen value per resource.
> - **Pause durations that matter**: multi-second garbage collection pauses and hypervisor stalls are routine, which is why this is a real problem rather than a theoretical one.

**Alternatives**

| Approach | Prevents stale writes by | Cost |
|---|---|---|
| Fencing tokens | Resource rejecting old epochs | Threading the token everywhere |
| Optimistic concurrency | Version mismatch on the row | Retry on conflict; single-entity only |
| Single-writer ownership | Routing all writes to one owner | Routing layer; still needs fencing |
| Idempotency | Making repeated writes harmless | Does not prevent overwriting newer data |
| Accept the risk | Nothing | Requires proving the impact is tolerable |

Idempotency is worth distinguishing: it makes a repeated write harmless, but it does not stop a stale write from overwriting a newer one. They solve different problems and are frequently confused — you often need both.

**In real systems**

- **Kubernetes lease objects** with resource versions provide exactly this pattern for controller leadership, and controllers are expected to carry the version through their writes.
- **Raft terms and etcd key revisions** are the canonical fencing tokens, available for free wherever consensus is already used for ownership.
- **The Redlock debate** centred on precisely this point: a lock algorithm without a fencing token cannot provide mutual exclusion in the presence of pauses, no matter how the lock is acquired.
- **Storage systems with per-shard epochs** — most modern distributed databases — reject writes from demoted owners at the storage layer rather than trusting the ownership layer.
- **HDFS NameNode fencing** historically used both epoch numbers and physical fencing such as power control, illustrating how far systems go when the resource cannot check an epoch itself.

**Common mistakes**

- **Using a distributed lock with no fencing token.**
- **Lengthening the lease** to avoid stale writes, which only widens the window.
- **Checking the token in client code**, where the stale client passes its own check.
- **Threading the token through some write paths but not all.**
- **Deriving the token from a wall clock**, which is not monotonic across nodes.
- **Confusing idempotency with fencing** — one prevents duplicates, the other prevents stale overwrites.
- **Assuming a third-party resource can be fenced** when it has no epoch concept.

**The staff-level view**

Distributed locking is one of the areas where a widely used pattern is quietly unsafe, and the fix is cheap but invasive.

- **Treat any distributed lock without fencing as a bug**, not a simplification. Mutual exclusion that only holds in the absence of pauses does not hold.
- **Enforce at the storage layer, and audit every write path.** Partial fencing gives the appearance of safety while leaving a hole, which is worse than none because it stops people looking.
- **Reuse the coordination layer's term or revision.** Inventing a separate token invites non-monotonicity bugs, and the existing one already changes exactly when ownership does.
- **Separate the two decisions explicitly**: the token provides safety, the lease duration provides availability. Teams conflate them and then lengthen leases hoping for safety.
- **Ask first whether a lock is needed at all.** A conditional write on the target row often gives the same guarantee with no lock service and no lease.

**Go deeper**

A process holding a distributed lock can be paused — by garbage collection, a hypervisor stall, or a partition — for longer than its lease. The lease expires, another process takes ownership, and then the first resumes and completes its write, overwriting newer data. No failure detector prevents this, because nothing failed; the process was merely slow, and it has no way to know time passed.

The fix is a fencing token: a monotonically increasing number issued with each grant, carried on every operation, and checked by the resource, which records the highest token seen and rejects anything lower. The stale process's write is refused and it learns it is stale. Critically, the check must live at the resource — the storage layer — because the client whose view is wrong is exactly the one that would be performing a client-side check.

Safety comes entirely from the token, not from the lease duration; duration is purely an availability decision, trading how long a crashed holder blocks others against how often a transient blip causes spurious ownership change. Where consensus is already used for ownership, the term or revision number is a free, correctly monotonic token. And some resources — third-party APIs with no epoch concept — simply cannot be fenced, which is a risk to name explicitly rather than to paper over.

Leases and fencing exist because of a fact that is easy to state and easy to forget: you cannot make a process stop. You can expire its lease and tell everyone else, but the process itself may be unreachable and will resume believing nothing has changed.

**The failure is routine, not exotic.** Multi-second garbage collection pauses, hypervisor stalls, page-fault storms, network partitions and even a breakpoint in a debugger all produce the same effect: a process that is alive, believes it holds a lease, and is wrong. From its own perspective no time passed. Failure detection cannot help because there is no failure to detect, and no amount of care in the locking protocol changes this — the problem is not in acquisition, it is in the gap between acquisition and the write arriving.

**Fencing moves enforcement to the only place it can work.** A monotonically increasing token is issued with each lease grant and carried on every operation. The resource records the highest token it has seen and refuses anything lower. A stale holder's write is rejected, it learns immediately, and no data is corrupted. The check cannot live in the client — that is the component with the wrong view — and it cannot live in the coordination service, which the stale process is no longer talking to. It has to be at the resource, which means fencing requires the storage layer's cooperation and is impossible for resources that cannot be modified to check an epoch.

**Duration and safety are different dials.** Teams commonly lengthen leases hoping to prevent stale writers, which does nothing for safety and makes both the exposure window and the post-crash unavailability longer. Safety comes from the token comparison alone and is independent of any clock. Duration governs availability: short leases mean a crashed holder blocks others only briefly but transient network variance causes spurious expiry and ownership churn; long leases are the reverse. Renewing at roughly a third of the lease interval is the usual compromise.

**Reuse what you already have.** Where consensus or a coordination service already grants ownership, its term or key revision is a correctly monotonic token that increases exactly when ownership changes, so inventing a separate one adds risk without benefit. The engineering effort is not in generating the token but in threading it through every write path and making the resource fail closed when it is absent — because partial fencing is worse than none, providing an appearance of safety that stops anyone looking for the remaining hole.

**Ask whether a lock is needed at all.** A great many uses of distributed locking are better served by an optimistic conditional write on the target row, where the version check *is* the fence: no lock service, no lease, no token plumbing, and the same guarantee for single-entity operations. Distributed locks earn their complexity only when exclusivity must span multiple resources or operations that cannot be expressed as one conditional write.

**When fencing is impossible, say so.** Third-party APIs and legacy systems with no epoch concept cannot reject a stale caller, so mutual exclusion over them is unachievable and every mitigation is partial — idempotency makes repeats harmless but does not stop stale overwrites, compensations require reversibility, shorter leases narrow but do not close the window. The decision that matters is whether the consequences are recoverable: overwriting a cache entry is tolerable, issuing a duplicate payment is not, and for the latter the right answer is to redesign so the critical operation goes through something that can be fenced.

**Prove it — interview questions**

1. **[Basic] Why isn't a distributed lock enough?**

   <details><summary>Model answer</summary>

   Because acquiring a lock tells you that you held it at that instant, not that you still hold it when your write arrives. A process can be paused by garbage collection, a hypervisor stall, or a network partition for longer than the lock's timeout; the lock expires, someone else takes it, and then the paused process resumes and completes its write with no knowledge that anything changed. No failure detector helps, because nothing failed — the process was merely slow.

   </details>

2. **[Basic] What is a fencing token?**

   <details><summary>Model answer</summary>

   A monotonically increasing number issued with each lease grant and carried on every operation the holder performs. The resource records the highest token it has seen and rejects anything lower. So a process that resumes after losing its lease presents an old token, the write is refused, and it learns it is stale instead of corrupting data. The essential detail is that the check happens at the resource, because the stale process's own view is precisely what is wrong.

   </details>

3. **[Senior] Why does lease duration not affect safety?**

   <details><summary>Model answer</summary>

   Because safety comes entirely from the token comparison. A longer lease does not make a paused process less likely to resume after expiry — it just makes the expiry happen later and the unavailability after a genuine crash longer. What duration controls is availability: a short lease means a crashed holder blocks others only briefly but a transient network blip can cause spurious expiry and unnecessary ownership churn; a long lease is the reverse. Teams routinely lengthen leases hoping to prevent stale writes, which is treating an availability dial as a safety mechanism.

   </details>

4. **[Senior] Where must the fencing check happen, and why?**

   <details><summary>Model answer</summary>

   At the resource being protected — the storage layer, the database, the file system. Not in the client, because the client whose view is stale is the one performing the check, and it will happily conclude it is still valid. Not in the coordination service, because the stale process is not talking to it. The only place that sees the actual write and has current knowledge of which epoch is authoritative is the resource, which is why fencing requires cooperation from the storage layer and why resources that cannot be modified to check an epoch cannot be safely fenced at all.

   </details>

5. **[Staff] How would you fence shard ownership in a distributed storage system?**

   <details><summary>Model answer</summary>

   The coordination layer — a Raft group or a coordination service — grants ownership of each shard along with the term at which it was granted, and the owner renews a short lease. Every write the owner issues carries the shard identifier and that term. The storage layer maintains the highest term it has seen per shard and rejects any write with a lower one, returning a stale-epoch error so the sender learns immediately and re-registers. The important property is that I do not need to invent a token: the consensus term already increases exactly when ownership changes, so reusing it guarantees monotonicity for free. The actual work is threading the term through every write path and making the storage check fail closed if the term is absent, because partial fencing gives an appearance of safety that stops people looking for the hole.

   </details>

6. **[Principal] What do you do when the resource cannot be fenced?**

   <details><summary>Model answer</summary>

   Acknowledge it as an accepted risk rather than pretending otherwise, and then reduce the blast radius. Third-party APIs, legacy systems and some external services have no epoch concept and cannot reject a stale caller, so mutual exclusion over them is unachievable. The mitigations are all partial. Idempotency keys make a repeated operation harmless, though they do not prevent a stale actor from overwriting newer state. Compensating actions let you detect and undo afterwards, which requires the operation to be reversible. Shortening the lease reduces the exposure window without eliminating it. And in some historical systems the answer has been physical fencing — cutting power or network to the suspected stale node — which is crude but is genuinely the only reliable option when software cannot enforce the epoch. The judgement I would make is whether the operation's consequences are recoverable: overwriting a cache entry is tolerable, issuing a duplicate payment is not, and for the latter I would redesign so the operation goes through something that *can* be fenced rather than accepting the risk.

   </details>

---

### Logical clocks and causal order

*Wall clocks lie across machines, so order events by what could have influenced what — using counters that capture causality rather than time.*

**Flow:** `Local event` → `Clock increment` → `Message metadata` → `Clock merge` → `Causal comparison`

> **The 30-second version**  
> Order events by what could have influenced what, not by wall clocks that drift and jump. Vector clocks detect genuine concurrency; wall-clock last-writer-wins silently discards it.

**The problem**

Two servers write to the same key. Which write happened later? The obvious answer — compare their timestamps — is wrong often enough to lose data. Clocks on different machines drift, are corrected by NTP in jumps that can move backwards, and can differ by tens or hundreds of milliseconds even when everything is working.

Worse, wall-clock comparison answers the wrong question. What matters for correctness is not which event occurred at a smaller number, but whether one event could have *influenced* the other. Two writes that neither knew about are genuinely concurrent, and picking a winner by timestamp silently discards one.

> **Last-writer-wins by wall clock is a data-loss mechanism**  
> With 100 ms of clock skew, an update made a genuinely later can carry an earlier timestamp and be discarded. Worse, the loss is silent: no conflict is reported, no error surfaces, and the data simply reverts. Clock skew becomes a correctness bug rather than a monitoring concern.

**Mental model**

Forget time. Ask instead: could event A have affected event B? If a message carrying knowledge of A reached the process that produced B before B happened, then A causally precedes B. If no such chain exists in either direction, the events are concurrent — and no ordering between them is meaningful.

1. **Happens-before** — A precedes B if they are on the same process in order, or if A is a send whose receive precedes B, or transitively through such chains.
2. **Lamport clock** — A single counter per process, incremented on every event and advanced to the maximum on message receipt. Gives a total order consistent with causality.
3. **Vector clock** — One counter per process, so comparison can distinguish “before”, “after” and “concurrent”. Detects conflicts that a Lamport clock hides.
4. **Version vector** — The same idea applied to data replicas rather than processes, which is what conflict detection in replicated stores uses.
5. **Hybrid logical clock** — A logical counter combined with a physical timestamp, giving causality plus rough correspondence to real time.

> **Lamport clocks order, vector clocks detect**  
> A Lamport clock gives you a total order in which causally related events appear correctly — but two concurrent events also get distinct values, so you cannot tell that they were concurrent. A vector clock keeps enough information to distinguish “A before B”, “B before A” and “neither”, which is exactly what you need to detect a conflict rather than fabricate a winner.

**How it works**

**Lamport clock versus vector clock**

```text
LAMPORT (one counter per process)
  local event:   c = c + 1
  send:          c = c + 1; attach c
  receive(t):    c = max(c, t) + 1

  if A -> B (causally) then L(A) < L(B)
  BUT L(A) < L(B) does NOT imply A -> B
  -> concurrent events get ordered anyway, invisibly

VECTOR CLOCK (one counter per process)
  process i local event:  V[i] = V[i] + 1
  send:                    attach V
  receive(W):              V = elementwise max(V, W); V[i]++

  A -> B   iff  V(A) <= V(B) elementwise AND V(A) != V(B)
  A || B   iff  neither dominates  -> CONCURRENT, detected

EXAMPLE  processes A, B
  A: [1,0] write x=1
  B: [0,1] write x=2        (neither saw the other)
  compare: [1,0] vs [0,1] -> neither dominates
  -> CONCURRENT -> surface both, do not pick one
```

1. **Use logical clocks wherever ordering affects correctness** — Conflict detection, causal delivery, snapshot consistency, and anything where “which came later” has consequences.
2. **Prefer vector or version vectors when conflicts must be detected** — A Lamport clock will order concurrent events arbitrarily, which is precisely the case where you needed to know they were concurrent.
3. **Accept that vector size grows with participants** — One entry per writer. Clients as writers is unbounded; replicas as writers is bounded and usually the right granularity.
4. **Prune carefully, if at all** — Dropping entries loses causality information and can turn a detectable conflict into a silent overwrite. Prune only with a scheme that preserves safety, such as server-side version vectors keyed by replica.
5. **Consider hybrid logical clocks for human-facing ordering** — They keep causality while staying close to physical time, so logs and debugging remain intelligible and cross-system comparison is roughly meaningful.
6. **Never use wall clocks for conflict resolution** — Use them for display, for retention, for rough sequencing where a mistake is harmless — never for deciding which of two writes survives.

**Why physical time cannot substitute**

```text
NTP-SYNCHRONISED CLOCKS in practice
  typical skew between machines:   1-100 ms
  after a correction:               can jump BACKWARDS
  in a VM under load:               can stall or leap
  across regions:                   worse

CONSEQUENCE
  two writes 20 ms apart in reality can carry timestamps
  in the wrong order
  -> last-writer-wins keeps the EARLIER one
  -> silent, unreportable data loss

WHEN PHYSICAL TIME IS FINE
  display and logging
  TTL and retention decisions
  rough bucketing for metrics
  anything where being 100 ms wrong costs nothing

WHEN IT IS NOT
  choosing between two conflicting versions
  deciding whether a lease has expired (use a token too)
  establishing that A caused B
```

> **Spanner's approach: make the uncertainty explicit**  
> Rather than pretending clocks agree, TrueTime returns an *interval* containing the true time, and the system waits out the uncertainty before committing. That converts an invisible correctness risk into a visible latency cost — a legitimate trade, but one that requires specialised hardware and a willingness to pay several milliseconds per commit.

**Worked example**

Conflict detection in a replicated key-value store, comparing the two approaches on the same history.

**Same events, two clock schemes, different outcomes**

```text
SCENARIO  a user edits their profile from phone and laptop
          while a network partition separates the replicas

HISTORY
  replica R1: phone sets bio = "Engineer"
  replica R2: laptop sets bio = "Engineer, London"
  neither replica saw the other's write

WITH WALL-CLOCK LWW
  phone timestamp  14:03:12.480
  laptop timestamp 14:03:12.455   (laptop clock is 40ms behind)
  -> "Engineer" wins although the laptop edit happened later
  -> user's newer edit silently vanishes
  -> nothing anywhere records that this happened

WITH VERSION VECTORS
  R1 version {R1:5, R2:3}
  R2 version {R1:4, R2:4}
  neither dominates -> CONCURRENT
  -> store both siblings
  -> on read, return both; the application (or the user)
     resolves: show a merge prompt, or apply a domain rule

THE DIFFERENCE IS NOT PERFORMANCE - IT IS WHETHER THE
CONFLICT IS VISIBLE AT ALL.
```

| Metric | Value | Note |
|---|---|---|
| Clock skew | 40 ms | ordinary NTP |
| LWW outcome | newer edit lost | **silently** |
| Vector outcome | conflict detected | both preserved |
| Cost | one entry per replica | bounded |

> **The choice is between silent loss and visible conflict**  
> Wall-clock last-writer-wins does not avoid conflicts; it hides them. The data still diverged, and the system still had to choose — it simply made the choice invisibly and sometimes wrongly. Version vectors do not eliminate the need to decide; they surface the decision so that a correct, domain-aware resolution is possible. Teams often perceive vectors as more complex, when in fact they are exposing complexity that already existed.

**When to use it**

- **Conflict detection in replicated stores**, where concurrent writes must be distinguished from sequential ones.
- **Causal message delivery**, ensuring a reply is never delivered before the message it replies to.
- **Distributed debugging and tracing**, where establishing what could have caused what matters more than clock alignment.
- **Consistent snapshots** across nodes without stopping the world.
- **Anywhere last-writer-wins is being considered**, as the alternative that makes the conflict visible.

**When to avoid it**

- **Do not use wall-clock timestamps to choose between conflicting versions**, at any scale.
- **Do not use Lamport clocks where conflicts must be detected**; they order concurrent events silently.
- **Do not attach a vector entry per client**, which is unbounded — key it by replica or server instead.
- **Do not prune vector entries casually**, which can convert a detectable conflict into a silent overwrite.
- **Do not use logical clocks for anything requiring real time**, such as TTLs, billing periods or SLAs.

**Advantages**

- **Correct by construction**, independent of clock synchronisation, drift or correction.
- **Vector clocks detect concurrency**, which is what makes conflict resolution possible rather than accidental.
- **Cheap** — counters and comparisons, no coordination required.
- **Composable with causal delivery**, preventing nonsensical orderings visible to users.
- **Hybrid logical clocks** give causality with approximate real-time meaning, which helps human debugging.

**Disadvantages**

- **Vector size grows with the number of writers**, which is a real cost at scale and must be bounded by design.
- **No relationship to real time** for pure logical clocks, so they cannot answer “how long ago”.
- **Concurrent events still need a resolution policy** — detection is not resolution.
- **Pruning is unsafe in general**, so metadata accumulates.
- **Conceptually unfamiliar**, so implementations often quietly degrade back to timestamps.

**Trade-offs**

**Clock schemes compared**

| Scheme | Detects concurrency | Size | Real-time meaning |
|---|---|---|---|
| Wall clock | No | 8 bytes | Yes, but unreliable across machines |
| Lamport clock | No | 8 bytes | None |
| Vector clock | Yes | Entry per participant | None |
| Version vector | Yes | Entry per replica | None |
| Hybrid logical clock | Partially | ~16 bytes | Approximate, and monotonic |

Hybrid logical clocks are the pragmatic modern default: bounded size, monotonic, causally correct for the orderings that matter, and close enough to physical time that logs and cross-system comparisons remain meaningful. They do not fully replace version vectors for conflict detection, but they remove the worst wall-clock hazards at negligible cost.

**How it fails**

**Clock-related failures**

| Failure | Cause | Fix |
|---|---|---|
| Newer write discarded | Wall-clock LWW with skew | Version vectors; detect and resolve explicitly |
| Conflict never surfaced | Lamport clock ordering concurrent events | Vector or version vectors |
| Metadata growth unbounded | Vector entry per client | Key by replica; server-assigned versions |
| Conflict silently lost after pruning | Vector entries dropped | Prune only with a safe scheme; prefer bounded participant sets |
| Reply delivered before its message | No causal delivery ordering | Attach and check causal metadata on delivery |
| Timestamps jump backwards | NTP correction | Use monotonic or hybrid clocks; never wall clock for ordering |
| Cross-system traces misordered | Comparing wall clocks across machines | Hybrid logical clocks or explicit causal links |

**Limits**

> **Practical numbers**
>
> - **NTP skew** between machines is typically 1–100 ms and can be far worse in virtualised environments.
> - **Vector clock size** is one counter per participant — bounded if keyed by replica, unbounded if keyed by client.
> - **Hybrid logical clocks** are roughly 16 bytes and stay within a bounded offset of physical time.
> - **Lamport clocks are total but not faithful**: L(A) < L(B) does not imply A caused B.
> - **Detection is not resolution** — a detected conflict still requires a domain-specific merge.

**Alternatives**

| Approach | Gives | Costs |
|---|---|---|
| Wall-clock LWW | Simplicity | Silent data loss under skew |
| Lamport clock | Total order consistent with causality | Cannot detect concurrency |
| Vector / version vector | Concurrency detection | Metadata size |
| Hybrid logical clock | Causality plus approximate time | Slightly larger; partial detection |
| Single writer per key | No concurrency to order | Routing; write ceiling |
| Consensus | A single agreed order | Coordination per operation |

The last two rows are worth remembering: much of the difficulty disappears if concurrent writes to the same entity cannot occur. Routing all writes for a key to one owner, or ordering them through consensus, removes the need to detect concurrency because there is none.

**In real systems**

- **Dynamo-style stores** use version vectors keyed by replica to detect concurrent writes and return siblings rather than guessing a winner.
- **Spanner's TrueTime** exposes clock uncertainty as an interval and waits it out, converting an invisible correctness risk into an explicit latency cost.
- **CockroachDB and MongoDB use hybrid logical clocks** to get causality with bounded size and approximate real-time meaning.
- **Git's commit graph** is a causal history: parent links establish happens-before, and concurrent branches are detected as genuine merges rather than silently ordered.
- **Distributed tracing systems** rely on explicit parent-child links rather than timestamps, because clocks across services cannot be compared reliably.

**Common mistakes**

- **Resolving conflicts by comparing wall-clock timestamps.**
- **Using Lamport clocks where concurrency must be detected**, so conflicts are ordered away invisibly.
- **Vector entries keyed by client**, growing without bound.
- **Pruning vector entries** and converting detectable conflicts into silent overwrites.
- **Treating detection as resolution**, leaving siblings to accumulate unresolved.
- **Using logical clocks for real-time decisions** such as expiry or billing.
- **Assuming NTP makes timestamps comparable**, when skew and backward corrections are routine.

**The staff-level view**

The practical value here is a single rule that prevents a class of silent data loss: never let a wall clock decide which version of data survives.

- **Ban wall-clock last-writer-wins in design review** for any data where losing an update matters. It is not simpler — it hides a decision the system is making anyway.
- **Bound the participant set deliberately.** Version vectors keyed by replica are bounded and practical; keyed by client they grow without limit.
- **Push toward hybrid logical clocks as a default** where full vectors are too costly, since they remove the worst hazards at negligible size.
- **Remember detection is not resolution.** Surfacing a conflict is progress only if someone has defined the merge; otherwise siblings accumulate and nobody resolves them.
- **Ask whether concurrency can be eliminated instead.** Single-writer ownership per key removes the whole problem, and is often cheaper than getting conflict resolution right.

**Go deeper**

Clocks on different machines disagree by milliseconds to tens of milliseconds and can jump backwards during correction, so comparing timestamps to decide which of two writes came later is unreliable. Worse, it answers the wrong question: what matters is whether one event could have influenced the other, and two writes that neither knew about are genuinely concurrent with no meaningful ordering between them.

Logical clocks capture this directly. A Lamport clock is a single counter that gives a total order consistent with causality, but it cannot distinguish concurrent events from sequential ones, so it orders conflicts away invisibly. A vector clock keeps a counter per participant, so comparison yields before, after, or neither — which is exactly what conflict detection needs. Version vectors apply the same idea keyed by replica, which keeps the metadata bounded.

The practical rule is never to let a wall clock decide which version of data survives. Last-writer-wins does not avoid conflicts, it hides them: the data diverged regardless, and the system chose a winner silently and sometimes wrongly. Hybrid logical clocks are a good default where full vectors are too costly, giving causality and monotonicity with approximate real-time meaning that keeps logs intelligible.

Ordering events across machines is a correctness problem disguised as a timekeeping problem, and treating it as the latter is how systems lose data quietly.

**Why wall clocks fail twice.** First mechanically: NTP-synchronised machines typically differ by milliseconds to tens of milliseconds, virtualised environments can stall or leap, and corrections move clocks backwards — so two writes genuinely twenty milliseconds apart can carry timestamps in the wrong order. Second conceptually: even with perfect clocks, the question that matters for correctness is not which event carried a smaller number, but whether one could have influenced the other. Two updates made without knowledge of each other are concurrent, and imposing an order on them is a fabrication rather than a discovery.

**Happens-before, and the two clock families.** A precedes B if they occur in order on one process, if A is a send whose receipt precedes B, or transitively. A Lamport clock — increment locally, take the maximum on receive — guarantees that causally related events get increasing values, producing a total order consistent with causality. But the converse fails: a smaller value does not imply causal precedence, so concurrent events are ordered indistinguishably from sequential ones, which is precisely the case where you needed to know. A vector clock keeps one counter per participant, and elementwise comparison yields three outcomes — before, after, or neither — making concurrency detectable.

**Bounding the metadata.** Vector size grows with the number of participants, and the common failure is keying entries by client, which is unbounded. The standard resolution is a version vector keyed by replica: the server increments its own entry on behalf of whichever client wrote, so size is bounded by the replication factor. Pruning entries is unsafe in general, because discarding causality information converts a conflict that would have been detected into a silent overwrite — so bounding the participant set at design time is far better than trimming later.

**Detection is not resolution.** Surfacing two concurrent versions is only progress if a merge policy exists. Union for sets, maximum for high-water marks, domain logic for documents, or a user-facing prompt — but something must resolve them, or siblings accumulate indefinitely and the system degrades. This is why teams perceive version vectors as complex: they expose a decision that last-writer-wins was making silently, and that decision genuinely requires domain knowledge.

**Hybrid logical clocks as the pragmatic default.** Combining a logical counter with a physical timestamp gives bounded size, monotonicity, correctness for causal orderings, and approximate correspondence to real time. That last property is not cosmetic: pure logical clocks make logs and cross-system traces hard for humans to interpret, which is a significant reason implementations quietly drift back to wall-clock comparison. Systems needing stronger guarantees can instead make uncertainty explicit — returning a time interval and waiting it out before committing — which converts an invisible correctness risk into a visible latency cost, at the price of specialised infrastructure.

**The strongest move is eliminating concurrency.** Much of this difficulty exists only because two writers can touch the same entity simultaneously. Routing all writes for a key to a single owner, or ordering them through consensus, means there is no concurrency to detect and no merge to define. Conflict detection should be chosen where multi-writer concurrency is a deliberate requirement — offline editing, multi-region writes, collaborative documents — rather than accepted as an accident of architecture that then demands merge semantics nobody has defined.

**Prove it — interview questions**

1. **[Basic] Why can't you order distributed events by timestamp?**

   <details><summary>Model answer</summary>

   Because clocks on different machines disagree. Typical NTP skew is milliseconds to tens of milliseconds, virtualised environments can be far worse, and corrections can move a clock backwards. So two writes twenty milliseconds apart in reality can carry timestamps in the wrong order. If you then resolve a conflict by keeping the larger timestamp, you discard the genuinely later update — silently, with nothing recording that it happened.

   </details>

2. **[Basic] What does happens-before mean?**

   <details><summary>Model answer</summary>

   Event A happens-before B if A could have influenced B: they are on the same process with A first, or A is a message send whose receipt precedes B, or there is a chain of such relations. If neither A happens-before B nor B happens-before A, they are concurrent — no causal influence was possible in either direction, and there is no meaningful ordering between them. That distinction is exactly what conflict detection requires.

   </details>

3. **[Senior] What is the difference between Lamport clocks and vector clocks?**

   <details><summary>Model answer</summary>

   A Lamport clock is a single counter that guarantees causally related events receive increasing values, so it produces a total order consistent with causality. But the converse does not hold: a smaller value does not mean the event actually came first, so two concurrent events are ordered arbitrarily and indistinguishably from causal ones. A vector clock keeps a counter per participant, so comparison can yield three outcomes — before, after, or neither — which is what lets you detect that two writes were genuinely concurrent and therefore constitute a conflict rather than a sequence.

   </details>

4. **[Senior] What is the cost of vector clocks and how do you bound it?**

   <details><summary>Model answer</summary>

   One counter per participant, so the metadata grows with the number of writers. The failure mode is keying by client, where the participant set is unbounded and the vector grows indefinitely. The standard fix is to key by replica instead: the server assigns and increments the version on behalf of whichever client wrote, so the vector's size is bounded by the replication factor, which is small and fixed. Pruning entries from a vector is generally unsafe, because removing causality information can turn a conflict that would have been detected into a silent overwrite — so bounding the participant set by design is preferable to trimming afterwards.

   </details>

5. **[Staff] When is last-writer-wins acceptable?**

   <details><summary>Model answer</summary>

   When the data has no meaningful concurrent-update semantics and losing one of two concurrent writes is genuinely harmless — a cached derived value, a presence indicator, a last-seen timestamp, a sensor reading where the newest is all that matters. Even then I would prefer a monotonic logical or hybrid clock over a wall clock, so that clock skew cannot make the earlier write win, which is the failure mode people do not anticipate. What makes it unacceptable is any case where the two writes carry independent information the user expects to be preserved — two items added to a cart, two edits to different parts of a document, two independent field updates — because there the loss is real and invisible. The framing I use in review is that last-writer-wins does not avoid the conflict, it hides it, and the question is only whether hiding it is acceptable for this data.

   </details>

6. **[Principal] How would you approach ordering across an organisation's distributed systems?**

   <details><summary>Model answer</summary>

   With one prohibition and one default. The prohibition is that no system may use wall-clock comparison to decide which of two conflicting versions survives; that single rule prevents a class of silent data loss that is extremely hard to detect after the fact, because nothing anywhere records that an update was discarded. The default I would push is hybrid logical clocks, which are bounded in size, monotonic, causally correct for the orderings that matter, and close enough to physical time that logs and cross-service comparisons remain intelligible to humans — that last property matters more than it sounds, because pure logical clocks make debugging harder and teams therefore quietly revert to timestamps. Beyond that, I would encourage designing concurrency out where possible: routing all writes for an entity to a single owner removes the need to detect concurrency at all, and is usually cheaper than implementing correct merge semantics. Conflict detection should be the answer where genuine multi-writer concurrency is a requirement, not where it is an accident of architecture.

   </details>

---

### CRDTs

*Data types whose concurrent updates always merge to the same result, giving convergence without coordination — for the restricted set of operations that admit it.*

**Flow:** `Replica A update` → `Replica B update` → `Merge rule` → `Converged state`

> **The 30-second version**  
> Data types whose merge is commutative, associative and idempotent, so replicas converge without coordination — at the cost of metadata growth and the inability to enforce any invariant.

**The problem**

Two replicas accept writes while partitioned. When they reconnect, their states differ and something must reconcile them. Last-writer-wins discards data. Version vectors detect the conflict but hand the problem to the application, which must then implement merge logic correctly for every data type and every edge case.

Conflict-free replicated data types remove the reconciliation step entirely by constraining the operations so that merging is mathematically guaranteed to converge. Any replica can accept any update at any time, replicas exchange state or operations in any order, and all of them arrive at the same value — with no coordination and no conflicts to resolve.

> **The property that makes it work**  
> Merge must be **commutative, associative and idempotent**. Order does not matter, grouping does not matter, and applying the same update twice changes nothing. Any operation satisfying those three properties can be replicated without coordination; any operation that does not, cannot. That is the entire boundary of what CRDTs can do.

**Mental model**

Think of state as something that only ever grows in a partial order, with merge defined as taking the least upper bound. Two replicas that have seen different updates both move upward when they merge, and they land in the same place regardless of the order in which they learned things.

1. **Grow-only counter (G-Counter)** — Each replica counts its own increments; merge takes the maximum per replica and sums. Increments only.
2. **PN-Counter** — Two G-Counters, one for increments and one for decrements. Supports both directions.
3. **Grow-only set (G-Set)** — Union on merge. Elements can be added, never removed.
4. **OR-Set (observed-remove)** — Each addition carries a unique tag; a removal removes the specific tags it observed. Add wins over a concurrent remove — which is usually what users expect.
5. **LWW-Register** — A single value with a timestamp. Converges, but by discarding one of two concurrent writes — it is a CRDT and still loses data.
6. **Sequence CRDTs (RGA, Logoot)** — Ordered lists for collaborative text, where each position has a stable identifier so concurrent insertions interleave deterministically.

> **Convergence is not correctness**  
> A CRDT guarantees that all replicas end up with the same value. It does not guarantee that the value is the one you wanted. A last-writer-wins register converges perfectly while discarding one of two concurrent updates; an OR-Set converges while resurrecting nothing but also while resolving add-versus-remove in a fixed direction you may not want. Choosing a CRDT means accepting its particular resolution semantics.

**How it works**

**Why a naive counter is not a CRDT, and a G-Counter is**

```text
NAIVE COUNTER (broken)
  both replicas hold count = 5
  A increments -> 6        B increments -> 6
  merge by taking max -> 6
  -> two increments, one counted. WRONG.

G-COUNTER (correct)
  state is a map: replica -> its own increment count
  A: {A:1, B:0}            B: {A:0, B:1}
  merge = elementwise MAX  -> {A:1, B:1}
  value  = sum             -> 2   CORRECT

  why it works: each replica only ever increases its OWN
  entry, so per-entry max never loses an update, and the
  merge is commutative, associative and idempotent.

PN-COUNTER  = two G-Counters (P for +, N for -)
  value = sum(P) - sum(N)
  decrements work; the state only ever grows.
```

1. **Choose the CRDT whose resolution semantics match the domain** — OR-Set resolves add-versus-remove in favour of add; 2P-Set makes removal permanent. Neither is universally right — pick the one whose bias matches user expectation.
2. **Expect metadata to grow** — Tombstones for removals, per-replica counters, position identifiers for sequences. This is the price of coordination-free merging and it must be bounded or garbage-collected.
3. **Distinguish state-based from operation-based** — State-based CRDTs merge whole states, tolerating any delivery order and duplication. Operation-based ones ship operations and require exactly-once, causally ordered delivery — a stronger transport requirement in exchange for smaller messages.
4. **Do not try to express constraints** — A CRDT cannot enforce “balance must not go negative” or “at most one owner”, because that requires seeing all concurrent updates before deciding. Invariants spanning replicas need coordination.
5. **Use delta-state CRDTs at scale** — Shipping the entire state on every merge is impractical for large structures; delta-state variants ship only the changes while preserving the merge properties.
6. **Garbage-collect tombstones carefully** — Removing a tombstone before every replica has seen the removal allows the element to resurrect. Collection generally requires knowing what all replicas have observed.

**OR-Set: why removal needs tags**

```text
NAIVE SET WITH REMOVAL (broken)
  A: add("x")        -> {x}
  B: (has x) remove  -> {}
  merge by union     -> {x}   the removal is undone

2P-SET (two-phase: add set + remove set)
  merge = (adds - removes)
  + removal sticks
  - an element can NEVER be re-added. Fatal for most uses.

OR-SET (observed-remove)
  add("x") at A  -> {x:(A,1)}
  add("x") at B  -> {x:(B,1)}
  B removes x, having observed only (B,1)
                 -> removes tag (B,1) only
  merge          -> {x:(A,1)}   -> x is still present
  -> a concurrent add WINS over a remove that did not
     observe it, which matches user intuition
  -> re-adding works, because it creates a NEW tag

COST: tags accumulate; removed tags must be tracked until
      all replicas have observed the removal.
```

> **CRDTs cannot enforce invariants**  
> The entire premise is that any replica may accept any update without consulting others. That means no replica can ever know whether a constraint is about to be violated by a concurrent update elsewhere. A balance can go negative, a capacity limit can be exceeded, and two replicas can both allocate the last item. Anything requiring a global invariant needs coordination, and no CRDT will provide it.

**Worked example**

A collaborative task list synchronising across offline mobile clients.

**Composing CRDTs for a real data structure**

```text
REQUIREMENTS
  clients edit offline and sync later
  adding a task while another client deletes it: the
    add should win (users are surprised when items vanish)
  editing a task's title concurrently: last edit is fine
  reordering: concurrent moves must not duplicate items

COMPOSITION
  task set          -> OR-Set of task ids
                       add wins over concurrent remove
  each task's title -> LWW-Register with a hybrid logical
                       clock (NOT a wall clock)
  each task's done  -> LWW-Register, or an enable-wins flag
  ordering          -> sequence CRDT with stable position
                       identifiers

WHAT CONVERGES AUTOMATICALLY
  concurrent adds, removes, completions, reorders
  offline edits merged on reconnect with no prompts

WHAT STILL NEEDS COORDINATION
  "this list may contain at most 100 tasks"
    -> two clients can each add the 100th task offline
    -> the limit is not enforceable without coordination
    -> either accept temporary violation and trim later,
       or enforce server-side at sync time

WHAT YOU ACCEPT
  title LWW discards one of two concurrent title edits
    -> acceptable here; NOT acceptable for document body,
       which needs a sequence CRDT
```

| Metric | Value | Note |
|---|---|---|
| Task set | OR-Set | add wins |
| Title | LWW-Register | **one edit lost** |
| Order | sequence CRDT | no duplication |
| Max 100 | not enforceable | needs coordination |

> **CRDTs are composed, not chosen**  
> Real data structures are built by combining CRDTs whose individual semantics suit each field: a set of ids, registers for scalar fields, a sequence for ordered content. The design work is deciding, field by field, which resolution bias is acceptable — and noticing which requirements, like a maximum count, fall outside what any composition can enforce.

**When to use it**

- **Offline-first and local-first applications**, where clients edit disconnected and must merge on reconnect.
- **Collaborative editing**, where sequence CRDTs give deterministic interleaving without a central serialiser.
- **Multi-region active-active writes**, where cross-region coordination is too slow.
- **Counters, sets, flags and presence**, which have natural conflict-free formulations.
- **Anywhere availability during partitions matters more than enforcing a global constraint.**

**When to avoid it**

- **Do not use CRDTs where an invariant must hold** — balances, capacity, uniqueness, allocation. They structurally cannot enforce one.
- **Do not use an LWW-Register for data where concurrent updates carry independent information**; it converges by discarding.
- **Do not use a 2P-Set** unless permanent removal is genuinely desired, since elements can never be re-added.
- **Do not ignore metadata growth**; tombstones and tags accumulate and need a garbage-collection strategy.
- **Do not use operation-based CRDTs without exactly-once causal delivery**, which they require to be correct.

**Advantages**

- **No coordination at all**, so any replica accepts writes during partitions.
- **Convergence is guaranteed mathematically**, not by careful application code.
- **Merge order does not matter**, so delivery can be out of order and duplicated.
- **Excellent for offline-first**, since a client can work disconnected for arbitrary periods.
- **Composable**, allowing complex structures from well-understood primitives.

**Disadvantages**

- **Cannot enforce invariants**, which rules out a large class of applications.
- **Metadata overhead** from tombstones, tags and per-replica counters.
- **Restricted operation set** — if an operation is not commutative, it cannot be a CRDT.
- **Resolution semantics are fixed by the type**, not by the situation, so the bias may not match every case.
- **Garbage collection is subtle**, since removing tombstones too early causes resurrection.
- **Convergence is not the same as being right**, and teams conflate the two.

**Trade-offs**

**Conflict handling approaches**

| Approach | Coordination | Data loss | Invariants |
|---|---|---|---|
| CRDT | None | Only where the type discards (e.g. LWW) | Cannot enforce |
| Version vectors + app merge | None | None, if merge is correct | Cannot enforce |
| Last-writer-wins | None | Silent, on every conflict | Cannot enforce |
| Single writer per key | Routing only | None | Enforceable locally |
| Consensus | Per operation | None | Enforceable |

The bottom two rows are the honest comparison. CRDTs buy availability by giving up the ability to enforce anything; single-writer ownership buys enforceability by giving up availability during a partition of that owner. Choosing between them is choosing which property the domain actually requires.

**How it fails**

**CRDT failure modes**

| Failure | Cause | Fix |
|---|---|---|
| Deleted item reappears | Tombstone garbage-collected before all replicas saw the removal | Collect only after confirming universal observation |
| Metadata larger than the data | Tags and tombstones accumulating | Delta-state CRDTs; bounded participant sets; scheduled collection |
| Concurrent edit silently lost | LWW-Register used where both edits mattered | Use a sequence or multi-value register |
| Element cannot be re-added | 2P-Set semantics | Use OR-Set |
| Invariant violated | Expecting a CRDT to enforce a constraint | Coordinate, or accept and repair after the fact |
| Operation-based CRDT diverges | Delivery not exactly-once or not causally ordered | State-based or delta-state CRDT; fix the transport |
| Counter loses increments | Naive max-merge instead of per-replica counters | G-Counter with per-replica entries |

**Limits**

> **Practical considerations**
>
> - **Merge must be commutative, associative and idempotent** — this is the complete admissibility test for an operation.
> - **Metadata grows** with replicas, removals and operations; delta-state variants reduce transmission but not necessarily storage.
> - **Tombstone collection** requires knowing what all replicas have observed, which is a coordination-like requirement in disguise.
> - **Operation-based CRDTs need exactly-once causal delivery**; state-based ones tolerate any delivery.
> - **No CRDT can enforce a constraint spanning replicas** — this is a theorem, not an implementation gap.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| CRDTs | Offline-first, collaborative, multi-region writes | No invariants; metadata growth |
| Operational transformation | Collaborative text with a central server | Complex transform functions; server required |
| Version vectors + application merge | Domain-specific resolution | Merge correctness is your problem |
| Single-writer ownership | Anything with invariants | Unavailable if the owner is unreachable |
| Consensus | Strong constraints across replicas | Coordination per operation |
| Last-writer-wins | Genuinely disposable data | Silent loss |

Operational transformation is the older approach to collaborative editing and remains in wide use; it achieves similar goals through transform functions applied against concurrent operations, typically with a central server ordering them. CRDTs trade more metadata for the removal of that central serialiser.

**In real systems**

- **Redis CRDTs and Riak data types** offer counters, sets, maps and registers as built-in replicated types with defined merge semantics.
- **Automerge and Yjs** implement sequence and map CRDTs for local-first collaborative applications, synchronising peer to peer or through a relay.
- **Figma and similar collaborative tools** use CRDT-like structures so that concurrent edits from many clients converge without a locking protocol.
- **Presence and typing indicators** in chat systems are natural CRDTs — sets with add-wins semantics and TTL-based expiry.
- **Multi-region counters** for view counts and metrics commonly use G-Counters, since increments from every region merge without coordination.

**Common mistakes**

- **Expecting a CRDT to enforce a constraint** such as a non-negative balance or a capacity limit.
- **Using an LWW-Register** where both concurrent updates carried information.
- **Merging a counter by maximum**, losing increments.
- **Using a 2P-Set**, making re-addition impossible.
- **Garbage-collecting tombstones too early**, resurrecting deleted items.
- **Using operation-based CRDTs over a transport without exactly-once causal delivery.**
- **Assuming “conflict-free” means “no data is ever discarded”.**

**The staff-level view**

CRDTs are a precise tool with a hard boundary, and most of the judgement is in recognising which side of that boundary a requirement sits on.

- **Test every requirement against the invariant question first.** If any requirement is a constraint that must hold across replicas, no CRDT will satisfy it and the design needs coordination for that part.
- **Be explicit that convergence is not correctness.** An LWW-Register converges perfectly while losing data; teams hear “conflict-free” and assume nothing is discarded.
- **Choose types by their resolution bias**, field by field, and write down what each one discards or favours so the product decision is visible.
- **Plan tombstone garbage collection from the start.** It is the operational problem CRDT deployments actually hit, and collecting too early causes deleted data to reappear.
- **Prefer single-writer ownership when it is available.** CRDTs earn their metadata cost when concurrent multi-writer access is a genuine requirement, not when it is an accident of architecture.

**Go deeper**

A CRDT constrains operations so that merging is guaranteed to converge: the merge must be commutative, associative and idempotent, meaning order, grouping and duplication of updates cannot change the result. Any replica can then accept any write during a partition, updates can propagate in any order, and all replicas reach the same state with no coordination and nothing to resolve.

The primitives are designed around that constraint. A G-Counter keeps per-replica increment counts and merges by elementwise maximum, so no increment is lost. An OR-Set tags each addition so a removal deletes only the tags it observed, giving add-wins semantics and allowing re-addition. Sequence CRDTs give stable position identifiers so concurrent insertions interleave deterministically. Real structures are composed from these, field by field, choosing the resolution bias that matches user expectation.

Two boundaries matter. CRDTs cannot enforce invariants — no replica can know whether a concurrent update elsewhere is about to break a balance or a capacity limit, so anything requiring a global constraint needs coordination. And convergence is not correctness: a last-writer-wins register is a perfectly valid CRDT that discards one of two concurrent updates every time. “Conflict-free” means replicas agree, not that information is preserved.

Conflict-free replicated data types trade generality for a very strong property: replicas that accept writes independently will converge to the same value without ever coordinating.

**The admissibility test.** Merge must be commutative, associative and idempotent. Order of learning, grouping of merges, and repeated delivery must all be irrelevant. Operations satisfying this can be replicated without coordination; operations that do not, cannot — and that is a theorem rather than an implementation limitation. Everything about CRDT design follows from reformulating desired operations so they satisfy it, which is why a counter becomes a map of per-replica counts and a set with removal becomes a set of uniquely tagged elements.

**The primitives and their biases.** A G-Counter stores each replica's own increments and merges elementwise by maximum, summing for the value, so concurrent increments cannot be lost. A PN-Counter is two of those. An OR-Set tags every addition uniquely and removes only observed tags, which makes a concurrent add win over a remove and permits re-addition — semantics that match user intuition, unlike a two-phase set where removal is permanent. Sequence CRDTs assign stable identifiers to positions so concurrent insertions interleave deterministically. Each type encodes a fixed resolution bias, and choosing one is accepting that bias for that field.

**Convergence is not correctness.** A last-writer-wins register satisfies every CRDT property and converges perfectly while discarding one of two concurrent updates on every conflict. Teams hear “conflict-free” and infer that nothing is lost, which is exactly wrong for that type. The honest framing is that CRDTs guarantee replicas agree; whether they agree on the value you wanted depends entirely on whether the type's resolution semantics match the domain.

**Invariants are structurally out of reach.** The premise is that any replica accepts any update without consulting others, so no replica can know whether a concurrent update elsewhere is about to violate a constraint. Two partitioned replicas can each permit the withdrawal that empties an account, each locally correct. No CRDT enforces a balance floor, a capacity ceiling, or uniqueness. The only options are coordination for that specific operation, or accepting violation and compensating afterwards — and recognising which requirements fall into this category is the most important design step.

**Metadata is the operational cost.** Tags, tombstones and per-replica counters accumulate, and in some workloads the metadata exceeds the data. Delta-state variants reduce what is transmitted without necessarily reducing what is stored. Tombstone garbage collection is the failure that deployments actually hit: removing a removal record before every replica has observed it allows the deleted element to resurrect, which is the symptom users notice most — and determining that every replica has observed something is itself a coordination-shaped problem hiding inside a coordination-free design.

**Where they genuinely earn their place.** Offline-first and local-first applications, collaborative editing, and multi-region active-active writes all have concurrent multi-writer access as a real requirement, and there the metadata buys availability nothing else provides. What warrants caution is proposing CRDTs where single-writer ownership would work: routing every write for an entity to one owner eliminates concurrency entirely, restores enforceable invariants, and costs only a routing layer. CRDTs should be chosen because the domain requires uncoordinated concurrent writes — not because the architecture happened to allow them.

**Prove it — interview questions**

1. **[Basic] What makes a data type conflict-free?**

   <details><summary>Model answer</summary>

   Its merge operation is commutative, associative and idempotent — the order in which replicas learn about updates does not matter, the grouping does not matter, and applying the same update twice has no additional effect. Given those three properties, replicas that have seen different subsets of updates in different orders will always converge to the same value once they have all seen everything, with no coordination required at any point.

   </details>

2. **[Basic] Why can't you merge a counter by taking the maximum?**

   <details><summary>Model answer</summary>

   Because concurrent increments would be lost. If both replicas hold five and each increments to six, taking the maximum gives six — two increments and one counted. A G-Counter fixes this by keeping a map from replica to that replica's own increment count: each replica only ever increases its own entry, so an elementwise maximum cannot lose anything, and the value is the sum of all entries. That structure is what makes the merge commutative and idempotent.

   </details>

3. **[Senior] Why does an OR-Set need tags?**

   <details><summary>Model answer</summary>

   Because a plain set merged by union cannot represent removal — the union simply restores anything deleted. A two-phase set fixes that with a separate remove set, but then an element can never be added again, which is fatal for most uses. An observed-remove set gives each addition a unique tag, and a removal deletes only the tags it actually observed. So a concurrent addition elsewhere survives, because its tag was never observed by the remover, and re-adding works because it creates a fresh tag. The semantics are add-wins over a concurrent remove, which generally matches what users expect.

   </details>

4. **[Senior] Why can't CRDTs enforce invariants?**

   <details><summary>Model answer</summary>

   Because the whole premise is that any replica may accept any update without consulting the others. To enforce a constraint like a non-negative balance or a maximum capacity, a replica would need to know about every concurrent update elsewhere before deciding — which is coordination, the thing CRDTs exist to avoid. So two disconnected replicas can each allow the withdrawal that takes the balance to zero, and both are locally correct. Anything requiring a global constraint must coordinate; the most you can do with a CRDT is detect the violation afterwards and compensate.

   </details>

5. **[Staff] How would you design synchronisation for an offline-first task list?**

   <details><summary>Model answer</summary>

   By composing CRDTs per field, with the resolution bias of each chosen to match user expectation. The set of task ids becomes an OR-Set so that adding a task while another client deletes it results in the task surviving — users are far more surprised by things vanishing than by things persisting. Each task's title becomes a last-writer-wins register driven by a hybrid logical clock rather than a wall clock, accepting that one of two concurrent title edits is lost, which is tolerable for a short field but would not be for a document body. Ordering needs a sequence CRDT with stable position identifiers so concurrent moves do not duplicate items. And I would flag explicitly what falls outside this: a rule like “at most one hundred tasks” cannot be enforced, because two offline clients can each add the hundredth, so the choice is to accept temporary violation and trim at sync time, or to enforce it server-side and reject — which is a product decision, not a technical one.

   </details>

6. **[Principal] When do CRDTs earn their complexity, and when do they not?**

   <details><summary>Model answer</summary>

   They earn it when concurrent multi-writer access is a genuine product requirement rather than an architectural accident. Offline-first applications, collaborative editing, and multi-region active-active writes all genuinely need replicas to accept writes without coordination, and in those cases the metadata cost buys availability that no other mechanism provides. What makes me cautious is the frequency with which CRDTs are proposed for situations where single-writer ownership would work — routing every write for an entity to one owner removes concurrency entirely, gives you enforceable invariants, and costs nothing but a routing layer. I would also want the team to be clear-eyed that convergence is not correctness: a last-writer-wins register is a CRDT and it discards data on every conflict, so “conflict-free” is a statement about replicas agreeing, not about information being preserved. And I would make tombstone garbage collection part of the initial design rather than a later problem, because it is the operational issue these deployments actually encounter, and collecting too eagerly makes deleted data come back — which is the failure users notice most.

   </details>

---
