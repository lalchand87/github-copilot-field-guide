# Case Study 30 — Multi-Region / Disaster Recovery Architecture

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: RPO vs RTO, Active-Active vs Active-Passive, Global Data Routing.

---

## Business Context

A payment platform that processes ₹100 crore/minute cannot afford a data centre failure. Disaster recovery is not just about backups — it is about how quickly you can resume serving customers and how much data you lose. For a regulated fintech: RBI mandates business continuity planning; downtime during market hours is a regulatory incident.

Multi-region also enables latency reduction (serve from nearest region) and data residency compliance (EU data stays in EU). These are different goals but share infrastructure.

**Business goals driving architecture:**
- Zero unplanned downtime (99.99% availability = 52 min/year max)
- RPO < 1 minute: lose at most 60 seconds of transactions in a catastrophic failure
- RTO < 5 minutes: be fully operational in a new region within 5 minutes
- Latency: serve Indian users from Mumbai; EU users from Frankfurt
- Data residency: EU data never leaves EU infrastructure
- Cost: active-active is 2× the infrastructure cost; is it justified?

---

## Recognition Framework

### RPO vs RTO

```
RPO (Recovery Point Objective):
  "How much data can we afford to lose?"
  RPO = 0: zero data loss (synchronous replication to standby before ACK)
  RPO = 60s: lose up to 60 seconds of transactions (async replication)
  RPO = 24h: daily backup (lose up to a day of data)
  
  Financial context:
    RPO = 0: synchronous replication → every write waits for remote ACK → +30–100ms latency
    RPO = 60s: async replication → no latency impact → risk of losing 60s of payments
    
    For a payment platform:
      User payments: RPO = 0 (cannot lose committed payments)
      Analytics data: RPO = 60s acceptable (lose 1 minute of metrics)
      Audit logs: RPO = 0 (compliance requirement; every audit event must survive)

RTO (Recovery Time Objective):
  "How quickly must we be serving requests after a failure?"
  RTO = 0: instant failover (active-active; no traffic loss)
  RTO = 5 min: fast failover (automated; pre-warmed standby)
  RTO = 1 hour: manual failover (ops team executes runbook)
  RTO = 24 hours: rebuild from backup (disaster scenario)
  
  For a payment platform:
    RTO = 5 minutes (automated failover; tolerable for most customers)
    RTO = 0 is achievable but requires active-active (2× cost)
```

### Active-Active vs Active-Passive

```
Active-Passive (Warm Standby):
  Primary region: handles ALL traffic
  Standby region: receives replicated data; no user traffic; ready to activate
  On primary failure: DNS failover to standby; standby becomes primary
  
  Pros: simpler; cheaper (standby can be smaller; no need to handle full traffic)
  Cons: RTO > 0 (DNS propagation + warmup takes 1–5 min); RPO depends on replication lag
  Best for: RTO = 5 min; RPO = 30s; cost-sensitive
  
  Used by: most payment platforms; banks; moderate availability requirements

Active-Active (Multi-Region):
  Both regions serve traffic simultaneously (e.g., 50/50 or 70/30 split)
  Data replication: bi-directional (both regions write; must handle conflicts)
  On region failure: other region absorbs ALL traffic (capacity planning required)
  
  Pros: zero downtime on region failure (traffic already flowing to both)
  Cons: complex (bi-directional replication; conflict resolution); expensive (2× capacity always)
        write conflicts: what if payment pay-abc is modified in both regions simultaneously?
  Best for: RTO = 0; latency-critical; truly global user base
  
  Used by: global banks; Stripe; companies with > $1B revenue dependent on availability

CHOSEN for this design:
  Intra-region: Active-Active within a region (multiple AZs in same region)
  Inter-region: Active-Passive with warm standby (DR)
  
  Rationale:
    Multi-AZ active-active: handles AZ failures (most common failure type) with zero downtime
    Multi-region active-passive: handles full region failure (rare; 5 min RTO acceptable)
    Cost: 1.5× vs 2× for full active-active
    Payment write conflicts (active-active cross-region) are complex enough to avoid
```

---

## Problem Statement

Multi-region architecture for a payment platform. Active-Active within region (multi-AZ). Active-Passive across regions (DR). RPO < 1 minute, RTO < 5 minutes. Data residency compliant. Automated failover.

---

## Functional Requirements
- Multi-AZ: payments continue serving despite AZ failure (zero downtime)
- Multi-region DR: failover to backup region within 5 minutes of primary region failure
- Data replication: payment DB replicated to DR region with < 60s lag
- Automated failover: no manual intervention for AZ failure; automated for region failure
- Data residency: EU requests always served by EU region
- Failback: return to primary region after recovery (without data loss)

## Non-Functional Requirements
- Availability: **99.99%** (52 min/year maximum downtime)
- RPO: **< 60 seconds** (< 1 minute of data loss in catastrophic failure)
- RTO: **< 5 minutes** (serving requests from DR within 5 minutes of failure detection)
- AZ failure: **RTO = 0** (multi-AZ active-active; zero downtime)
- Latency impact of replication: **< 5ms** (async; no payment latency impact)

---

## Capacity Estimation
```
Primary region (Mumbai):
  50K TPS peak; 100M transactions/day
  DB: 200 GB/day new data

DR region (Hyderabad / Singapore):
  Must handle 50K TPS if primary fails (same capacity as primary)
  Active Passive: DR region is warm (can serve) but doesn't receive traffic normally
  Replication lag: async PostgreSQL streaming replication → typically < 1s; target < 60s

Replication bandwidth:
  200 GB/day = ~2.3 MB/sec WAL stream to DR region
  PostgreSQL streaming replication overhead: minimal (< 5% of DB IOPS)
  Network cost: ~2.3 MB/sec × 86400 × $0.08/GB = $16/day = $480/month (cross-region data transfer)
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Multi-AZ Deployment | AZ failure (most common) handled with zero downtime |
| Active-Passive (Warm Standby) | DR for full region failure; 5 min RTO; cheaper than active-active |
| PostgreSQL Streaming Replication | Async WAL replication to DR; < 60s lag for RPO target |
| Global Load Balancer (Route53 / Cloudflare) | DNS-based failover; health checks trigger routing change |
| Pre-Warmed Standby | DR app servers are running (not stopped); fast activation |
| Read Replica in DR | Serve read traffic from DR even in normal operation |
| Runbook Automation | Failover playbook automated to < 5 min execution |

---

## Architecture

### Primary Region (Mumbai) — Normal Operation

```
Region: ap-south-1 (Mumbai)
  3 Availability Zones: AZ-1a, AZ-1b, AZ-1c

Application Tier (deployed across all 3 AZs):
  payment-service:   3 instances per AZ = 9 total
  fraud-service:     2 instances per AZ = 6 total
  api-gateway:       2 instances per AZ = 6 total
  All behind ALB (Application Load Balancer): health-checks per AZ; routes around failures

Database Tier:
  PostgreSQL cluster (Patroni):
    Primary:   1 instance in AZ-1a
    Standby 1: 1 instance in AZ-1b  (synchronous replication; RPO=0 within region)
    Standby 2: 1 instance in AZ-1c  (synchronous replication)
    
  On AZ-1a failure:
    Patroni promotes Standby 1 (AZ-1b) to primary automatically (< 30 seconds)
    Application: reconnects to new primary (connection string via PgBouncer; transparent)
    Result: payments continue; zero data loss (sync replication); < 30s service interruption
    
  DR Standby (in DR region):
    Async streaming replication from primary to DR
    This is the one allowed to lag (< 60s RPO for cross-region)

Redis Cluster:
  3-node Redis Cluster (1 primary + 2 replicas per shard)
  Spread across 3 AZs
  AZ failure: replica in another AZ auto-promoted (Redis Sentinel/Cluster)

Kafka:
  3-broker cluster (1 per AZ)
  RF=3: messages replicated to all 3 AZs
  AZ failure: 2 remaining brokers continue; messages were replicated before failure

S3:
  Auto-multi-AZ (S3 standard): objects replicated across AZs within region automatically
  No additional action needed
```

### DR Region (Hyderabad/Singapore) — Normal Operation (Standby)

```
Region: ap-south-2 (Hyderabad) or ap-southeast-1 (Singapore)

DR Application Tier:
  All services are RUNNING (warm standby; not stopped)
  But receiving ZERO user traffic (LB health check points to primary region only)
  Why running: cold start takes 2–5 min; pre-warmed = < 30s activation time
  Cost: ~50% of primary (smaller instances; no traffic means less auto-scaling)

DR Database:
  PostgreSQL read replica (streaming replication from Mumbai primary)
  Read-only: cannot accept writes (streaming replication; not primary)
  Used for: read-only analytics, compliance reports, DR testing (non-disruptive)
  Replication lag: typically < 5s; alert if > 60s

DR Redis:
  Snapshot restore from primary (not real-time sync; Redis AOF backups every 60s)
  RDB snapshot every 5 min (lower cost; acceptable since Redis is mostly cache)

DR Kafka:
  MirrorMaker 2 (Kafka cross-cluster replication) mirrors critical topics from primary
  Lag: < 30s for critical topics (payment.events, audit.log)

DR S3:
  S3 Cross-Region Replication: primary S3 bucket → DR S3 bucket automatically
  Lag: < 15 min (S3 CRR is near-real-time but not instantaneous)

DNS / Global LB:
  Route53 (or Cloudflare) health checks: HTTP health check to payment API every 10s
  If primary region fails: health check fails 3× → failover record activated → traffic routes to DR
  TTL: 60 seconds (DNS propagation within 1 minute globally)
```

---

## Detailed Components

### Failover Procedure (< 5 Minutes)

```
Failure detected:
  Time T=0: Primary region becomes unavailable (AZ failure → multi-AZ handles)
            OR full region failure (power, networking, AWS incident)

Detection (T=0 to T+30s):
  Route53 health check fails 3 consecutive times (30s)
  Monitoring: payment-service request rate drops to 0 (Prometheus alert fires immediately)
  On-call paged: "Primary region health check failed; investigate DR readiness"

Automated failover (T+30s to T+5min):
  Step 1 (T+30s): Route53 DNS failover record activates → traffic begins routing to DR
    DR services are already running (pre-warmed) → traffic handled immediately
    DNS TTL 60s → most clients see new IP within 60s
    Immediate traffic: clients with cached old IP hit primary (failing) → retry → connect to DR

  Step 2 (T+1min): DR PostgreSQL promoted to primary
    Trigger: automated (runbook Lambda function fires on Route53 health alarm)
    Action: Patroni promote DR replica to primary
    Duration: < 30 seconds (Patroni promotion is fast)
    Risk: any transactions committed to primary in last 60s (replication lag) are lost
    
  Step 3 (T+2min): DR services verify database connectivity
    Automated health check: payment-service health endpoint verifies DB write succeeds
    If DB write fails: alert; ops team investigates

  Step 4 (T+3min): Verify DR traffic serving correctly
    Smoke test: automated payment simulation against DR
    Check: metrics show payments processing in DR region
    
  Step 5 (T+5min): DR fully operational; on-call declares incident contained

Total: < 5 minutes with automation; 10–15 minutes without automation

Post-recovery (T+recovery to T+failback):
  Investigate primary region failure
  Fix root cause
  Sync DR data back to primary (primary is now behind)
  Perform failback (planned; less urgent than the original failover)
  Announce post-incident review
```

### PostgreSQL Replication Setup
```bash
# Primary (Mumbai):
# postgresql.conf
wal_level = replica
max_wal_senders = 10
wal_keep_size = 10GB          # keep 10GB of WAL for replica reconnect
synchronous_commit = local    # async to DR; sync within region

# DR replica (Hyderabad):
# recovery.conf / postgresql.conf
primary_conninfo = 'host=primary.mumbai.internal port=5432 user=replication'
primary_slot_name = 'dr_slot'  # replication slot prevents WAL deletion
hot_standby = on              # allow reads from DR replica

# Monitor replication lag:
# On primary:
SELECT
  client_addr,
  state,
  write_lag,
  flush_lag,
  replay_lag         -- this is the effective RPO
FROM pg_stat_replication
WHERE application_name = 'dr_replica';
-- Alert if replay_lag > interval '60 seconds'

# Replication slot:
# Without slot: if DR replica disconnects > 10GB WAL accumulation → WAL deleted → replica must rebuild
# With slot: primary keeps WAL until DR replica confirms receipt
# Risk: if DR goes down for days → primary WAL accumulates indefinitely → disk fill → OUTAGE
# Mitigation: alert if slot lag > 1 hour; consider dropping slot if replica gone > 24h
```

### Chaos Engineering for DR
```python
# DR drill automation: test failover quarterly without real failure
class DRDrill:
    def run_quarterly_drill(self):
        """Simulate primary region failure and measure RTO/RPO."""
        
        # 1. Record current state
        baseline_metrics = self.capture_payment_metrics()
        last_committed_txn = self.get_last_transaction_id()

        # 2. Block traffic to primary (simulate region failure)
        # Use: block Route53 health check endpoint; DO NOT actually terminate infra
        self.simulate_primary_unhealthy()

        # 3. Measure time to failover
        t_failover_start = time.now()
        self.wait_for_dr_serving(timeout=300)  # 5 min timeout
        t_failover_complete = time.now()

        rto_measured = t_failover_complete - t_failover_start

        # 4. Measure data loss (RPO)
        # Get first transaction committed in DR
        first_dr_txn = self.get_first_dr_transaction()
        rpo_measured = last_committed_txn.timestamp - first_dr_txn.timestamp

        # 5. Verify DR is serving correctly
        self.run_smoke_tests_against_dr()

        # 6. Report
        self.generate_drill_report({
            'rto_seconds': rto_measured.total_seconds(),
            'rto_target': 300,
            'rpo_seconds': rpo_measured.total_seconds(),
            'rpo_target': 60,
            'passed': rto_measured.total_seconds() < 300 and rpo_measured.total_seconds() < 60
        })

        # 7. Failback (restore primary as primary)
        self.restore_primary_routing()
        self.sync_dr_to_primary()

# Run quarterly (automated); results reviewed in capacity planning meetings
```

### Data Residency + Global Routing
```
EU User (Frankfurt-based):
  DNS query → Cloudflare → returns Frankfurt ELB IP (geolocation-based routing)
  All processing: Frankfurt region (eu-central-1)
  Data: stored in Frankfurt PostgreSQL only
  Never touches: Mumbai or Singapore infrastructure

India User (Mumbai-based):
  DNS query → Cloudflare → returns Mumbai ELB IP
  All processing: Mumbai region (ap-south-1)
  Data: stored in Mumbai PostgreSQL

Global User (US-based merchant):
  DNS query → Cloudflare → geolocation → nearest region with service
  Or: explicit region selection (merchant configures in API settings)

Routing table (Cloudflare or Route53 Geolocation):
  EU countries → eu-central-1 (Frankfurt)
  India → ap-south-1 (Mumbai)
  SEA → ap-southeast-1 (Singapore)
  Default/US → us-east-1

Data residency enforcement:
  1. DNS/LB: initial routing (soft enforcement; CDN can be bypassed)
  2. API Gateway: tenant.region check (hard enforcement; returns 302 if wrong region)
  3. DB: data physically in region's RDS (strongest enforcement; data never moves)
  
  Audit: weekly check → any EU tenant data in non-EU storage? → compliance alert
```

---

## Scaling Strategy
```
Phase 1 (single region, single AZ):
  No DR; no multi-AZ
  RTO/RPO: best effort; accept downtime
  Acceptable for: pre-product-market-fit; < 100 customers

Phase 2 (single region, multi-AZ):
  Deploy across 3 AZs; PostgreSQL Patroni (3 nodes)
  AZ failure: zero downtime; automated Patroni promotion
  RTO: 30s for AZ failure; no cross-region DR yet
  Acceptable for: growth stage; < $1M/month transactions

Phase 3 (multi-region, active-passive):
  Primary region (Mumbai) + DR region (Hyderabad)
  Async streaming replication to DR
  RTO: 5 minutes; RPO: 60s
  Automated failover via Route53 + runbook Lambda
  Acceptable for: Series B/C; regulatory compliance requiring BCP

Phase 4 (multi-region, active-active):
  Two regions serving traffic simultaneously (50/50 or by geography)
  Conflict resolution: payment_id is globally unique; routing consistency
  Write routing: user always routed to same region (sticky by user_id hash)
  RPO: ~0 (any committed payment visible in both regions within 1s via sync replication)
  Acceptable for: public company; > $1B revenue; global user base
```

---

## Reliability Strategy
- **Automated failover**: Route53 health check → Lambda function → Patroni promote DR → within 3 minutes without human
- **Pre-warmed DR**: all services running in DR at reduced capacity; scale-out takes 2 min (not cold start 5 min)
- **Replication slot guard**: alert if DR replication slot lag > 30 min; drop slot if DR offline > 24h
- **No split-brain**: during failover window (30–60s after primary goes down): traffic may fail (acceptable); DNS converges
- **Quarterly DR drills**: automated test of failover; measured RTO/RPO must meet targets

---

## Security Considerations
- **DR access**: DR region has same security controls as primary (same IAM policies, same OPA rules, same SPIFFE trust domain)
- **Failover admin access**: breaking-glass procedure to access DR console during failover; elevated access auto-revokes after 4 hours
- **Data sovereignty**: during failover from EU to non-EU (unacceptable) → EU must have its own DR region within EU
- **Replication security**: streaming replication over TLS; replication user has replication privilege only
- **WAL encryption**: WAL contains financial data; encrypt at rest (RDS encryption) and in transit (TLS)

---

## Observability
```
DR health metrics:
  replication_lag_seconds             (alert if > 30s; critical if > 60s)
  dr_replica_connected               (boolean; alert if false)
  rpo_current_estimate_seconds       (derived from replication_lag; this IS the RPO right now)
  dr_health_check_status             (can DR serve traffic right now?)
  primary_region_health_score        (composite; feeds Route53 failover decision)

Failover metrics (during event):
  failover_trigger_time
  dr_serving_time (first successful request in DR)
  rto_achieved_seconds
  rpo_achieved_seconds
  transactions_lost_count (RPO impact in real terms)

Normal operation:
  cross_region_bandwidth_mbps         (replication stream size)
  dr_smoke_test_success_rate          (daily synthetic test of DR)
  db_slot_lag_bytes                   (replication slot WAL accumulation; alert > 10 GB)
```

---

## Chaos Testing
```
Experiment 1: AZ Failure (Most Common Failure Type)
  Inject:    Block all traffic to AZ-1a (isolate the AZ)
  Expected:  ALB: routes away from AZ-1a instances immediately (health check)
  Expected:  Patroni: if primary was in AZ-1a → promotes AZ-1b replica (< 30s)
  Expected:  Payment processing: continuous (brief blip during Patroni promotion)
  Expected:  RTO: < 30 seconds; RPO: 0 (synchronous in-region replication)
  Red flag:  Complete payment failure for > 1 minute (multi-AZ not working)

Experiment 2: Full Region Failure Simulation (Quarterly DR Drill)
  Inject:    Block Route53 health check responses from Mumbai
  Expected:  Route53: 3 failures in 30s → activates failover record (DR DNS)
  Expected:  DNS propagation: < 60s for most clients
  Expected:  DR PostgreSQL promotion: < 30s (automated Lambda trigger)
  Expected:  First successful payment in DR: < 3 minutes from failure
  Expected:  Total RTO: < 5 minutes
  Expected:  RPO: < 60 seconds (transactions in replication lag window lost)
  Red flag:  RTO > 5 minutes (automation failed; requires manual intervention)

Experiment 3: Replication Lag Spike
  Inject:    Throttle replication bandwidth to 10% of normal
  Expected:  Replication lag grows: 5s → 30s → (alert at 30s) → 60s → (critical at 60s)
  Expected:  Alert fires: "DR replication lag at 45 seconds; RPO at risk"
  Expected:  Incident response: identify cause (network throttle in this case)
  Expected:  Resolution: un-throttle → lag catches up → alert clears
  Red flag:  No alert until lag > 60s (alert threshold too loose)

Experiment 4: Split-Brain Prevention
  Inject:    Introduce network partition: primary cannot reach DR; DR cannot reach primary
             Simultaneously, trigger automated failover (DR thinks primary is down)
  Expected:  Both regions try to accept writes (split-brain scenario)
  Expected:  FENCING: primary detects it cannot reach DR → does NOT stop serving
             (primary doesn't know if it's a network issue or real failure)
  Expected:  Route53: only SWITCHES to DR when health check fails (not replication)
  Expected:  DR only accepts writes after Route53 points to it AND after Patroni promote
  Expected:  If both regions are promoted simultaneously: fencing token prevents split-brain
  Red flag:  Two writable primaries accepting payment writes simultaneously
```

---

## Monthly Cost Estimate
```
Scale: 50K TPS primary; DR region at 50% capacity (warm standby)

Primary Region (Mumbai):
  Already costed in respective case studies
  Estimate: ~$100,000/month for full payment platform

DR Region (Hyderabad) — 50% capacity warm standby:
  Application tier (50% of primary): $25,000/mo
  DB standby (same size as primary for failover capability): $9,600/mo
  Redis (snapshot restore; smaller): $1,500/mo
  Kafka MirrorMaker 2: $900/mo
  Monitoring + DR tooling: $1,000/mo
  Cross-region replication bandwidth ($0.08/GB): $480/mo
  Route53 health checks + DNS: $50/mo
─────────────────────────────────────────────────────────────────
DR Region Total: ~$38,530/month

Total with DR: $100,000 + $38,530 = $138,530/month
DR overhead: 38.5% additional cost for 99.99% availability target

Active-Active comparison (for reference):
  Both regions at 100% capacity: $200,000/month (2×)
  Active-Active overhead: 100% additional (vs 38.5% for active-passive)
  Verdict: Active-Passive is the right choice for this availability target
  Active-Active only justified if: RTO = 0 is a hard requirement AND revenue > $1M/month
```

---

## Architecture Review Checklist
```
□ RTO and RPO clearly defined and TESTED:
  → RTO < 5 min: measured in quarterly DR drill; automated alert if target missed
  → RPO < 60s: replication lag metric; alert if > 30s (half of RPO target)
  → Both measured end-to-end; not estimated

□ Multi-AZ within primary (handles most failures):
  → 3 AZs; all services deployed in all 3
  → DB: Patroni with sync replication across AZs (RPO=0 within region)
  → AZ failure: automated; no human intervention; < 30s

□ Cross-region (DR for catastrophic failure):
  → Async replication (< 60s RPO; no primary latency impact)
  → Pre-warmed standby (all services running in DR at reduced capacity)
  → Automated promotion (Patroni + Lambda; not manual)
  → DNS failover (Route53 health check; 60s TTL; < 2 min for most clients)

□ Data residency compliance:
  → EU primary → EU DR (Frankfurt → Amsterdam or Dublin)
  → India primary → India DR (Mumbai → Hyderabad)
  → No cross-border data movement even during failover

□ Split-brain prevention:
  → Only one writable primary at any time (Patroni fencing tokens)
  → Route53 + Patroni promotion are sequenced (Patroni after DNS switch)
  → Network partition test in chaos experiments

□ Failback plan:
  → After primary region recovers: sync data from DR back to primary
  → Planned failback (not automated; to avoid yo-yo effect)
  → Verify data consistency before switching back

□ SCC LENS:
  STATE: primary PostgreSQL (authoritative), DR PostgreSQL (eventually consistent replica)
  COORDINATION: Route53 health check (failover trigger), Patroni (promotion), replication slots (WAL delivery guarantee)
  CONCENTRATION: primary region handles all writes → replication stream is the single bottleneck for RPO
```

---

## Staff Engineer Discussion Points

**"How do you handle the 60-second RPO window — what exactly is lost?"**
Any payment that was committed to the primary PostgreSQL but not yet replicated to DR in those 60 seconds is lost. In practice: (1) payments in the AUTHORISED state (card pre-authorised but not captured) — the authorisation is lost; the card network still holds the reserve; customer will see the hold but no payment record → needs reconciliation; (2) payments that were CAPTURED in that window — merchant does not receive money; customer was charged → refund needed. This is why we have reconciliation jobs (Case 21): the card network settlement file becomes our source of truth to identify which payments need to be recreated after failover. The RPO is not just a latency number — it has a financial recovery procedure attached.

**"Why not make the DB synchronous cross-region to achieve RPO=0?"**
Synchronous replication means: every DB write waits for the DR region to confirm receipt before returning. For a Mumbai → Hyderabad distance of ~1,400km, that's ~7ms round-trip latency added to every payment. At 50K TPS, every checkout adds 7ms. For a payment that should complete in < 2 seconds, adding 7ms might be acceptable. But: (1) network jitter can spike this to 50–100ms, (2) if DR region is experiencing high load, it slows your primary, (3) DR network partition = primary DB hangs waiting for sync ACK. The recommendation: synchronous within a region (between AZs, ~2ms RTT); asynchronous between regions. Accept the 60-second RPO and invest in excellent reconciliation tooling.

**"How do you test DR readiness without risking a real outage?"**
Three approaches of increasing aggressiveness: (1) Non-disruptive: connect to DR replica and run read queries; verify data is fresh; confirm services are running in DR. Zero risk. (2) Shadow mode: duplicate a small percentage of production traffic to DR region; verify responses match primary. Slight cost; some complexity. (3) Full drill (our quarterly test): block Route53 health check responses from primary; measure actual failover. This is real failover; primary traffic is disrupted for the drill duration (~5 min). The key: only do full drills during scheduled maintenance windows; communicate to customers. The value: you find gaps (automation failure, WAL slot misconfiguration, service dependencies) before a real disaster, not during one.

---

## Cheat Sheet Tie-in

```
RPO vs RTO (always define both):
  RPO = how much data can we lose? (tied to replication lag)
  RTO = how fast must we recover? (tied to failover automation)
  RPO=0 + RTO=0 = active-active (expensive; complex)
  RPO<60s + RTO<5min = active-passive warm standby (most payment platforms)

MULTI-AZ vs MULTI-REGION:
  Multi-AZ: same city; handles hardware/power failures in one datacenter
            Active-Active; synchronous replication; RPO=0; RTO<30s
  Multi-Region: different geography; handles full datacenter city outage
                Active-Passive; async replication; RPO=60s; RTO=5min

FAILOVER SEQUENCE:
  1. Health check fails (30s)
  2. DNS failover → DR region (60s TTL → clients start using DR)
  3. Patroni promotes DR replica → DR is now writable (30s)
  4. DR services verify DB connectivity (1min)
  5. Smoke test confirms DR serving correctly (1min)
  Total: < 5 minutes automated

REPLICATION SLOT CAVEAT:
  Replication slot prevents WAL cleanup → if DR offline for days → primary disk fills → OUTAGE
  Mitigation: alert if slot lag > 1h; drop slot if DR offline > 24h
  Without slot: DR can't reconnect if too far behind (WAL deleted) → must rebuild from scratch

SCC LENS:
  STATE: primary DB (authoritative/writable), DR DB (async replica/readable)
  COORDINATION: Route53 + Patroni (failover coordination), replication slot (WAL delivery)
  CONCENTRATION: replication stream is the bottleneck (1 stream per region pair) → monitor lag

VS ACTIVE-ACTIVE:
  Active-Passive: 1.38× cost; RPO=60s; RTO=5min; manageable complexity
  Active-Active:  2.0× cost;  RPO~0;   RTO=0;    bi-directional conflict resolution needed
  Choose active-active when: revenue loss per minute of downtime > monthly cost of 2× infra
```

---

*End of Advanced Track — Case Studies 21–30 Complete*
*System Design Fundamentals Handbook: 30 case studies across Foundational, Intermediate, and Advanced tracks.*
