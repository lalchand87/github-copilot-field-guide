# Curriculum · Reliability

[← System Design index](../README.md)

> 8 lessons in **Reliability**. Part of the Curriculum — Build the fundamentals.

## Contents

- **Reliability** (8): [SLOs and error budgets](#slos-and-error-budgets) · [Circuit breakers](#circuit-breakers) · [Bulkheads and isolation](#bulkheads-and-isolation) · [Load shedding and admission control](#load-shedding-and-admission-control) · [Timeouts, deadlines and cancellation](#timeouts-deadlines-and-cancellation) · [Health, readiness and liveness](#health-readiness-and-liveness) · [Disaster recovery and restore](#disaster-recovery-and-restore) · [Graceful degradation](#graceful-degradation)

## Reliability

### SLOs and error budgets

*Define a measurable reliability target from the user's perspective, and use the allowed shortfall as a budget that governs how aggressively you may change the system.*

**Flow:** `User-visible SLI` → `Target SLO` → `Error budget` → `Burn rate` → `Release decision`

> **The 30-second version**  
> Measure a user-visible indicator, set a target below perfection, treat the shortfall as a budget, alert on burn rate, and agree in advance what happens when it runs out.

**The problem**

Two teams argue about reliability. One wants to ship features weekly; the other wants a change freeze because of last month's incident. Neither has a shared definition of how reliable the service should be, so the argument is about temperament rather than about facts, and it recurs every quarter.

Meanwhile the dashboard says the service is ninety-nine point nine per cent available, measured as the proportion of requests returning a non-error status at the load balancer. Users are complaining about timeouts. Both statements are true, because the metric does not measure what users experience.

> **An error budget converts reliability from an opinion into an allowance**  
> If the target is 99.9 per cent over thirty days, then 0.1 per cent of that period — about forty-three minutes — is budgeted unreliability. Spending it is not failure; it is using the allowance. What the budget provides is a shared, factual answer to the question of whether to ship or to stabilise, replacing a recurring argument with a number both sides already agreed to.

**Mental model**

An indicator measures something users actually experience. An objective sets the target for that indicator. The gap between the target and perfection is a budget that gets consumed by incidents and degraded performance, and its remaining balance drives decisions.

1. **SLI** — A measurement of user-visible behaviour — success rate, latency below a threshold, freshness.
2. **SLO** — The target for that indicator over a window, such as 99.9 per cent over 30 days.
3. **Error budget** — The permitted shortfall — what remains of 100 per cent after the objective.
4. **Burn rate** — How fast the budget is being consumed relative to the window.
5. **Policy** — What happens when the budget is exhausted, agreed in advance rather than argued during an incident.

> **A 100 per cent target is neither achievable nor desirable**  
> Perfect reliability would require never changing anything, and even then dependencies fail. More importantly, the cost of each additional nine rises steeply while the user-perceived benefit falls — beyond a point, the network between the user and the service is less reliable than the service itself, so further investment improves nothing anyone can perceive. Choosing a target below perfection is what makes the budget exist.

**How it works**

**What the objective actually permits**

```text
ERROR BUDGET OVER 30 DAYS

  99%      = 7 hours 12 minutes of failure
  99.9%    = 43 minutes
  99.95%   = 21 minutes
  99.99%   = 4 minutes 20 seconds
  99.999%  = 26 seconds

-> each additional nine costs roughly 10x more
   engineering effort
-> and at 99.99%, a single unlucky deploy consumes the
   entire month's budget

REQUEST-BASED vs TIME-BASED
  request-based: failed requests / total requests
    -> a 5-minute outage at 3 a.m. costs little budget
    -> matches user impact better
  time-based: minutes where the service was unhealthy
    -> simpler to explain
    -> treats quiet and peak outages identically

-> request-based is usually the honest choice, because
   it weights failures by how many users experienced
   them

BURN RATE - THE ACTIONABLE NUMBER
  burn rate 1 = budget exactly exhausted at window end
  burn rate 2 = exhausted in half the window
  burn rate 14.4 = a 30-day budget gone in ~2 days

ALERT ON BURN RATE, NOT ON THRESHOLD CROSSINGS
  fast burn  (14.4x over 1 hour)  -> page someone now
  slow burn  (1x over 6 hours)    -> raise a ticket
  -> this is what stops alerting on every blip while
     still catching real degradation early
```

1. **Measure what users experience, not what servers report** — A load balancer's success rate misses client timeouts, slow responses and partial failures — the things users actually notice.
2. **Set the target from user tolerance, not from ambition** — The right number is the point beyond which users cannot perceive improvement, which is usually lower than teams assume.
3. **Include latency in the definition of success** — A response that arrives after the user has given up is a failure, so successful means fast enough as well as correct.
4. **Alert on burn rate, with multiple windows** — Fast burn pages immediately; slow burn raises a ticket. Threshold alerts either fire constantly or too late.
5. **Agree the policy before you need it** — What happens at budget exhaustion must be decided when nobody is under pressure, or it will be renegotiated during every incident.
6. **Exclude what you do not control, honestly** — Dependency outages still affect users; excluding them makes the number comfortable rather than true.

**Choosing indicators that mean something**

```text
BAD SLI: server-side HTTP success rate
  misses: client timeouts, DNS failures, slow responses
         that users abandoned, partial page failures
  -> can read 99.99% while users are unable to work

BETTER SLI: proportion of requests that completed
successfully within 500 ms, measured as close to the
user as possible

THE THREE COMMON SHAPES
  availability  successful / total requests
  latency       requests faster than a threshold /
                total requests
  freshness     data younger than a threshold /
                total reads

A LATENCY SLI IS A RATIO, NOT AN AVERAGE
  not "p99 latency is 300 ms"
  but "99% of requests complete within 500 ms"
  -> the second is budgetable: each slow request
     consumes budget
  -> the first is a statistic that cannot be spent

CRITICAL USER JOURNEYS, NOT ENDPOINTS
  measure "user can complete checkout" rather than
  "POST /api/orders returns 200"
  -> a journey can fail while every endpoint succeeds
  -> and a non-critical endpoint failing should not
     consume the checkout budget
```

> **An error budget nobody enforces is a reporting exercise**  
> The mechanism works because exhausting the budget changes behaviour — typically pausing feature releases in favour of reliability work until the balance recovers. If the response to exhaustion is a discussion that concludes with shipping anyway, then the budget is a dashboard rather than a control, and the original argument about whether to ship returns unchanged. The policy, and management's commitment to it, is the part that matters.

**Worked example**

An e-commerce checkout service, with the objective derived rather than asserted.

**Defining and operating the SLO**

```text
CRITICAL USER JOURNEY
  "customer completes a purchase"

SLI
  proportion of checkout attempts that succeed within
  3 seconds, measured from client-side instrumentation
  -> client-side, because a server that returns 200
     after the user gave up did not succeed

SLO  99.9% over 30 days
  why not 99.99%?
    -> the payment provider's own SLA is 99.95%
    -> promising more than a hard dependency delivers
       is not a target, it is a fiction
  why not 99%?
    -> 7 hours of failed checkouts per month is
       commercially unacceptable

ERROR BUDGET  43 minutes per 30 days

BURN-RATE ALERTS
  14.4x over 1 hour   -> page  (budget gone in 2 days)
  6x over 6 hours     -> page
  1x over 3 days      -> ticket

POLICY, AGREED IN ADVANCE
  budget > 50% remaining -> ship freely
  budget < 25% remaining -> only low-risk changes
  budget exhausted       -> feature freeze; reliability
                            work only until it recovers

WHAT THIS CHANGED
  the ship-or-stabilise argument became a lookup
  the payment dependency became visible as a ceiling
  client-side measurement revealed failures the
    server-side dashboard had never shown
```

| Metric | Value | Note |
|---|---|---|
| Target | 99.9% / 30 d | 43 min budget |
| Measured | client-side | **what users see** |
| Ceiling | payment 99.95% | dependency bound |
| Policy | pre-agreed | no incident debates |

> **Your SLO cannot exceed your dependencies' reliability**  
> A service that must call a payment provider with a 99.95 per cent SLA cannot credibly promise 99.99 per cent for any journey requiring that call, unless it can complete without it. Enumerating hard dependencies and their reliability produces an upper bound on what is achievable, and doing that arithmetic often reveals that the ambitious target under discussion was never possible — which is a far more useful conversation than trying to reach it.

**When to use it**

- **Any service with users who notice when it fails**, which is most of them.
- **When ship-versus-stabilise arguments recur**, since the budget resolves them factually.
- **Prioritising reliability work**, by showing where budget is actually consumed.
- **Setting customer expectations**, with an internal objective stricter than any external commitment.
- **Evaluating dependencies**, whose reliability bounds your own.

**When to avoid it**

- **Do not set objectives on metrics users do not experience**, which produces comfortable numbers and unhappy users.
- **Do not target 100 per cent**, which eliminates the budget and forbids change.
- **Do not define an SLO without a policy**, which reduces it to reporting.
- **Do not promise externally what you target internally**; the internal objective should be stricter.
- **Do not apply one objective to everything**, since critical journeys and background features differ.

**Advantages**

- **Turns reliability into a shared, factual decision** rather than a recurring argument.
- **Makes the cost of change explicit**, with velocity and stability visibly connected.
- **Focuses effort** on what actually consumes budget.
- **Burn-rate alerting** catches real degradation without firing on every blip.
- **Exposes dependency limits**, bounding what is achievable.
- **Gives permission to fail** within an agreed allowance, which reduces incident anxiety.

**Disadvantages**

- **Only as good as the indicator**, and choosing a genuinely user-visible one is hard.
- **Requires organisational commitment**, since an unenforced budget changes nothing.
- **Client-side measurement is harder** than scraping server metrics.
- **Can be gamed** by defining success loosely or excluding inconvenient failures.
- **Windowing effects** mean a large incident distorts the budget for the rest of the period.

**Trade-offs**

**Target selection trade-offs**

| Target | Budget / 30 d | Cost | Appropriate for |
|---|---|---|---|
| 99% | 7 h 12 m | Low | Internal tools; batch systems |
| 99.9% | 43 m | Moderate | Most user-facing services |
| 99.95% | 21 m | High | Revenue-critical paths |
| 99.99% | 4 m 20 s | Very high | Infrastructure others depend on |
| 99.999% | 26 s | Extreme | Rarely justified above the network's own reliability |

The steepness of that cost curve is the argument for choosing carefully. Each additional nine is roughly ten times the effort, and past a certain point the user's own network is less reliable than the service, so the improvement is undetectable to the people it was meant to serve.

**How it fails**

**SLO programme failures**

| Failure | Cause | Fix |
|---|---|---|
| Dashboard healthy, users unhappy | Indicator measures servers, not user experience | Client-side measurement of critical journeys |
| Budget exhausted and nothing changes | No agreed policy, or no commitment to it | Pre-agreed policy with management backing |
| Alerts fire constantly | Threshold alerting rather than burn rate | Multi-window burn-rate alerts |
| Real degradation detected late | Only slow-burn alerting configured | Add a fast-burn window that pages |
| Target impossible to meet | Objective exceeds dependency reliability | Compute the ceiling from hard dependencies |
| Good numbers, bad experience | Latency excluded from success | Define success as correct and fast enough |
| Budget permanently exhausted | Target set aspirationally rather than from tolerance | Re-set the target to what users require |

**Limits**

> **Budget arithmetic**
>
> - **99.9% over 30 days** is 43 minutes; **99.99%** is 4 minutes 20 seconds.
> - **Each additional nine** costs roughly an order of magnitude more effort.
> - **Fast-burn alert**: around 14.4× burn over an hour consumes a 30-day budget in two days.
> - **Dependency ceiling**: your achievable objective is bounded by the product of your hard dependencies' reliability.
> - **Internal targets** should be stricter than external commitments, leaving room to breach internally without breaching contractually.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| SLOs with error budgets | Balancing velocity and reliability | Requires commitment and good indicators |
| Uptime percentage only | Simple reporting | Misses latency and partial failure |
| Incident count | Rough trend sense | Not user-weighted; easily gamed |
| Contractual SLAs | Customer commitments | Legal rather than operational |
| MTTR and MTBF | Operational maturity tracking | Not a release-decision mechanism |
| No formal target | Very early projects | Recurring unresolvable arguments |

Service-level agreements are a different instrument entirely: they are contractual commitments with financial consequences, and the internal objective should always be stricter so that an internal breach is a warning rather than a liability.

**In real systems**

- **Error budget policies that pause feature work** are the mechanism's original form, and the enforcement is what distinguishes a functioning programme from a dashboard.
- **Multi-window burn-rate alerting** has largely replaced static threshold alerting, because it catches fast degradation without firing on brief blips.
- **Client-side measurement** consistently reveals failures invisible to server metrics, which is why journey-level instrumentation matters.
- **Dependency reliability as a ceiling** is standard practice when setting objectives, since promising more than a hard dependency provides is not achievable.
- **Internal objectives stricter than external agreements** give a margin between operational concern and contractual breach.

**Common mistakes**

- **Measuring server-side success** rather than user experience.
- **Targeting 100 per cent**, which eliminates the budget entirely.
- **No agreed policy** for budget exhaustion, so nothing changes.
- **Ignoring latency**, counting slow responses as successes.
- **Setting targets above dependency reliability**, making them unachievable.
- **Threshold alerts instead of burn rate**, producing noise or lateness.
- **One objective for all functionality**, conflating critical journeys with background features.

**The staff-level view**

SLO programmes fail on indicator choice and on organisational commitment far more often than on arithmetic.

- **Insist the indicator reflects a user journey.** Endpoint success rates can read beautifully while users cannot complete the task, and that gap is where trust in the whole programme is lost.
- **Get the policy agreed before the first breach.** Decided under pressure, the answer is always to ship anyway, and the budget becomes decorative from that point onward.
- **Compute the dependency ceiling early.** It frequently shows the proposed target was never attainable, which redirects the conversation from effort to architecture.
- **Include latency in success.** A slow response that the user abandoned is a failure, and objectives that ignore this report health during exactly the degradations users complain about.
- **Use burn-rate alerting with at least two windows.** One catches sudden severe degradation, the other catches slow erosion, and threshold alerting does neither well.

**Go deeper**

A service level objective sets a target for an indicator that measures what users actually experience, and the gap between that target and perfection is an error budget. At 99.9 per cent over thirty days the budget is about forty-three minutes of failure — an allowance to be spent, not a line never to be crossed. Its balance gives a factual answer to whether the team should be shipping or stabilising, which replaces an argument that otherwise recurs indefinitely.

The indicator is where most programmes succeed or fail. Server-side success rates can read 99.99 per cent while users cannot complete their task, because they miss client timeouts, abandoned slow responses and partial failures. Good indicators are measured near the client, expressed as ratios, framed around critical journeys rather than endpoints, and treat latency as part of success. The achievable target is also bounded by hard dependencies — promising more reliability than a required provider offers is not a goal but a fiction.

Operationally, alerting should use multi-window burn rates rather than thresholds: a fast window pages on severe degradation, a slow one raises a ticket on gradual erosion. And the policy for budget exhaustion must be agreed in advance with organisational backing, because a budget whose exhaustion changes nothing is a dashboard, and the decision reached under incident pressure is always to ship anyway.

Service level objectives make reliability measurable and negotiable, replacing a recurring argument between velocity and stability with a shared allowance.

**The budget is the mechanism.** Choosing a target below perfection creates a quantity of permitted unreliability — forty-three minutes per month at 99.9 per cent — and reframes failure as spending rather than as fault. That reframing is what makes it useful: a team with budget remaining can ship confidently, and a team that has exhausted it has an objective reason to stabilise. Both sides of the perennial argument agreed to the number in advance, so the decision becomes a lookup rather than a negotiation.

**The indicator decides whether any of it means anything.** Metrics gathered where they are easiest to collect — load balancer status codes, server-side success rates — systematically miss the failures users notice: client timeouts, DNS problems, partial page failures, and responses that arrived long after the user gave up. An objective built on such a metric reads healthy during exactly the degradations that generate complaints, and once that discrepancy is visible the programme loses credibility entirely. Indicators should be measured close to the client, expressed as ratios of good events to total events, defined around critical user journeys rather than individual endpoints, and must treat latency as part of success.

**Dependencies impose a ceiling.** A journey that cannot complete without a provider offering 99.95 per cent cannot itself achieve 99.99 per cent, and the arithmetic across all hard dependencies frequently shows that an ambitious target under discussion was never attainable. Establishing this early converts an unproductive conversation about effort into a productive one about architecture — whether the dependency can be made optional, cached, or degraded around.

**Burn rate is the operational form.** Alerting on the objective itself is either too late, because the window is long, or too noisy, because brief blips cross thresholds. Burn rate expresses consumption relative to the window, so a fast window catches severe degradation that would exhaust a month's budget in days and pages immediately, while a slower window catches gradual erosion and raises a ticket. This two-tier structure is what allows meaningful alerting without the fatigue that threshold alarms produce.

**Steep cost, diminishing perceptibility.** Each additional nine costs roughly an order of magnitude more engineering effort, while the benefit users can detect diminishes — beyond a certain point the user's own network is less reliable than the service, making further improvement invisible to the people it was intended for. This asymmetry is the case for setting targets from user tolerance rather than from ambition, and for accepting that different functionality warrants different objectives: a critical revenue journey and a background feature should not share a target.

**Enforcement is what separates a control from a report.** The budget works because exhausting it changes behaviour — conventionally pausing feature work in favour of reliability until the balance recovers. If the answer to exhaustion is a discussion that ends with shipping anyway, the mechanism has no force and the original argument returns with additional overhead. The policy must therefore be agreed while nobody is under pressure and explicitly backed by whoever can pause feature delivery, because the decision made during an incident is always the expedient one. Of everything in a service level programme, this commitment and the quality of the indicator are the two things that determine whether it functions at all.

**Prove it — interview questions**

1. **[Basic] What is an error budget?**

   <details><summary>Model answer</summary>

   The permitted amount of unreliability implied by an objective. If the target is 99.9 per cent over thirty days, then 0.1 per cent — about forty-three minutes — is budgeted failure. The reframing matters: spending it is not a failure but using the allowance, and the remaining balance gives a factual answer to whether the team should be shipping features or working on reliability. That replaces a recurring argument about temperament with a number both sides agreed to beforehand.

   </details>

2. **[Basic] Why not target 100 per cent reliability?**

   <details><summary>Model answer</summary>

   Because it is unachievable and would be the wrong goal even if it were not. Any change risks failure, so perfect reliability means never changing anything, and dependencies fail regardless. More practically, each additional nine costs roughly ten times the effort while delivering less perceptible benefit, and past a point the user's own network is less reliable than the service — so further investment improves nothing anyone can detect. Targeting below perfection is also what creates the budget in the first place.

   </details>

3. **[Senior] What makes a good service level indicator?**

   <details><summary>Model answer</summary>

   That it measures what users experience. A server-side success rate can read 99.99 per cent while users are unable to work, because it misses client timeouts, DNS failures, partial page failures and responses that arrived after the user gave up. A good indicator is measured as close to the user as possible, expressed as a ratio of good events to total events, and framed around a critical journey — whether a customer could complete checkout — rather than around whether a particular endpoint returned a success status. It should also treat latency as part of success, since a slow response the user abandoned was not a success.

   </details>

4. **[Senior] Why alert on burn rate rather than on the objective itself?**

   <details><summary>Model answer</summary>

   Because the objective is evaluated over a long window, so alerting on it is either too late or too noisy. Burn rate expresses how fast the budget is being consumed relative to the window: a rate of one exhausts it exactly at the end, while a rate of fourteen exhausts a thirty-day budget in two days. That lets you configure a fast window that pages on severe sudden degradation and a slower window that raises a ticket on gradual erosion — catching both kinds of problem at appropriate urgency, which threshold alerting on the raw number does badly in both directions.

   </details>

5. **[Staff] How would you set an SLO for a checkout service?**

   <details><summary>Model answer</summary>

   I would start from the user journey rather than the endpoints: the indicator is the proportion of checkout attempts that complete successfully within a few seconds, measured client-side, because a server returning success after the customer abandoned the page did not succeed. Then I would bound the target by dependencies — if the payment provider commits to 99.95 per cent and checkout cannot complete without it, then promising 99.99 per cent is a fiction, and that arithmetic often ends the debate before it starts. Within that ceiling I would choose from commercial tolerance: seven hours of failed checkouts a month at 99 per cent is unacceptable, so 99.9 per cent giving forty-three minutes is the reasonable landing point. Alerting would be multi-window burn rate — a fast window that pages when the month's budget would be gone in two days, a slow window that raises a ticket. And critically, the policy would be agreed in advance and backed by management: ship freely above half the budget, low-risk changes only below a quarter, feature freeze on exhaustion. That last part is what makes it a control rather than a dashboard.

   </details>

6. **[Principal] Why do SLO programmes fail in practice?**

   <details><summary>Model answer</summary>

   Two reasons, and neither is technical. The first is indicator choice: teams instrument what is easy to collect rather than what users experience, so the objective tracks load-balancer status codes and reads beautifully during exactly the degradations that generate complaints. Once that gap is visible to anyone outside the team, trust in the entire programme collapses and it becomes a reporting ritual. The second is the absence of enforced policy. The budget only works because exhausting it changes behaviour, so if the response to exhaustion is a discussion that concludes with shipping anyway, then the original argument about velocity versus stability has simply returned with extra dashboards attached. Both failures are avoidable at the outset and nearly impossible to correct later, which is why I would spend the initial effort on two things above all: instrumenting a genuine user journey as close to the client as possible, and getting the exhaustion policy agreed and explicitly backed by whoever has the authority to pause feature work — before the first breach, when the answer can be reached calmly.

   </details>

---

### Circuit breakers

*Stop calling a dependency that is failing, so the caller fails fast instead of exhausting its own resources waiting on something that will not answer.*

**Flow:** `Closed state` → `Failure threshold` → `Open state` → `Half-open probe` → `Recovery`

> **The 30-second version**  
> Detect a failing dependency and reject calls to it immediately, protecting the caller's resources and the callee's recovery — with tight timeouts doing most of the work and a fallback providing the value.

**The problem**

A downstream service becomes slow — not failing, just taking thirty seconds instead of fifty milliseconds. Every caller thread waiting on it is blocked. Within a minute the caller's thread pool is exhausted, so requests that have nothing to do with that dependency also start failing. A localised problem has become a total outage.

The instinct to retry makes it worse. A struggling service receiving three times its normal request volume from clients retrying has less chance of recovering, so the outage extends rather than resolves.

> **Failing fast protects the caller and the callee simultaneously**  
> If a dependency is clearly failing, calling it again wastes the caller's resources and adds load the callee cannot handle. A circuit breaker detects this condition and rejects calls immediately without attempting them — preserving the caller's capacity for work it can actually complete, and giving the failing service the quiet it needs to recover.

**Mental model**

The breaker watches calls to a dependency. While they mostly succeed it stays closed and calls pass through. When failures exceed a threshold it opens and rejects immediately. After a cooldown it lets a probe through to test whether the dependency has recovered.

1. **Closed** — Normal operation; calls pass and outcomes are recorded.
2. **Threshold** — The failure condition that trips the breaker — a rate over a window, not a raw count.
3. **Open** — Calls are rejected instantly without being attempted.
4. **Half-open** — After a cooldown, a limited number of probes test whether recovery has occurred.
5. **Fallback** — What the caller does instead, which is what determines whether opening helps or merely relocates the failure.

> **A circuit breaker without a fallback just fails faster**  
> Opening the circuit converts a slow failure into an immediate one, which protects resources but does not help the user unless there is something to do instead — cached data, a default, a degraded response, or an honest error. Teams often add breakers and then discover the user experience is unchanged, because the interesting design work was never in the breaker but in deciding what happens when it is open.

**How it works**

**States and transitions**

```text
CLOSED  (normal)
  calls pass through; outcomes recorded in a window
  failure rate exceeds threshold -> OPEN

OPEN  (failing fast)
  every call rejected immediately - no attempt made
  no resources consumed, no load added downstream
  after the cooldown -> HALF-OPEN

HALF-OPEN  (testing)
  allow a small number of probe calls
  probes succeed -> CLOSED
  probes fail    -> OPEN, restart the cooldown

WHY HALF-OPEN EXISTS
  without it, recovery requires either staying open
  forever or reopening the floodgates at once
  -> a service that just recovered cannot absorb full
     traffic instantly
  -> probes test the water with a handful of requests

THE KEY PARAMETERS
  failure RATE, not count
    50% of requests failing over 20+ requests
    -> a count trips on low-traffic endpoints
       after a few unlucky calls
  minimum volume
    do not evaluate the rate until enough samples exist
    -> otherwise 1 failure out of 1 call = 100%
  cooldown
    long enough for recovery to be plausible
    -> typically 10-60 seconds
  timeout matters more than the breaker
    -> a 30-second timeout means each failing call
       occupies a thread for 30 seconds even with a
       breaker present
```

1. **Trip on failure rate over a minimum volume** — Raw counts trip on quiet endpoints after a couple of unlucky calls, producing spurious open circuits.
2. **Set aggressive timeouts first** — The breaker limits how many slow calls happen; the timeout limits how long each one occupies a thread, and the timeout matters more.
3. **Use one breaker per dependency, not per service** — A shared breaker means one failing dependency blocks calls to healthy ones.
4. **Design the fallback before the breaker** — The fallback is what determines whether opening helps users or only helps the caller's thread pool.
5. **Count timeouts as failures** — A call that times out is a failure whether or not it eventually returns; excluding timeouts defeats the purpose.
6. **Make breaker state observable and alertable** — An open breaker is a significant event, and a breaker that flaps indicates parameters that do not match reality.

**What actually goes wrong**

```text
CASCADE WITHOUT A BREAKER
  dependency slows to 30 s
  caller has 200 threads
  200 requests/s to that dependency
  -> in 1 second all 200 threads are blocked
  -> requests to OTHER dependencies also fail
  -> the caller is fully down because of one
     dependency

WITH A BREAKER (and a 1-second timeout)
  failures detected within seconds
  breaker opens
  subsequent calls rejected in microseconds
  -> threads stay free for other work
  -> the failing dependency stops receiving load
  -> callers of the healthy paths are unaffected

THE TIMEOUT DOES MOST OF THE WORK
  with a 30 s timeout, even a breaker lets 200 threads
  block for 30 s before it has enough signal
  with a 1 s timeout, the same detection happens with
  1/30th of the damage
  -> a breaker without tight timeouts is decoration

RETRY INTERACTION - GET THIS RIGHT
  retries INSIDE the breaker: 3 attempts per logical
    call -> the breaker sees 3x the failures, trips
    sooner, and you have tripled load on a struggling
    service
  retries OUTSIDE the breaker: the breaker rejects
    instantly, so retries are cheap and harmless
  -> place the breaker between the retry logic and
     the network
```

> **A breaker that flaps is worse than none at all**  
> If the threshold is too tight or the cooldown too short, the breaker opens on transient noise and closes again immediately, producing intermittent rejections that are harder to diagnose than a consistent failure. Symptoms appear and vanish, some requests succeed, and the cause is invisible unless breaker state is instrumented. Parameters should be tuned from observed failure patterns rather than chosen from defaults.

**Worked example**

A product page depending on a recommendations service, showing why the fallback is the real design.

**Configuration and behaviour**

```text
DEPENDENCY  recommendations service
CRITICALITY  enhancement, not essential

TIMEOUT  200 ms
  -> product pages must render fast regardless
  -> this alone bounds the damage

BREAKER
  failure rate threshold: 50%
  minimum volume: 20 requests in 10 s
    -> avoids tripping on a couple of unlucky calls
  cooldown: 30 s
  half-open probes: 3

FALLBACK  render the page without recommendations
  -> THIS is what makes the breaker valuable
  -> users see a slightly less useful page, not an error

INCIDENT WALKTHROUGH
  t+0    recommendations latency rises to 5 s
  t+0.2  calls begin timing out at 200 ms
  t+8    50% failure rate over 20+ requests
         -> breaker OPENS
  t+8+   product pages render instantly without
         recommendations
         recommendations service receives no traffic
  t+38   half-open: 3 probes
         still failing -> OPEN again
  t+95   probes succeed -> CLOSED
         recommendations reappear

USER IMPACT  slightly reduced page content for ~90 s
WITHOUT THE BREAKER  product pages timing out for
  every user for as long as the dependency was slow

PER-DEPENDENCY BREAKERS
  recommendations, reviews, inventory each have their
  own
  -> one failing does not block the others
```

| Metric | Value | Note |
|---|---|---|
| Timeout | 200 ms | bounds each call |
| Threshold | 50% of 20+ | no false trips |
| Fallback | render without | **user still served** |
| Impact | reduced content | not an error |

> **Criticality determines the fallback, and the fallback determines the value**  
> For an enhancement like recommendations, the fallback is to omit it and the user barely notices. For something essential like payment authorisation, there is no useful fallback and the breaker only converts a slow failure into a fast one. That difference should be established before adding a breaker, because it tells you whether you are designing graceful degradation or merely protecting your own thread pool — both are worthwhile, but they are different things.

**When to use it**

- **Calls to any remote dependency**, particularly ones that can become slow rather than simply fail.
- **Where a useful fallback exists**, so degradation is graceful rather than merely fast.
- **Protecting a struggling dependency** from retry amplification.
- **Preventing cascading failure**, where one slow dependency exhausts a caller's resources.
- **Non-critical enhancements**, which should never be able to take down the critical path.

**When to avoid it**

- **Do not use a breaker without tight timeouts**, since the timeout limits the damage each call causes.
- **Do not share one breaker across dependencies**, which lets one failure block healthy paths.
- **Do not trip on raw failure counts**, which produces false opens on low-traffic endpoints.
- **Do not place retries inside the breaker**, which multiplies observed failures and downstream load.
- **Do not add breakers to local in-process calls**, where the failure mode does not apply.

**Advantages**

- **Prevents resource exhaustion** in the caller when a dependency slows.
- **Reduces load on a failing service**, improving its chance of recovery.
- **Fails in microseconds** rather than after a timeout, preserving capacity.
- **Enables graceful degradation** when paired with a fallback.
- **Automatic recovery** through half-open probing, with no manual intervention.
- **Contains failures** so one dependency cannot take down unrelated functionality.

**Disadvantages**

- **Adds a failure mode of its own** — an incorrectly tuned breaker rejects healthy traffic.
- **Parameters need tuning** against real failure patterns, not defaults.
- **Flapping is confusing** and harder to diagnose than consistent failure.
- **Only helps with a fallback**, or it relocates the failure rather than removing it.
- **Per-instance state** means behaviour differs across a fleet unless coordinated.
- **Can mask persistent problems** by degrading quietly if the open state is not alerted.

**Trade-offs**

**Tuning trade-offs**

| Parameter | Too low or short | Too high or long | Reasonable start |
|---|---|---|---|
| Failure threshold | Trips on transient noise | Cascade before opening | 50% over a minimum volume |
| Minimum volume | False trips on quiet endpoints | Slow to detect | 20 requests in the window |
| Cooldown | Flapping; hammering a recovering service | Slow recovery after the fault clears | 10–60 s |
| Timeout | Healthy slow calls counted as failures | Threads blocked; breaker ineffective | Just above p99 |
| Half-open probes | Insufficient signal | Floods a recovering service | A handful |

Timeout deserves the most attention of these, because it bounds the damage of every individual call. A breaker with a thirty-second timeout still allows the thread pool to fill before it has enough signal to open, which is why tight timeouts are the prerequisite rather than the companion.

**How it fails**

**Circuit breaker failures**

| Failure | Cause | Fix |
|---|---|---|
| Cascade despite a breaker | Timeout too long; threads block before detection | Aggressive timeouts near p99 |
| Healthy traffic rejected | Threshold too tight or volume minimum too low | Rate-based threshold over sufficient volume |
| Intermittent unexplained errors | Breaker flapping | Longer cooldown; instrument breaker state |
| Failure relocated, not mitigated | No fallback behind the breaker | Design the degraded path first |
| Healthy dependencies unreachable | One breaker shared across dependencies | Per-dependency breakers |
| Struggling service hammered | Retries inside the breaker | Breaker between retry logic and the network |
| Problem hidden for hours | Open state not alerted | Alert when a breaker opens |

**Limits**

> **Starting parameters**
>
> - **Timeout**: just above the dependency's p99 under healthy conditions — the single most important setting.
> - **Threshold**: around 50% failure rate, evaluated only over a minimum volume such as 20 requests.
> - **Cooldown**: 10–60 s, long enough that recovery is plausible before probing.
> - **Half-open probes**: a small number, since a just-recovered service cannot absorb full traffic.
> - **Scope**: one breaker per dependency, and per instance unless state is deliberately shared.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Circuit breaker | Remote dependencies that can degrade | Tuning; needs a fallback |
| Timeouts alone | Simple protection | Every call still waits the full timeout |
| Bulkheads | Isolating resource pools | Complementary, not a substitute |
| Load shedding | Protecting yourself when overloaded | Different direction — inbound rather than outbound |
| Retry with backoff | Transient failures | Amplifies load on persistent failures |
| Service mesh breakers | Uniform policy without code changes | Less application-specific fallback logic |

Breakers, timeouts and bulkheads are complements rather than alternatives: timeouts bound each call, bulkheads bound how much of your resources any one dependency can consume, and breakers stop calling something that is clearly broken. Deployed together they cover the failure mode from three directions.

**In real systems**

- **Service meshes** provide circuit breaking as infrastructure policy, applying it uniformly without application changes — though the fallback still has to live in the application.
- **Non-critical enhancements behind breakers** is a standard pattern, ensuring recommendations, personalisation and similar features cannot take down a core page.
- **Rate-based thresholds with minimum volume** are the norm, since count-based tripping misfires badly on low-traffic paths.
- **Half-open probing** is universal, because reopening to full traffic tends to re-break a service that has just recovered.
- **Alerting on breaker state** is standard practice, since an open breaker means a dependency is down and silent degradation hides it.

**Common mistakes**

- **Adding a breaker without tightening timeouts**, so cascades happen anyway.
- **No fallback**, converting a slow failure into a fast one with no user benefit.
- **Count-based thresholds**, tripping falsely on low-traffic endpoints.
- **Retries inside the breaker**, amplifying load on a failing dependency.
- **One breaker for all dependencies**, blocking healthy calls.
- **Cooldowns too short**, causing flapping that is hard to diagnose.
- **No alerting on open state**, hiding a real outage behind quiet degradation.

**The staff-level view**

Circuit breakers are frequently added as a checkbox and tuned as an afterthought, which inverts where the value is.

- **Fix timeouts before adding breakers.** The timeout bounds the damage of each individual call, and a breaker behind a thirty-second timeout lets the thread pool fill before it has the signal to act.
- **Design the fallback first.** The breaker is trivial; deciding what the user gets when the dependency is unavailable is the actual work, and without it you have only protected your own thread pool.
- **Classify dependencies by criticality.** Enhancements should degrade invisibly; essential dependencies have no useful fallback, and knowing which is which determines what a breaker is even for.
- **Place the breaker between retry logic and the network.** Retries inside it multiply observed failures and triple load on a service that is already struggling.
- **Alert on open breakers.** Graceful degradation that nobody is told about is a dependency outage running silently, sometimes for hours.

**Go deeper**

A circuit breaker watches calls to a dependency and, when failures exceed a rate threshold, opens and rejects further calls without attempting them. After a cooldown it admits a few probes to test recovery. This prevents the classic cascade in which a slow dependency blocks every caller thread until requests unrelated to it also start failing, and it simultaneously relieves a struggling service of load it cannot handle.

Two things matter more than the breaker itself. Timeouts bound the damage of each individual call, and a breaker behind a thirty-second timeout allows the thread pool to fill before it has enough signal to open — so tight timeouts are the prerequisite. And the fallback determines whether opening helps anyone: without something to do instead, a breaker converts a slow failure into a fast one and the user sees an error either way.

Tuning should use failure rates evaluated over a minimum volume, since raw counts misfire badly on low-traffic paths. Breakers should be per dependency rather than shared, and placed between retry logic and the network so retries hit an open circuit cheaply instead of multiplying load. Finally, an open breaker means a dependency is down, so it warrants an alert — graceful degradation nobody is told about is a silent outage.

Circuit breakers address the failure mode in which calling a broken dependency harms the caller more than the failure itself does.

**Slowness is the dangerous case.** A dependency returning errors quickly is survivable: the caller learns immediately and moves on. A dependency responding in thirty seconds instead of fifty milliseconds holds a thread for each in-flight call, so a modest request rate exhausts the pool within seconds, and requests to entirely unrelated dependencies begin failing. The caller is then completely down because of one partially degraded downstream service — the cascade that this pattern exists to interrupt.

**Timeouts do most of the protecting.** The breaker cannot act until it has observed enough failures to be confident, and with a loose timeout the resource exhaustion occurs during that observation period. Setting timeouts just above the dependency's healthy p99 means each failing call costs a fraction of a second, the signal accumulates rapidly, and the breaker opens before capacity is consumed. This ordering — timeouts first, breaker second — is frequently reversed, which is why cascades occur in systems that have breakers configured.

**The fallback is where the value lives.** Opening the circuit converts a slow failure into an instant one, which preserves the caller's capacity but changes nothing for the user unless there is an alternative: cached data, a default, a page rendered without the missing component, or an honest immediate error. Deciding what that is requires classifying the dependency's criticality. An enhancement can be omitted and barely noticed; an essential dependency has no useful substitute, and there the breaker protects infrastructure rather than experience. Both are legitimate, but conflating them leads teams to expect graceful degradation from a mechanism that was only ever going to protect a thread pool.

**Parameters must be rate-based and volume-qualified.** Tripping on a raw count behaves inconsistently across traffic levels, opening spuriously on quiet endpoints where a handful of unlucky calls constitutes the entire sample. Evaluating a failure rate only once a minimum number of requests has been observed gives stable behaviour everywhere. Cooldowns need to be long enough that recovery is plausible, and half-open probing must be limited, because a service that has just recovered cannot absorb full traffic instantly and will be re-broken by an abrupt return.

**Composition errors are common and consequential.** A single breaker shared across several dependencies means one failure blocks calls to healthy services, converting isolation into coupling. Retries placed inside the breaker multiply the failures it observes and triple the load arriving at a service already struggling, so the breaker belongs between the retry logic and the network. And per-instance breaker state means behaviour varies across a fleet, which is usually acceptable but is worth knowing when diagnosing partial symptoms.

**Silent degradation is still an outage.** A breaker doing its job produces a system that appears healthy while a dependency is entirely unavailable, and without instrumentation and alerting on breaker state that condition can persist for hours. The same instrumentation catches the other characteristic failure — flapping, where a tight threshold or short cooldown causes the breaker to open and close repeatedly, producing intermittent errors that are considerably harder to diagnose than a consistent failure would have been.

**Prove it — interview questions**

1. **[Basic] What does a circuit breaker do?**

   <details><summary>Model answer</summary>

   It stops calling a dependency that is clearly failing. While calls mostly succeed it stays closed and traffic passes through; when the failure rate crosses a threshold it opens and rejects calls immediately without attempting them; after a cooldown it allows a few probes through to test recovery. The point is that calling a broken dependency wastes the caller's resources and adds load the callee cannot handle, so failing fast protects both sides.

   </details>

2. **[Basic] Why is a slow dependency more dangerous than a failing one?**

   <details><summary>Model answer</summary>

   Because a failing dependency returns errors quickly and the caller moves on, whereas a slow one holds a thread for the duration. With a thirty-second response time and a couple of hundred threads, a modest request rate exhausts the entire pool within seconds — and then requests that have nothing to do with that dependency start failing too. A localised problem becomes a complete outage, which is the cascade that breakers and tight timeouts exist to prevent.

   </details>

3. **[Senior] Why do timeouts matter more than the breaker itself?**

   <details><summary>Model answer</summary>

   Because the timeout bounds the damage of every individual call, and the breaker only acts once it has enough signal. With a thirty-second timeout, two hundred threads can all be blocked before the breaker has observed enough failures to open — so the cascade happens regardless. With a timeout just above the healthy p99, each failing call costs a fraction of a second, the failure signal accumulates quickly, and the breaker opens before resources are exhausted. A breaker behind a loose timeout is decoration.

   </details>

4. **[Senior] Why trip on failure rate rather than failure count?**

   <details><summary>Model answer</summary>

   Because a count has no denominator, so it behaves completely differently depending on traffic. On a high-traffic endpoint, five failures might be a rounding error; on a quiet one it might be every call made that minute — or simply five unlucky calls out of a thousand spread over an hour. Rate-based thresholds evaluated only once a minimum volume has been observed give consistent behaviour across traffic levels and avoid the classic failure of breakers opening on low-traffic paths for no real reason.

   </details>

5. **[Staff] How would you protect a product page that calls recommendations, reviews and inventory?**

   <details><summary>Model answer</summary>

   Separate breakers per dependency, because a shared one means a recommendations failure blocks inventory calls that are working perfectly. Then classify by criticality, because that determines the fallback and the fallback is the actual design: recommendations and reviews are enhancements, so the fallback is to render the page without them and the user barely notices; inventory may be essential, in which case there is no useful fallback and the breaker only protects the thread pool. Timeouts set just above each dependency's healthy p99 — for enhancements, aggressively tight, perhaps two hundred milliseconds, since a product page should not wait on a nice-to-have. Rate-based thresholds around fifty per cent evaluated over a minimum volume so quiet periods do not cause false trips, with cooldowns of tens of seconds and a handful of half-open probes so a recovering service is not immediately flooded. Retry logic sits outside the breaker so retries hit an open circuit cheaply rather than tripling load on something already struggling. And breaker state is instrumented and alerted, because a page silently rendering without reviews for six hours is an outage nobody has been told about.

   </details>

6. **[Principal] What do teams misunderstand about circuit breakers?**

   <details><summary>Model answer</summary>

   They treat the breaker as the solution when it is the cheapest part of the solution. Adding a breaker is a library call; deciding what the user receives when the dependency is unavailable is genuine product and architecture work, and skipping it means you have converted a slow failure into a fast one without improving anything the user sees — you protected your thread pool, which is worthwhile but is not what anyone thought they were buying. The second misunderstanding is that the breaker does the protecting, when in practice tight timeouts do most of it: the breaker cannot act until it has observed failures, and with a loose timeout the resource exhaustion happens during that observation window. The third is scope — one breaker across several dependencies, or retries placed inside it, both of which convert a protective mechanism into an amplifying one. What I would press for in review is the ordering: classify dependencies by criticality, decide the degraded experience for each, set timeouts from observed healthy latency, and only then add the breaker. Done in that order it is a genuinely powerful pattern; done in reverse it is a configuration file that makes people feel safer than they are.

   </details>

---

### Bulkheads and isolation

*Partition shared resources so that one workload's failure consumes only its own allocation and cannot sink the whole service.*

**Flow:** `Shared pool` → `Partitioned allocations` → `Failure containment` → `Independent capacity` → `Blast radius`

> **The 30-second version**  
> Divide shared resources by failure domain so one workload's exhaustion affects only its own allocation — paying in utilisation for a bounded blast radius, and remembering that the weakest shared component defines the real isolation.

**The problem**

A service handles requests for a hundred customers from one thread pool and one connection pool. One customer runs a query that takes thirty seconds. Their traffic occupies every thread, and the other ninety-nine customers see a service that is completely down — despite nothing being wrong with their requests.

The same pattern appears within a single application: a slow reporting endpoint consuming the pool that serves the login page, a batch import starving interactive traffic, a background job holding every database connection.

> **Shared resources transmit failure; partitioned resources contain it**  
> Any pool shared between independent workloads is a channel through which one workload's problem reaches all the others. Dividing the pool so each workload has its own allocation means a workload that exhausts its share affects only itself. The capacity given up to partitioning is the price of bounding the blast radius, and it is almost always worth paying.

**Mental model**

Named for a ship's watertight compartments: a hull breach floods one compartment rather than the vessel. Resources — threads, connections, memory, instances — are divided so that exhausting one partition does not exhaust the others.

1. **Shared resource** — The pool through which failure propagates: threads, connections, memory, CPU.
2. **Partition** — A dedicated allocation for a workload, tenant or dependency.
3. **Containment** — The property that exhausting one partition leaves the others functional.
4. **Blast radius** — How much of the system a single failure can affect — what partitioning bounds.
5. **Utilisation cost** — The capacity lost because a partition's idle resources cannot serve another's overflow.

> **The most damaging shared resource is usually the one nobody listed**  
> Teams partition thread pools and then discover that a shared database, a shared cache, a shared rate limiter or a shared deployment pipeline carried the failure anyway. Isolation is only as strong as the least-isolated shared component on the path, so the exercise that matters is enumerating everything two workloads have in common — including things that are not obviously resources, like a configuration store or a release process.

**How it works**

**Levels of isolation and what each contains**

```text
WEAKEST                                      STRONGEST

1  SEPARATE POOLS IN ONE PROCESS
   distinct thread and connection pools per
   dependency or workload
   contains: pool exhaustion
   does NOT contain: memory exhaustion, crashes,
     CPU starvation, a deploy that breaks the process

2  SEPARATE PROCESSES
   contains: crashes, memory exhaustion
   does NOT contain: host failure, noisy CPU neighbours

3  SEPARATE HOSTS OR NODES
   contains: host failure, resource contention
   does NOT contain: shared datastore, shared network
     path, shared control plane

4  SEPARATE CLUSTERS / CELLS
   contains: nearly everything, including bad deploys
     if releases are staggered per cell
   does NOT contain: a shared global dependency
     (DNS, auth, a global database)

5  SEPARATE REGIONS
   contains: regional infrastructure failure
   cost: replication, data residency, complexity

THE RULE
  isolation is only as strong as the least-isolated
  shared component
  -> separate thread pools sharing one database still
     fail together when the database is saturated
```

1. **Enumerate everything shared before partitioning anything** — The failure will travel through whatever was not on the list, and configuration stores and deploy pipelines are shared resources too.
2. **Partition by failure domain, not by convenience** — The right boundary is whatever set of things should fail together; dividing on any other axis gives cost without containment.
3. **Size partitions for their own peak, not the average** — A partition sized for average load fails during its own normal peak, which turns isolation into self-inflicted unavailability.
4. **Accept the utilisation cost explicitly** — Idle capacity in one partition cannot serve another's overflow; that is the mechanism working, not a defect.
5. **Isolate by tenant where a single tenant can be pathological** — Multi-tenant systems fail through one customer's unusual workload more often than through any other cause.
6. **Use cells for the strongest practical containment** — Independent stacks serving subsets of traffic, with staggered deploys, contain bad releases as well as resource failures.

**Partitioning strategies**

```text
BY DEPENDENCY
  one thread/connection pool per downstream service
  payment pool: 20 threads
  search pool:  50 threads
  email pool:   10 threads
  -> a slow search service exhausts 50 threads, not 200
  -> payments keep working

BY WORKLOAD CLASS
  interactive:  80% of capacity
  batch/report: 15%
  background:   5%
  -> a runaway report cannot starve the login page
  -> the classic source of "the site is down because
     someone ran an export"

BY TENANT
  per-customer quotas on threads, connections, rate
  -> one customer's pathological query affects only
     them
  -> essential in multi-tenant systems, where this is
     the single most common outage cause

BY CELL  (strongest common form)
  independent full stacks, each serving a subset of
  customers
  cell 1: customers A-F   cell 2: customers G-M  ...
  -> a failure affects one cell
  -> deploys roll cell by cell, so a bad release is
     contained too
  -> cost: operational multiplicity, cross-cell
     routing, capacity per cell

SIZING MATTERS
  4 partitions of 50 threads != 1 pool of 200
  each partition must handle ITS peak alone
  -> total capacity usually has to increase
```

> **Partitioning reduces utilisation, and that is the trade being made**  
> One pool of two hundred threads absorbs a spike in any workload; four pools of fifty cannot, even when three are idle. The lost efficiency is precisely what buys containment, so the question is not how to avoid it but how much to pay. Partitions sized too tightly cause failures that would not otherwise have happened, which is the characteristic way isolation is implemented badly.

**Worked example**

A multi-tenant analytics platform, where one customer's query can be arbitrarily expensive.

**Isolation design**

```text
THE RISK
  queries are customer-authored; some are pathological
  a single 10-minute scan can occupy a worker entirely

LAYER 1 - WORKLOAD CLASS
  interactive query pool:  70% of workers
  scheduled report pool:   20%
  data ingestion pool:     10%
  -> a flood of scheduled reports cannot block the
     interactive dashboard

LAYER 2 - PER-TENANT LIMITS
  max concurrent queries per tenant: 5
  max query duration: 60 s (then cancelled)
  -> one tenant occupies at most 5 workers
  -> with 100 workers, 20 tenants must misbehave
     simultaneously to exhaust the pool

LAYER 3 - CELLS
  tenants sharded across 4 independent cells
  each with its own workers, cache and database
  -> a cell failure affects 25% of tenants
  -> deploys roll cell by cell: a bad release is
     caught at 25% blast radius

WHAT IS STILL SHARED  (the honest part)
  authentication service      - global
  DNS and the routing layer   - global
  the control plane           - global
  -> these are the remaining single points, and
     naming them is the point of the exercise

COST
  ~30% more capacity than a single undivided pool
  4x the operational surface
  cross-cell tenant migration is a real project
```

| Metric | Value | Note |
|---|---|---|
| Per tenant | 5 concurrent | bounded |
| Classes | 70/20/10 | interactive protected |
| Cells | 4 | **25% blast radius** |
| Cost | ~30% capacity | the price of containment |

> **Cells contain bad deployments, which pools cannot**  
> Thread pools isolate resource exhaustion but share a code path — a release with a serious defect breaks every partition simultaneously. Cells with staggered deployment contain that too, because a bad release reaches one cell and stops. Since bad deploys cause a large share of serious incidents, this is often the strongest argument for cellular architecture, and it is one that resource-level partitioning cannot address at all.

**When to use it**

- **Multi-tenant systems**, where one customer's workload can be pathological.
- **Mixed workload classes**, so interactive traffic is protected from batch work.
- **Multiple downstream dependencies**, giving each its own pool so one cannot exhaust the caller.
- **High-consequence services**, where bounding the blast radius justifies the capacity cost.
- **Staged deployment**, where cells let a bad release affect a fraction of traffic.

**When to avoid it**

- **Do not partition when workloads are homogeneous and trusted**, where the cost buys nothing.
- **Do not size partitions below their own peak**, which manufactures failures that pooling would have absorbed.
- **Do not partition one layer and share the next**, which leaves the failure path intact.
- **Do not adopt cells for a small system**, where the operational multiplicity exceeds the benefit.
- **Do not partition on an axis unrelated to failure domains**, which costs utilisation without containing anything.

**Advantages**

- **Bounds the blast radius** of any single failure.
- **Protects critical workloads** from non-critical ones.
- **Contains pathological tenants** in multi-tenant systems.
- **Cells contain bad deployments** as well as resource exhaustion.
- **Makes capacity attributable**, since each partition's usage is visible.
- **Enables differentiated guarantees** per workload class or customer tier.

**Disadvantages**

- **Lower utilisation**, since idle capacity in one partition cannot serve another.
- **More total capacity required**, as each partition must handle its own peak.
- **Operational multiplicity**, particularly with cells.
- **Only as strong as the least-isolated shared component** on the path.
- **Rebalancing is hard**, as moving a tenant between cells is a real migration.
- **More configuration surface**, and more ways to size something wrongly.

**Trade-offs**

**Isolation level trade-offs**

| Level | Contains | Cost | Appropriate for |
|---|---|---|---|
| Separate pools | Pool exhaustion | Minimal | Per-dependency isolation |
| Separate processes | Crashes, memory | Low | Plugin or untrusted code |
| Separate hosts | Host failure, contention | Moderate | Noisy-neighbour problems |
| Cells | Most failures, bad deploys | High | Large multi-tenant systems |
| Regions | Regional outages | Very high | Global availability requirements |

Most systems benefit enormously from the cheapest level — separate pools per dependency and workload class — and only large multi-tenant platforms genuinely need cells. Reaching for cellular architecture before exhausting pool-level isolation is a common way to spend operational budget without a proportionate reduction in incidents.

**How it fails**

**Isolation failure modes**

| Failure | Cause | Fix |
|---|---|---|
| One tenant takes down everyone | Shared unpartitioned resource pool | Per-tenant concurrency and rate limits |
| Interactive traffic starved by batch | Single pool for all workload classes | Separate pools per class with reserved capacity |
| Partitions isolated but service still fails | Shared database, cache or control plane | Enumerate all shared components; isolate the real path |
| Failures that pooling would have absorbed | Partitions sized for average, not peak | Size each partition for its own peak |
| Bad deploy breaks everything | Isolation at resource level only | Cells with staggered deployment |
| Capacity wasted | Over-partitioned with many idle allocations | Fewer, larger partitions on real failure boundaries |
| Hot cell | Tenant distribution uneven | Rebalance; size cells for their actual load |

**Limits**

> **Sizing guidance**
>
> - **Total capacity** typically rises 20–40% relative to a single pool, because each partition must cover its own peak.
> - **Partition count**: fewer and larger is usually better; every additional boundary costs utilisation.
> - **Per-tenant limits** should allow normal use comfortably while bounding pathological use.
> - **Cells**: four to a few dozen is common; blast radius is one over the cell count.
> - **Weakest link**: any shared component on the request path defines the real isolation level, regardless of how many pools exist.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Resource partitioning | Containing pool exhaustion | Lower utilisation |
| Cells | Containing most failures including deploys | Operational multiplicity |
| Rate limiting per tenant | Bounding tenant impact cheaply | Does not contain crashes |
| Circuit breakers | Containing dependency failures | Complementary, not a substitute |
| Dedicated instances per customer | Strongest tenant isolation | Very expensive; poor utilisation |
| Shared everything | Small, trusted, homogeneous workloads | Any failure is total |

Rate limiting per tenant is the cheapest meaningful step and often the highest-return one in a multi-tenant system: it bounds how much of any shared resource a single customer can consume without requiring the resource to be physically partitioned at all.

**In real systems**

- **Cellular architectures** are standard in large multi-tenant platforms, sized so one cell's failure affects a known fraction of customers.
- **Per-dependency connection and thread pools** are basic practice, preventing one slow downstream service from exhausting a caller entirely.
- **Per-tenant concurrency limits** are the most common defence against a single customer's pathological workload.
- **Staggered deployment across cells** is what converts cellular isolation from resource containment into release containment.
- **Shared control planes** are the usual residual single point of failure, and naming them explicitly is part of any honest isolation review.

**Common mistakes**

- **Partitioning pools while sharing the database**, leaving the failure path intact.
- **Sizing partitions for average load**, causing failures during normal peaks.
- **No per-tenant limits** in a multi-tenant system, the most common outage cause there.
- **Over-partitioning**, wasting capacity on boundaries that contain nothing.
- **Relying on resource isolation to contain bad deploys**, which it cannot.
- **Partitioning on a convenient axis** rather than a failure domain.
- **Ignoring the shared control plane** and believing isolation is complete.

**The staff-level view**

Isolation work is frequently done at the visible layer and undone by something shared further down.

- **Enumerate shared components before partitioning.** The failure travels through whatever was not on the list, and the list must include configuration stores, deploy pipelines and control planes, not just pools.
- **Size every partition for its own peak.** Under-sizing converts an isolation mechanism into a source of failures that a shared pool would have absorbed, which discredits the whole approach.
- **Start with per-tenant limits and per-dependency pools.** They are cheap, they address the most common causes of multi-tenant outages, and they should be exhausted before considering cells.
- **Justify cells by deployment containment.** Their distinguishing benefit over pools is bounding a bad release, and since bad releases cause a large share of serious incidents, that is usually the real argument.
- **State the utilisation cost openly.** Partitioning trades efficiency for containment, and pretending otherwise leads to partitions sized too tightly to survive their own traffic.

**Go deeper**

Bulkheads partition shared resources — threads, connections, instances, whole stacks — so that a workload consuming its allocation entirely cannot consume anyone else's. Without partitioning, any shared pool is a channel through which one customer's slow query or one runaway batch job becomes a total outage for everyone, despite nothing being wrong with the other requests.

The cost is utilisation: a single large pool absorbs a spike anywhere, while partitions cannot lend capacity to one another, so total provisioning typically rises by twenty to forty per cent. That inefficiency is precisely what buys containment, and the characteristic implementation error is sizing partitions for average load rather than their own peak, which manufactures failures a shared pool would have absorbed.

Isolation is only as strong as the least-isolated shared component, so the valuable work is enumerating everything two workloads have in common — including databases, caches, control planes and deployment pipelines. Cells, meaning independent full stacks each serving a subset of customers, are the strongest practical form because staggered deploys contain bad releases as well as resource failures, which resource-level partitioning cannot do at all.

Bulkheading partitions resources so that failure is contained within a compartment rather than propagating through everything that shares a pool.

**Shared resources are failure channels.** Any finite pool that independent workloads draw from can be consumed entirely by one of them. A customer issuing thirty-second queries occupies threads for thirty seconds each, and at modest volume every thread is theirs, so every other customer receives nothing despite their requests being ordinary. The same shape appears within a single application when a reporting endpoint starves the login path, or a background import holds every database connection. Partitioning closes the channel by making each workload's exhaustion its own problem.

**Utilisation is the price, and it must be paid honestly.** One pool of two hundred absorbs a spike anywhere; four pools of fifty reject a spike in one while three sit idle. Total provisioning therefore rises, typically by twenty to forty per cent, because each partition must independently cover its own peak. This is not a defect to be optimised away — it is the mechanism functioning — and the most common way isolation is implemented badly is sizing partitions for average load, which produces failures during normal traffic that a shared pool would never have exhibited, and then discredits the approach.

**The weakest shared component defines the real containment.** Perfectly partitioned thread pools mean nothing when the workloads share a saturated database, a common cache, a single rate limiter or one authentication service. The diagram shows isolation; the failure path does not care. The exercise with actual value is enumerating everything two workloads have in common, deliberately including things not usually classed as resources — configuration stores, DNS, control planes, deployment pipelines — because that enumeration, not the pool configuration, describes what is genuinely contained.

**Partition on failure domains, not on convenience.** The correct boundary is the set of things that should be permitted to fail together. Dividing along any other axis spends utilisation without containing anything, and over-partitioning multiplies that waste while adding configuration surface and more opportunities to size something wrongly. Fewer, larger partitions placed on real boundaries almost always outperform many small ones.

**Cells add the one thing pools cannot provide.** Resource partitions share a code path, so a release containing a serious defect breaks every partition simultaneously — and since bad deployments account for a large fraction of severe incidents, that gap matters. Independent cells, each a full stack serving a subset of customers with deploys rolled cell by cell, bound a bad release to one cell's worth of traffic. This deployment containment is usually the strongest argument for cellular architecture, and it comes alongside real costs: operational multiplicity, cross-cell routing, uneven load requiring rebalancing, and tenant migration becoming a genuine project.

**Start cheap and escalate only when justified.** Per-tenant concurrency and rate limits, plus separate pools per dependency and workload class, address the most common multi-tenant outages at very low cost and should be exhausted before considering cells. Escalating to cellular architecture before doing so spends substantial operational budget without a proportionate reduction in incidents — and a team that struggles to operate one stack reliably will not be helped by operating eight of them.

**Prove it — interview questions**

1. **[Basic] What is a bulkhead in software?**

   <details><summary>Model answer</summary>

   A partition of a shared resource so that one workload's failure cannot consume everything. The name comes from ship compartments: a breach floods one section rather than sinking the vessel. In practice it means separate thread pools, connection pools, or entire stacks per dependency, workload class or tenant — so that a workload which exhausts its allocation affects only itself, and everything else keeps running.

   </details>

2. **[Basic] Why does sharing a pool transmit failure?**

   <details><summary>Model answer</summary>

   Because the pool is a finite resource that any workload can consume entirely. If one customer issues queries that take thirty seconds, their requests occupy threads for thirty seconds each, and at modest volume every thread is theirs — so every other customer gets nothing, despite their requests being perfectly ordinary. The shared pool is the channel through which one workload's problem becomes everyone's, and partitioning it closes that channel at the cost of some utilisation.

   </details>

3. **[Senior] What is the utilisation cost of partitioning?**

   <details><summary>Model answer</summary>

   One pool of two hundred threads can absorb a spike in any workload because all of it is available to whichever workload needs it. Four pools of fifty cannot: a spike in one is rejected while three sit idle. So total capacity typically has to rise by twenty to forty per cent, because each partition must independently handle its own peak rather than borrowing from the others. That lost efficiency is exactly what buys containment — it is the mechanism working, not a defect — and the common implementation error is sizing partitions too tightly and thereby causing failures that a shared pool would have absorbed.

   </details>

4. **[Senior] Why is isolation only as strong as the least-isolated component?**

   <details><summary>Model answer</summary>

   Because failure travels along whatever is still shared. A service can have perfectly partitioned thread pools per tenant and still fail entirely when the shared database saturates, or when a shared cache is evicted, or when a shared rate limiter throttles. The partitioning did real work at one layer and none at the next. This is why the valuable part of the exercise is enumerating everything two workloads have in common — including things not usually thought of as resources, such as a configuration store, a DNS zone, an authentication service or a deployment pipeline — because that list defines the actual containment, not the pool diagram.

   </details>

5. **[Staff] Design isolation for a multi-tenant analytics platform where customers write their own queries.**

   <details><summary>Model answer</summary>

   The defining risk is that query cost is customer-controlled and effectively unbounded, so I would work in three layers. First, per-tenant limits: maximum concurrent queries and a hard query duration after which the query is cancelled. This is the cheapest and highest-return control, because it means a single pathological customer occupies a small, known share of workers rather than all of them. Second, workload class pools: interactive queries, scheduled reports and ingestion get separate allocations, so a flood of scheduled reports cannot starve someone's dashboard — that specific failure, an export taking down the interactive path, is one of the most common in this kind of system. Third, cells: tenants sharded across several independent stacks, each with its own workers, cache and database, which bounds a cell failure to a known fraction of customers and, crucially, lets deploys roll cell by cell so a bad release is caught at that same fraction. I would also state plainly what remains shared — authentication, DNS, the control plane — because those are the residual single points, and an isolation design that does not name them is describing containment it does not have. The cost is roughly thirty per cent more capacity and several times the operational surface, which is the honest price.

   </details>

6. **[Principal] When is cellular architecture worth its cost, and when is it premature?**

   <details><summary>Model answer</summary>

   The distinguishing benefit of cells over cheaper partitioning is that they contain bad deployments. Thread pools isolate resource exhaustion but share a code path, so a release with a serious defect breaks every partition at once — and bad releases cause a large share of serious incidents, so containing them is often the real justification rather than any resource argument. That makes cells worth it when the blast radius of a bad deploy is commercially unacceptable and staggered rollout across independent stacks is the only way to bound it, which in practice means large multi-tenant platforms with many customers and high consequences. It is premature when cheaper isolation has not been exhausted: per-tenant limits and per-dependency pools address the most common multi-tenant outages at almost no cost, and reaching past them to cells spends substantial operational budget without a proportionate reduction in incidents. The other caution is that cells create their own work — cross-cell routing, tenant rebalancing as load becomes uneven, and running many copies of everything — so a team that cannot comfortably operate one stack will not be helped by operating eight.

   </details>

---

### Load shedding and admission control

*Reject excess work at the edge so the work you accept completes, rather than accepting everything and failing at all of it.*

**Flow:** `Incoming load` → `Capacity estimate` → `Admission decision` → `Shed low priority` → `Stable throughput`

> **The 30-second version**  
> Refuse excess work at the entrance, by priority and before it consumes resources, so the work you accept completes within its deadline instead of everything failing together.

**The problem**

Traffic doubles unexpectedly. The service accepts every request, queues grow, latency rises, clients time out and retry, which adds more load. Eventually the system is spending all its capacity on requests whose callers have already given up, and goodput — useful work completed — falls to near zero while the machines are at full utilisation.

This is the characteristic overload collapse, and its defining feature is that it is worse than simply being slow. A system at capacity that accepts everything does not degrade gracefully; it falls off a cliff and cannot climb back without shedding load, because the retry traffic sustains the overload.

> **Accepting work you cannot complete is worse than refusing it**  
> Every request admitted beyond capacity consumes resources and produces nothing, while also delaying the requests that could have succeeded. Rejecting quickly returns those resources to useful work and gives the caller an immediate answer they can act on. The counter-intuitive result is that a system which refuses ten per cent of requests can serve far more successful requests than one which accepts them all.

**Mental model**

An admission controller sits at the entrance and decides, per request, whether the system can complete it. Under normal load everything is admitted; under overload, requests are refused by priority so that the most valuable work still completes.

1. **Capacity signal** — Something that indicates saturation — queue depth, latency, concurrency, utilisation.
2. **Admission decision** — Accept or reject, made before the work begins rather than after it has consumed resources.
3. **Priority** — Which requests to shed first, which is what makes shedding a product decision rather than a random one.
4. **Fast rejection** — Refusal must be cheap, or shedding consumes the capacity it was protecting.
5. **Client behaviour** — How callers respond to rejection determines whether shedding stabilises or amplifies.

> **Retries turn shedding into amplification unless clients cooperate**  
> A client that immediately retries a rejected request has not reduced load; it has converted one request into several. Under overload, aggressive retrying multiplies the traffic precisely when the system can least afford it. Effective shedding requires clients to back off exponentially with jitter, and ideally to be told how long to wait — otherwise the mechanism protects nothing.

**How it works**

**Choosing the saturation signal**

```text
QUEUE DEPTH  (good)
  if the queue exceeds N, shed
  + directly reflects work waiting
  + responds quickly
  - needs a queue to observe

LATENCY  (good, and user-relevant)
  if p99 exceeds the SLO threshold, shed
  + measures what users experience
  - a lagging indicator: by the time latency rises,
    the queue is already deep

CONCURRENCY LIMIT  (best general answer)
  cap in-flight requests at N
  + directly bounds resource use
  + no tuning of thresholds, only of N
  + adaptive variants adjust N from observed latency

CPU UTILISATION  (poor alone)
  - a service can be saturated on locks, connections
    or downstream capacity at 40% CPU
  - and healthy at 90% CPU

QUEUE TIME  (the sharpest signal)
  how long a request waited before being picked up
  -> if it waited longer than the client's timeout,
     the work is ALREADY WASTED
  -> drop it without processing: this is pure gain

RULE
  prefer signals that reflect work waiting, not
  resources consumed
```

1. **Reject before doing work, not after** — A request rejected after parsing, authenticating and querying has already consumed most of what it would have cost.
2. **Make rejection cheap** — If refusing costs meaningfully less than serving, shedding buys capacity; if not, it does nothing.
3. **Shed by priority** — Dropping health checks and background refreshes before checkout requests turns a technical mechanism into a product-aware one.
4. **Drop requests whose deadline has passed** — Work that expired while queued is guaranteed waste, so discarding it is free capacity.
5. **Tell the client how long to wait** — A retry-after signal converts an unpredictable retry storm into staggered, survivable load.
6. **Test shedding under real overload** — Untested shedding frequently fails in the direction of not engaging, or of shedding everything.

**Why shedding increases goodput**

```text
SERVICE CAPACITY  1,000 requests/s
INCOMING          2,000 requests/s
CLIENT TIMEOUT    1 s

WITHOUT SHEDDING
  all 2,000 accepted
  queue grows without bound
  after a few seconds, queue time > 1 s
  -> every request completes AFTER its client gave up
  -> goodput: ~0
  -> CPU: 100%
  -> and clients retry, adding more load
  -> the system cannot recover without intervention

WITH SHEDDING AT A CONCURRENCY LIMIT
  admit up to the concurrency the service can sustain
  reject the remainder immediately
  -> ~1,000 requests/s complete within the timeout
  -> goodput: 1,000/s
  -> rejected clients get an immediate, actionable
     answer

THE COUNTER-INTUITIVE RESULT
  refusing half the traffic serves INFINITELY more
  successful requests than accepting all of it

PRIORITY SHEDDING REFINES IT FURTHER
  if 200 req/s are checkout and 1,800 are background
  refresh, shedding by priority means 100% of checkouts
  succeed
  -> same capacity, far better outcome
```

> **Shedding must not be the first thing to break**  
> The admission control path has to remain functional precisely when everything else is saturated, so it must not depend on a database lookup, a remote policy service, or anything else under load. Shedding logic that requires a healthy system to decide what to shed fails exactly when it is needed, and this is a surprisingly common design error.

**Worked example**

An API gateway protecting a service during a traffic surge.

**Admission control design**

```text
SIGNAL  adaptive concurrency limit
  start at 200 in-flight requests
  raise while latency stays below target
  lower when latency rises
  -> the limit tracks real capacity as it changes with
     deploys, dependency health and query mix

PRIORITY CLASSES
  P0  payment, checkout            never shed
  P1  interactive reads            shed at 90% of limit
  P2  background sync, prefetch    shed at 70%
  P3  analytics, non-urgent        shed at 50%
  -> classification comes from the product, not from
     the request path

DEADLINE AWARENESS
  each request carries a deadline
  if queue time already exceeds it, drop WITHOUT
    processing
  -> no capacity spent on work nobody will read

REJECTION
  HTTP 429 with Retry-After
  cost: microseconds, no downstream calls
  -> rejection must be far cheaper than service

CLIENT CONTRACT
  exponential backoff with jitter
  honour Retry-After
  -> without this, shedding converts into amplification

SURGE BEHAVIOUR
  t+0   traffic 3x normal
  t+1   concurrency limit reached
        P3 then P2 shed
  t+5   P1 partially shed
        P0 unaffected throughout
  t+300 traffic normalises; limit rises again
  RESULT: all checkouts succeeded; the system never
    entered collapse
```

| Metric | Value | Note |
|---|---|---|
| Signal | adaptive concurrency | tracks real capacity |
| P0 | never shed | **checkout protected** |
| Expired | dropped unprocessed | free capacity |
| Client | backoff + Retry-After | no amplification |

> **Priority makes shedding a product decision rather than a technical one**  
> Shedding uniformly means a random ten per cent of checkouts fail alongside a random ten per cent of analytics calls. Shedding by priority means every checkout succeeds and analytics waits. The capacity is identical; the business outcome is entirely different. That classification cannot be derived from the request path — it has to come from someone who knows what the system is for.

**When to use it**

- **Any service that can receive more traffic than it can handle**, which is nearly all of them.
- **Protecting critical paths** by shedding lower-priority work first.
- **Guarding against retry storms** and traffic surges.
- **Multi-tenant systems**, combined with per-tenant limits.
- **Anywhere queue growth leads to collapse**, which is the default without admission control.

**When to avoid it**

- **Do not shed without priorities** where request value varies, since uniform shedding drops valuable work at random.
- **Do not shed when queueing would suffice**, such as short bursts within a bounded queue.
- **Do not use CPU alone as the signal**, since saturation frequently occurs elsewhere.
- **Do not shed without client backoff**, which converts protection into amplification.
- **Do not make admission control depend on loaded components**, or it fails when needed.

**Advantages**

- **Maintains goodput under overload** instead of collapsing to near zero.
- **Fast, actionable failure** for rejected clients rather than a timeout.
- **Protects critical work** when shedding is priority-aware.
- **Prevents queue-growth collapse**, from which recovery is otherwise difficult.
- **Bounds latency** for admitted requests.
- **Recovers automatically** as load subsides.

**Disadvantages**

- **Some users are refused service**, which is a real cost even when it is the right trade.
- **Requires a good saturation signal**, and poor ones shed too early or too late.
- **Priority classification is product work**, not derivable from the code.
- **Client cooperation is required**, and clients you do not control may not back off.
- **Tuning is ongoing**, since capacity changes with deploys and dependencies.
- **Rarely exercised**, so it is often broken when first needed.

**Trade-offs**

**Signal choice trade-offs**

| Signal | Responsiveness | Accuracy | Notes |
|---|---|---|---|
| Queue depth | Fast | Good | Needs an observable queue |
| Queue time vs deadline | Fast | Excellent | Expired work is guaranteed waste |
| Latency threshold | Lagging | User-relevant | Queue is already deep by then |
| Concurrency limit | Immediate | Good | Adaptive variants self-tune |
| CPU utilisation | Fast | Poor | Saturation often lies elsewhere |

Adaptive concurrency limiting has become the default recommendation because it self-tunes: the limit rises while latency is healthy and falls when it is not, so it tracks actual capacity as that changes with deployments, dependency health and workload mix — none of which a static threshold accommodates.

**How it fails**

**Load shedding failures**

| Failure | Cause | Fix |
|---|---|---|
| Goodput collapses under surge | No admission control; unbounded queueing | Concurrency limit with fast rejection |
| Critical requests dropped | Uniform shedding | Priority classes with protected tiers |
| Rejection made things worse | Clients retry immediately | Backoff with jitter; Retry-After |
| Shedding never engages | Signal is CPU, but saturation is elsewhere | Queue depth or concurrency signal |
| Everything shed at once | Threshold too tight; no hysteresis | Graduated thresholds per priority class |
| Admission control itself fails | Depends on a loaded database or remote policy | Keep the decision local and cheap |
| Capacity wasted on dead requests | No deadline awareness | Drop requests whose deadline has passed |

**Limits**

> **Practical guidance**
>
> - **Rejection cost** must be orders of magnitude below service cost, or shedding consumes what it protects.
> - **Concurrency limit** should be adaptive, since fixed limits go stale with every deployment.
> - **Priority tiers**: a handful is enough; more becomes unmanageable to reason about.
> - **Graduated thresholds**: shed low priority well before the limit, high priority only at it.
> - **Client backoff** is a precondition — without it, shedding amplifies rather than protects.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Load shedding | Protecting goodput under overload | Some users refused |
| Rate limiting | Bounding per-client usage | Not responsive to actual capacity |
| Queueing with bounds | Smoothing short bursts | Collapses if the burst persists |
| Autoscaling | Sustained load growth | Too slow for sudden surges |
| Backpressure to upstream | Internal pipelines | Needs a cooperative producer |
| Over-provisioning | Predictable peaks | Expensive; still finite |

Autoscaling and shedding solve different timescales and both are needed: scaling responds to sustained load over minutes, while shedding responds to a surge in seconds. A system relying on autoscaling alone will collapse during the interval before capacity arrives.

**In real systems**

- **Adaptive concurrency limiting** is widely used because it self-tunes to capacity that changes with deploys and dependency health.
- **Deadline propagation with early dropping** discards requests whose callers have already timed out, which is pure capacity recovery.
- **Priority-aware shedding** protects revenue-critical paths while lower-value traffic absorbs the reduction.
- **Retry-After with client backoff** is the standard contract that stops rejection from becoming amplification.
- **Load shedding at the edge or gateway** keeps the rejection path cheap and independent of the services under load.

**Common mistakes**

- **No admission control at all**, allowing queue growth to collapse goodput.
- **Uniform shedding**, dropping checkout requests alongside analytics.
- **CPU as the saturation signal**, missing saturation elsewhere.
- **Expensive rejection**, consuming the capacity shedding was meant to protect.
- **No client backoff**, turning rejection into amplification.
- **Admission control depending on loaded components**, failing when needed.
- **Never testing it**, so its first real execution is also its first test.

**The staff-level view**

Load shedding is the mechanism most likely to be absent from a design and most likely to be the difference between degradation and collapse.

- **Insist on priority classification.** Uniform shedding drops valuable work at random; classification is product knowledge and must come from someone who knows what the system is for.
- **Choose a signal that reflects waiting work.** CPU saturation is a poor proxy because services frequently saturate on locks, connections or downstream capacity while CPU looks fine.
- **Keep the admission path independent.** Shedding logic that consults a database or remote policy service fails exactly when everything is loaded, which is a recurring and avoidable design error.
- **Make the client contract explicit.** Without backoff and honouring retry hints, rejection multiplies load, so the protection depends on behaviour you must specify and, where possible, enforce.
- **Exercise it deliberately.** Shedding runs rarely, so it is usually broken the first time it matters; load testing to actual collapse is the only way to know it engages correctly.

**Go deeper**

Beyond capacity, accepting a request does not serve it: queues grow, work completes after callers have timed out, and the system runs at full utilisation producing nothing useful. Worse, retry traffic sustains the overload so it does not resolve on its own. Admission control rejects excess work immediately, which returns capacity to requests that can actually complete — a service refusing part of its traffic serves far more successful requests than one accepting all of it.

The signal should reflect waiting work rather than consumed resources: queue depth, in-flight concurrency, or queue time against the request deadline, which identifies work that is already guaranteed waste. CPU is a poor proxy, since saturation commonly occurs on locks, connections or downstream capacity. Adaptive concurrency limiting is the general default because it self-tunes as capacity changes with deploys and dependency health.

Two things determine whether shedding helps. Priority classification turns it from a technical mechanism into a product-aware one — analytics absorbing the reduction while checkout is untouched, rather than both failing at the same rate. And client backoff is a precondition, because a caller that retries immediately has multiplied load rather than reduced it. The admission path must also be local and cheap, since logic that consults a loaded database fails exactly when it is needed.

Load shedding accepts that capacity is finite and decides deliberately which work to refuse, rather than accepting everything and failing at all of it.

**Overload collapse is qualitatively different from slowness.** As queues grow past the point where queue time exceeds client timeouts, every completed request arrives after its caller has abandoned it — so the system is fully utilised and producing nothing. Timed-out clients retry, sustaining the overload, which means the state does not resolve as load subsides in the way a merely slow system would. This non-recovery property is what makes admission control a design requirement rather than an optimisation.

**The signal should measure waiting work.** Queue depth and in-flight concurrency directly indicate whether the system is keeping up. Latency is user-relevant but lagging — by the time it rises, the queue is already deep. CPU utilisation is actively misleading, because services commonly saturate on lock contention, connection pools or downstream capacity while CPU appears comfortable, and conversely run healthily at high utilisation. The sharpest available signal is queue time compared against the request's deadline: work that has already waited longer than its caller will tolerate is certain waste, so discarding it without processing is free capacity.

**Adaptive limits beat fixed thresholds.** Real capacity changes with every deployment, every dependency's health, and every shift in workload mix, so a threshold tuned once is wrong shortly afterwards. An adaptive concurrency limit that rises while latency stays within target and falls when it does not tracks actual capacity continuously, which removes most of the ongoing tuning burden and handles the cases a static number cannot anticipate.

**Priority is what makes shedding worth doing well.** Uniform shedding reduces load but drops work at random, so a ten per cent reduction fails ten per cent of checkouts alongside ten per cent of analytics requests. Graduated thresholds per priority class — shedding background and analytics traffic well before the limit, protecting revenue-critical paths entirely — deliver the same capacity relief with an entirely different business outcome. The classification cannot be inferred from the code; it is product knowledge about what the system exists to do, which makes this a conversation rather than an implementation detail.

**Rejection must be cheap and the client must cooperate.** If refusing a request costs a meaningful fraction of serving it — because it happens after parsing, authentication and a database lookup — then shedding consumes the capacity it was protecting. And rejection only reduces load if callers wait: a client retrying immediately has converted one request into several at precisely the wrong moment. Exponential backoff with jitter, plus a retry hint the server can use to stagger returning traffic, is the contract that makes the mechanism function, and where clients are outside your control it must be enforced through per-client limits rather than assumed.

**The admission path must survive the conditions it exists for.** Shedding logic that consults a database, a remote policy service or anything else under load fails exactly when everything is saturated — a recurring design error that produces a protection mechanism unavailable during the only scenario it addresses. Related, and equally common: because shedding executes rarely, it is frequently broken the first time it genuinely matters, either failing to engage or engaging so aggressively that it becomes the outage. Load testing to actual collapse is the only way to establish that it does what was intended, and it is the step most often skipped.

**Prove it — interview questions**

1. **[Basic] Why reject requests instead of trying to serve them all?**

   <details><summary>Model answer</summary>

   Because beyond capacity, accepting a request does not serve it — it consumes resources, produces nothing, and delays the requests that could have succeeded. As queues grow, work completes only after its caller has timed out, so the system runs at full utilisation producing essentially zero useful output. Rejecting quickly returns that capacity to requests that can complete within their deadlines, which is why a service that refuses part of its traffic can serve far more successful requests than one that accepts everything.

   </details>

2. **[Basic] What makes a good saturation signal?**

   <details><summary>Model answer</summary>

   Something that reflects work waiting rather than resources consumed. Queue depth and in-flight concurrency are direct measures of whether the system is keeping up. CPU utilisation is a poor proxy, because a service can be completely saturated on locks, connection pools or a slow downstream dependency while CPU sits at forty per cent, and equally can be perfectly healthy at ninety. The sharpest signal of all is queue time compared against the request's deadline: work that has already waited longer than its caller will tolerate is guaranteed waste and can be discarded for free.

   </details>

3. **[Senior] Why does shedding require client cooperation?**

   <details><summary>Model answer</summary>

   Because rejection only reduces load if the client does not immediately ask again. A caller that retries instantly has converted one request into several, so under overload aggressive retrying multiplies traffic at exactly the moment the system can least absorb it, and the shedding mechanism protects nothing. Effective shedding requires exponential backoff with jitter, and ideally a retry hint so the server can stagger returning clients. Where you do not control the clients, you need enforcement — per-client rate limits — rather than a contract.

   </details>

4. **[Senior] Why is priority classification essential?**

   <details><summary>Model answer</summary>

   Because without it, shedding drops valuable work at random. If a system sheds ten per cent uniformly, then ten per cent of checkouts fail alongside ten per cent of analytics calls — same capacity, needlessly bad outcome. With priority classes, background refreshes and analytics absorb the entire reduction while every checkout still succeeds. The important point is that this classification cannot be derived from the code or the request path; it is product knowledge about what the system is for, and it has to be supplied by someone who holds that knowledge.

   </details>

5. **[Staff] Design admission control for an API gateway facing unpredictable traffic surges.**

   <details><summary>Model answer</summary>

   An adaptive concurrency limit as the signal, because a fixed threshold goes stale with every deploy and every change in dependency health — the limit should rise while latency stays within target and fall when it does not, so it tracks real capacity rather than an assumption about it. Requests classified into a few priority tiers with graduated thresholds: analytics shed at half the limit, background sync at seventy per cent, interactive reads at ninety, and payment and checkout never shed. Deadline propagation so that any request whose queue time has already exceeded its caller's timeout is dropped without being processed, which is pure capacity recovery because that work was guaranteed to be discarded anyway. Rejection returns immediately with a retry hint, and must cost microseconds with no downstream calls — if refusing is not dramatically cheaper than serving, shedding consumes what it protects. The client contract is exponential backoff with jitter, honouring the retry hint, backed by per-client rate limits for callers I do not control. And critically, the admission decision must be local and cheap, never consulting a database or remote policy service, because that dependency will be loaded at precisely the moment the decision matters most.

   </details>

6. **[Principal] Why is load shedding so often missing, and what is the consequence?**

   <details><summary>Model answer</summary>

   It is missing because it only matters in a state most teams never deliberately reach. Everything works in testing, autoscaling handles growth, and the failure mode — accepting work you cannot complete — looks like success right up until it does not. Then the consequence is qualitatively different from ordinary slowness: queue growth means requests complete after their callers have abandoned them, so utilisation is at a hundred per cent while useful output is near zero, and retry traffic from timed-out clients sustains the overload so the system cannot climb back out without intervention. That non-recovery property is what makes it worth designing for in advance rather than adding afterwards. The two things I would insist on in review are priority classification, because uniform shedding wastes the mechanism's main benefit by dropping revenue-critical work at random, and deliberate testing to actual collapse, because shedding logic executes rarely and is therefore usually broken the first time it is genuinely needed — either failing to engage, or engaging so aggressively that it becomes the outage.

   </details>

---

### Timeouts, deadlines and cancellation

*Bound how long any operation may take, propagate the remaining budget through the call chain, and stop work whose result nobody will read.*

**Flow:** `Request deadline` → `Propagated budget` → `Per-hop timeout` → `Cancellation signal` → `Released resources`

> **The 30-second version**  
> Set a deadline from the user's tolerance, propagate the remaining budget down the chain, cancel work that outlives it, and treat every timeout as an unknown outcome rather than a failure.

**The problem**

A request arrives with a three-second client timeout. It calls service A, which calls B, which calls C. Each has a thirty-second timeout configured because thirty seconds was the framework default. The client gives up after three seconds, but the work continues through the chain for another twenty-seven, holding threads, connections and database locks for a result nobody will ever read.

Multiply that by a traffic surge and the system is spending most of its capacity on abandoned work. Meanwhile a service with no timeout at all can wait forever, which means one unresponsive dependency can permanently consume every thread in the caller.

> **A deadline is the request's property; a timeout is each hop's share of it**  
> The client's tolerance is a single number that belongs to the whole request. Configuring independent per-hop timeouts means their sum bears no relation to it, and the ones deeper in the chain outlive the client's patience. Propagating the remaining budget down the chain makes every hop aware of how much time is actually left, and allows work to stop the moment it becomes pointless.

**Mental model**

The client sets a deadline. Each service computes how much of it remains, spends part on its own work, and passes the remainder to its dependencies. When the deadline passes, everything downstream stops.

1. **Deadline** — An absolute point in time by which the result is useless — a property of the request.
2. **Budget** — The time remaining at any hop, computed from the deadline rather than configured.
3. **Per-hop timeout** — The bound on one call, derived from the budget and the work still to do.
4. **Cancellation** — The signal that propagates so downstream work actually stops rather than merely being ignored.
5. **Resource release** — What cancellation is for: returning threads, connections and locks to use.

> **Timeouts without cancellation only free the caller**  
> A caller that times out stops waiting, but unless a cancellation signal reaches the callee, the downstream work continues to completion — still holding its thread, still executing its database query, still consuming capacity for a result that will be discarded. Under load this is the difference between a system that recovers and one that remains saturated with abandoned work long after the traffic subsided.

**How it works**

**Deadline propagation versus independent timeouts**

```text
INDEPENDENT TIMEOUTS  (the common, wrong shape)
  client   3 s
  gateway  30 s  (framework default)
  service  30 s
  database 30 s
  -> client gives up at 3 s
  -> the chain works on for up to 27 more seconds
  -> holding threads, connections, locks
  -> for a result that will be thrown away

DEADLINE PROPAGATION  (correct)
  client sets deadline = now + 3 s, sends it
  gateway: 2,950 ms remain -> calls service with that
  service: spends 200 ms, 2,700 ms remain -> passes on
  database: given 2,700 ms; if it cannot finish,
    it is cancelled
  deadline reached anywhere -> everything stops

THE ARITHMETIC THAT MATTERS
  per-hop timeouts must SUM to less than the client's
  tolerance, allowing for network time and retries
  -> 3 s client budget with 3 sequential hops
     is NOT 3 s per hop
  -> and a retry doubles the hop it retries

CHECK ON ARRIVAL, NOT ONLY ON TIMEOUT
  a request that waited 3.2 s in a queue is already
  expired before any work begins
  -> reject it immediately
  -> this is the cheapest capacity recovery available
```

1. **Set a deadline at the entry point** — It is a property of the request and the user's patience, not of any individual service.
2. **Propagate the remaining budget on every call** — Each hop should know how much time is left, not just what its own configured limit is.
3. **Check the deadline before starting work** — A request that expired while queued costs nothing to reject and everything to process.
4. **Ensure cancellation actually stops work** — Cancelling a wrapper while the query continues frees nothing; the signal must reach the resource holder.
5. **Budget retries within the deadline** — A retry consumes budget, so the same deadline must cover both attempts or the retry is guaranteed to expire.
6. **Derive timeouts from observed latency** — A timeout set just above p99 catches genuine failures without failing healthy slow requests.

**Choosing the number**

```text
TOO SHORT
  healthy-but-slow requests fail
  retries amplify load
  the p99.9 case becomes an error rather than a delay
  -> a timeout below p99 creates failures that did not
     otherwise exist

TOO LONG
  threads held on hopeless calls
  cascading exhaustion
  failure detection delayed
  -> the classic default-30-seconds problem

REASONABLE STARTING POINT
  timeout ~= p99 latency + margin
  -> healthy requests almost never hit it
  -> genuine failures are detected in roughly p99 time

DIFFERENT TIMEOUTS FOR DIFFERENT THINGS
  connection establishment   short (hundreds of ms)
  request completion         from p99 of that operation
  idle socket                minutes
  total request deadline     the user's tolerance
  -> a single timeout value for all of these is wrong
     for most of them

BUDGET SPLITTING EXAMPLE  (3 s deadline)
  gateway overhead        50 ms
  auth call               200 ms
  primary service         2,000 ms
    of which DB query     1,500 ms
  response serialisation  50 ms
  margin                  700 ms
  -> the margin absorbs variance; without it, any
     slow hop breaks the whole chain
```

> **Retries multiply the time budget, not just the load**  
> A hop with a one-second timeout that retries twice can consume three seconds, so a chain whose timeouts sum neatly to the deadline blows past it as soon as anything retries. Retries must be budgeted inside the deadline rather than layered on top of it — which usually means the retry is only attempted if enough budget remains to have a realistic chance of completing.

**Worked example**

A checkout request with a three-second user tolerance, budgeted end to end.

**Budget allocation and expiry behaviour**

```text
DEADLINE  now + 3,000 ms, set at the gateway

ALLOCATION
  gateway routing and auth       250 ms
  checkout service               2,400 ms
    inventory check              400 ms
    payment authorisation        1,200 ms
    order write                  500 ms
    margin within the service    300 ms
  response                        50 ms
  global margin                  300 ms

EACH CALL RECEIVES THE REMAINING BUDGET
  not a fixed configured timeout
  -> if auth was slow and took 600 ms, the checkout
     service is told it has 2,050 ms, and it adapts
     rather than blindly using 2,400

EXPIRY BEHAVIOUR
  deadline passes during payment authorisation
  -> the call is cancelled
  -> the cancellation propagates to the provider
     client, which aborts the connection
  -> the order write never begins
  -> the user gets a clear failure, not a hang

IMPORTANT CORRECTNESS POINT
  payment authorisation may have SUCCEEDED remotely
  before cancellation arrived
  -> cancellation does not undo side effects
  -> so the operation needs an idempotency key and a
     reconciliation path
  -> timeouts create ambiguity, and ambiguity must be
     resolved deliberately

QUEUE ADMISSION
  any request whose deadline has already passed while
  queued is rejected without processing
```

| Metric | Value | Note |
|---|---|---|
| Deadline | 3,000 ms | user tolerance |
| Propagated | remaining, not fixed | adapts to reality |
| Margin | ~20% | absorbs variance |
| Ambiguity | idempotency key | **cancellation ≠ undo** |

> **A timeout leaves the outcome unknown, which is a correctness problem**  
> When a call times out, the operation may have succeeded, failed, or still be running. Cancellation stops waiting; it does not undo. Any operation with side effects therefore needs an idempotency key so a retry cannot double-charge, and a reconciliation path for the case where the caller believes it failed and the callee believes it succeeded. Treating a timeout as a failure is the common and dangerous simplification.

**When to use it**

- **Every remote call**, since an unbounded call can consume a thread indefinitely.
- **Multi-hop request chains**, where independent timeouts cannot sum correctly.
- **User-facing paths**, where the deadline is genuinely the user's patience.
- **Resource-constrained services**, where holding threads on abandoned work is expensive.
- **Anywhere retries occur**, since retries must fit inside the same budget.

**When to avoid it**

- **Do not set timeouts below p99**, which turns healthy slow requests into failures.
- **Do not use one timeout value for connection, request and idle**, which are different concerns.
- **Do not treat a timeout as a definite failure**, since the operation may have succeeded.
- **Do not retry without checking remaining budget**, which guarantees the retry expires.
- **Do not rely on timeouts alone** when cancellation does not actually stop the work.

**Advantages**

- **Bounds resource consumption** per request, preventing indefinite thread occupation.
- **Stops abandoned work**, recovering capacity that would otherwise be wasted.
- **Makes failure fast and predictable** rather than a hang.
- **Prevents cascading exhaustion** when a dependency slows.
- **Deadline propagation adapts** to how much time each hop has actually consumed.
- **Enables free capacity recovery** by rejecting already-expired queued work.

**Disadvantages**

- **Creates ambiguity** about whether the operation completed.
- **Requires idempotency** for any side-effecting operation.
- **Needs plumbing through every layer**, and any layer that drops the deadline breaks the chain.
- **Tuning requires latency data**, and wrong values cause failures in both directions.
- **Cancellation support varies** — some libraries and drivers ignore it entirely.
- **Clock alignment matters** when deadlines are absolute across machines.

**Trade-offs**

**Timeout strategy trade-offs**

| Approach | Behaviour | Cost | Best for |
|---|---|---|---|
| No timeout | Waits indefinitely | Thread exhaustion | Never |
| Fixed per-hop timeouts | Simple | Sum unrelated to user tolerance | Simple single-hop calls |
| Deadline propagation | Chain-aware; adapts | Plumbing through every layer | Multi-hop systems |
| Deadline plus cancellation | Work actually stops | Library support required | Resource-constrained services |
| Aggressive timeouts | Fast failure detection | Healthy slow requests fail | Non-critical enhancements |

The deciding factor is chain depth. A single call is served adequately by a fixed timeout set from observed latency; a chain of three or more services cannot be, because independent timeouts have no relationship to the client's tolerance and the deepest hops always outlive it.

**How it fails**

**Timeout and cancellation failures**

| Failure | Cause | Fix |
|---|---|---|
| Capacity spent on abandoned work | No cancellation propagation | Propagate cancellation to the resource holder |
| Thread pool exhausted by one dependency | No timeout, or default too long | Timeouts near p99; tight connection timeouts |
| Healthy requests failing | Timeout below p99 | Derive from observed latency with margin |
| Deep hops outlive the client | Independent per-hop timeouts | Deadline propagation |
| Duplicate side effects | Retry after an ambiguous timeout | Idempotency keys; reconciliation |
| Retry guaranteed to expire | Retry not budgeted within the deadline | Attempt only if sufficient budget remains |
| Cancellation ignored | Driver or library does not support it | Verify support; use query-level timeouts |

**Limits**

> **Practical values**
>
> - **Request timeout**: p99 of the operation plus margin — below p99 creates failures that did not exist.
> - **Connection timeout**: hundreds of milliseconds, much shorter than request timeouts.
> - **Margin**: roughly 20% of the deadline, absorbing variance across hops.
> - **Retries**: must fit inside the deadline; attempt only with enough budget remaining.
> - **Expired queue entries**: reject before processing — the cheapest capacity available.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Deadline propagation | Multi-hop chains | Plumbing through every layer |
| Fixed timeouts | Single calls | No chain awareness |
| Circuit breakers | Persistent dependency failure | Complementary, not a substitute |
| Async with callbacks | Long operations | Different programming model |
| Queue with deadline checks | Batch work | Requires deadline metadata |
| No bound | Nothing | Guaranteed eventual exhaustion |

Timeouts and circuit breakers address different halves of the same problem: the timeout bounds the cost of one slow call, while the breaker stops making calls once enough of them have failed. Neither substitutes for the other, and a breaker behind a loose timeout is largely ineffective.

**In real systems**

- **RPC frameworks with built-in deadline propagation** pass the remaining budget automatically, which is why they avoid the independent-timeout problem by default.
- **Context-based cancellation** in modern languages carries the deadline and cancellation signal through call chains as a first-class value.
- **Query-level timeouts in databases** matter because cancelling at the application layer does not necessarily stop a running query.
- **Idempotency keys on payment operations** exist precisely because a timeout leaves the outcome unknown and retries must be safe.
- **Rejecting expired queued work** is standard in high-throughput systems, since processing it is guaranteed waste.

**Common mistakes**

- **Framework default timeouts** far longer than the client's tolerance.
- **Independent per-hop timeouts** in multi-hop chains.
- **Timeouts without cancellation**, freeing the caller but not the callee.
- **Treating a timeout as definite failure**, causing duplicate side effects on retry.
- **Retrying without budget checks**, guaranteeing expiry.
- **One timeout value** for connection, request and idle.
- **Processing requests whose deadline already passed** while queued.

**The staff-level view**

Timeout configuration is usually inherited from defaults and is one of the most consequential things nobody owns.

- **Audit the timeout arithmetic across chains.** Independent per-hop values almost never sum to the client's tolerance, and the deepest hops routinely outlive the user's patience by an order of magnitude.
- **Verify that cancellation reaches the resource holder.** Cancelling an application-level wrapper while the database query continues frees nothing, and this gap is common enough to be worth checking directly.
- **Treat timeouts as ambiguity, not failure.** Any side-effecting operation behind a timeout needs an idempotency key and a reconciliation path, and omitting this is how duplicate charges happen.
- **Derive values from measured latency.** Timeouts set below p99 create failures that did not otherwise exist, and thirty-second defaults create cascades — both are avoidable with data that is usually already collected.
- **Reject expired work before processing it.** It is the cheapest capacity recovery available and is almost always absent.

**Go deeper**

A deadline is a property of the request — the point after which its result is useless — while a timeout is one hop's bound. Configuring independent per-hop timeouts means their sum has no relationship to the client's tolerance, so deep hops continue working long after the caller has abandoned the request, holding threads, connections and locks for results nobody will read. Propagating the remaining budget makes every hop aware of how much time actually remains.

Timeouts alone free only the caller. Unless a cancellation signal reaches whatever holds the resource — the database driver, the HTTP client — the downstream work runs to completion regardless. Under load that distinction decides whether a system recovers when traffic subsides or remains saturated with abandoned work, so verifying that cancellation propagates to the actual resource holder is the part most worth checking.

Values should come from measured latency: just above p99 catches real failures without turning healthy slow requests into errors, while framework defaults of tens of seconds permit cascading exhaustion. And a timeout is an ambiguous outcome rather than a failure — the operation may have succeeded after the caller gave up — so any side-effecting operation behind one needs an idempotency key and a reconciliation path.

Timeouts bound how long any single operation may consume resources; deadlines bound the whole request; cancellation is what makes either of them actually free anything.

**Independent timeouts cannot be correct in a chain.** Each service configures a value that was locally sensible, usually a framework default measured in tens of seconds, and nobody computes whether their sum relates to what the user will tolerate. The consequence is systematic: a client abandons after three seconds while the chain works on for another twenty-seven, so under load a large share of capacity produces results that are discarded. This is invisible in ordinary metrics because utilisation is high and nothing errors. Propagating an absolute deadline, with each hop passing on what remains, replaces the arithmetic with a value that adapts to how much time has actually been consumed upstream.

**Cancellation is the part that frees resources.** A timeout stops the caller waiting; it does not stop the callee working. Without a signal reaching the component that actually holds the resource — the database driver executing the query, the HTTP client holding the connection — the work runs to completion while its result goes nowhere. Cancelling an application-level wrapper while the underlying query continues is a common and consequential gap, and it is worth verifying directly rather than assuming the framework handles it.

**Values must come from measurement.** A timeout below p99 converts healthy slow requests into failures and triggers retries that amplify load — creating an outage from a latency distribution that was fine. A timeout at a default of thirty seconds permits thread exhaustion and cascading failure long before detection. Just above p99 gives the useful property that healthy requests almost never hit it while genuine failures are detected quickly. Different kinds of timeout also need different values: connection establishment in hundreds of milliseconds, request completion from the operation's own distribution, idle sockets in minutes.

**Retries consume budget, not just capacity.** A hop that retries twice under a one-second timeout can occupy three seconds, so a chain whose timeouts sum neatly to the deadline overruns it the moment anything retries. Retries therefore belong inside the deadline, attempted only when enough budget remains for a realistic chance of completion — otherwise the retry is guaranteed to expire and has consumed resources to produce nothing.

**A timeout is ambiguity, and ambiguity is a correctness problem.** When a call times out the operation may have failed, may still be running, or may have succeeded moments after the caller gave up. Cancellation stops waiting; it does not undo completed side effects. Retrying a payment that actually succeeded charges the customer twice, so anything with side effects behind a timeout requires an idempotency key making the retry recognisable as the same operation, plus a reconciliation path for when caller and callee disagree. Treating timeouts as definite failures is the simplification that produces duplicate charges.

**Expired work should be rejected before it starts.** A request that waited longer than its deadline while sitting in a queue is guaranteed waste, so checking the deadline on arrival and discarding it without processing is the cheapest capacity recovery available anywhere in a system. It costs a comparison and it is almost always absent — which, alongside the timeout arithmetic and the cancellation gap, makes this one of the three checks that reliably surfaces real problems in an otherwise healthy-looking service.

**Prove it — interview questions**

1. **[Basic] What is the difference between a timeout and a deadline?**

   <details><summary>Model answer</summary>

   A timeout is a duration applied to one operation; a deadline is an absolute point in time by which the whole request's result becomes useless. The deadline belongs to the request and reflects the user's tolerance, while each hop's timeout should be derived from how much of that deadline remains. The distinction matters because independently configured per-hop timeouts bear no relationship to the client's patience, so the deeper hops routinely keep working long after the caller has given up.

   </details>

2. **[Basic] Why does a timeout alone not free resources?**

   <details><summary>Model answer</summary>

   Because timing out only stops the caller waiting. Unless a cancellation signal actually reaches the callee, the downstream work continues to completion — still holding its thread, still running its database query, still consuming capacity for a result that will be discarded. Under load that is the difference between a system that recovers when traffic subsides and one that stays saturated with abandoned work, so cancellation propagation is the part that matters and it is frequently the part that is missing.

   </details>

3. **[Senior] How do you choose a timeout value?**

   <details><summary>Model answer</summary>

   From observed latency rather than from a default. A timeout just above p99 means healthy requests almost never hit it, while genuine failures are detected in roughly p99 time. Setting it below p99 manufactures failures that would not otherwise have occurred and triggers retries that amplify load; setting it to a framework default of thirty seconds allows thread exhaustion and cascading failure long before detection. It is also worth distinguishing the kinds of timeout — connection establishment should be hundreds of milliseconds, request completion comes from the operation's own p99, and idle socket timeouts are minutes — because a single value is wrong for most of them.

   </details>

4. **[Senior] Why is a timeout an ambiguous outcome?**

   <details><summary>Model answer</summary>

   Because it tells you that you stopped waiting, not what happened. The operation may have failed, may still be running, or may have completed successfully just after you gave up — and cancellation stops the waiting, it does not undo work already performed. For anything with side effects this is a correctness problem rather than a performance one: retrying a payment that actually succeeded charges the customer twice. The resolution is an idempotency key so the retry is recognised as the same operation, plus a reconciliation path for the case where caller and callee disagree about what happened.

   </details>

5. **[Staff] Design timeout handling for a checkout flow with a three-second user tolerance.**

   <details><summary>Model answer</summary>

   The deadline is set once at the gateway as an absolute time and propagated on every downstream call, so each hop is told how much budget remains rather than using a configured constant — which means that if authentication was unusually slow, the checkout service adapts instead of blindly assuming its full allocation. I would budget explicitly: routing and auth a couple of hundred milliseconds, the checkout service most of the remainder with its own internal split across inventory, payment and the order write, and roughly twenty per cent held back as margin, because without margin any single slow hop breaks the entire chain. Retries are budgeted inside the deadline and attempted only if enough time remains to have a realistic chance, since a retry that is guaranteed to expire is pure waste. Cancellation must propagate to the actual resource holder — the database driver and the payment client — because cancelling an application wrapper while the query runs on frees nothing. And critically, payment authorisation carries an idempotency key with a reconciliation path, because a timeout there means the charge may or may not have happened, and treating that ambiguity as a failure is how customers get charged twice. At the queue, any request whose deadline has already passed is rejected without processing.

   </details>

6. **[Principal] Why are timeouts so often wrong in production systems?**

   <details><summary>Model answer</summary>

   Because they are inherited rather than chosen, and nobody owns the arithmetic. Each service configures what its framework defaulted to, usually tens of seconds, and each was locally reasonable when written; nobody computes whether the sum across a chain bears any relationship to what the user will wait. The result is a system where the deepest hops routinely work for thirty seconds on requests abandoned after three, so under load a substantial share of capacity is consumed producing results nobody reads — and because utilisation looks high and nothing errors, it is invisible in ordinary metrics. Two compounding problems make it worse. Cancellation is often absent or incomplete, so even correct timeouts free only the caller while the database query grinds on. And timeouts are treated as failures rather than as ambiguity, which quietly introduces duplicate side effects wherever retries exist on side-effecting operations. What I would push for is concrete: measure the actual timeout budget across each critical chain, verify cancellation reaches the resource holder rather than a wrapper, and require idempotency on anything retried past a timeout — three checks that are cheap to perform and that consistently surface real problems.

   </details>

---

### Health, readiness and liveness

*Distinguish whether a process should be restarted from whether it should receive traffic, because conflating them turns a dependency blip into a cluster-wide restart loop.*

**Flow:** `Liveness probe` → `Readiness probe` → `Startup probe` → `Traffic routing` → `Restart decision`

> **The 30-second version**  
> Separate the restart decision from the routing decision: liveness checks only what a restart could fix, readiness checks whether traffic can be served, and a startup probe protects slow initialisation.

**The problem**

A service exposes one health endpoint that checks the database. The database becomes briefly unavailable. Every instance reports unhealthy, the orchestrator restarts all of them simultaneously, and when the database recovers there is nothing left running to serve traffic — and the cold restarts hammer the database further as every instance reconnects at once.

The endpoint answered a question nobody asked. Being unable to reach a dependency does not mean the process is broken and should be killed; it means the instance cannot serve requests right now. Those are different conditions with opposite correct responses.

> **Restart and route are separate decisions needing separate signals**  
> Liveness asks whether the process is beyond recovery and should be killed. Readiness asks whether it can serve traffic at this moment. A dependency outage makes an instance not ready but perfectly alive — removing it from rotation is right, restarting it is actively harmful. One endpoint cannot express both answers, and using it for both is how a brief dependency blip becomes a self-inflicted outage.

**Mental model**

Three distinct questions, each with a different consequence: is the process irrecoverably broken, can it serve traffic now, and has it finished starting up?

1. **Liveness** — Is this process broken beyond recovery? Failing means kill and restart.
2. **Readiness** — Can this instance serve requests right now? Failing means remove from rotation.
3. **Startup** — Has initialisation finished? Failing means keep waiting, do not kill.
4. **Consequence** — The action taken on failure, which is what distinguishes the probes.
5. **Blast radius** — Whether a probe can fail for every instance simultaneously, which determines how dangerous it is.

> **A liveness probe that checks dependencies will kill your entire fleet**  
> If liveness depends on a database, then a database outage restarts every instance at once — and restarting does not fix a database. Worse, the fleet comes back cold and reconnects simultaneously, adding load to the dependency that was already struggling. Liveness must check only conditions that a restart can actually repair, which in practice means the process's own internal state and almost nothing else.

**How it works**

**What each probe should check**

```text
LIVENESS  -> failure means KILL
  check: can this process still make progress at all?
    - the event loop or request handler responds
    - no deadlock in a core internal lock
    - no unrecoverable internal state
  DO NOT check: databases, caches, downstream services,
    anything external
  rule: only check what a RESTART would fix
  -> a restart does not fix a database outage
  -> so a database check has no place here

READINESS  -> failure means REMOVE FROM ROTATION
  check: can this instance serve a request right now?
    - required dependencies reachable
    - connection pools initialised
    - caches warm enough to serve within SLO
    - not shutting down
  this is where dependency checks belong
  -> and it is safe, because being removed from
     rotation is reversible and harmless

STARTUP  -> failure means KEEP WAITING
  check: has initialisation completed?
  exists because slow starts otherwise trip liveness
  -> without it, a service taking 90 s to warm caches
     is killed at 30 s, forever, in a restart loop

THE DANGEROUS COMBINATION
  readiness checking dependencies  = correct and safe
  liveness checking dependencies   = fleet-wide restart
```

1. **Check only restart-fixable conditions in liveness** — A restart cannot repair anything external, so external checks in liveness convert dependency problems into fleet destruction.
2. **Put dependency checks in readiness** — Removal from rotation is reversible and proportionate; killing the process is neither.
3. **Use a startup probe for slow initialisation** — Without one, a service that legitimately takes a minute to warm is killed repeatedly and never starts.
4. **Make probes cheap and independent** — A probe that queries the database on every check adds load precisely when the database is struggling.
5. **Fail readiness during shutdown before closing** — Removal from rotation must precede refusing connections, or in-flight requests fail during every deploy.
6. **Avoid correlated readiness failure** — If every instance checks the same dependency, a blip empties the entire rotation at once.

**Failure modes by probe type**

```text
SCENARIO: the database becomes unavailable for 60 s

IF LIVENESS CHECKS THE DATABASE
  all instances fail liveness
  orchestrator kills all of them
  new instances start, cannot reach the database,
    fail liveness again
  -> CrashLoopBackOff across the fleet
  -> when the database returns, everything is cold
  -> simultaneous reconnection storm
  -> a 60 s dependency blip becomes a 20-minute outage

IF READINESS CHECKS THE DATABASE
  all instances fail readiness
  removed from rotation; processes keep running
  connection pools, caches and JIT state preserved
  -> the database returns
  -> readiness passes within seconds
  -> traffic resumes immediately, warm
  -> a 60 s blip is a 60 s outage

SAME DEPENDENCY, SAME OUTAGE, VERY DIFFERENT RESULT

THE SUBTLER PROBLEM: CORRELATED READINESS
  even correct readiness empties the whole rotation
  when every instance depends on the same thing
  -> consider whether the instance can serve DEGRADED
     traffic without that dependency
  -> if it can, stay ready and degrade rather than
     disappearing entirely
```

> **Readiness that fails everywhere at once is still an outage**  
> Correctly placing dependency checks in readiness avoids restart loops but does not avoid the outage: if every instance depends on the same component, they all leave rotation together and there is nothing to route to. The question worth asking is whether the instance could serve something useful without that dependency — cached data, a degraded response, a subset of endpoints — because staying in rotation and degrading is often better than vanishing entirely.

**Worked example**

A web service with a database, a cache and an optional recommendations dependency.

**Probe design**

```text
LIVENESS  /internal/alive
  returns 200 if the request handler is responsive and
  no core lock is deadlocked
  checks NOTHING external
  period 10 s, failure threshold 3
  -> ~30 s to kill a genuinely wedged process
  -> and no dependency can ever trigger it

READINESS  /internal/ready
  database: connection pool has a usable connection
            (from cached pool state, NOT a live query)
  cache:    reachable OR degraded mode acceptable
  recommendations: NOT checked - it is optional and
            has a fallback
  not shutting down
  period 5 s, failure threshold 2
  -> ~10 s to leave rotation

STARTUP  /internal/started
  configuration loaded, pools initialised, caches warm
  period 5 s, failure threshold 24
  -> allows up to 2 minutes to start
  -> liveness does not begin until this passes

SHUTDOWN SEQUENCE
  1  receive termination signal
  2  readiness starts failing IMMEDIATELY
  3  wait for the load balancer to notice (~10 s)
  4  THEN stop accepting new connections
  5  finish in-flight requests
  6  exit
  -> skipping step 3 means failed requests on every
     deploy, which is the most common cause of
     deploy-time errors

PROBE COST
  readiness reads cached pool state, no queries
  -> 1,000 instances x every 5 s would otherwise be
     200 queries/s of pure probe load on a database
     that may already be struggling
```

| Metric | Value | Note |
|---|---|---|
| Liveness | internal only | **never dependencies** |
| Readiness | dependencies | reversible |
| Startup | 2 min allowance | no kill during warm-up |
| Shutdown | unready first | then close |

> **Failing readiness before closing connections is what makes deploys invisible**  
> Most deploy-time errors come from a process that stops accepting connections before the load balancer has noticed it is going away, so in-flight routing sends requests to a socket that is already closed. Failing readiness first, waiting longer than the load balancer's detection interval, and only then closing is a small sequencing change that removes an entire category of user-visible errors.

**When to use it**

- **Any orchestrated deployment**, where the platform makes restart and routing decisions automatically.
- **Load-balanced services**, where readiness controls which instances receive traffic.
- **Services with slow initialisation**, which need a startup probe to avoid restart loops.
- **Rolling deployments**, where readiness gates progression to the next instance.
- **Graceful shutdown**, where readiness is the mechanism for draining traffic.

**When to avoid it**

- **Do not check dependencies in liveness**, which turns dependency outages into fleet-wide restarts.
- **Do not use one endpoint for both probes**, since the correct answers differ.
- **Do not run expensive queries in probes**, which adds load exactly when the dependency is struggling.
- **Do not check optional dependencies in readiness**, which removes capacity for a feature that has a fallback.
- **Do not close connections before failing readiness**, which causes errors on every deployment.

**Advantages**

- **Automatic recovery** from genuinely wedged processes.
- **Traffic routed only to instances that can serve**, without manual intervention.
- **Slow starts tolerated** without restart loops.
- **Graceful deploys** with no user-visible errors when sequencing is right.
- **Rolling updates gated on readiness**, so a broken release stops progressing.
- **Clear separation** between restart-worthy and route-worthy conditions.

**Disadvantages**

- **Easy to misconfigure catastrophically**, since dependency checks in liveness destroy fleets.
- **Probe load** can be significant across a large fleet.
- **Correlated readiness failure** empties the entire rotation at once.
- **Tuning thresholds** trades detection speed against flapping.
- **Probes can pass while the service is useless**, if they check the wrong things.
- **Shutdown sequencing is subtle** and frequently wrong.

**Trade-offs**

**Probe configuration trade-offs**

| Setting | Too aggressive | Too lenient | Reasonable |
|---|---|---|---|
| Liveness threshold | Healthy processes killed on a blip | Wedged processes persist | 3 failures at 10 s |
| Readiness threshold | Flapping in and out of rotation | Traffic to instances that cannot serve | 2 failures at 5 s |
| Startup allowance | Restart loop during warm-up | Broken starts undetected for ages | Generous, e.g. 2 minutes |
| Probe depth | Load on dependencies; false failures | Passes while broken | Cached state, not live queries |
| Shutdown delay | Errors during deploys | Slow rollouts | Longer than LB detection interval |

The asymmetry worth internalising is that readiness failures are cheap and reversible while liveness failures are destructive, so readiness can afford to be sensitive and liveness should be extremely conservative.

**How it fails**

**Health check failures**

| Failure | Cause | Fix |
|---|---|---|
| Fleet-wide restart loop | Liveness checks a dependency | Liveness checks internal state only |
| Service never starts | Slow initialisation trips liveness | Startup probe with a generous allowance |
| Errors on every deploy | Connections closed before readiness fails | Fail readiness first, then wait, then close |
| Entire rotation empties | Correlated readiness on a shared dependency | Degrade and stay ready where possible |
| Dependency overwhelmed by probes | Live queries in readiness across a large fleet | Use cached pool state |
| Capacity lost for an optional feature | Readiness checks a dependency with a fallback | Check only required dependencies |
| Probes pass while the service is broken | Probe checks nothing meaningful | Check the ability to serve, not just process liveness |

**Limits**

> **Typical settings**
>
> - **Liveness**: 10 s period, 3 failures — roughly 30 s to kill a wedged process, and never dependency-based.
> - **Readiness**: 5 s period, 2 failures — about 10 s to leave rotation.
> - **Startup**: generous, often a minute or two, since it only gates the transition to liveness checking.
> - **Shutdown delay**: longer than the load balancer's detection interval, typically 10–30 s.
> - **Probe cost**: cached state rather than live queries, since fleet-wide probe load can rival real traffic.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Separate liveness/readiness/startup | Orchestrated deployments | Configuration complexity |
| Single health endpoint | Very simple services | Conflates restart and route decisions |
| External synthetic monitoring | End-to-end verification | Slower; outside the orchestrator |
| Load balancer health checks only | Non-orchestrated deployments | No restart capability |
| Manual intervention | Small static fleets | Does not scale; slow |
| Degraded-mode readiness | Dependency-tolerant services | Requires a real fallback path |

Degraded-mode readiness deserves more attention than it usually receives: if an instance can serve cached results or a subset of endpoints without a failing dependency, remaining in rotation and degrading beats disappearing, because an empty rotation serves nobody at all.

**In real systems**

- **Orchestrators with three distinct probe types** exist precisely because conflating restart and routing decisions caused widespread fleet-destroying outages.
- **Startup probes** were introduced because slow-initialising services were being killed by liveness checks before they could finish warming.
- **Failing readiness before closing connections** is standard graceful-shutdown practice and removes most deploy-time errors.
- **Cached dependency state in readiness** avoids probe traffic multiplying against a dependency that is already under strain.
- **Degraded-mode operation with a still-passing readiness check** keeps capacity available when a non-essential dependency fails.

**Common mistakes**

- **Dependency checks in liveness**, causing fleet-wide restart loops.
- **One endpoint for liveness and readiness**, conflating opposite decisions.
- **No startup probe** for slow-starting services, producing permanent restart loops.
- **Closing connections before failing readiness**, causing errors on every deploy.
- **Live queries in probes**, multiplying load on a struggling dependency.
- **Checking optional dependencies in readiness**, losing capacity for a feature with a fallback.
- **Probes that pass while the service cannot actually serve**, giving false confidence.

**The staff-level view**

Health check misconfiguration is one of the few things that reliably converts a minor dependency problem into a major outage.

- **Audit liveness probes for external checks.** A database check in liveness means a database blip restarts the fleet, returns it cold, and hammers the recovering dependency — the single most destructive health check mistake there is.
- **Require a startup probe wherever initialisation is slow.** Without it, a service that legitimately takes ninety seconds to warm enters a permanent restart loop and never serves anything.
- **Check the shutdown sequence directly.** Failing readiness before closing connections eliminates most deploy-time errors, and the wrong ordering is extremely common because it produces only intermittent symptoms.
- **Ask whether readiness failure should be degradation instead.** Correlated readiness on a shared dependency empties the entire rotation, so serving something reduced is usually better than serving nothing.
- **Keep probes cheap.** Across a large fleet, probe traffic against a dependency can rival real traffic, and it arrives exactly when that dependency is struggling.

**Go deeper**

Liveness and readiness answer different questions with opposite consequences. Liveness failing means the process is killed; readiness failing means it is removed from rotation while continuing to run. A dependency outage makes an instance unable to serve but entirely functional as a process — so it belongs in readiness, and putting it in liveness means a database blip restarts the whole fleet, brings it back cold, and directs a reconnection storm at the recovering dependency.

A startup probe exists because slow initialisation is indistinguishable from a wedged process. Without one, a service that legitimately takes ninety seconds to warm caches is killed at thirty and enters a permanent restart loop. The startup probe gates when liveness checking begins, so its allowance can be generous without weakening detection later.

Two further details matter operationally. Probes must be cheap — live dependency queries across a large fleet add substantial load exactly when that dependency is struggling, so cached state is preferable. And shutdown must fail readiness before closing connections, waiting longer than the load balancer's detection interval, because the reverse ordering produces failed requests on every deployment and is the most common source of deploy-time errors.

Health checking separates two decisions an orchestrator makes automatically — whether to restart a process and whether to send it traffic — which have different correct answers and very different costs when wrong.

**Liveness must check only restart-fixable conditions.** A restart repairs internal state: a deadlocked lock, an unresponsive handler, corrupted in-process data. It repairs nothing external. Checking a database in liveness therefore means that any database incident causes every instance to fail simultaneously, be killed, restart, fail again because the database is still unavailable, and loop — while the recovered dependency is then hit by the entire fleet reconnecting at once from a cold start. A brief dependency blip becomes a prolonged self-inflicted outage, and it is the single most destructive health-check misconfiguration.

**Readiness is where dependency awareness belongs**, because its consequence is proportionate and reversible. An instance removed from rotation keeps its connection pools, warm caches and accumulated runtime state, so when the dependency returns it resumes serving within seconds. The same outage that produces a twenty-minute recovery through liveness produces a sixty-second one through readiness, with identical underlying facts.

**Correlated readiness failure is the residual problem.** Correct placement avoids restart loops but not the outage: if every instance checks the same dependency, all of them leave rotation together and there is nothing left to route to. The question worth asking is whether the instance could still serve something useful — cached results, a subset of endpoints, a degraded response — because staying in rotation and degrading serves more users than vanishing entirely. This turns readiness from a binary into a product decision about what partial service is acceptable.

**Startup probes resolve an ambiguity that otherwise breaks slow services.** A process ninety seconds into cache warming and a process permanently wedged look identical to a liveness check. Without a startup probe, the liveness threshold must be long enough to accommodate the slowest legitimate start, which makes detection of genuinely broken processes correspondingly slow. Separating the two lets startup be generous and liveness be tight, which is strictly better than either compromise.

**Probe cost is real at fleet scale.** A readiness check issuing a live database query, multiplied by a thousand instances at a five-second interval, is hundreds of queries per second of pure overhead — arriving precisely when the database is under strain and most likely to be the thing being checked. Reading cached connection-pool state gives nearly the same signal at no cost to the dependency, and this substitution is one of the easier significant improvements available.

**Shutdown sequencing removes an entire category of deploy errors.** A process that stops accepting connections on receiving a termination signal is still being routed to until the load balancer's next health check notices, so those requests hit a closed socket. Failing readiness first, waiting longer than the detection interval, then closing and draining in-flight work means traffic has already moved away before anything refuses it. The wrong order produces intermittent errors on every deployment that are easily attributed to something else, which is why the sequence is worth verifying explicitly rather than assumed to be handled.

**Prove it — interview questions**

1. **[Basic] What is the difference between liveness and readiness?**

   <details><summary>Model answer</summary>

   They answer different questions with different consequences. Liveness asks whether the process is broken beyond recovery, and failing it means the process is killed and restarted. Readiness asks whether the instance can serve traffic right now, and failing it means removal from the load balancer rotation while the process keeps running. A dependency outage makes an instance not ready but entirely alive — taking it out of rotation is correct, restarting it is harmful.

   </details>

2. **[Basic] Why does a startup probe exist?**

   <details><summary>Model answer</summary>

   Because slow initialisation otherwise looks identical to a wedged process. A service that legitimately takes ninety seconds to load configuration, establish connection pools and warm caches will be killed by a liveness probe configured for thirty seconds, restart, and be killed again — a permanent restart loop in which it never successfully starts. A startup probe gates the transition: liveness checking does not begin until initialisation has completed, so the allowance can be generous without weakening liveness detection afterwards.

   </details>

3. **[Senior] Why must liveness never check dependencies?**

   <details><summary>Model answer</summary>

   Because a restart cannot fix anything external. If liveness checks the database and the database becomes unavailable, every instance fails simultaneously and the orchestrator kills all of them. The replacements also cannot reach the database, so they fail too — a fleet-wide restart loop. When the database recovers there is nothing warm left running, and every instance reconnects at once, adding load to a component that has just come back. A sixty-second dependency blip becomes a twenty-minute outage entirely of your own making. The rule is that liveness should check only conditions a restart would actually repair, which in practice means internal process state and nothing else.

   </details>

4. **[Senior] Why fail readiness before closing connections during shutdown?**

   <details><summary>Model answer</summary>

   Because the load balancer needs time to notice. If a process stops accepting connections the moment it receives a termination signal, the balancer is still routing to it for however long its health-check interval takes, and those requests hit a closed socket. Failing readiness first, waiting longer than the balancer's detection interval, and only then refusing new connections means traffic has already been drained before anything closes. This sequencing is the most common cause of deploy-time errors and the fix is small — which is why it is worth checking directly rather than assuming.

   </details>

5. **[Staff] Design health checks for a service with a database, a cache, and an optional recommendations dependency.**

   <details><summary>Model answer</summary>

   Three separate probes with clearly different scopes. Liveness checks only internal state — that the request handler responds and no core lock is deadlocked — and checks nothing external whatsoever, because a restart cannot repair a database and a dependency check there would restart the entire fleet during any database incident. Readiness checks required dependencies: whether the connection pool holds a usable database connection, read from cached pool state rather than by issuing a live query, since a thousand instances probing every few seconds would otherwise add hundreds of queries per second to a database that may already be struggling. The cache is checked only if degraded operation without it is unacceptable, and recommendations is deliberately not checked at all, because it is optional with a fallback and failing readiness for it would remove capacity to protect a feature that already degrades gracefully. A startup probe with a generous allowance gates liveness so warm-up cannot trigger restarts. And shutdown fails readiness first, waits longer than the load balancer's detection interval, then stops accepting connections and drains in-flight work — which is what makes deploys invisible to users. I would also ask whether readiness failure on the database could instead be degraded service from cache, because correlated readiness failure empties the whole rotation and an empty rotation serves nobody.

   </details>

6. **[Principal] Why is health check configuration disproportionately dangerous?**

   <details><summary>Model answer</summary>

   Because it is the one piece of configuration that grants a machine authority to destroy your fleet, and the destructive setting looks entirely reasonable. Checking the database in a health endpoint seems obviously correct — surely a service that cannot reach its database is unhealthy — but in a liveness probe it means a transient dependency problem kills every instance simultaneously, brings them back cold, and directs a reconnection storm at the component that just recovered. The blast radius is total and self-inflicted, and it converts a brief incident into a long one. What makes this hard to catch is that the configuration is correct on the readiness side and catastrophic on the liveness side while looking nearly identical, and it is usually written once by someone following an example. So the review questions I treat as non-negotiable are: does liveness touch anything external, is there a startup probe for anything slow to initialise, and does shutdown fail readiness before closing connections. Those three account for the overwhelming majority of health-check-caused outages, and all three are cheap to verify and cheap to fix before they matter.

   </details>

---

### Disaster recovery and restore

*Define how much data you may lose and how long you may be down, then prove by rehearsal that you can actually meet both — because an untested backup is a hypothesis.*

**Flow:** `Backup strategy` → `RPO and RTO` → `Restore procedure` → `Rehearsal` → `Verified recovery`

> **The 30-second version**  
> Derive backup and replication decisions from how much data you may lose and how long you may be down — then rehearse the restore on a schedule, because the measured recovery time is usually nothing like the assumed one.

**The problem**

A database is corrupted by a bad migration at two in the afternoon. Backups exist — nightly, to object storage, running successfully for three years. The restore begins, and then the questions start: how long does restoring two terabytes take, does the backup include the schema, do the credentials still work, what about everything written since midnight, and who knows the procedure.

Nobody has answers, because nobody has ever done it. The backup job succeeding proved only that a file was written; it proved nothing about whether that file can become a working system, or how long that would take.

> **A backup is a hypothesis until a restore has been rehearsed**  
> Backup success metrics measure whether data was copied, not whether it can be recovered. The gap between those two things is where disasters live: unreadable archives, missing schema, expired credentials, restore times measured in days rather than hours, and procedures that exist only in one person's memory. Rehearsal is what converts a hypothesis into a capability, and it is the step almost universally skipped.

**Mental model**

Two numbers define the requirement: how much recent data you can afford to lose, and how long you can afford to be down. Every backup and replication decision follows from those, and a rehearsal is what proves the decisions were adequate.

1. **RPO** — Recovery point objective — the maximum acceptable data loss, measured in time.
2. **RTO** — Recovery time objective — the maximum acceptable downtime.
3. **Backup strategy** — What is captured, how often, and where it is stored — derived from the RPO.
4. **Restore procedure** — The documented, executable steps that turn backups into a running system.
5. **Rehearsal** — Periodic practice that verifies both numbers are actually achievable.

> **Backups in the same failure domain as the data are not backups**  
> A snapshot in the same account, the same region, or reachable with the same credentials as the primary system shares its failure modes. Ransomware that encrypts the database encrypts the backups; an account compromise deletes both; a regional outage takes out both. Isolation — separate credentials, separate account, separate region, ideally immutable retention — is what distinguishes a backup from a second copy waiting to be destroyed alongside the first.

**How it works**

**From objectives to strategy**

```text
RPO drives BACKUP FREQUENCY
  RPO 24 h  -> nightly backup
  RPO 1 h   -> hourly, or continuous log archiving
  RPO 5 min -> continuous archiving / streaming
  RPO ~0    -> synchronous replication (and that is
               replication, not backup)

RTO drives RESTORE MECHANISM
  RTO 24 h  -> restore from cold archive storage
  RTO 4 h   -> restore from warm storage, practised
  RTO 1 h   -> standby replica ready to promote
  RTO 5 min -> hot standby with automatic failover

THE COST CURVE IS STEEP IN BOTH DIRECTIONS
  nightly backup to object storage      cheap
  continuous archiving                  moderate
  warm standby in another region        ~2x infra
  hot multi-region active-active        3x+ and a
                                        large ongoing
                                        complexity tax

REPLICATION IS NOT BACKUP
  replication copies everything, including the
  DELETE that destroyed your data, within seconds
  -> it protects against hardware and site failure
  -> it does NOT protect against corruption, bad
     migrations, bugs or malice
  -> you need both, for different threats
```

1. **Derive RPO and RTO from business impact, not from what backups happen to provide** — The current backup schedule is an implementation detail; the objectives are a business decision that should drive it.
2. **Keep backups in a separate failure domain** — Separate credentials, account and region, because a backup destroyed with the primary is worthless.
3. **Use immutable retention for a window** — Ransomware and malicious deletion both target backups first; retention that cannot be shortened is the defence.
4. **Automate and document the restore path** — A procedure that lives in one person's memory fails when that person is unavailable, which is disproportionately likely during a crisis.
5. **Rehearse on a schedule and time it** — The rehearsal produces the actual RTO, which is frequently several times the assumed one.
6. **Verify restored data, not just restore completion** — A restore that produces a corrupt or partial database has completed successfully and delivered nothing.

**What rehearsal reveals**

```text
TYPICAL FIRST REHEARSAL FINDINGS

TIME
  assumed RTO: 2 hours
  actual: 11 hours
    download 2 TB from cold storage      4 h
    decompress and restore                3 h
    rebuild indexes                       2 h
    replay logs since the snapshot        1 h
    verify and cut over                   1 h
  -> nobody had ever measured any of these

COMPLETENESS
  schema included? often not
  sequences and auto-increment state? frequently lost
  blob/object storage? backed up separately, or
    not at all
  secrets and configuration? almost never

ACCESS
  credentials expired
  the restore role lacks a permission added last year
  the runbook references a tool that no longer exists

DEPENDENCIES
  the service cannot start without a config store that
    was also lost
  DNS still points at the destroyed primary
  downstream systems cache stale connection details

-> none of these appear in a backup success metric
-> all of them appear in the first rehearsal
```

> **Restore time is dominated by steps nobody counts**  
> Teams estimate restore time from the database restore command and forget retrieval from cold storage, decompression, index rebuilding, log replay, verification, DNS propagation and downstream reconnection. The measured figure is routinely several times the assumed one, which means an RTO commitment made without a rehearsal is essentially a guess — and one that will be discovered to be wrong at the worst possible moment.

**Worked example**

An e-commerce platform deriving its recovery strategy from stated business tolerance.

**Objectives, strategy and verification**

```text
BUSINESS TOLERANCE
  losing orders:      unacceptable -> RPO ~0 for orders
  losing analytics:   1 day acceptable
  downtime:           1 hour maximum for checkout

STRATEGY BY DATA CLASS
  orders database
    synchronous replica in a second availability zone
    continuous log archiving to a separate account
    -> RPO seconds, RTO ~15 min via promotion
  product catalogue
    hourly snapshots, cross-region, immutable 30 days
    -> RPO 1 h, RTO 2 h
  analytics warehouse
    nightly, cold storage
    -> RPO 24 h, RTO 24 h - and that is fine

ISOLATION
  backups written to a separate account
  write-only credentials from production
  immutable retention 30 days - cannot be deleted
    early even with full production compromise

REHEARSAL PROGRAMME
  monthly:   restore the catalogue to a scratch
             environment, verify row counts and
             checksums, record elapsed time
  quarterly: full failover of the orders database,
             including DNS and downstream
             reconnection
  annually:  region-loss exercise

WHAT THE FIRST QUARTERLY EXERCISE FOUND
  promotion worked in 12 min
  but DNS TTL was 1 hour -> real RTO was 70 min
  -> TTL reduced to 60 s; retested; RTO now 15 min
  -> this is the entire value of rehearsal
```

| Metric | Value | Note |
|---|---|---|
| Orders | RPO ~0 / RTO 15 m | sync replica |
| Catalogue | RPO 1 h / RTO 2 h | cross-region |
| Analytics | RPO 24 h | cold, cheap |
| Found by drill | DNS TTL 1 h | **RTO was 70 m** |

> **Different data classes deserve different objectives**  
> Applying the strictest requirement to everything is enormously expensive and usually unnecessary. Orders may warrant synchronous replication while the analytics warehouse is perfectly well served by a nightly backup to cold storage. Classifying data by what its loss would actually cost lets expensive mechanisms be spent where they matter, and it usually reduces total cost while improving protection where it counts.

**When to use it**

- **Any system holding data whose loss would matter**, which is essentially all production systems.
- **Regulated environments**, where recovery capability is often a compliance requirement.
- **Before major migrations**, where a verified restore path is the rollback plan.
- **Multi-region architectures**, where regional failure is an explicit design consideration.
- **Whenever objectives are stated**, since an unrehearsed objective is not a commitment.

**When to avoid it**

- **Do not rely on replication as backup**, since it faithfully copies corruption and deletion.
- **Do not store backups in the same failure domain**, where they share the primary's fate.
- **Do not apply the strictest objective to all data**, which wastes money without improving what matters.
- **Do not claim an RTO that has never been measured**, which is a guess presented as a commitment.
- **Do not treat backup job success as verification**, since it proves only that a file was written.

**Advantages**

- **Bounded, known loss and downtime** rather than an unknown outcome.
- **Protection against corruption and deletion**, which replication cannot provide.
- **Confidence from rehearsal**, converting a hypothesis into a measured capability.
- **A rollback path** for risky migrations and releases.
- **Tiered cost**, with expensive mechanisms applied only where justified.
- **Defence against ransomware** when retention is immutable.

**Disadvantages**

- **Ongoing cost** in storage, replication and rehearsal effort.
- **Rehearsals take real time** from teams who always have more urgent work.
- **Strict objectives are expensive**, and near-zero RPO usually implies synchronous replication.
- **Procedures decay** as systems change, so documentation needs continual maintenance.
- **Backup storage is a security surface** holding a complete copy of production data.
- **Complexity grows** with the number of data stores that must be recovered consistently.

**Trade-offs**

**Objective and cost trade-offs**

| RPO / RTO | Mechanism | Relative cost | Appropriate for |
|---|---|---|---|
| 24 h / 24 h | Nightly backup to cold storage | Very low | Analytics; derived data |
| 1 h / 4 h | Hourly snapshots, warm storage | Low | Catalogues; content |
| 5 min / 1 h | Continuous archiving, standby | Moderate | Most transactional systems |
| ~0 / 15 min | Synchronous replica, promotion | High | Orders; payments |
| ~0 / seconds | Active-active multi-region | Very high | Systems where any downtime is unacceptable |

Moving from hours to minutes of RTO usually means keeping a standby running, which roughly doubles infrastructure for that component — so the question is always which specific data justifies it, rather than what the platform-wide standard should be.

**How it fails**

**Disaster recovery failures**

| Failure | Cause | Fix |
|---|---|---|
| Restore takes far longer than the stated RTO | Never measured; steps uncounted | Timed rehearsals; publish the measured figure |
| Backups unusable | Never verified beyond job success | Restore and verify data, not just completion |
| Backups destroyed with the primary | Same account, region or credentials | Separate failure domain; immutable retention |
| Corruption replicated instantly | Replication mistaken for backup | Point-in-time backups alongside replication |
| Restore blocked by expired access | Credentials and permissions drifted | Rehearsal exercises the real access path |
| System restored but cannot start | Configuration and secrets not backed up | Include configuration in scope |
| Failover works but traffic does not move | Long DNS TTLs; downstream caching | Short TTLs; rehearse the full cutover |

**Limits**

> **Planning figures**
>
> - **Restore time** is typically several times the database restore step alone once retrieval, indexing, log replay and verification are counted.
> - **Cold storage retrieval** can take hours before any restore begins.
> - **Immutable retention**: 30 days is a common window against ransomware and malicious deletion.
> - **Rehearsal cadence**: monthly for partial restores, quarterly for full failover.
> - **DNS TTL** bounds cutover time and is a frequent hidden contributor to RTO.

**Alternatives**

| Approach | Protects against | Does not protect against |
|---|---|---|
| Point-in-time backups | Corruption, deletion, bugs | Long restore times |
| Synchronous replication | Hardware and site failure | Corruption, deletion, bad migrations |
| Asynchronous replication | Site failure, with some lag | Corruption; recent writes |
| Immutable snapshots | Ransomware, malicious deletion | Application-level corruption if propagated before the snapshot |
| Multi-region active-active | Regional failure | Logical corruption; very high cost |
| Backup only, no rehearsal | Nothing reliably | Everything, discovered during the incident |

Replication and backup answer different threats and are not substitutes: replication protects against losing hardware, while backups protect against losing data correctness. A system with excellent replication and no point-in-time recovery is fully protected against the failure that is easiest to survive and defenceless against the one that most often causes real data loss.

**In real systems**

- **Immutable backup retention** has become standard practice specifically because ransomware operators target backups before encrypting primary data.
- **Cross-account backup storage with write-only production credentials** prevents a full production compromise from also destroying recovery data.
- **Scheduled restore rehearsals** are the only reliable way organisations discover that their measured RTO differs substantially from the assumed one.
- **Point-in-time recovery via continuous log archiving** provides fine-grained RPO without the cost of synchronous replication.
- **DNS TTL reduction before planned failover** is routine, because propagation time frequently dominates the cutover.

**Common mistakes**

- **Treating backup job success as proof of recoverability.**
- **Backups in the same account, region or credential scope** as the primary.
- **Relying on replication**, which copies corruption faithfully.
- **Stating an RTO nobody has ever measured.**
- **Omitting configuration and secrets** from backup scope, so restored systems cannot start.
- **Ignoring DNS TTL and downstream caching** in cutover planning.
- **Never rehearsing**, so the first restore attempt is during the disaster.

**The staff-level view**

Disaster recovery is the area where stated capability and actual capability diverge most, and the divergence is only ever discovered at the worst time.

- **Insist that any stated RTO has been measured.** An unrehearsed objective is a guess, and the measured figure is routinely several times larger once retrieval, indexing, log replay and cutover are counted.
- **Check the failure domain of the backups.** Same account, same credentials or same region means the backup shares the primary's fate, and this is depressingly common.
- **Separate replication from backup in the conversation.** Teams with strong replication often believe they are covered, while being fully exposed to the corruption and deletion cases that cause most real data loss.
- **Classify data by loss cost.** Applying the strictest objective everywhere is expensive and usually reduces the budget available for the data that genuinely warrants it.
- **Schedule rehearsals as work, not as aspiration.** They compete with everything urgent and will not happen otherwise, and the first one always finds something that would have been fatal.

**Go deeper**

Recovery planning starts from two business decisions: the recovery point objective, meaning how much recent data may be lost, and the recovery time objective, meaning how long downtime may last. The RPO determines backup frequency or replication mode; the RTO determines whether recovery means restoring from cold storage or promoting a standby that is already warm. Applying the strictest objective to all data is expensive and usually starves the data that genuinely warrants it.

Replication and backup address different threats and neither substitutes for the other. Replication protects against hardware and site failure while faithfully copying corruption, deletion and bad migrations within seconds — which are the causes of most real data loss. Backups must also sit in a separate failure domain: a different account, different credentials, a different region, with immutable retention, because ransomware and compromise both target recovery data first.

The decisive practice is rehearsal. Backup job success proves only that a file was written — not that it is readable, complete, includes configuration and secrets, or can be restored by someone available at the time. Measured restore times are routinely several times the assumed figure once cold-storage retrieval, index rebuilding, log replay, verification and DNS cutover are counted, so an unrehearsed objective is a guess rather than a commitment.

Disaster recovery is the discipline of bounding two quantities — data loss and downtime — and then proving the bounds are achievable rather than assuming it.

**The objectives are business decisions that drive everything technical.** How much recent data may be lost determines whether nightly backups suffice, continuous log archiving is needed, or synchronous replication is required. How long downtime may last determines whether recovery means retrieving from cold archive or promoting a standby already running. Deriving these from what the current backup schedule happens to provide inverts the relationship and produces objectives that describe existing practice rather than actual requirements.

**Replication and backup protect against different things.** Replication guards against losing hardware or a site, and it copies every write faithfully — including the bad migration, the accidental deletion and the application bug that corrupted a table. Those propagate to every replica within seconds. Point-in-time recovery is what allows returning to a state before the damage, and it addresses the failure mode that causes most real data loss. A system with excellent replication and no point-in-time recovery is comprehensively protected against the easier problem and defenceless against the harder one, which is a common and comfortable position to be in.

**Backups must not share the primary's failure domain.** A snapshot in the same account, reachable with the same credentials, or resident in the same region, is destroyed by the same events that destroy the original: ransomware, account compromise, regional outage. Separation means a distinct account, credentials that permit writing but not deletion from production, a different region, and immutable retention for a defined window. Immutability specifically matters because ransomware operators target backups before encrypting primary data, so retention that cannot be shortened even with full production access is the meaningful defence rather than mere separation.

**Restore time is dominated by steps nobody counts.** Estimates are typically made from the database restore command, omitting retrieval from cold storage, decompression, index rebuilding, log replay since the snapshot, data verification, DNS propagation and downstream reconnection. Each is substantial and their sum is frequently several times the assumed figure. This is why an RTO that has never been measured is a guess, and why the first timed rehearsal so reliably produces an uncomfortable number.

**Rehearsal is what converts a hypothesis into a capability.** It surfaces the things backup metrics cannot: archives that are unreadable, schemas or sequence state that were never captured, configuration and secrets omitted so the restored system cannot start, credentials that expired, runbook steps referencing tools that no longer exist, and procedures that live in the memory of someone who may not be available. None of these appear in a green dashboard and all of them appear in the first drill — which is also where the real elapsed time is measured and the published objective can be corrected.

**The organisational failure is more stubborn than the technical one.** Backup job success stays green for years, so capability is assumed; rehearsal is expensive, disruptive and always deferrable against work with deadlines, so it never happens; and both facts are discovered simultaneously during an incident. The most effective intervention is procedural: require every published recovery objective to carry the date it was last verified by rehearsal, and treat one without a recent verification date as an aspiration rather than a commitment. That convention makes an invisible gap visible, which is generally the only thing that gets rehearsal onto a schedule and keeps it there.

**Prove it — interview questions**

1. **[Basic] What are RPO and RTO?**

   <details><summary>Model answer</summary>

   Recovery point objective is the maximum amount of recent data you can afford to lose, expressed as a time window — an RPO of one hour means losing up to an hour of writes is acceptable. Recovery time objective is the maximum tolerable downtime. They are business decisions rather than technical ones, and everything else follows from them: the RPO determines backup frequency or replication mode, while the RTO determines whether you restore from cold storage or promote a standby that is already running.

   </details>

2. **[Basic] Why is replication not a backup?**

   <details><summary>Model answer</summary>

   Because it copies everything faithfully, including the mistakes. A bad migration, an accidental deletion, application-level corruption or a malicious action propagates to every replica within seconds, so replication protects against losing hardware or a site while offering no protection at all against losing data correctness. Point-in-time backups let you go back to before the damage. Both are needed because they address genuinely different threats, and teams with excellent replication frequently believe they are covered when they are exposed to the more common failure.

   </details>

3. **[Senior] Why does an untested backup not count?**

   <details><summary>Model answer</summary>

   Because backup success only proves a file was written. It does not prove the archive is readable, that the schema is included, that sequence state survived, that configuration and secrets are captured, that the restore credentials still work, or that the procedure is executable by anyone currently employed. Most significantly, it does not establish how long a restore takes — and measured restore times are routinely several times the assumed figure once cold-storage retrieval, decompression, index rebuilding, log replay, verification and DNS cutover are counted. Every one of those gaps surfaces in the first rehearsal and none of them appear in a backup metric.

   </details>

4. **[Senior] Why must backups live in a separate failure domain?**

   <details><summary>Model answer</summary>

   Because anything that can destroy the primary can destroy a backup that shares its context. Ransomware encrypting the database encrypts snapshots reachable with the same credentials; a compromised account deletes both; a regional failure removes both. Backups belong in a separate account with credentials that allow writing but not deletion from production, in a different region, with immutable retention for a defined window so they cannot be shortened even by an attacker with full production access. Ransomware operators specifically target backups first, which is why immutability rather than mere separation has become the standard.

   </details>

5. **[Staff] Design disaster recovery for an e-commerce platform.**

   <details><summary>Model answer</summary>

   I would start by classifying data against business tolerance rather than choosing a platform-wide standard, because the strictest requirement applied everywhere is expensive and usually starves the data that actually needs it. Orders cannot tolerate loss, so synchronous replication to a second availability zone plus continuous log archiving, giving an RPO near zero and an RTO of minutes via promotion. The product catalogue tolerates an hour, so hourly cross-region snapshots with immutable thirty-day retention. Analytics tolerates a day, so nightly to cold storage — and being explicit that this is fine is what frees budget for the orders path. All backups go to a separate account with write-only credentials from production and immutable retention, so a full production compromise cannot destroy them. Then the part that actually determines whether any of it works: a rehearsal programme with monthly partial restores verifying row counts and checksums, quarterly full failover of the orders database including DNS and downstream reconnection, and an annual region-loss exercise. The measured elapsed time from those rehearsals becomes the published RTO, replacing the assumed one — in my experience the first full failover exercise finds something like a one-hour DNS TTL that quietly made the real RTO five times the stated figure.

   </details>

6. **[Principal] Why do organisations consistently overestimate their recovery capability?**

   <details><summary>Model answer</summary>

   Because the metrics they watch measure the wrong thing and the verifying activity competes with work that always feels more urgent. Backup dashboards report job success, which establishes that bytes were copied and nothing else — not readability, not completeness, not whether the restore credentials still function, and crucially not how long recovery takes. Since that metric is green for years, the capability is assumed. Meanwhile rehearsal is the only activity that would reveal otherwise, and it is expensive, disruptive, and always deferrable in favour of something with a deadline, so it is deferred indefinitely. The result is an organisation with a documented RTO that nobody has measured and a procedure that exists in one person's memory, discovering both facts simultaneously during an incident — when the person may be unavailable and the measured time turns out to be several times the committed one. The intervention I would push hardest for is procedural rather than technical: require that any published recovery objective carries the date it was last verified by rehearsal, and treat an objective without a recent verification date as an aspiration rather than a commitment. That single convention converts an invisible gap into a visible one, which is the only reliable way I know to get rehearsal scheduled.

   </details>

---

### Graceful degradation

*Design the system so that losing a component removes a capability rather than the whole service, with the degraded behaviour decided in advance rather than improvised.*

**Flow:** `Core capability` → `Optional enhancements` → `Dependency failure` → `Reduced feature set` → `Continued service`

> **The 30-second version**  
> Classify each dependency as required or optional, define what the user gets when each optional one fails, keep optional timeouts tight, and alert on degraded states so a hidden outage is not running quietly.

**The problem**

A product page calls seven services: catalogue, pricing, inventory, reviews, recommendations, personalisation and analytics. If any one of them fails and the page treats every call as required, the page fails. The probability of all seven being healthy simultaneously is lower than the availability of any individual one, so adding capabilities has quietly reduced reliability.

The mathematics are unforgiving. Seven dependencies each at 99.9 per cent give a combined availability of about 99.3 per cent — roughly five hours of downtime a month arising purely from composition, without any single service performing badly.

> **Treating optional dependencies as required is how availability is lost**  
> Most of those seven services are enhancements. A product page without recommendations is still a product page; without a price it is not. Classifying each dependency as required or optional, and designing an explicit degraded path for the optional ones, means the combined availability approaches that of the genuinely required subset rather than the product of all of them.

**Mental model**

The system has a core capability that defines what it is for, surrounded by enhancements that improve the experience. Failure should remove enhancements from the outside in, leaving the core intact for as long as possible.

1. **Core** — The minimum that makes the service meaningful — losing this is an outage.
2. **Enhancement** — Anything that improves but does not define the experience.
3. **Degraded path** — The predetermined behaviour when an enhancement is unavailable.
4. **Detection** — Recognising failure fast enough that degradation happens instead of waiting.
5. **Visibility** — Knowing the system is degraded, since silent degradation is an unreported outage.

> **Degradation that nobody notices is an outage running quietly**  
> The mechanism works by making failures invisible to users, which means it also makes them invisible to operators unless degraded states are explicitly instrumented and alerted. A service can run for days with recommendations entirely unavailable, looking perfectly healthy on every dashboard, because the degradation succeeded. Alerting on degraded mode is as important as implementing it.

**How it works**

**Classifying dependencies**

```text
FOR EACH DEPENDENCY, ANSWER TWO QUESTIONS
  1  is it required for the core capability?
  2  if not, what does the user get without it?

PRODUCT PAGE EXAMPLE
  catalogue        REQUIRED   no page without it
  pricing          REQUIRED   cannot sell without it
  inventory        OPTIONAL   show the product, omit
                              stock status
  reviews          OPTIONAL   omit the section
  recommendations  OPTIONAL   omit the carousel
  personalisation  OPTIONAL   show the generic version
  analytics        OPTIONAL   fire and forget

AVAILABILITY ARITHMETIC
  all 7 treated as required, each 99.9%
    -> 0.999^7 = 99.3%  (~5 h/month)
  2 required, 5 optional with degraded paths
    -> 0.999^2 = 99.8%  (~1.4 h/month)
  -> the same components, 3.5x less downtime,
     purely from classification

THE DEGRADED PATH MUST BE EXPLICIT
  not "the call failed so the field is null and the
  template throws"
  but "reviews unavailable -> render without the
  reviews section, log it, continue"
  -> the difference is whether someone decided
```

1. **Classify every dependency as required or optional** — Unclassified dependencies are treated as required by default, which is how the arithmetic goes wrong.
2. **Define the degraded behaviour for each optional one** — The value is in what the user gets instead, not in the failure handling itself.
3. **Detect failure fast** — A degraded path reached after a thirty-second timeout has already cost the user the page.
4. **Prefer stale data over no data** — A cached price from ten minutes ago is usually far better than an error, provided staleness is bounded and acknowledged.
5. **Degrade progressively, not all at once** — Shedding enhancements in priority order preserves more value than a single fallback for everything.
6. **Alert on degraded states** — Successful degradation hides a real outage, which can then persist unnoticed for days.

**Degradation strategies by situation**

```text
OMIT
  drop the feature entirely
  reviews section missing from the page
  -> best when the feature is visually separable

STALE
  serve the last known good value
  cached prices, cached catalogue entries
  -> bounded staleness is usually far better than
     an error
  -> but NEVER for data where staleness is unsafe
     (account balances, inventory at checkout)

DEFAULT
  substitute a generic value
  generic recommendations instead of personalised
  popular items instead of tailored ones
  -> the user gets something reasonable

DEFER
  accept now, process later
  queue the analytics event, queue the email
  -> works when the operation is not
     synchronously required

REDUCE
  offer a smaller version of the capability
  search without faceting, results without ranking
  -> partial capability beats none

REFUSE HONESTLY
  for required dependencies, fail clearly and fast
  -> an immediate honest error beats a hang
  -> and beats a page that half-renders and confuses
```

> **Stale data is only acceptable where staleness is safe**  
> Serving a cached price during an outage is usually right; serving a cached account balance or cached inventory at the point of sale is not, because the user acts on it and the action is wrong. The degraded path must be chosen per data type against what the user will do with it — and the cases where staleness causes real harm are exactly the ones where a clear failure is the correct answer.

**Worked example**

A product page with seven dependencies, each classified and given a behaviour.

**Degradation design**

```text
REQUIRED  (failure = honest error page)
  catalogue   product identity and description
  pricing     cannot transact without it
  -> 99.8% combined, and that is the page's ceiling

OPTIONAL  (failure = degrade, page still renders)
  inventory        -> omit stock badge
                      checkout re-checks authoritatively
  reviews          -> omit the section
  recommendations  -> omit the carousel
  personalisation  -> generic content
  analytics        -> fire and forget, never awaited

TIMEOUT BY CRITICALITY
  required    800 ms  (worth waiting for)
  optional    150 ms  (not worth delaying the page)
  -> optional calls must not be able to slow the core
  -> this is as important as the fallback itself

PROGRESSIVE SHEDDING UNDER LOAD
  load high    -> drop personalisation and
                  recommendations first
  load higher  -> drop reviews
  load extreme -> catalogue and pricing only
  -> the page survives at every level

OBSERVABILITY
  metric per dependency: degraded requests / total
  alert if any optional dependency is degraded for
    more than 5 minutes
  -> otherwise recommendations can be down for a week
     and nobody knows

OUTCOME
  the page's availability tracks its REQUIRED
  dependencies, not the product of all seven
```

| Metric | Value | Note |
|---|---|---|
| All required | 99.3% | ~5 h/month |
| Classified | 99.8% | **~1.4 h/month** |
| Optional timeout | 150 ms | cannot delay core |
| Alert | degraded > 5 min | no silent outage |

> **Timeouts for optional dependencies matter as much as the fallbacks**  
> A fallback that engages after a thirty-second timeout has already destroyed the page load. Optional dependencies need aggressive timeouts — a few hundred milliseconds — precisely because they are optional: waiting for something the user does not need is strictly worse than omitting it. Getting this wrong means a service that technically degrades gracefully while still being unusable.

**When to use it**

- **Composite pages and responses** built from several dependencies.
- **Systems with clear core and enhancement separation**, which is most user-facing products.
- **Under load**, where shedding enhancements preserves the core capability.
- **Where partial service has real value**, which covers most read paths.
- **High-dependency architectures**, where availability arithmetic makes composition the dominant risk.

**When to avoid it**

- **Do not degrade where correctness matters**, such as payment authorisation or inventory at checkout.
- **Do not serve stale data the user will act on**, where being wrong causes real harm.
- **Do not degrade silently**, which hides an outage from operators as well as users.
- **Do not degrade a genuinely required dependency**, where an honest error is the correct response.
- **Do not build degraded paths that are never exercised**, since they will be broken when needed.

**Advantages**

- **Availability approaches that of required dependencies** rather than the product of all of them.
- **Partial service instead of total failure**, which usually preserves most of the value.
- **Load shedding by capability**, protecting the core under pressure.
- **Bounded blast radius**, so one failing component affects one feature.
- **Better user experience** than an error page in almost every case.

**Disadvantages**

- **More code paths**, each needing implementation and testing.
- **Degraded paths rot** because they run rarely.
- **Hides problems** unless degraded states are instrumented and alerted.
- **Classification is product work**, requiring judgement about what matters.
- **Inconsistent experience**, which can confuse users who notice missing features.
- **Stale data carries risk** where the user acts on it.

**Trade-offs**

**Degradation strategy trade-offs**

| Strategy | User experience | Risk | Best for |
|---|---|---|---|
| Omit the feature | Slightly reduced | None | Visually separable enhancements |
| Serve stale data | Usually unnoticeable | Acting on old data | Slow-changing values |
| Generic default | Less relevant | None | Personalisation and ranking |
| Defer processing | Unchanged | Backlog if prolonged | Asynchronous side effects |
| Reduced capability | Noticeably simpler | None | Search, filtering, ranking |
| Honest failure | Clear error | None | Required dependencies |

The last row is not a lesser option. For a genuinely required dependency, an immediate clear failure is the correct design — better than a hang, and better than a half-rendered page that leaves the user unsure whether anything worked.

**How it fails**

**Degradation failures**

| Failure | Cause | Fix |
|---|---|---|
| One optional failure breaks the page | Dependency treated as required by default | Explicit classification with fallbacks |
| Page slow despite degrading | Optional dependency timeout too long | Aggressive timeouts on optional calls |
| Outage unnoticed for days | No alerting on degraded state | Metric and alert per dependency |
| Fallback path broken when needed | Never exercised | Test degraded paths; exercise them deliberately |
| User acted on stale data | Staleness served where it is unsafe | Per-type decision; fail rather than serve stale |
| Everything degrades at once | No priority ordering | Progressive shedding by enhancement priority |
| Confusing partial render | Degradation without user-visible acknowledgement | Clear, quiet indication where appropriate |

**Limits**

> **Availability arithmetic**
>
> - **Composition**: N required dependencies at availability a give a combined a^N — seven at 99.9% is 99.3%.
> - **Classification gain**: reducing required dependencies from seven to two takes that to 99.8%, roughly 3.5× less downtime.
> - **Optional timeouts**: a few hundred milliseconds, since waiting for the unnecessary is worse than omitting it.
> - **Degraded alerting**: minutes, not hours, because successful degradation hides the outage.
> - **Staleness bounds** must be chosen per data type against what the user will do with the value.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Graceful degradation | Composite services with enhancements | More paths to build and test |
| Fail fast entirely | Simple services; strict correctness | Total failure from partial problems |
| Retry until success | Transient failures | Latency; amplification under real outages |
| Redundant dependencies | Critical capabilities | Cost; still shares logical failures |
| Cached-everything read path | Read-heavy systems | Staleness; cache warming |
| Static fallback page | Total failure of dynamic systems | Very limited capability |

A static fallback is worth having as the last tier: when even required dependencies are unavailable, serving a cached or static version of the most important pages preserves some value and is far better than an error, particularly for content that changes slowly.

**In real systems**

- **Large product pages** are routinely assembled so that recommendations, reviews and personalisation failures remove sections rather than the page.
- **Aggressive timeouts on optional calls** are standard, because an enhancement that delays the core is worse than one that is absent.
- **Progressive feature shedding under load** protects core capability during traffic surges rather than degrading everything uniformly.
- **Degraded-mode metrics and alerts** exist because successful degradation otherwise conceals complete dependency outages for extended periods.
- **Static or cached fallback pages** serve as a final tier when even required dependencies are unavailable.

**Common mistakes**

- **Treating every dependency as required**, multiplying failure probabilities unnecessarily.
- **Long timeouts on optional calls**, so the page is slow even when degrading.
- **No alerting on degraded mode**, hiding complete outages.
- **Untested fallback paths**, broken when they finally matter.
- **Serving stale data where the user acts on it**, causing real harm.
- **Uniform degradation**, dropping valuable and trivial features together.
- **Half-rendered pages** that leave users unsure what happened.

**The staff-level view**

Graceful degradation is mostly a classification exercise, and the classification is where the availability is won.

- **Make dependency classification explicit and reviewed.** Anything unclassified is required by default, and that default is what turns seven healthy services into five hours of monthly downtime.
- **Set optional timeouts aggressively.** A fallback reached after thirty seconds has already lost the user, so the timeout matters as much as the fallback and is more often wrong.
- **Alert on degraded state.** The mechanism succeeds by making outages invisible, so without instrumentation a dependency can be entirely down for a week while every dashboard is green.
- **Exercise the degraded paths deliberately.** They run rarely, so they rot quietly and are broken precisely when first needed.
- **Decide staleness per data type against user action.** Stale prices are usually fine, stale balances and stale checkout inventory are not, and that distinction cannot be made globally.

**Go deeper**

When a composite response treats every dependency as required, failure probabilities multiply: seven services at 99.9 per cent give a combined 99.3 per cent, about five hours of monthly downtime from composition alone. Classifying most of them as enhancements with explicit degraded behaviour means availability tracks the genuinely required subset instead — the same components with several times less downtime, purely from the classification.

The degraded behaviour must be decided per dependency: omit the feature, serve bounded stale data, substitute a generic default, defer the work, or offer a reduced version. For genuinely required dependencies the right answer is an immediate honest failure, which beats both a hang and a half-rendered page. Staleness in particular must be judged against what the user will do with the value — stale prices are usually fine, stale balances and checkout inventory are not.

Two operational details decide whether it works. Optional dependencies need aggressive timeouts, because a fallback reached after thirty seconds has already lost the page, and this is more often wrong than the fallback itself. And degraded states must be instrumented and alerted, because the mechanism succeeds by making failure invisible — which conceals it from operators as effectively as from users.

Graceful degradation converts component failure into capability reduction, so that a service loses a feature rather than becoming unavailable.

**The availability arithmetic is the motivation.** Dependencies treated as required compose multiplicatively: seven at 99.9 per cent yield roughly 99.3 per cent, which is about five hours of downtime each month produced entirely by composition, with every individual service performing to target. Reclassifying five of them as optional with real degraded paths takes the combined figure to roughly 99.8 per cent. Nothing about the components changed — only the decision about which failures are permitted to propagate.

**Classification is product judgement, not engineering.** Deciding that reviews are optional and pricing is not requires knowing what the service exists to do, and engineers left to decide alone reasonably default to treating every dependency as required, which is precisely the choice that loses the availability. Making classification explicit, reviewed and documented is where most of the benefit is captured; the code that implements a fallback is comparatively trivial.

**The degraded behaviour must be chosen per dependency.** Omission suits visually separable features; bounded staleness suits slow-changing values; a generic default suits personalisation and ranking; deferral suits asynchronous side effects; a reduced capability suits search and filtering. Staleness in particular demands per-type judgement against what the user will do with the value — a cached price is almost always better than an error, while a cached account balance or cached inventory at the point of sale leads the user to act on something false, which is worse than an honest failure.

**Timeouts on optional calls matter as much as the fallbacks.** A fallback that engages only after a long timeout has already cost the user the experience it existed to protect, producing a service that degrades correctly and is still unusable. Because these dependencies are by definition not needed, waiting for them is strictly worse than omitting them, which argues for timeouts measured in a few hundred milliseconds. This is the detail most often left at a default and is a common reason degradation fails to deliver what was expected.

**Progressive shedding preserves more than uniform degradation.** Under load, dropping enhancements in priority order — personalisation and recommendations before reviews, reviews before anything core — keeps the most valuable capability available longest. Treating all optional dependencies identically discards valuable features alongside trivial ones at the same threshold, which wastes the ordering information the classification already produced.

**The mechanism hides failure from operators too, and the paths decay.** A degraded service looks healthier on every dashboard than an erroring one, so without per-dependency degraded-ratio metrics and alerts on sustained degradation, a component can be entirely unavailable for days unnoticed — the better-engineered system being the one where outages hide most effectively. Compounding this, degraded paths execute rarely, escape ordinary test coverage, and rot silently as surrounding code changes, so they are frequently broken at the moment they are first genuinely needed. Both problems are addressable through instrumentation and deliberate exercise, and both are outside the code that implements the fallback, which is why they are the parts most often skipped.

**Prove it — interview questions**

1. **[Basic] What is graceful degradation?**

   <details><summary>Model answer</summary>

   Designing so that losing a component removes a capability rather than the whole service. A product page whose recommendations service fails should render without the recommendations carousel rather than returning an error, because a page without recommendations is still a useful page. The essential work is deciding in advance which dependencies are genuinely required and what the user receives when each optional one is unavailable.

   </details>

2. **[Basic] Why does treating everything as required hurt availability?**

   <details><summary>Model answer</summary>

   Because failure probabilities multiply. Seven dependencies each at 99.9 per cent availability give a combined figure of about 99.3 per cent — roughly five hours of downtime a month arising purely from composition, with no individual service performing badly. Classifying five of those as optional with degraded paths means availability tracks the two required ones instead, around 99.8 per cent. Same components, same individual reliability, roughly three and a half times less downtime, purely from the classification.

   </details>

3. **[Senior] Why do timeouts matter as much as fallbacks?**

   <details><summary>Model answer</summary>

   Because a fallback that engages after thirty seconds has already destroyed the experience it was meant to protect. If an optional dependency is allowed to delay the response for that long, the page is effectively broken whether or not it eventually renders without the feature. Optional calls therefore need aggressive timeouts — a few hundred milliseconds — precisely because they are optional: waiting for something the user does not need is strictly worse than omitting it. A system that technically degrades gracefully while still being unusably slow is a common and frustrating outcome.

   </details>

4. **[Senior] Why must degraded states be alerted?**

   <details><summary>Model answer</summary>

   Because the mechanism succeeds by making failures invisible. When degradation works, users see a slightly reduced experience and no error, dashboards show healthy responses, and error rates stay flat — so a dependency can be entirely unavailable for days with nothing indicating it. Emitting a metric per dependency for the proportion of degraded requests, and alerting when any optional dependency stays degraded for more than a few minutes, is what keeps a successful degradation from becoming an unreported outage.

   </details>

5. **[Staff] Design degradation for a product page with seven backing services.**

   <details><summary>Model answer</summary>

   The first step is classification with the product, not with the code: catalogue and pricing are genuinely required, because there is no page without product identity and no transaction without a price, while inventory, reviews, recommendations, personalisation and analytics are all enhancements. That alone moves the page's availability ceiling from the product of seven services to the product of two. Each optional dependency then gets an explicit behaviour — omit the stock badge and let checkout re-check authoritatively, omit the reviews section, omit the carousel, serve generic content instead of personalised, fire analytics without awaiting it. Timeouts differ by criticality: required calls are worth waiting several hundred milliseconds for, optional calls get around a hundred and fifty, because an enhancement must never be able to slow the core. Under load I would shed progressively — personalisation and recommendations first, then reviews, leaving catalogue and pricing last — rather than degrading everything uniformly. And each dependency emits a degraded-request ratio with an alert if it stays degraded for more than a few minutes, because otherwise the entire mechanism quietly conceals a week-long outage behind a green dashboard.

   </details>

6. **[Principal] What makes graceful degradation hard in practice, given the idea is simple?**

   <details><summary>Model answer</summary>

   Three things, none of them technical. The first is that classification is product judgement rather than engineering: deciding that reviews are optional and pricing is not requires knowing what the service is for, and engineers left to decide alone will default to treating everything as required, which is exactly the choice that loses the availability. The second is decay — degraded paths execute rarely, so they are not covered by ordinary testing, they break silently as the surrounding code changes, and they are discovered broken at the moment they first matter. That argues for deliberately exercising them, whether through fault injection or scheduled drills, which teams rarely do. The third and most insidious is that the mechanism works by hiding failure, so it hides it from operators too: a service running degraded looks healthier than one erroring, which means the better-engineered system is the one where an outage can persist unnoticed. All three are addressable, but none of them are fixed by the code that implements the fallback — they need a classification review, a testing commitment and an alerting discipline, and those are the parts that get skipped.

   </details>

---
