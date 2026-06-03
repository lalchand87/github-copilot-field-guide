# Case Study 40 — Staff Engineer Leadership: Design Reviews, Incidents, and Technical Strategy

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Architecture review criteria, Incident command, RFC process, Technical strategy.

---

## Business Context

Staff Engineer is not just a senior developer. It is a leadership role with technical scope spanning multiple teams. The difference between a Senior Engineer and a Staff Engineer: Senior engineers solve well-defined technical problems. Staff engineers identify which problems are worth solving, align teams on solutions, and ensure the technical direction of the platform is coherent over years — not just sprints.

This case study covers the non-coding parts of the Staff Engineer job: design reviews, incident command, RFC processes, technical strategy, and the behavioural signals that interviewers look for.

---

## Recognition Framework

### What Staff Engineers Actually Do

```
30% Individual contribution:
  Deep technical work on the hardest problems
  Proof-of-concept for new technologies
  Code review on critical systems
  Technical documentation (ADRs, RFCs)

40% Multiplying others:
  Architecture review (CS21–30 case studies are the knowledge base)
  Mentoring senior engineers
  Design reviews (catch problems before code is written)
  Hiring bar: calibrating what "senior" means

30% Technical leadership:
  Technical strategy (3-year roadmap)
  Cross-team alignment
  Incident command
  Representing engineering in product/business discussions
```

---

## Architecture Design Review

### What to Look for in a Design Review

```
Level 1: Correctness questions (must answer before anything else)
  1. What is the source of truth for each piece of data?
  2. What happens when this component fails?
  3. Is this operation safe to retry? (idempotent?)
  4. What are the consistency requirements? (strong? eventual? which operations need which?)

Level 2: Scale questions
  5. What's the worst-case load? (peak × 2 for headroom)
  6. Are there hot partitions? (celebrity problem, hot merchant problem)
  7. What breaks first? (identify the bottleneck component)
  8. Can this scale horizontally? (or is there a single-threaded bottleneck?)

Level 3: Operational questions
  9. How do you know when this is broken? (observability: what metrics, what alerts?)
  10. How do you debug this at 3am? (trace, logs, runbook)
  11. How long does deployment take? (is it zero-downtime?)
  12. How do you roll back a bad deployment in 5 minutes?

Level 4: Security and compliance questions (payment platform specific)
  13. Does any PII flow through this system? Is it encrypted? Is access logged?
  14. Can one tenant access another tenant's data? (multi-tenant isolation)
  15. Is there an audit trail for all financial state changes?
  16. Are secrets handled correctly? (no plaintext, no env vars in code)
```

### Sample Architecture Review: New Payment Service Proposal

```
Proposal: "We want to add a Buy Now Pay Later (BNPL) service that splits payments into EMIs."

Staff Engineer review questions:

Q1 (Correctness): Where is the EMI schedule authoritative?
  → Is it in the payment DB, a new BNPL DB, or the lending service?
  → If the EMI schedule changes (customer requests restructuring): how does that propagate?
  → What prevents creating a BNPL plan without corresponding ledger entries?

Q2 (Consistency): What happens if the authorisation succeeds but EMI creation fails?
  → Do we charge the customer for the full amount (not EMI)?
  → Do we void the authorisation and show an error?
  → Saga pattern needed here: authorise → create EMIs → capture OR void

Q3 (Scale): How many active BNPL plans will there be?
  → 1M users × 3 active plans avg = 3M plans
  → Each plan has 12 EMI schedules = 36M schedule rows
  → Is the existing payment DB able to handle this or do we need a new service?

Q4 (Regulatory): Has RBI lending regulations been consulted?
  → BNPL is regulated lending; requires NBFC license or banking partnership
  → KYC requirements differ from payment; credit bureau checks required
  → This needs compliance review before architecture review

Q5 (Failure mode): What happens if a customer's EMI payment fails?
  → Late fee? Collections process? Credit bureau reporting?
  → These are business decisions that must be made before system design

Q6 (Existing systems): Can we reuse the existing payment gateway (Case 21)?
  → EMI collection is a recurring payment (mandate pattern from Case 23)
  → Yes: UPI AutoPay mandate + payment gateway can handle EMI collection
  → New service needs: EMI scheduling, credit scoring, and risk management
  → Core payment infrastructure: reuse existing

Outcome: Proposed architecture is directionally correct but needs:
  1. Regulatory approval before build
  2. Saga pattern for atomic authorisation + EMI creation
  3. Separate BNPL service (not in payment-service) for clean domain separation
  4. UPI mandate for EMI collection (reuse existing infrastructure)
```

---

## RFC (Request for Comments) Process

### When to Write an RFC
```
Write an RFC when:
  - The decision affects more than one team
  - There are multiple technically valid approaches (tradeoffs need discussion)
  - The change is irreversible or costly to undo (new DB, API breaking change)
  - Significant infrastructure cost ($10K+/month)
  - New external dependency (new vendor, new protocol)

Don't write an RFC for:
  - Implementation details within a single service
  - Refactoring that doesn't change external behavior
  - Bug fixes
  - Minor performance optimizations
```

### RFC Template
```markdown
# RFC: [Title]
Date: [YYYY-MM-DD]
Author: [Name]
Status: [Draft | In Review | Accepted | Rejected | Superseded]
RFC-number: RFC-042

## Summary (3 sentences max)
What are we proposing? Why now? What's the expected outcome?

## Motivation
What problem does this solve? Why can't we solve it with existing tools?
Include metrics: "payment retry rate is 2%; idempotency would reduce it to 0.1%"

## Design
Describe the proposed solution in enough detail for engineers to evaluate.
Include: data model, API design, sequence diagrams, failure modes.

## Alternatives Considered
What other approaches were evaluated? Why were they rejected?
This section prevents "why didn't you consider X?" in review.

## Implementation Plan
Phases, timeline, rollback strategy.

## Open Questions
What is still uncertain? What needs input from reviewers?

## Success Metrics
How will we know if this worked? What are we measuring?
```

---

## Incident Management

### Incident Command Structure
```
Incident Commander (IC):
  Role: owns the incident; drives to resolution; communicates with stakeholders
  NOT: the person with the most technical knowledge (that's the tech lead)
  IS:  the person who keeps the room calm, delegates tasks, and makes decisions

Tech Lead:
  Role: diagnoses the problem; proposes fixes; executes changes
  Reports to IC; doesn't get distracted by stakeholder questions

Communications Lead:
  Role: writes status updates to stakeholders (Slack, status page)
  Frees tech lead to focus on technical problem

Scribe:
  Role: documents all actions taken (timeline; what was tried; what worked)
  Post-incident review depends on accurate timeline

Severity definition (payment platform):
  P0: payment processing completely unavailable (ALL payments failing)
  P1: payments > 5% failure rate or > 2s latency for > 5 minutes
  P2: specific payment method unavailable (e.g., UPI down; card working)
  P3: degraded performance; within SLO but trending bad
```

### Incident Command in Action
```
T=0: Alert fires — "payment success rate dropped to 85%"

T+2min: IC joins incident bridge
  IC: "What's the impact? How many users affected?"
  Tech Lead: "50K TPS × 15% failure = 7,500 failures/min. All payment methods."
  IC: "This is P0. Declare major incident. Comms lead: post to status page now."

T+5min: IC drives hypothesis generation (not solving; generating)
  IC: "What changed in the last 30 minutes?"
  Tech Lead: "Payment-service v1.2.4 deployed at T-25 minutes."
  IC: "What's the rollback time?"
  Dev: "3 minutes to roll back."
  IC: "Roll back NOW. Don't investigate further — the correlation is clear."

T+8min: Rollback in progress
  IC: "While rollback runs: what else changed? Any infrastructure changes?"
  SRE: "No infra changes. Rollback at 50% now."

T+11min: Rollback complete
  Tech Lead: "Success rate back to 99.8%. Recovering."
  IC: "Good. Comms lead: update status page — resolved, investigating cause."
  IC: "Post-incident review in 48 hours. Scribe: send me the timeline now."

Key staff engineer behaviors shown:
  1. Rollback before full root cause analysis (bias toward resolution)
  2. Clear delegation (IC manages room; tech lead fixes problem)
  3. Document everything in real-time (post-mortem depends on it)
  4. Communicate early and often (status page updated every 5 min)
```

### Post-Incident Review (Blameless)
```markdown
# Post-Incident Review: Payment Success Rate Drop
Date: 2024-01-16
Severity: P0 (15 minutes duration)
Impact: ~112,500 payment failures; estimated revenue impact ₹5.6M

## Timeline
T-25min: payment-service v1.2.4 deployed (Argo Rollouts canary at 5%)
T=0: Alert fires (success rate 85%)
T+2: IC joined; P0 declared
T+5: Decision to roll back
T+11: Rollback complete; success rate 99.8%

## Root Cause
v1.2.4 introduced a config change that set DB connection pool max = 5 (was 50).
At 5% canary: 5% × 7,500 failures/min = 375 failures/min → below alert threshold
When canary promoted to 50%: failures crossed alert threshold

## What Went Well
- Alert fired within 3 minutes of P1 threshold breach
- Rollback decision made quickly (bias toward resolution)
- Status page updated throughout

## What Went Wrong
- Canary analysis only checked error rate; did not check latency or pool metrics
- DB connection pool config change was not flagged in code review
- Canary analysis threshold was too low: required 1% sustained error (should be 0.1%)

## Action Items (owners + deadlines)
1. Add DB connection pool metrics to canary analysis gates [SRE, T+7 days]
2. Add config change detection to PR review checklist [Platform, T+14 days]
3. Lower canary error threshold from 1% to 0.1% [SRE, T+3 days]
4. Add load test in staging before every deployment [Dev, T+30 days]

## Blameless framing
The system failed, not the engineer. The engineer followed the process correctly.
The process (canary thresholds) needed improvement. That's what we're fixing.
```

---

## Technical Strategy (3-Year View)

### Staff Engineer's Planning Horizon
```
1-week horizon (developer mindset): "which ticket should I work on?"
3-month horizon (senior mindset): "what should our team accomplish this quarter?"
1-year horizon (lead/staff mindset): "where should our platform be in 12 months?"
3-year horizon (staff/principal mindset): "what technical bets do we need to make today that will pay off in 3 years?"

Technical debt vs technical investment:
  Technical debt: something we did that's now slowing us down
  Technical investment: something we're building now that will accelerate us later
  
  Staff engineer role: identify which debts have high interest (slowing us down now)
                       and which investments have high returns (will accelerate most)
```

### Example: Payment Platform Technical Strategy 2024–2027

```
Observation: our payment infrastructure is a monolith. We have 3 engineers who understand it.
             Every change takes 3 weeks because of coupling. 40% of developer time is spent
             understanding existing code rather than writing new features.

12-month goal: extract core payment domain into isolated service with clean API
  Milestone 1 (Q1): define payment domain API; freeze interface
  Milestone 2 (Q2): extract payment-service behind API; existing code calls API
  Milestone 3 (Q3): migrate all callers to new API; decommission old coupling
  Milestone 4 (Q4): document; hire for payment-service team

24-month goal: all critical services independently deployable; < 5 min deploy time
36-month goal: platform serves 10× current scale without architectural changes

How to frame this for management:
  "Our current architecture limits deploy frequency to 1/week (too many dependencies).
   The market demands faster iteration (competitors deploy daily).
   This plan will get us to daily deploys in 12 months and 10× scale in 36 months.
   Cost: 2 engineer-quarters of focused work. Benefit: 5 engineer-quarters saved per year ongoing."
  
  Technical argument + business case = approved investment
  Technical argument alone = "add it to the backlog"
```

---

## Staff Engineer Interview Signals

### What Interviewers Look for at Staff Level
```
1. System design breadth AND depth:
   Breadth: knows all 40 case studies in this handbook
   Depth: can go deep on any one (e.g., "how exactly does Kafka exactly-once work?")
   
2. Technical judgment (tradeoffs, not just patterns):
   Bad answer: "we should use Kafka for everything"
   Good answer: "for this use case, Kafka's replay is valuable, but the ops overhead
                 means we'd start with SQS and migrate to Kafka when we need fan-out"

3. Ambiguity handling:
   Bad: "I can't answer without more requirements"
   Good: "Let me make some assumptions and design to those; tell me which to revisit"
   Staff engineers start with what they know and refine

4. End-to-end thinking:
   Bad: only thinks about the happy path
   Good: "What happens when the card network times out? What if the DB is unavailable
          during this window? Who gets paged? What's the runbook?"

5. Non-technical skills:
   "How have you influenced technical direction without authority?"
   "Tell me about a time you changed an engineer's design without overruling them."
   "How have you handled a situation where you disagreed with your team's technical decision?"

6. SCC lens fluency:
   Can describe any system design in: STATE (what's stored where), 
   COORDINATION (how consistency is maintained), CONCENTRATION (what's the bottleneck)
   This framework signals systems thinking vs feature thinking
```

### Staff Engineer Self-Assessment Rubric
```
Technical:
  □ Can design any of the 40 systems in this handbook in 45 minutes
  □ Can identify the bottleneck in any given architecture in 5 minutes
  □ Can explain the tradeoffs of 3 alternatives for any component decision
  □ Knows when NOT to build (use existing tool) vs when to build custom

Leadership:
  □ Has run an architecture review that changed a team's approach
  □ Has written and gotten consensus on an RFC
  □ Has led an incident to resolution as IC or tech lead
  □ Has defined technical success metrics for a project

Communication:
  □ Can explain any system design to a PM or business stakeholder
  □ Can write a blameless post-incident review
  □ Can represent engineering in technical tradeoff discussions with business
  □ Can give constructive technical feedback without demotivating engineers

Missing any of these → specific growth area to work on before Staff interview
```

---

## Architecture Review Checklist (Master Checklist)
```
This integrates all 40 case studies into a universal review framework:

DATA AND STATE:
  □ Source of truth: one authoritative store per data type
  □ Consistency model: strong where required (financial); eventual where acceptable
  □ Immutability: append-only for audit, financial, event data
  □ Idempotency: all mutating operations safe to retry (Case 36)
  □ Cache coherence: stale data acceptable? for which operations? (Case 37)

AVAILABILITY AND RESILIENCE:
  □ Failure mode for each component: what degrades? what fails hard?
  □ Circuit breaker: external dependencies protected (Card network, NPCI, banks)
  □ Retry strategy: exponential backoff + jitter + max retries defined
  □ Multi-AZ: all services deployed across ≥ 2 AZs
  □ DR plan: RPO and RTO defined and tested quarterly (Case 30)
  □ PDB: no service can be fully drained in one kubectl drain (Case 32)

SCALE AND PERFORMANCE:
  □ Bottleneck identified: what breaks first under 10× load?
  □ Hot partitions: celebrity/merchant/user problem handled
  □ Read/write ratio: caching strategy appropriate per data type (Case 37)
  □ HPA configured: services autoscale on the right metric (Case 32)
  □ DB indexes: queries have appropriate indexes; EXPLAIN ANALYZE reviewed

SECURITY AND COMPLIANCE:
  □ PII identified: encrypted at rest and in transit; access logged
  □ Secrets: in Vault/Secrets Manager; not in Git/env vars (Case 34)
  □ mTLS: inter-service traffic authenticated and authorised (Case 27)
  □ Audit log: every financial state change, access control decision logged (Case 10)
  □ PCI scope: card data handled only in tokenisation vault (Case 21)
  □ Multi-tenant: tenant isolation verified at DB, app, and network layer (Case 26)

OBSERVABILITY:
  □ Golden signals: latency, traffic, errors, saturation — per service (Case 17)
  □ Distributed traces: end-to-end trace across all hops (Case 29)
  □ Alerts: symptom-based, with runbooks, at correct severity (Case 17)
  □ Dead man's switch: monitoring system failure is itself alerted (Case 17)
  □ SLOs defined: error budget and burn rate alerts configured

OPERATIONS:
  □ Deployment: zero-downtime rolling; canary analysis gates (Case 33)
  □ Rollback: git revert → production in < 5 minutes
  □ Runbook: every alert has a written runbook
  □ On-call: clear ownership; escalation policy defined
  □ DR drill: quarterly test of failover (Case 30)
```

---

*End of System Design Fundamentals Handbook — 40 Case Studies Complete*

```
HANDBOOK STRUCTURE:
  CS01–10: Foundational (URL shortener, Cache, Rate limiter, CDN, API Gateway, 
           Consistent hashing, Load balancer, Feature flags, Key-value store, Audit log)
  CS11–20: Intermediate (Notification, News feed, Chat, Slack, Collab editor,
           Search, Monitoring, Analytics, Recommendation, Job scheduler)
  CS21–30: Advanced (Payment gateway, Fraud detection, UPI, Ledger, Rate limiter advanced,
           Multi-tenant, Zero trust, CQRS, Observability, Multi-region DR)
  CS31–40: Expert (Service mesh, Kubernetes, CI/CD GitOps, Secrets/PKI, API gateway advanced,
           Idempotency, Caching, DB sharding, Event streaming, Staff leadership)
```
