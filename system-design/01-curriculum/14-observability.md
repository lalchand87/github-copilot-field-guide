# Curriculum · Observability

[← System Design index](../README.md)

> 8 lessons in **Observability**. Part of the Curriculum — Build the fundamentals.

## Contents

- **Observability** (8): [Metrics and golden signals](#metrics-and-golden-signals) · [Latency histograms and percentiles](#latency-histograms-and-percentiles) · [Structured logging](#structured-logging) · [Distributed tracing](#distributed-tracing) · [Telemetry sampling](#telemetry-sampling) · [Cardinality and label design](#cardinality-and-label-design) · [Actionable alerting and burn rates](#actionable-alerting-and-burn-rates) · [Profiling and resource attribution](#profiling-and-resource-attribution)

## Observability

### Metrics and golden signals

*Instrument a small set of user-facing signals — traffic, errors, latency and saturation — so that a dashboard answers whether the service is working, not merely what it is doing.*

**Flow:** `Traffic` → `Errors` → `Latency` → `Saturation` → `Alerting signal`

> **The 30-second version**  
> Measure traffic, errors, latency as a distribution split by outcome, and saturation on the real constraining resource — page on those symptoms, and keep everything else for explaining them.

**The problem**

A service has four hundred dashboards covering cache hit ratios, garbage collection pauses, connection pool sizes, thread counts and queue depths. During an incident, nobody can tell whether users are affected, because none of those metrics answers that question directly and the one that might is buried among the others.

The opposite failure is equally common: a service with CPU and memory graphs and nothing else, where a complete functional outage produces no visible change because the machines were never the problem.

> **A small set of user-facing signals beats a large set of internal ones**  
> Four measurements — how much work is arriving, how much of it fails, how long it takes, and how close resources are to their limits — answer the question that matters during an incident: is the service working for users, and if not, in what way. Internal metrics explain why once you know there is a problem; golden signals are what tell you there is one.

**Mental model**

Traffic tells you what is being asked of the system. Errors and latency tell you whether users are being served. Saturation tells you how much headroom remains before they stop being served.

1. **Traffic** — Demand — requests per second, messages consumed, connections opened.
2. **Errors** — The proportion of requests that failed, including failures the server considers successes.
3. **Latency** — How long requests take, as a distribution, separated by success and failure.
4. **Saturation** — How full the most constrained resource is — the leading indicator of trouble.
5. **Per-service, per-endpoint** — Signals aggregated across everything hide the failures that matter.

> **Averaged latency actively conceals the problem**  
> If ninety-nine per cent of requests take ten milliseconds and one per cent takes ten seconds, the mean is about a hundred milliseconds — a number that describes no actual request and looks healthy. The one per cent experiencing ten-second responses are precisely the users complaining. Latency must be recorded as a distribution and reported at high percentiles, because the average is mathematically guaranteed to hide the tail that constitutes the incident.

**How it works**

**The four signals and what each catches**

```text
TRAFFIC  - demand
  requests/s, by endpoint and status
  catches: surges, drops, shifts in mix
  a sudden DROP is often the clearest outage signal:
    traffic falling to zero usually means clients
    cannot reach you at all

ERRORS  - failure rate
  failed requests / total requests
  catches: functional breakage
  MUST include the failures the server does not count:
    timeouts, connection resets, 200 responses
    containing an error payload
  -> server-side status codes routinely miss most of
     what users experience as failure

LATENCY  - speed, as a DISTRIBUTION
  p50, p90, p99, p99.9 - never the mean
  separate SUCCESSFUL from FAILED requests:
    fast failures drag the distribution down and make
    a broken service look quick
  catches: degradation before it becomes failure

SATURATION  - headroom
  how full is the most constrained resource?
  thread pool utilisation, queue depth, connection
  pool usage, disk fill rate
  catches: trouble BEFORE users see it
  -> the only leading indicator of the four

RULE
  the first three describe user experience
  saturation predicts it
```

1. **Measure errors as users experience them** — Timeouts, resets and success responses containing error payloads are all failures, and server status codes miss them.
2. **Record latency as a histogram** — Percentiles cannot be computed from an average, and averages cannot be recovered into percentiles.
3. **Separate successful and failed latency** — A service failing instantly looks fast; mixing the two makes an outage appear as an improvement.
4. **Identify the real saturation point** — It is rarely CPU — thread pools, connection pools, queue depth and lock contention are more common limits.
5. **Aggregate by endpoint, not just by service** — One broken endpoint among fifty disappears entirely in a service-wide error rate.
6. **Alert on symptoms, instrument for causes** — Pages should fire on user impact; internal metrics exist to explain it once you are looking.

**Why percentiles and why the mean lies**

```text
REQUEST LATENCY, 10,000 REQUESTS
  9,900 requests at 10 ms
    100 requests at 10,000 ms

  mean:  ~110 ms     "looks fine"
  p50:    10 ms      "looks great"
  p99:    10,000 ms  "1% of users wait 10 seconds"

-> the mean describes no request that occurred
-> the p99 is the incident

WHY p99 IS NOT ENOUGH EITHER
  a page assembling 20 backend calls sees the p99 of
  at least one call with probability
    1 - 0.99^20 = 18%
  -> nearly one page in five hits a tail request
  -> so the tail is the typical page experience,
     not an edge case

SEPARATING SUCCESS FROM FAILURE
  dependency goes down; calls fail in 5 ms
  overall p99 latency IMPROVES
  -> the dashboard shows the service getting faster
     during a total outage
  -> this is why latency must be split by outcome

SATURATION IS THE LEADING SIGNAL
  thread pool at 60% -> fine
  thread pool at 85% -> latency starting to rise
  thread pool at 95% -> queueing, latency cliff
  -> saturation warns before errors and latency do
```

> **CPU utilisation is usually the wrong saturation metric**  
> Services far more often saturate on thread pools, connection pools, lock contention, queue depth or a downstream dependency's capacity than on CPU. A service can be completely saturated at forty per cent CPU because every thread is blocked on I/O, and perfectly healthy at ninety per cent because it is genuinely compute-bound. Identifying the actual constraining resource is the work; graphing CPU because it is available is not.

**Worked example**

An order service instrumented so that its dashboard answers the question that matters.

**Instrumentation plan**

```text
TRAFFIC
  orders_requests_total{endpoint, method, status}
  -> rate by endpoint
  -> alert on a sharp DROP as well as a spike;
     zero traffic is usually a worse signal than
     high traffic

ERRORS
  orders_requests_total{status=~"5.."} / total
  PLUS client-observed failures:
    timeouts, connection errors, and 200 responses
    whose body indicates failure
  -> the gap between server-observed and
     client-observed error rate is itself a useful
     metric

LATENCY
  orders_request_duration_seconds  (histogram)
  labels: endpoint, outcome
  -> p50/p90/p99/p99.9 per endpoint
  -> success and failure separated, always

SATURATION
  thread pool in use / capacity
  database connection pool utilisation
  inbound queue depth
  -> NOT CPU: this service is I/O bound and
     saturates on the database pool at ~45% CPU

ALERTING
  page:   error-budget burn rate on the checkout
          endpoint
  page:   p99 above SLO threshold, sustained
  ticket: saturation above 80% for 15 minutes
  -> no pages on internal metrics; they explain,
     they do not wake people

DASHBOARD, TOP TO BOTTOM
  1  is the service working?     errors, latency
  2  how much work is arriving?  traffic
  3  how much headroom is left?  saturation
  4  everything else             below the fold
```

| Metric | Value | Note |
|---|---|---|
| Latency | histogram | **never the mean** |
| Errors | client-observed too | status codes miss |
| Saturation | DB pool, not CPU | real constraint |
| Pages | symptoms only | causes explain |

> **Alert on symptoms; instrument everything else for explanation**  
> Paging on internal metrics produces alerts that fire when nothing is wrong and stay silent when everything is. Pages should fire on user-visible symptoms — errors and latency against the objective — because those are what actually matter and they catch causes nobody anticipated. The hundreds of internal metrics remain valuable, but their job is to explain a symptom once a human is already looking, not to decide when that human is woken.

**When to use it**

- **Every production service**, as the baseline instrumentation.
- **Incident response**, where a small set of user-facing signals gives orientation quickly.
- **Capacity planning**, where saturation trends predict when headroom runs out.
- **Defining objectives**, since error and latency signals are what service level indicators are built from.
- **Comparing services**, where uniform signals allow meaningful comparison.

**When to avoid it**

- **Do not rely on averaged latency**, which hides the tail that constitutes the incident.
- **Do not use server-side status codes alone** as the error signal, since they miss timeouts and resets.
- **Do not page on internal metrics**, which fire when nothing is wrong and miss what is.
- **Do not treat CPU as saturation** without confirming it is the constraining resource.
- **Do not aggregate only at service level**, which hides a single broken endpoint completely.

**Advantages**

- **Small, comparable set** applicable uniformly across services.
- **Directly reflects user experience** rather than internal mechanics.
- **Saturation gives warning** before users are affected.
- **Forms the basis of objectives**, since error and latency indicators come straight from it.
- **Reduces alert noise** by focusing pages on symptoms.
- **Fast orientation during incidents**, since four graphs tell you the shape of the problem.

**Disadvantages**

- **Says what is wrong, not why**, so deeper instrumentation is still needed.
- **Histograms cost more** in storage and cardinality than simple counters.
- **Client-side error measurement is harder** than scraping server metrics.
- **Identifying the true saturation point requires understanding the system**, not just collecting numbers.
- **Per-endpoint aggregation multiplies series**, which has cardinality consequences.

**Trade-offs**

**Metric choice trade-offs**

| Choice | Reveals | Hides | Verdict |
|---|---|---|---|
| Mean latency | Nothing useful | The entire tail | Never sufficient |
| p50 | Typical experience | Tail entirely | Useful alongside others |
| p99 | Tail experience | The worst 1% | The standard working signal |
| p99.9 | Severe tail | Noisy at low volume | For high-traffic critical paths |
| Server status codes | Application failures | Timeouts, resets, soft errors | Insufficient alone |
| Client-observed errors | Actual user failure | Requires client instrumentation | The honest signal |

The gap between server-observed and client-observed error rates is itself worth tracking: it quantifies how much user-visible failure the server-side view is missing, and a widening gap often indicates network, proxy or timeout problems that no server metric would reveal.

**How it fails**

**Metrics failures**

| Failure | Cause | Fix |
|---|---|---|
| Dashboards healthy during an outage | Server-side metrics only; no client view | Measure errors and latency as users experience them |
| Latency looks fine, users complain | Averaged rather than distributed | Histograms with high percentiles |
| Service appears faster during an outage | Fast failures mixed into latency | Separate latency by outcome |
| Saturation alert never fires | Watching CPU when the limit is elsewhere | Identify the real constraining resource |
| One broken endpoint invisible | Only service-level aggregation | Per-endpoint labels |
| Alert fatigue | Paging on internal metrics | Page on symptoms; use causes for diagnosis |
| Outage detected by customers first | No alert on traffic dropping to zero | Alert on absence of traffic too |

**Limits**

> **Practical guidance**
>
> - **Percentiles**: p50, p90, p99 as standard; p99.9 only where volume makes it meaningful.
> - **Tail exposure**: a page making 20 backend calls hits at least one p99 request about 18% of the time.
> - **Saturation thresholds**: warn around 80%, since latency typically degrades well before 100%.
> - **Histogram cost**: more series per metric than counters, so bucket choice and label cardinality matter.
> - **Server versus client error gap**: track it explicitly; it quantifies what server metrics miss.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Golden signals | Universal baseline | Explains what, not why |
| RED method (rate, errors, duration) | Request-driven services | No saturation signal |
| USE method (utilisation, saturation, errors) | Resources and infrastructure | Not user-facing |
| Detailed internal metrics | Diagnosis | Poor for alerting; high volume |
| Distributed tracing | Per-request causal detail | Sampling; higher cost |
| Logs only | Detail and ad-hoc queries | Expensive to aggregate; poor for trends |

The request-oriented and resource-oriented methods are complementary views of the same system: one describes what users are getting, the other describes what the machinery is doing. Golden signals essentially combine them, which is why they work as a single baseline.

**In real systems**

- **Latency histograms rather than averages** are standard in modern monitoring precisely because averages hide the tail that constitutes user-visible incidents.
- **Client-side instrumentation** consistently reveals failure rates higher than server metrics show, since timeouts and network errors never reach the server's counters.
- **Symptom-based alerting** has largely replaced cause-based alerting, because pages on internal metrics produce noise and miss unanticipated failures.
- **Saturation on thread and connection pools** is the usual real constraint for service workloads, not CPU.
- **Alerting on traffic dropping to zero** catches outages where clients cannot reach the service at all, which no error-rate metric would show.

**Common mistakes**

- **Averaged latency**, hiding the tail entirely.
- **Latency not split by outcome**, so outages look like speed improvements.
- **Server-side error rates only**, missing timeouts and resets.
- **CPU as the saturation signal** when the constraint is elsewhere.
- **Service-level aggregation only**, hiding a single broken endpoint.
- **Paging on internal metrics**, producing noise and blind spots.
- **No alert on traffic disappearing**, missing total-connectivity outages.

**The staff-level view**

Most observability problems are not missing data but the wrong four graphs at the top of the dashboard.

- **Insist on distributions for latency.** An average is mathematically guaranteed to conceal the tail, and the tail is what users report — this is the single most common instrumentation defect.
- **Split latency by outcome.** A service whose dependency has failed returns errors in milliseconds and therefore appears to get faster during a total outage, which has misled more than one incident response.
- **Measure errors where the user is.** Server status codes miss timeouts, connection resets and success responses carrying error payloads, and the gap between server and client views is worth tracking as its own signal.
- **Find the actual saturation point.** Graphing CPU because it is available produces a metric that stays flat while the service saturates on a connection pool at forty per cent CPU.
- **Page on symptoms only.** Cause-based alerts fire when nothing is wrong and stay silent for failures nobody predicted; internal metrics belong in diagnosis, not in the paging path.

**Go deeper**

Four signals answer whether a service is working: traffic shows demand, errors show what proportion fails, latency shows how long it takes, and saturation shows how much headroom remains. The first three describe current user experience while saturation is the only leading indicator, rising before errors and latency do. Internal metrics explain problems; these detect them.

Two instrumentation details matter disproportionately. Latency must be recorded as a histogram and reported at high percentiles, because an average is mathematically certain to conceal the tail that constitutes the incident — and it must be split by outcome, since fast failures otherwise make a service appear to get quicker during a total outage. Errors must be measured as users experience them, because server status codes miss timeouts, connection resets and success responses carrying error payloads.

Saturation should be measured on whatever actually constrains the service, which is usually a thread pool, connection pool or queue rather than CPU — a service can be fully saturated at forty per cent CPU with every thread blocked on I/O. And alerting should page on symptoms only: cause-based alerts fire when nothing is wrong and stay silent for failures nobody anticipated, while symptom-based pages catch novel causes by construction.

Golden signals provide a small, uniform instrumentation baseline that answers whether a service is serving its users, as distinct from what its internals are doing.

**Four measurements, one leading.** Traffic characterises demand and its shape; errors and latency describe what users are currently receiving; saturation describes how close the system is to being unable to serve them. Saturation is the only one that warns in advance, which makes it the signal worth watching between incidents, while the other three are what orient responders during one. Everything else in a monitoring system explains these rather than replacing them.

**Averages are not an approximation of the tail, they are its opposite.** A distribution where nearly all requests complete in ten milliseconds and one per cent take ten seconds produces a mean of about a hundred milliseconds — a value no request exhibited, comfortably below any threshold, and describing precisely nobody. The affected one per cent are the users filing complaints. Because percentiles cannot be recovered from an average after collection, the decision to record histograms rather than means is made once at instrumentation time and determines whether the tail is ever visible.

**The tail is more common than it sounds.** A page assembling twenty backend calls encounters at least one p99 request roughly eighteen per cent of the time, so what appears as a one-in-a-hundred event at the service level is nearly a one-in-five event at the page level. This is why high percentiles rather than medians are the working signal for composite systems, and why improving tail latency often improves perceived performance far more than improving the median.

**Latency must be split by outcome or it inverts during outages.** Failures are frequently fast — a dependency that is down returns errors in milliseconds — so a combined distribution improves as the service breaks. Dashboards then show latency falling during a total outage, which has misled real incident responses. Separating successful from failed requests keeps the successful distribution meaningful as a description of what served users experienced.

**Errors must be counted where the user is.** Server-side status codes systematically miss the failures that matter most: client timeouts where no response was ever received, connection resets, DNS failures, and success responses whose payload indicates an error. A service can report a 99.99 per cent success rate while a material share of users cannot complete their task. Instrumenting the client view, and tracking the gap between server-observed and client-observed error rates as its own metric, quantifies exactly how much the server-side picture is missing — and a widening gap is itself diagnostic of network, proxy or timeout problems.

**Alert on symptoms, instrument for causes.** Cause-based alerting — paging on cache hit ratio, garbage collection time, queue depth — fires when nothing is user-visible and stays silent for failure modes nobody anticipated, producing fatigue and blind spots simultaneously. Symptom-based pages on error rate and latency against the objective catch novel causes by construction, because any cause that matters manifests as user impact. The large body of internal metrics remains valuable and should be retained, but its role is to explain a symptom once a human is already investigating, not to determine when that human is woken — and getting that hierarchy right matters more than collecting anything additional.

**Prove it — interview questions**

1. **[Basic] What are the golden signals?**

   <details><summary>Model answer</summary>

   Traffic, errors, latency and saturation. Traffic is how much work is arriving, errors is what proportion of it fails, latency is how long it takes as a distribution, and saturation is how full the most constrained resource is. The first three describe what users are experiencing right now; saturation is the only leading indicator, warning that trouble is coming before it becomes visible. Together they answer whether the service is working, which is the question that matters during an incident.

   </details>

2. **[Basic] Why is average latency misleading?**

   <details><summary>Model answer</summary>

   Because it describes no actual request. If ninety-nine per cent of requests take ten milliseconds and one per cent takes ten seconds, the mean is about a hundred milliseconds — a comfortable-looking number, while one user in a hundred waits ten seconds and complains. The mean is mathematically guaranteed to hide the tail, and the tail is the incident. Latency has to be recorded as a histogram so that high percentiles can be reported, and percentiles cannot be recovered from an average after the fact.

   </details>

3. **[Senior] Why separate latency by success and failure?**

   <details><summary>Model answer</summary>

   Because failures are often fast. When a dependency goes down and calls start failing in five milliseconds, the combined latency distribution improves — so the dashboard shows the service getting faster during a total outage, which is precisely backwards and has genuinely misled incident responders. Splitting the histogram by outcome means successful-request latency reflects the experience of users who are being served, and failure latency is tracked separately where it belongs.

   </details>

4. **[Senior] Why is CPU usually the wrong saturation metric?**

   <details><summary>Model answer</summary>

   Because services rarely saturate on CPU. A request-handling service is typically bound by its thread pool, its database connection pool, queue depth, lock contention, or a downstream dependency's capacity — so it can be completely saturated at forty per cent CPU with every thread blocked on I/O, and perfectly healthy at ninety per cent when genuinely compute-bound. Graphing CPU because it is easy to collect produces a metric that stays flat through the saturation that matters. The work is identifying the actual constraining resource for that specific service.

   </details>

5. **[Staff] How would you instrument a new service from scratch?**

   <details><summary>Model answer</summary>

   Four signals first, everything else afterwards. Traffic as a counter labelled by endpoint, method and status, with an alert on a sharp drop as well as a spike — traffic falling to zero is often the clearest possible outage signal and no error-rate metric would show it. Errors measured as users experience them, which means server status codes plus client-observed timeouts, connection resets and success responses carrying error payloads; I would also track the gap between the server and client views, because a widening gap points at network or timeout problems invisible from the server. Latency as a histogram labelled by endpoint and outcome, reported at p50, p90 and p99, never as an average and never mixing successes with failures. Saturation on whatever the real constraint is, which for a typical I/O-bound service is the database connection pool rather than CPU. Alerting pages only on symptoms — error-budget burn and sustained p99 against objective — with saturation raising a ticket at eighty per cent. Everything else gets instrumented for diagnosis and lives below the fold on the dashboard, because its job is to explain a symptom once someone is already looking, not to decide when they get woken.

   </details>

6. **[Principal] Why do organisations end up with hundreds of dashboards and poor visibility?**

   <details><summary>Model answer</summary>

   Because instrumentation grows by accretion during incidents while nobody removes anything. Each postmortem adds the metric that would have helped that time, so the collection grows monotonically and its signal-to-noise ratio falls, until during the next incident the useful graph exists but cannot be found among four hundred others. The deeper issue is that most of what accumulates measures internal mechanics rather than user experience — cache hit ratios and garbage collection pauses explain a problem beautifully once you know there is one, but none of them tell you there is one. So the corrective is structural rather than additive: put a small number of user-facing signals at the top of every service dashboard, page exclusively on those, and treat everything else explicitly as diagnostic material reached after a symptom has fired. That ordering also fixes the alerting problem, because cause-based alerts fire when nothing is wrong and stay silent for the failures nobody anticipated, whereas a symptom-based page catches novel causes by construction. The hundreds of metrics are not the problem; their position in the hierarchy is.

   </details>

---

### Latency histograms and percentiles

*Record latency as bucketed distributions so tail behaviour is visible and aggregatable, and understand why percentiles cannot be averaged.*

**Flow:** `Bucketed observations` → `Cumulative counts` → `Percentile estimate` → `Aggregation across instances` → `Tail visibility`

> **The 30-second version**  
> Record latency as bucketed counts so distributions merge exactly and any percentile can be computed later — never average percentiles, and put a bucket boundary on the threshold that matters.

**The problem**

A service reports p99 latency per instance. There are twenty instances. Someone builds a fleet-wide graph by averaging the twenty p99 values, and the resulting number is wrong in a way that is difficult to detect — it is not the fleet's p99, it is not any meaningful quantity, and it can move in the opposite direction to the real figure.

A related failure: the team wants to know the p99.9 for the checkout endpoint over the last week, but only pre-computed p50, p95 and p99 were stored. The raw data is gone, and no post-hoc computation can recover what was never recorded.

> **Store distributions, compute percentiles at query time**  
> A histogram records counts per latency bucket, and those counts are additive: summing them across instances or time gives the distribution of the combined population, from which any percentile can be computed. Pre-computed percentiles are not additive, so a system storing only p99 values has thrown away the ability to answer almost every question it will later be asked.

**Mental model**

Rather than storing every measurement or a summary statistic, count how many observations fell into each latency range. Those counts merge cleanly, and percentiles are estimated from where the cumulative count crosses the target fraction.

1. **Buckets** — Latency ranges, usually with exponentially increasing widths.
2. **Counts** — How many observations fell into each bucket — the stored data.
3. **Additivity** — Bucket counts sum across instances and time; percentiles do not.
4. **Estimation** — A percentile is found by locating where the cumulative count crosses the target fraction.
5. **Bucket resolution** — Determines how precise that estimate can be.

> **Averaging percentiles produces a number with no meaning**  
> The mean of twenty instances' p99 values is not the fleet p99 and does not approximate it. If nineteen instances are fast and one is catastrophically slow, the fleet p99 may be dominated by that instance while the average of the p99s barely moves. The operation is not merely imprecise — it computes a quantity that corresponds to nothing, and it can trend in the opposite direction to the truth.

**How it works**

**How a histogram stores and answers**

```text
BUCKETS (cumulative, exponentially spaced)
  le=0.005   count 4,200
  le=0.010   count 8,100
  le=0.025   count 9,400
  le=0.050   count 9,700
  le=0.100   count 9,850
  le=0.250   count 9,930
  le=0.500   count 9,970
  le=1.000   count 9,990
  le=+Inf    count 10,000

ESTIMATING p99
  target = 0.99 x 10,000 = 9,900 observations
  9,850 are <= 100 ms
  9,930 are <= 250 ms
  -> p99 lies between 100 ms and 250 ms
  -> interpolate within the bucket for an estimate
  -> RESOLUTION IS BOUNDED BY BUCKET WIDTH

WHY ADDITIVITY MATTERS
  instance A: le=0.100 -> 9,850
  instance B: le=0.100 -> 9,100
  fleet:      le=0.100 -> 18,950
  -> exact, for any number of instances
  -> then compute the fleet p99 from the merged counts

CONTRAST WITH PRE-COMPUTED PERCENTILES
  instance A p99 = 120 ms
  instance B p99 = 900 ms
  average       = 510 ms
  actual fleet p99 = ? UNKNOWABLE from these numbers
  -> could be 850 ms if B serves most traffic
  -> could be 130 ms if B serves almost none
```

1. **Choose buckets covering the plausible range** — Everything above the largest bucket collapses into one bin, making the extreme tail unmeasurable.
2. **Use exponential spacing** — Latency spans orders of magnitude, so linear buckets waste resolution where it is not needed and lack it where it is.
3. **Place bucket boundaries near thresholds that matter** — If the objective is 500 ms, a boundary exactly there makes the compliance ratio exact rather than interpolated.
4. **Never average percentiles** — Merge the histograms and compute the percentile from the merged counts.
5. **Mind the series cost** — Each bucket is a separate time series per label combination, so buckets multiply cardinality.
6. **Prefer histograms over summaries for aggregation** — Client-computed quantiles cannot be merged across instances, which is usually what you need.

**Bucket design in practice**

```text
BAD: LINEAR BUCKETS
  0-100ms, 100-200ms, ..., 900ms-1s
  -> 10 buckets, all resolution in a range where
     almost nothing happens
  -> p50 at 8 ms is indistinguishable from p50 at
     95 ms: both land in the first bucket
  -> the tail beyond 1 s is one undifferentiated bin

GOOD: EXPONENTIAL BUCKETS
  5ms, 10, 25, 50, 100, 250, 500, 1s, 2.5s, 5s, 10s
  -> fine resolution where typical requests land
  -> coverage across three orders of magnitude
  -> the tail is still differentiated

ALIGN A BUCKET WITH THE OBJECTIVE
  SLO: 99% of requests under 300 ms
  -> include a boundary at exactly 300 ms
  -> the count at that boundary divided by the total
     IS the compliance ratio, with no estimation error
  -> this is the cheapest accuracy improvement
     available

CARDINALITY ARITHMETIC
  11 buckets x 20 endpoints x 5 status codes
    = 1,100 series for ONE metric
  add instance and region labels and this explodes
  -> bucket count and label cardinality multiply
  -> this is the main cost of histograms

RESOLUTION LIMIT
  p99 estimated between 250 ms and 500 ms buckets
  -> interpolation assumes uniform distribution within
     the bucket, which is rarely true
  -> report it as an estimate, not a precise value
```

> **The highest bucket determines whether the extreme tail is visible**  
> Observations above the largest bucket boundary land in an unbounded overflow bin, so if the top bucket is one second and some requests take thirty, the histogram cannot distinguish a one-second request from a thirty-second one. Any percentile falling in that region is unbounded above. Since severe tail behaviour is usually the reason for looking, bucket ranges must extend well past normal operation into the territory that only occurs during incidents.

**Worked example**

Instrumenting an API where the objective is 99 per cent of requests under 300 milliseconds.

**Histogram configuration**

```text
BUCKET BOUNDARIES (seconds)
  0.005, 0.010, 0.025, 0.050, 0.100, 0.200,
  0.300,   <- exactly the SLO threshold
  0.500, 1.0, 2.5, 5.0, 10.0

WHY 0.300 IS EXPLICIT
  compliance = count(le=0.300) / count(le=+Inf)
  -> exact, no interpolation
  -> the SLO reading is not an estimate

WHY THE RANGE EXTENDS TO 10 s
  normal p99 is ~250 ms
  but incidents produce multi-second latency
  -> without high buckets, an incident shows
     "p99 > 1 s" and nothing more
  -> with them, the shape of the degradation is
     visible

LABELS  endpoint, outcome
  deliberately NOT instance:
    the histogram is aggregated server-side and
    per-instance detail is rarely worth the
    cardinality
  deliberately NOT user or path parameters:
    unbounded cardinality would break the system

SERIES COST
  12 buckets x 15 endpoints x 2 outcomes = 360 series
  acceptable

QUERYING
  fleet p99:  merge all instances' bucket counts,
              then compute
  NEVER:      average each instance's p99
```

| Metric | Value | Note |
|---|---|---|
| Buckets | exponential | 3 orders of magnitude |
| SLO boundary | exact 300 ms | **no interpolation** |
| Top bucket | 10 s | incident visibility |
| Series | 360 | cardinality controlled |

> **Aligning a bucket boundary with the objective makes compliance exact**  
> If the objective is that ninety-nine per cent of requests complete under three hundred milliseconds and a bucket boundary sits exactly there, then the compliance ratio is simply that bucket's count divided by the total — no interpolation, no estimation error. It costs one bucket and removes the main source of inaccuracy from the number the entire objective depends on, which makes it the cheapest accuracy improvement available in any histogram configuration.

**When to use it**

- **Any latency measurement**, since averages hide the behaviour that matters.
- **Multi-instance services**, where only additive histograms aggregate correctly.
- **Service level indicators**, where the compliance ratio is a count over a threshold.
- **Any distribution**, including payload sizes, queue times and batch sizes.
- **Long-term retention**, where raw measurements are impractical but distributions compress well.

**When to avoid it**

- **Do not average percentiles across instances or time**, which computes a meaningless quantity.
- **Do not use linear buckets** for latency, which spans orders of magnitude.
- **Do not set the top bucket near normal operation**, which makes incident tails invisible.
- **Do not add high-cardinality labels**, since buckets multiply every label combination.
- **Do not use client-computed quantile summaries** where cross-instance aggregation is needed.

**Advantages**

- **Additive across instances and time**, so aggregation is exact.
- **Any percentile computable at query time**, not just those decided in advance.
- **Compact**, storing counts rather than individual observations.
- **Threshold ratios are exact** when a boundary is aligned with the objective.
- **Shows distribution shape**, revealing multimodality that percentiles alone hide.

**Disadvantages**

- **Percentiles are estimates**, bounded by bucket resolution.
- **Bucket boundaries must be chosen in advance** and are awkward to change later.
- **Series count multiplies** with buckets times label cardinality.
- **The overflow bucket is unbounded**, so the extreme tail is uncharacterised.
- **Interpolation assumes uniformity** within buckets, which is rarely accurate.

**Trade-offs**

**Representation trade-offs**

| Approach | Aggregatable | Precision | Cost |
|---|---|---|---|
| Average | Yes | Useless for tails | Minimal |
| Client-computed summary | No | High per instance | Low |
| Histogram | Yes | Bounded by buckets | Buckets × labels |
| Raw observations | Yes | Exact | Prohibitive at scale |
| Sketch structures | Yes | High, bounded error | Moderate; less tooling support |

Sketch-based structures give relative-error guarantees across the whole range with far better tail precision than fixed buckets, at the cost of narrower ecosystem support. Where extreme tail accuracy genuinely matters, they are worth knowing about as the alternative to adding ever more buckets.

**How it fails**

**Histogram and percentile failures**

| Failure | Cause | Fix |
|---|---|---|
| Fleet percentile wrong or misleading | Averaging per-instance percentiles | Merge bucket counts, then compute |
| Cannot answer a new percentile question | Only pre-computed percentiles stored | Store histograms, compute at query time |
| Tail invisible during incidents | Top bucket too low | Extend bucket range well past normal operation |
| Percentile estimate inaccurate | Buckets too coarse in the relevant range | Add boundaries where the values land |
| Metrics system overloaded | Buckets multiplied by high-cardinality labels | Reduce labels; fewer buckets |
| SLO compliance figure disputed | Interpolated across a wide bucket | Align a boundary with the threshold |
| Bimodal behaviour hidden | Only percentiles examined | Inspect the full bucket distribution |

**Limits**

> **Design guidance**
>
> - **Bucket count**: 10–15 exponentially spaced boundaries typically covers three orders of magnitude adequately.
> - **Series cost**: buckets × endpoints × other labels — the dominant cost of histogram instrumentation.
> - **Resolution**: percentile precision is bounded by the width of the bucket it falls in.
> - **Top bucket**: should exceed worst observed incident latency, or the tail is uncharacterised.
> - **Threshold alignment**: one boundary at the objective makes compliance exact rather than interpolated.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Fixed-bucket histograms | General latency measurement | Resolution bounded by buckets |
| Sketch structures | High tail accuracy | Less tooling support |
| Client-computed summaries | Single-instance precision | Cannot aggregate |
| Raw traces | Individual request detail | Sampling; storage cost |
| Exemplars with histograms | Linking a bucket to example traces | Requires tracing integration |
| Averages | Nothing involving tails | Hides the incident |

Exemplars are a useful addition rather than an alternative: attaching a sample trace identifier to histogram buckets lets you jump from noticing that the p99 bucket is populated straight to an actual slow request, which closes the gap between aggregate metrics and per-request diagnosis.

**In real systems**

- **Metric systems expose histograms with cumulative buckets** precisely because bucket counts aggregate correctly across instances while quantiles do not.
- **Bucket boundaries aligned with service level thresholds** turn compliance measurement into an exact ratio rather than an interpolation.
- **Sketch-based percentile structures** are used where fixed buckets give insufficient tail accuracy, offering bounded relative error across the range.
- **Exemplars linking buckets to traces** connect aggregate tail behaviour to specific slow requests for diagnosis.
- **Label cardinality limits on histograms** are enforced in practice because buckets multiply every label combination.

**Common mistakes**

- **Averaging percentiles** across instances or time windows.
- **Storing only pre-computed percentiles**, foreclosing every later question.
- **Linear buckets** for a quantity spanning orders of magnitude.
- **Top bucket too low**, making incident tails uncharacterised.
- **No boundary at the objective threshold**, leaving compliance interpolated.
- **High-cardinality labels on histograms**, multiplying series uncontrollably.
- **Reading percentiles only**, missing bimodal distributions entirely.

**The staff-level view**

Percentile handling is one of the few observability topics where a common practice is simply mathematically invalid.

- **Eliminate averaged percentiles wherever they appear.** The operation produces a quantity corresponding to nothing and can move opposite to the truth, which makes dashboards built on it actively misleading during incidents.
- **Align one bucket boundary with each latency objective.** It converts the compliance figure from an interpolation into an exact ratio at the cost of a single bucket, and it is the number the whole objective rests on.
- **Extend bucket ranges into incident territory.** Buckets sized for healthy operation report only that the tail exceeded the top boundary, which is exactly when detail is most needed.
- **Budget cardinality deliberately.** Buckets multiply every label combination, so histograms are where metric systems are most often overwhelmed, and the discipline has to be applied at instrumentation time.
- **Look at the distribution, not only the percentiles.** Bimodal behaviour — two distinct populations of fast and slow requests — is visible in bucket counts and invisible in any single percentile.

**Go deeper**

A histogram stores how many observations fell into each latency bucket. Those counts are additive, so merging them across instances or time gives the exact distribution of the combined population, from which any percentile can be computed at query time. Pre-computed percentiles have neither property: they cannot be merged, and they foreclose every question that was not anticipated when the metric was defined.

Averaging percentiles across instances is not an approximation — it computes a quantity that corresponds to nothing and can move in the opposite direction to the true fleet percentile. If one instance of twenty is severely degraded, the average of the p99 values barely shifts while the actual fleet p99 may be dominated by it. The correct operation is always to merge bucket counts first.

Bucket design carries two deliberate choices. A boundary aligned exactly with a latency objective makes the compliance ratio a precise count rather than an interpolation, which matters because that number is what the objective rests on. And the top bucket must extend well past normal operation into incident territory, or a degraded service reports only that the tail exceeded the highest boundary. The constraint is cardinality, since every bucket becomes a series for each label combination.

Histograms record latency as counts per bucket, which is what makes distributions aggregatable and tail behaviour recoverable.

**Additivity is the defining property.** Bucket counts from different instances or time windows sum to give the exact distribution of the combined population, so a fleet percentile is obtained by merging first and computing second. Percentiles themselves have no such property: they are positions within a distribution, and combining them requires knowledge of the underlying distributions that the percentile values do not carry. This is why storing summary quantiles forecloses the aggregate questions that are usually the ones being asked.

**Averaging percentiles is invalid, not imprecise.** With nineteen instances at a p99 of a hundred milliseconds and one at nine hundred, the mean is a hundred and forty, while the true fleet p99 could be near either extreme depending on traffic distribution. The computed value corresponds to no quantity, and it can trend opposite to the real figure — a dashboard built on it will show improvement during a degradation affecting a subset of instances. Since the operation is easy to perform and the result looks plausible, this remains one of the most common genuine errors in production monitoring.

**Resolution is bounded by bucket width.** A percentile falling between the two-hundred-and-fifty and five-hundred-millisecond boundaries is estimated by interpolating within that range, which assumes a uniform distribution inside the bucket — rarely true. The estimate should be treated as such. Adding boundaries where values actually land improves precision directly, and the highest-value placement is a boundary exactly at any latency objective, because compliance then becomes a count divided by a total with no estimation involved.

**The top bucket determines whether incidents are legible.** Observations above the largest boundary fall into an unbounded overflow bin, so a histogram topping out at one second cannot distinguish a one-second request from a thirty-second one. Since the reason to examine latency closely is usually that something is wrong, buckets sized only for healthy operation fail precisely when needed — reporting that the tail exceeded the maximum and nothing more about its shape or trajectory.

**Cardinality is the real cost.** Every bucket is a distinct time series for each combination of labels, so a dozen buckets across fifteen endpoints and two outcomes is already several hundred series for a single metric, and adding instance, region or any user-derived label multiplies that further. Histograms are consequently where metric systems are most often overwhelmed, and since bucket boundaries and label sets are decided at instrumentation time and awkward to change afterwards, the discipline has to be applied up front.

**The distribution shows what percentiles cannot.** A p50 of ten milliseconds alongside a p99 of two seconds is consistent both with a smooth long-tailed distribution and with two separate populations — cache hits and misses, for instance — which have entirely different causes and remedies. Bucket counts reveal that bimodality immediately while no combination of percentiles distinguishes it, so investigations that begin from percentiles alone frequently start with the wrong model of the system. Reading the full distribution when investigating, and reserving percentiles for alerting and reporting where a single figure is required, uses each for what it is actually good at.

**Prove it — interview questions**

1. **[Basic] Why store histograms rather than percentiles?**

   <details><summary>Model answer</summary>

   Because bucket counts are additive and percentiles are not. Summing bucket counts across instances or time windows gives the exact distribution of the combined population, from which any percentile can be computed at query time. Storing pre-computed percentiles means you can only ever answer the questions you anticipated — if p95 and p99 were stored, p99.9 is unrecoverable — and it means fleet-wide figures cannot be derived at all, because there is no valid way to combine per-instance percentiles.

   </details>

2. **[Basic] How is a percentile estimated from a histogram?**

   <details><summary>Model answer</summary>

   By finding where the cumulative count crosses the target fraction. For a p99 over ten thousand observations, you look for the point at which nine thousand nine hundred observations have accumulated; if nine thousand eight hundred and fifty fall below one hundred milliseconds and nine thousand nine hundred and thirty fall below two hundred and fifty, the p99 lies somewhere between those boundaries and is interpolated within that bucket. The precision is therefore bounded by bucket width, which is why bucket placement matters.

   </details>

3. **[Senior] Why can percentiles not be averaged?**

   <details><summary>Model answer</summary>

   Because a percentile is a position in a distribution, not a quantity that combines linearly. If nineteen instances report a p99 of one hundred milliseconds and one reports nine hundred, their average is one hundred and forty — but the true fleet p99 depends entirely on how traffic is distributed across those instances, and could plausibly be anywhere from just above one hundred to near nine hundred. The averaged number corresponds to nothing real, and it can trend in the opposite direction to the actual fleet percentile, which makes it worse than no metric. The correct operation is to merge the bucket counts and compute the percentile from the combined distribution.

   </details>

4. **[Senior] How should bucket boundaries be chosen?**

   <details><summary>Model answer</summary>

   Exponentially spaced, covering roughly three orders of magnitude, with two deliberate placements. First, a boundary exactly at any latency objective threshold, because then the compliance ratio is that bucket's count over the total — exact rather than interpolated, which matters when the number is what an objective is measured against. Second, a top bucket well beyond normal operation and into the range that occurs during incidents, because otherwise a degraded service reports only that the tail exceeded the highest boundary, with no information about whether that means one second or thirty. The constraint pulling the other way is cardinality: every bucket is a separate series for each label combination.

   </details>

5. **[Staff] Design latency instrumentation for an API with a 300 millisecond objective.**

   <details><summary>Model answer</summary>

   Exponential buckets from five milliseconds up to ten seconds, with an explicit boundary at exactly three hundred milliseconds. That boundary is the important one: the compliance figure becomes the count below it divided by the total, with no interpolation error, and since the entire objective rests on that number it is worth a dedicated bucket. The range extending to ten seconds is the other deliberate choice — normal p99 might be two hundred and fifty milliseconds, but incidents produce multi-second latency, and a histogram topping out at one second would tell me only that things were bad without telling me how bad or whether they were improving. Labels are endpoint and outcome, splitting success from failure so that fast failures cannot make the service look quicker during an outage, and deliberately excluding instance identifiers and any path parameters, because buckets multiply every label combination and that is where metric systems get overwhelmed. Twelve buckets across fifteen endpoints and two outcomes is a few hundred series, which is comfortable. And fleet percentiles are computed by merging bucket counts across instances, never by averaging per-instance percentiles.

   </details>

6. **[Principal] What does a histogram show that percentiles do not?**

   <details><summary>Model answer</summary>

   The shape of the distribution, which is frequently where the explanation lives. A p50 of ten milliseconds and a p99 of two seconds is consistent with a smooth distribution having a long tail, and it is equally consistent with two entirely separate populations — say ninety per cent of requests hitting a warm cache in eight milliseconds and ten per cent missing and taking two seconds. Those have completely different causes and completely different fixes, and no set of percentiles distinguishes them, while the bucket counts show the bimodality immediately. In my experience that pattern is common — cache hits versus misses, requests with and without a particular code path, traffic from clients on different network conditions — and reading only percentiles means the investigation starts from the wrong model of what is happening. So the practical habit I would encourage is to look at the full bucket distribution when starting to investigate latency, and to use percentiles for alerting and reporting where a single number is required. They serve different purposes and substituting one for the other loses information in a direction that is hard to notice.

   </details>

---

### Structured logging

*Emit logs as machine-parseable records with consistent fields, so they can be queried and correlated rather than grepped and guessed at.*

**Flow:** `Event fields` → `Correlation ids` → `Severity` → `Aggregation` → `Query`

> **The 30-second version**  
> Emit logs as typed key-value records with consistent field names and a propagated correlation id, redact centrally, define levels by required action, and sample debug detail while keeping every failed trace.

**The problem**

An incident requires finding every request from one customer that failed in the last hour. The logs are free-form strings written by six teams over four years, so the customer identifier appears as user_id in some lines, customerId in others, embedded mid-sentence in a third, and absent entirely from the lines that matter most.

The investigation becomes a sequence of increasingly desperate regular expressions, and the conclusion is reached from partial evidence — not because the information was missing, but because it was recorded in a form that cannot be queried.

> **A log line is a data record, not a sentence**  
> Written as key-value fields, a log entry can be filtered, aggregated and joined: every event for a customer, every error for an endpoint, the distribution of a field across a time window. Written as prose, it can only be searched for substrings. The difference is not stylistic — it determines whether logs are a queryable dataset or an archive of text that happens to be stored.

**Mental model**

Each log entry is a structured event with a timestamp, a severity, a message and a set of typed fields. Consistency of field names across services is what makes correlation possible.

1. **Event** — One structured record describing something that happened.
2. **Fields** — Typed key-value pairs — identifiers, durations, outcomes — that make the record queryable.
3. **Correlation id** — A value threaded through every log line for one request, so a distributed flow can be reassembled.
4. **Severity** — The level, which should indicate required action rather than the author's mood.
5. **Context propagation** — Carrying identifiers across service boundaries so correlation survives the hop.

> **Logs are where sensitive data leaks, quietly and durably**  
> Logging a request body captures passwords, tokens and personal data; logging a full URL captures signed access credentials in query strings. Log storage is usually replicated, retained for months, and accessible to far more people than the production database. Redaction must be structural — a deny-list of field names applied at the logging layer — because relying on every developer to remember at every call site has a predictable failure rate.

**How it works**

**Unstructured versus structured**

```text
UNSTRUCTURED
  2024-03-14 10:23:41 ERROR Failed to process order
  12345 for customer alice@example.com after 3
  retries: connection timeout

  queryable by: substring search
  NOT queryable by: customer, duration, retry count,
    error type, or any aggregate over these

STRUCTURED
  {
    "ts": "2024-03-14T10:23:41Z",
    "level": "error",
    "msg": "order processing failed",
    "order_id": "12345",
    "customer_id": "c_9f2a",
    "retries": 3,
    "error_type": "connection_timeout",
    "duration_ms": 4520,
    "trace_id": "abc123",
    "service": "order-processor"
  }

  now answerable:
    all failures for customer c_9f2a
    count of connection_timeout by hour
    p99 duration_ms for failed orders
    every service involved in trace abc123

NOTE WHAT CHANGED IN THE FIELDS
  customer_id is an opaque id, NOT an email address
  -> personal data does not belong in logs when an
     identifier will do
```

1. **Standardise field names across services** — Correlation requires that the same concept has the same key everywhere; three spellings of customer id makes joining impossible.
2. **Propagate a correlation id through every hop** — Without it, reconstructing a request across services means guessing from timestamps.
3. **Use severity to mean required action** — Error should mean something is broken and needs attention, not that the author found the situation notable.
4. **Redact structurally at the logging layer** — A deny-list of field names catches what individual discipline will eventually miss.
5. **Log identifiers, not personal data** — An opaque customer id supports every investigation an email address would, without the retention and exposure liability.
6. **Sample high-volume debug logs** — Full-fidelity logging at high request rates is frequently more expensive than the service it observes.

**Severity that means something**

```text
A LEVEL SHOULD ANSWER: what must someone do?

ERROR   something is broken; a human should look
        -> if errors fire routinely and nobody looks,
           the level is wrong, not the people
WARN    unexpected but handled; worth investigating
        if frequent
INFO    significant business events: request served,
        order placed, job completed
DEBUG   detail for diagnosis; off or sampled in
        production

THE COMMON FAILURE
  everything logged at ERROR because it felt important
  -> thousands of errors per hour
  -> nobody reads them
  -> the one that matters is indistinguishable
  -> error-rate alerts become useless

TEST FOR "ERROR"
  would you want to be paged for this?
  if not, it is WARN or INFO.

VOLUME AND COST
  1,000 req/s x 5 log lines x 500 bytes
    = 2.5 MB/s = ~216 GB/day
  at typical ingestion pricing this frequently exceeds
  the compute cost of the service being logged
  -> INFO for business events, DEBUG sampled,
     ERROR always kept in full
```

> **Log volume can cost more than the service it observes**  
> At a thousand requests per second, a handful of log lines per request produces hundreds of gigabytes per day, and ingestion pricing for observability platforms is typically charged per gigabyte. Teams regularly discover that logging costs exceed the compute cost of the application. The remedy is deliberate: full retention for errors, sampling for debug and high-cardinality detail, and the recognition that metrics answer aggregate questions far more cheaply than logs do.

**Worked example**

A payment service's logging design, showing the fields that make investigation possible.

**Logging standard**

```text
EVERY LINE CARRIES
  ts, level, msg, service, version, environment
  trace_id, span_id           <- from the trace context
  request_id                  <- per inbound request
  customer_id                 <- opaque, never an email

DOMAIN FIELDS WHERE RELEVANT
  payment_id, amount_minor, currency, provider,
  outcome, duration_ms, retry_count, error_type

NEVER LOGGED  (deny-list enforced in the logger)
  card numbers, CVV, full request bodies,
  Authorization headers, signed URLs with query
  strings, email addresses, physical addresses
  -> enforced centrally, because per-call-site
     discipline fails eventually and the failure is
     durable

LEVELS
  ERROR  payment failed for a reason needing attention
  WARN   provider retry succeeded; degraded path used
  INFO   payment attempted, payment succeeded
  DEBUG  provider request/response shape, sampled 1%

VOLUME MANAGEMENT
  INFO on business events only, not per function call
  DEBUG sampled at 1%, but ALWAYS retained in full for
    any trace that ended in an error
  -> sampling that keeps the interesting cases

INVESTIGATION THIS ENABLES
  "all failures for customer c_9f2a today"
    -> filter customer_id + level
  "did the provider timeout spike at 14:00?"
    -> count by error_type over time
  "what happened in this one request?"
    -> filter trace_id across all services
```

| Metric | Value | Note |
|---|---|---|
| Fields | consistent names | join across services |
| Identity | opaque id | **never email** |
| Debug | 1% sampled | 100% on error traces |
| Redaction | central deny-list | not per call site |

> **Sample debug logs, but always keep the traces that failed**  
> Uniform one per cent sampling discards ninety-nine per cent of the detail for the requests that actually failed — precisely the ones needed. Retaining full debug output for any trace that ended in an error, while sampling the rest, keeps the interesting cases at a fraction of the volume. This requires the sampling decision to be made at the end of the request rather than the beginning, which is a small architectural difference with a large practical payoff.

**When to use it**

- **Every production service**, as the baseline form of logging.
- **Distributed systems**, where correlation identifiers are the only way to reassemble a flow.
- **Debugging specific requests**, which metrics cannot address.
- **Audit and compliance**, where structured records are required evidence.
- **Any investigation needing per-request detail**, which is what logs uniquely provide.

**When to avoid it**

- **Do not use logs for aggregate metrics**, which are far cheaper and faster as actual metrics.
- **Do not log personal data or credentials**, given retention, replication and access breadth.
- **Do not log at full fidelity at high request rates** without sampling.
- **Do not use ERROR for anything not requiring attention**, which destroys the level's meaning.
- **Do not rely on per-call-site redaction discipline**, which fails and fails durably.

**Advantages**

- **Queryable and aggregatable** by field rather than by substring.
- **Correlatable across services** via propagated identifiers.
- **Machine-processable**, enabling automated analysis and alerting.
- **Consistent across teams** when field names are standardised.
- **Per-request detail** that metrics fundamentally cannot provide.
- **Central redaction** is possible because fields are named.

**Disadvantages**

- **More verbose on the wire** than plain text, though compression mitigates this.
- **Less readable raw**, requiring tooling to view comfortably.
- **Requires cross-team discipline** on field naming to be useful.
- **Expensive at volume**, sometimes exceeding the cost of the service.
- **A persistent leak surface** for sensitive data.
- **High-cardinality fields** can strain indexing in log platforms.

**Trade-offs**

**Signal type trade-offs**

| Signal | Answers | Cost | Best for |
|---|---|---|---|
| Metrics | Aggregate questions | Very low | Trends, alerting, dashboards |
| Structured logs | What happened in this case | High at volume | Per-request investigation |
| Traces | Where time went across services | Moderate with sampling | Latency and dependency analysis |
| Unstructured logs | Substring searches | Same as structured | Nothing, given the alternative |
| Debug logs unsampled | Complete detail | Often prohibitive | Low-volume or incident-scoped |

The most cost-effective change most teams can make is moving aggregate questions out of logs and into metrics. Counting occurrences by scanning log lines is orders of magnitude more expensive than incrementing a counter, and the answer is worse — logs are for the specific case, metrics for the general one.

**How it fails**

**Logging failures**

| Failure | Cause | Fix |
|---|---|---|
| Cannot correlate a request across services | No propagated correlation id | Thread trace and request ids through every hop |
| Cannot join logs between teams | Inconsistent field names | Shared schema for common fields |
| Errors ignored | Everything logged at ERROR | Levels defined by required action |
| Credentials found in log storage | Bodies or full URLs logged | Central deny-list; log identifiers only |
| Logging costs exceed compute | Full-fidelity logging at high volume | Sample debug; move aggregates to metrics |
| Failed requests lack detail | Uniform sampling discarded them | Retain full detail for error traces |
| Log platform degraded | High-cardinality fields indexed | Limit indexed fields; keep the rest as payload |

**Limits**

> **Volume and cost figures**
>
> - **Volume**: 1,000 req/s × 5 lines × 500 bytes is roughly 216 GB/day.
> - **Cost**: ingestion pricing frequently makes logging more expensive than the service generating it.
> - **Sampling**: 1–10% for debug is common, with full retention for error traces.
> - **Retention**: errors and audit records long; debug detail short.
> - **Cardinality**: indexed fields should be bounded; unbounded values belong in the payload, not the index.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Structured logs | Per-request investigation | Cost at volume |
| Metrics | Aggregates and alerting | No per-request detail |
| Distributed tracing | Cross-service latency attribution | Sampling; instrumentation effort |
| Event streams | Business events as data | Different retention and tooling |
| Profiling | Resource attribution within a process | Not request-scoped |
| Unstructured logs | Legacy compatibility | Not queryable by field |

Logs, metrics and traces are complementary rather than competing: metrics tell you something is wrong, traces tell you where the time went, and logs tell you what happened in the specific case. Using any one of them for another's job is where most observability cost is wasted.

**In real systems**

- **JSON-formatted logs with a shared field schema** are the practical standard, because cross-service correlation depends on consistent key names.
- **Trace and span identifiers embedded in every log line** connect logs to traces, turning two separate tools into one investigation.
- **Tail-based sampling that retains failed traces in full** keeps the useful detail without the volume of unsampled logging.
- **Central redaction deny-lists in logging libraries** exist because per-call-site discipline reliably fails at some point, and the leak persists in storage.
- **Migration of counting queries from logs into metrics** is a routine cost-reduction exercise, since aggregation over log lines is expensive and slow.

**Common mistakes**

- **Free-form log messages**, searchable only by substring.
- **Inconsistent field names** across services, preventing correlation.
- **No propagated correlation id**, making distributed investigation guesswork.
- **Everything at ERROR**, rendering the level meaningless.
- **Logging request bodies and full URLs**, leaking credentials and personal data.
- **Using logs for aggregate counting**, which metrics do far better and cheaper.
- **Head-based sampling**, discarding exactly the failed requests that matter.

**The staff-level view**

Logging is usually adequate for the team that wrote it and useless across teams, which is exactly when it is needed.

- **Standardise the common fields organisationally.** Correlation across services is impossible when the same concept carries three different key names, and this cannot be fixed per team after the fact.
- **Enforce redaction centrally.** Sensitive data in logs is durable, widely accessible and replicated, and relying on every developer at every call site has a failure rate that approaches certainty over time.
- **Define severity by required action.** When everything is an error, error-rate alerting is worthless and the one line that mattered is indistinguishable from thousands that did not.
- **Move aggregate questions to metrics.** Counting by scanning log lines is expensive, slow and a common reason logging bills exceed compute bills.
- **Sample at the end of the request, not the start.** Uniform sampling discards the failures, which are the only records anyone will want.

**Go deeper**

Structured logging treats each entry as a data record — typed fields with consistent names — rather than a sentence. That makes logs filterable, aggregatable and joinable: every event for a customer, the distribution of durations for failed requests, the full path of one request across services. Free-form messages contain the same information but allow only substring search, which makes most investigative questions unanswerable in practice.

Three disciplines determine whether the logs are useful across an organisation. Field names must be standardised, because correlation between teams is impossible when the same concept carries three spellings. A correlation identifier must be propagated through every hop, or reconstructing a distributed request means guessing from timestamps. And severity must be defined by required action, because when everything is an error, error-rate alerting is worthless and the one line that mattered is lost.

Two costs need active management. Sensitive data leaks into logs through request bodies and full URLs, and log storage is replicated, long-retained and broadly accessible — so redaction must be a central deny-list rather than per-call-site discipline. And volume is expensive enough that logging bills frequently exceed compute bills, which argues for sampling debug output with the decision made at the end of the request, so failed traces are retained in full.

Structured logging converts log output from text that happens to be stored into a dataset that can be queried, which is what makes it useful during investigation rather than merely available.

**A record, not a sentence.** Named typed fields permit filtering, aggregation and joining — all failures for a given customer, counts by error type over time, the latency distribution of requests that failed. The same facts embedded in prose support only substring matching, so questions that are trivially expressible against structured data become impossible, and investigations conclude from partial evidence gathered by increasingly speculative pattern matching.

**Field naming is an organisational concern, not a per-service one.** The value of structured logs comes largely from correlating across service boundaries, and that requires the same concept to carry the same key everywhere. Three spellings of a customer identifier across three teams makes joining impossible, and this cannot be repaired retrospectively without reprocessing history. Agreeing a core schema — identifiers, service, version, environment, trace context — is therefore a prerequisite rather than a refinement.

**Correlation identifiers connect everything else.** A trace identifier threaded through every hop and included in every log line means one filter returns the complete story of a single request across all services in order. Including trace and span identifiers additionally links the log store to the tracing system, so an investigation can move between aggregate latency attribution and specific event detail rather than being confined to whichever tool it began in.

**Severity should express required action.** When error means anything the author found notable, error volume becomes high enough that nobody reads it, the genuinely broken case is indistinguishable, and alerting built on error rate is worthless. Defining the levels by what someone must do — and applying a test such as whether the event would justify a page — keeps the signal usable. This is a discipline problem rather than a technical one, which is why it degrades steadily without explicit convention.

**Logs are a durable, broadly accessible copy of whatever passes through them.** Request bodies contain credentials, full URLs contain signed access tokens in query strings, user objects contain personal data — and log storage is typically replicated, retained for months, and readable by far more people than the production database. One careless call site therefore creates a lasting exposure. Redaction must be structural: a deny-list of field names enforced in the logging layer, combined with a convention of logging opaque identifiers rather than personal attributes, because per-call-site vigilance across a large codebase and a long period has an effective failure rate of one.

**Volume management determines the bill, and sampling order determines the value.** Several log lines per request at a thousand requests per second produces hundreds of gigabytes daily, which at typical ingestion pricing frequently exceeds the compute cost of the service. The two effective remedies are moving aggregate questions to metrics — counting by scanning log lines is orders of magnitude more expensive than a counter and gives a worse answer — and sampling debug output. Crucially, that sampling decision should be made at the end of the request rather than the start, because uniform head-based sampling discards ninety-nine per cent of the detail for exactly the failed requests anyone will later want to examine.

**Prove it — interview questions**

1. **[Basic] What makes a log structured?**

   <details><summary>Model answer</summary>

   That it is a record of typed key-value fields rather than a sentence. A structured entry carries a timestamp, level and message alongside named fields — customer identifier, duration, error type, trace identifier — so the log store can filter, aggregate and join on them. A free-form line containing the same information can only be substring-searched, which means questions like the distribution of durations for failed requests, or every event for one customer, are unanswerable even though the data is technically present.

   </details>

2. **[Basic] Why does severity matter?**

   <details><summary>Model answer</summary>

   Because it determines whether anyone acts. A level should answer what someone must do: error means something is broken and a human should look, warn means unexpected but handled, info records significant business events, debug is diagnostic detail. The common failure is logging everything notable at error, which produces thousands of errors an hour that nobody reads — and then the one that genuinely matters is indistinguishable, and any alerting built on error rate becomes useless. A workable test is whether you would want to be paged for it.

   </details>

3. **[Senior] Why is a correlation identifier essential?**

   <details><summary>Model answer</summary>

   Because without one, reconstructing what happened to a single request across several services means correlating by timestamp and hoping. With a trace identifier propagated through every hop and included in every log line, a single filter returns every event from every service for that one request, in order. That is the difference between an investigation that takes minutes and one that produces a plausible guess from partial evidence. It is also why embedding trace and span identifiers in logs matters — it connects the log store to the tracing system, so one investigation spans both.

   </details>

4. **[Senior] Why is logging a security concern?**

   <details><summary>Model answer</summary>

   Because logs capture whatever passes through them and then keep it. Logging a request body captures passwords and tokens; logging a full URL captures signed access credentials embedded in query strings; logging user objects captures personal data. Log storage is then replicated, retained for months, and readable by far more people than the production database — so a single careless call site creates a durable, broadly accessible copy of sensitive data. The mitigation has to be structural: a deny-list of field names enforced in the logging layer, plus a convention of logging opaque identifiers rather than personal attributes, because per-call-site discipline across a large codebase fails eventually and the failure persists.

   </details>

5. **[Staff] Design a logging standard for a payment service.**

   <details><summary>Model answer</summary>

   Every line carries a consistent core: timestamp, level, message, service, version, environment, trace and span identifiers, request identifier, and an opaque customer identifier — never an email address, because an opaque id supports every investigation an email would without the retention liability. Domain fields where relevant: payment id, amount, currency, provider, outcome, duration, retry count and error type, all with names agreed across the organisation so that logs from different teams can actually be joined. A deny-list enforced in the logging library covering card data, request bodies, authorization headers and signed URLs, because that is the class of mistake that is durable and broadly exposed once made. Levels defined by required action, with the page test applied to anything proposed as an error. On volume, info for business events only rather than per function call, debug sampled at around one per cent — but with the sampling decision made at the end of the request so that any trace ending in an error is retained in full, since uniform head-based sampling throws away exactly the records anyone will want. And aggregate questions answered by metrics rather than by counting log lines, because at a payment service's volume that difference is frequently the majority of the observability bill.

   </details>

6. **[Principal] How do you think about the boundary between logs, metrics and traces?**

   <details><summary>Model answer</summary>

   By what question each answers, because using one for another's job is where most observability cost and frustration originates. Metrics answer aggregate questions — how many, how fast, what proportion — extremely cheaply, because they are pre-aggregated counters and histograms; they are what alerting should be built on. Traces answer where time went across service boundaries for a request, which is the question metrics cannot address and logs address badly. Logs answer what specifically happened in one case, with detail neither of the others carries. The failure mode I see most often is teams counting occurrences by scanning log lines, which is orders of magnitude more expensive than a counter, slower to query, and produces a worse answer — and it is usually how logging bills come to exceed compute bills. The second failure is the reverse: trying to diagnose a specific customer's problem from aggregate metrics, which cannot be done at all. So the discipline I would push is explicit: when adding instrumentation, ask which of the three questions it answers, and put it in the corresponding signal. That framing also makes the correlation identifiers obviously important, since they are what lets an investigation move between the three rather than being stuck in whichever one it started in.

   </details>

---

### Distributed tracing

*Follow one request across every service it touches, recording where time was spent and which call caused which, so latency and failure can be attributed rather than guessed.*

**Flow:** `Trace context` → `Spans` → `Parent-child causality` → `Propagation` → `Latency attribution`

> **The 30-second version**  
> Record each request as a tree of causally linked spans propagated across every hop, sample so that slow and failed traces are kept, and link traces to logs so latency can be attributed rather than guessed.

**The problem**

A checkout request takes four seconds. Metrics show every individual service reporting healthy p99 latency. Logs from each service look normal. Nobody can say where the four seconds went, because no single service saw more than its own fragment and nothing connects the fragments.

The problem compounds with fan-out. A request touching twelve services in a partly parallel, partly sequential pattern has a critical path that is not obvious from any individual service's measurements, and optimising the wrong service produces no improvement at all.

> **A trace is the causal structure of one request, not a collection of timings**  
> Recording start and end times per service gives fragments. Recording parent-child relationships gives structure: which call triggered which, what ran in parallel, and therefore what the critical path actually is. That structure is what turns a set of durations into an explanation, and it is the reason tracing answers questions that metrics and logs cannot.

**Mental model**

A trace is a tree of spans. Each span represents one operation with a start, a duration and attributes; each knows its parent, so the tree reconstructs causality across process and network boundaries.

1. **Trace** — One request's complete journey, identified by a trace id.
2. **Span** — A single operation — an RPC, a query, a handler — with timing and attributes.
3. **Parent-child** — The causal link that makes the collection a tree rather than a list.
4. **Propagation** — Passing trace context across process boundaries, usually in headers.
5. **Sampling** — Deciding which traces to keep, since recording all of them is rarely affordable.

> **One service that does not propagate context breaks the trace**  
> Trace context must be passed through every hop — HTTP headers, message metadata, job payloads. A single service that drops it produces traces ending at that boundary, so everything downstream becomes invisible and appears to take no time. This is why partial instrumentation gives disproportionately little value, and why asynchronous boundaries such as queues are the usual place traces silently break.

**How it works**

**Structure and what it reveals**

```text
TRACE abc123, total 4,200 ms

gateway                          [====================] 4,200
  auth                           [=]                       50
  checkout-service               [===================] 4,100
    inventory-check              [==]                     180
    payment-provider             [================]     3,600  <-
    order-write                  [=]                     120
    notification-enqueue         [.]                      15
  response-render                [.]                      30

-> the critical path is payment-provider
-> every other service is healthy and irrelevant
-> NO individual service's metrics would have shown
   this: the payment provider is external, and
   checkout-service's own p99 includes the wait

WHAT THE TREE ADDS OVER A LIST OF TIMINGS
  which calls ran in PARALLEL
    -> parallel calls do not sum; the slowest dominates
  which call CAUSED which
    -> a slow database query under a specific handler
  where time was spent NOT in any child span
    -> gaps indicate local work, queueing, or
       uninstrumented calls

SPAN ATTRIBUTES CARRY THE CONTEXT
  http.status_code, db.statement (parameterised),
  error, retry.count, provider.name
  -> these turn "this span was slow" into "this span
     was slow because it retried three times"
```

1. **Propagate context at every boundary** — Including queues and background jobs, which are where traces most often break silently.
2. **Instrument the boundaries first** — RPC calls, database queries and external calls explain most latency; internal function spans add noise before they add value.
3. **Use tail-based sampling where possible** — Deciding after the request completes lets you keep the slow and failed traces rather than a random sample of successful ones.
4. **Attach meaningful attributes** — A span showing a duration answers what; attributes answer why.
5. **Link traces to logs and metrics** — Embedding the trace id in log lines and exemplars in histograms makes the three signals one investigation.
6. **Watch span cardinality and cost** — Traces are large relative to metrics, so sampling strategy determines whether the system is affordable.

**Sampling strategies**

```text
HEAD-BASED  (decide at the start)
  sample 1% of traces at the entry point
  + simple; decision propagates naturally
  + bounded cost, known in advance
  - keeps a RANDOM 1%, which is overwhelmingly
    successful fast requests
  - the slow and failed ones are discarded at 99%
  -> you sampled away the reason you built it

TAIL-BASED  (decide at the end)
  buffer spans, decide once the trace completes
  keep: all errors, all slow traces, a small
    percentage of normal ones
  + retains exactly the interesting cases
  - requires buffering all spans somewhere until the
    trace finishes
  - more infrastructure; more memory

HYBRID, AND USUALLY RIGHT
  head-based sample at a generous rate
  plus: always sample if the request is already
    marked interesting (error flag, debug header)
  plus: tail-based retention where affordable

COST INTUITION
  1,000 req/s, 10 spans/request, ~1 KB/span
    = 10 MB/s unsampled = ~860 GB/day
  at 1% = ~8.6 GB/day
  -> sampling is not optional at scale
  -> so WHICH 1% you keep is the whole question
```

> **Head-based sampling discards the traces you built tracing to find**  
> Sampling one per cent uniformly at the entry point keeps a random selection, and since the overwhelming majority of requests succeed quickly, that is what you retain. The rare slow request and the failing one — the entire reason for having traces — are kept with the same one per cent probability as everything else. Tail-based sampling, deciding after completion, is what makes the retained sample useful rather than merely representative.

**Worked example**

Tracing a checkout flow, showing the investigation it makes possible.

**Instrumentation and findings**

```text
PROPAGATION
  HTTP: W3C traceparent header
  message queue: trace context in message metadata
  background jobs: context stored with the job record
  -> the queue hop is the one most often forgotten,
     and it disconnects the whole asynchronous tail

SPANS CREATED AT
  inbound request handler
  each outbound HTTP or RPC call
  each database query
  each cache operation (if slow enough to matter)
  each queue publish and consume
  NOT: every internal function, which produces
    thousands of spans and obscures the structure

ATTRIBUTES
  http.method, http.route, http.status_code
  db.system, db.operation, db.table
  error=true plus error.type on failures
  retry.count, provider.name
  customer tier (bounded values only)

SAMPLING
  head-based 5% baseline
  always sampled if: an error occurred, the request
    exceeded 1 s, or a debug header was supplied
  tail-based retention for errors and slow traces

INVESTIGATION IT ENABLED
  complaint: checkout occasionally takes 4 s
  metrics:   every service p99 looks normal
  traces:    filter duration > 3 s
             -> 90% of slow traces show a
                payment-provider span of 3.5 s+
             -> and retry.count = 2 on those spans
             -> the provider was timing out and being
                retried, invisibly, inside a span that
                metrics reported as one slow call
  fix:       reduce provider timeout, add a breaker
```

| Metric | Value | Note |
|---|---|---|
| Critical path | payment 3.6 s | **others irrelevant** |
| Queue hop | context in metadata | commonly missed |
| Sampling | 5% + all slow/errors | keeps what matters |
| Attributes | retry.count | explains the why |

> **Traces explain what metrics can only report**  
> A metric can tell you that checkout p99 is four seconds. Only a trace can tell you that three and a half of those seconds were a payment provider call that silently retried twice, while every other service was healthy. Metrics detect and quantify; traces attribute and explain. Teams that add tracing expecting better dashboards are disappointed, while teams that add it expecting faster root-cause analysis are not.

**When to use it**

- **Microservice architectures**, where no single service sees the whole request.
- **Latency investigation**, where the critical path is not obvious from individual metrics.
- **Dependency mapping**, since traces reveal the actual call graph rather than the documented one.
- **Debugging specific slow or failed requests**, which aggregate metrics cannot address.
- **Understanding fan-out**, where parallel and sequential structure determines the outcome.

**When to avoid it**

- **Do not use traces for aggregate questions**, which metrics answer far more cheaply.
- **Do not instrument every internal function**, which buries the structure in noise.
- **Do not rely on head-based sampling alone**, which discards the interesting traces.
- **Do not adopt tracing partially**, since one non-propagating service breaks everything downstream.
- **Do not put unbounded values in span attributes**, which causes the same cardinality problems as metrics.

**Advantages**

- **Attributes latency to specific operations** rather than to services.
- **Reveals causality and parallelism**, showing the real critical path.
- **Discovers the actual dependency graph**, including calls nobody documented.
- **Explains individual slow requests**, which metrics fundamentally cannot.
- **Connects to logs and metrics** through trace identifiers and exemplars.
- **Surfaces hidden behaviour** such as retries inside a single logical call.

**Disadvantages**

- **Requires instrumentation everywhere**, with partial coverage giving little value.
- **Expensive unsampled**, so sampling strategy becomes a design decision.
- **Tail-based sampling needs buffering infrastructure.**
- **Context propagation breaks silently**, particularly across asynchronous boundaries.
- **Overhead per span**, small individually but real at high span counts.
- **Not suited to aggregate analysis**, which needs metrics alongside.

**Trade-offs**

**Sampling trade-offs**

| Strategy | Keeps | Cost | Infrastructure |
|---|---|---|---|
| Head-based 1% | A random 1%, mostly fast successes | Low, predictable | Minimal |
| Head-based 100% | Everything | Prohibitive at scale | Minimal |
| Tail-based | Errors and slow traces | Moderate | Buffering required |
| Hybrid | Interesting plus a baseline | Moderate | Some buffering |
| Debug-triggered | Specific requests on demand | Negligible | Header plumbing |

Debug-triggered sampling is worth having regardless of the main strategy: a header that forces a trace to be recorded lets a specific reported problem be investigated directly, which turns an unreproducible customer complaint into a concrete trace.

**How it fails**

**Tracing failures**

| Failure | Cause | Fix |
|---|---|---|
| Traces stop at a service boundary | Context not propagated | Instrument propagation at every hop |
| Asynchronous work invisible | Context not carried through the queue | Store trace context in message metadata |
| Slow requests never sampled | Head-based uniform sampling | Tail-based or interest-triggered sampling |
| Trace structure unreadable | Every internal function instrumented | Span at boundaries, not everywhere |
| Tracing backend overwhelmed | Unbounded attribute cardinality or no sampling | Bound attribute values; sample |
| Cannot connect a trace to its logs | Trace id absent from log lines | Include trace and span ids in logs |
| Retries hidden inside a span | No retry attribute recorded | Record retry counts and outcomes as attributes |

**Limits**

> **Cost and design figures**
>
> - **Volume**: 1,000 req/s × 10 spans × ~1 KB is roughly 860 GB/day unsampled.
> - **Sampling**: single-digit percentages are typical, with errors and slow traces retained preferentially.
> - **Span granularity**: boundaries and external calls, not every function.
> - **Propagation**: every hop including queues, or downstream becomes invisible.
> - **Attributes**: bounded value sets only, for the same cardinality reasons as metric labels.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Distributed tracing | Cross-service latency attribution | Instrumentation and sampling cost |
| Metrics | Aggregates and alerting | No causal detail |
| Structured logs with correlation ids | Event detail per request | No timing structure or parallelism |
| Profiling | Within-process attribution | Not cross-service |
| Synthetic transactions | End-to-end checks | Not real user traffic |
| Service mesh telemetry | Automatic RPC visibility | No application-level spans |

Service mesh telemetry is a useful starting point because it produces spans for RPC hops without application changes, giving a cross-service view immediately — but it cannot see inside a process, so database queries, cache calls and retry behaviour remain invisible until application instrumentation is added.

**In real systems**

- **W3C trace context headers** provide a standard propagation format, which matters because traces only work if every service and library agrees on it.
- **Tail-based sampling** is widely adopted precisely because head-based sampling discards the slow and failing traces that motivated tracing in the first place.
- **Trace identifiers embedded in log lines** connect the two signals so an investigation moves between aggregate structure and specific detail.
- **Exemplars linking metric buckets to traces** allow jumping from a populated tail bucket to an actual slow request.
- **Queue and job context propagation** is routinely the last thing instrumented and the most common reason asynchronous work is invisible in traces.

**Common mistakes**

- **Context not propagated through queues**, hiding all asynchronous work.
- **Head-based uniform sampling**, discarding the slow and failed traces.
- **Partial instrumentation**, so traces end at the first uninstrumented service.
- **Spans for every internal function**, burying the structure.
- **Unbounded attribute values**, overwhelming the tracing backend.
- **No trace id in logs**, leaving the two signals disconnected.
- **Using traces for aggregate analysis**, which metrics do better and far cheaper.

**The staff-level view**

Tracing pays off when coverage is complete and sampling keeps the right traces; partial efforts on either dimension deliver very little.

- **Treat propagation as all-or-nothing across the critical path.** One service that drops context makes everything downstream invisible, so partial rollouts should follow request paths rather than team boundaries.
- **Instrument asynchronous boundaries deliberately.** Queues and background jobs are where traces break silently, and the asynchronous tail is often exactly the part nobody can otherwise explain.
- **Insist on interest-based sampling.** Uniform head-based sampling retains a representative sample of fast successful requests, which is the one population nobody needed to investigate.
- **Keep spans at boundaries.** Instrumenting every function produces thousands of spans per trace, obscuring the structure that made tracing valuable.
- **Link the signals.** Trace ids in logs and exemplars in histograms turn three separate tools into one investigation, and this integration is worth more than additional instrumentation depth.

**Go deeper**

A trace follows one request across every service, recording spans with durations and parent-child relationships. That structure is what distinguishes tracing from a collection of per-service timings: it shows which calls ran in parallel, which caused which, and therefore where the critical path actually lies — questions no individual service's metrics can answer, since each sees only its own fragment.

Two things determine whether the investment pays off. Context must propagate across every hop, because a single service that drops it makes everything downstream invisible — and asynchronous boundaries such as queues and background jobs are where this breaks most often and most silently. And sampling must be interest-based: uniform head-based sampling retains a random selection, which at typical success rates means keeping fast successful requests and discarding the slow and failing traces that motivated the system.

Spans belong at boundaries — inbound handlers, outbound calls, database queries, queue operations — rather than on every internal function, which buries the structure in noise. Attributes carry the explanation: a retry count on a slow span turns one slow call into two silent timeouts. And embedding trace identifiers in log lines is what connects aggregate structure to specific detail, making three signals one investigation.

Distributed tracing reconstructs the causal structure of a single request across services, which is the information no individual service possesses.

**The tree is the point.** Recording durations per service produces disconnected fragments; recording parent-child relationships produces structure. That structure distinguishes sequential from parallel work — parallel branches do not sum, so the slowest dominates — identifies which call triggered which, and exposes time spent outside any child span, which indicates local work, queueing or uninstrumented calls. Without it, a four-second request in which every service reports healthy latency remains unexplainable.

**Propagation is all-or-nothing along a path.** A trace ends at the first service that fails to pass context, so everything beyond appears to take no time and the investigation is directed away from the actual cause. This makes partial instrumentation disproportionately unrewarding, and it argues for rolling out along request paths rather than by team ownership: one complete critical path is far more useful than five disconnected services.

**Asynchronous boundaries are where traces break silently.** HTTP propagation is typically handled by libraries, but a message published to a queue and consumed later by another process requires trace context in the message metadata and re-establishment by the consumer. That work is routinely last on the list, with the result that the entire asynchronous tail is invisible — frequently the part that was hardest to explain to begin with. Background and scheduled jobs share the problem.

**Sampling strategy decides whether the data is useful.** Unsampled tracing at meaningful request rates produces hundreds of gigabytes daily, so sampling is mandatory; the question is which traces survive. Head-based uniform sampling keeps a representative selection, which given typical success rates means overwhelmingly fast successful requests — precisely the population nobody needed to examine. Tail-based sampling, deciding once the trace completes, retains every error and slow trace alongside a baseline, at the cost of buffering infrastructure. Hybrid approaches with forced sampling for already-interesting requests capture most of the benefit more cheaply.

**Granularity and attributes determine readability.** Spans at boundaries — handlers, outbound calls, queries, queue operations — produce a tree whose shape is interpretable; instrumenting every internal function produces thousands of spans per trace and obscures exactly the structure that justified the effort. Attributes then supply causation rather than mere timing: a retry count on a slow span converts one apparently slow call into two silent timeouts and a success, which is a completely different problem with a completely different fix. Attribute values must be bounded for the same cardinality reasons that apply to metric labels.

**The value is attribution, not aggregation.** Metrics detect and quantify; traces explain individual cases. Teams adopting tracing in the expectation of better dashboards are generally disappointed, because it is a per-request tool. Framed as the mechanism that turns an unexplainable tail latency into a specific external call that retried twice, the proposition is clear — and so is the judgement about when it is worth the cost, which depends on fan-out and depth rather than on whether the architecture is described as microservices. Linking trace identifiers into logs and exemplars into histograms is what completes the picture, allowing an investigation to move from an aggregate anomaly to a specific trace to the log lines explaining it.

**Prove it — interview questions**

1. **[Basic] What does a trace show that metrics do not?**

   <details><summary>Model answer</summary>

   Where time went within a single request, across every service it touched, with the causal structure intact. Metrics can tell you checkout p99 is four seconds, but each service reports only its own fragment and all of them can look healthy — including the one that spent three and a half seconds waiting on an external provider. A trace shows the tree of calls with durations and parent-child relationships, which identifies the critical path and reveals which parallel branches did not matter.

   </details>

2. **[Basic] Why does context propagation matter so much?**

   <details><summary>Model answer</summary>

   Because a trace is only as complete as the chain of services that passed the context along. One service that fails to propagate produces traces that stop at its boundary, so everything downstream is invisible and appears to consume no time — which can point the investigation at exactly the wrong place. This is why partial instrumentation gives disproportionately little value, and why the highest-value fix is usually completing propagation along a request path rather than adding more spans within services that already have it.

   </details>

3. **[Senior] Why is head-based sampling problematic?**

   <details><summary>Model answer</summary>

   Because it keeps a random sample, and the overwhelming majority of requests are fast successes — so sampling one per cent at the entry point retains one per cent of the boring traffic and one per cent of the slow and failed requests that motivated building tracing. You have sampled away the reason for the system. Tail-based sampling, which buffers spans and decides after the request completes, can retain every error and every slow trace alongside a small baseline of normal ones, producing a sample that is useful rather than merely representative. The cost is buffering infrastructure, which is why hybrid approaches — a generous head-based baseline plus forced sampling when a request is already known to be interesting — are common.

   </details>

4. **[Senior] Where do traces typically break?**

   <details><summary>Model answer</summary>

   At asynchronous boundaries. HTTP propagation is usually handled by instrumentation libraries, but a message published to a queue and consumed minutes later by a different process requires the trace context to be carried in the message metadata and re-established by the consumer — and that is normally the last thing anyone instruments. The result is a trace that ends when the request returns, with the entire asynchronous tail invisible, which is frequently the part nobody can otherwise explain. Background jobs and scheduled work have the same issue and the same fix.

   </details>

5. **[Staff] How would you introduce tracing to an existing microservice system?**

   <details><summary>Model answer</summary>

   Along request paths rather than by team, because a trace stops at the first service that does not propagate — so instrumenting five services that are not adjacent gives five disconnected fragments, while instrumenting one complete critical path gives a usable picture immediately. I would start with the highest-value path, typically checkout or whatever has the worst-understood latency, propagating standard trace context headers and creating spans at boundaries only: inbound handlers, outbound calls, database queries and queue operations. Explicitly including queue publish and consume, because that is where traces break silently and where the unexplained time usually is. Attributes on spans carry the why rather than just the what — status codes, error types, retry counts, provider names — with bounded value sets to avoid the cardinality problems that affect metric labels equally. On sampling, a modest head-based baseline plus forced sampling whenever a request has errored, exceeded a latency threshold, or carries a debug header, moving to tail-based retention where the buffering infrastructure is affordable. And I would insist on embedding trace identifiers in log lines from the start, because the integration between signals is worth more than additional instrumentation depth — it is what lets an investigation move from a slow trace to the specific log detail explaining it.

   </details>

6. **[Principal] When does tracing not pay for itself?**

   <details><summary>Model answer</summary>

   When coverage is partial, when sampling is uniform, or when the system is not actually distributed enough for the causal structure to be the mystery. The first two are self-inflicted and fixable: fragments of traces across non-adjacent services explain nothing, and a representative sample of fast successful requests answers no question anybody had. The third is a genuine judgement call — a system with two services and a database has a call graph everyone already understands, so the marginal value of tracing over good metrics and correlated logs is small relative to the instrumentation and infrastructure cost. The honest framing is that tracing earns its cost when the critical path is genuinely unclear, which is a function of fan-out and depth rather than of architectural fashion. I would also push back on the common expectation that tracing improves dashboards: it does not, because it is a per-request tool rather than an aggregate one, and teams that adopt it hoping for better overview metrics tend to conclude it was not worth it. Framed correctly — as the thing that turns an unexplainable four-second p99 into a specific provider call that silently retried twice — the value proposition is much clearer, and so is the decision about whether a given system needs it.

   </details>

---

### Telemetry sampling

*Keep a deliberately chosen subset of observability data so cost stays bounded while the rare and failing cases — the ones worth keeping — survive.*

**Flow:** `Full telemetry` → `Sampling decision` → `Retained subset` → `Preserved rare events` → `Bounded cost`

> **The 30-second version**  
> Sample inversely to frequency — keep every error and slow request, a small baseline of normal traffic, and almost no health checks — deciding once per request and taking rates from metrics rather than from the sample.

**The problem**

A service handling ten thousand requests per second produces traces, logs and detailed telemetry for each one. Retaining all of it costs more than running the service, and the overwhelming majority of it describes requests that succeeded quickly and that nobody will ever examine.

The obvious fix — keep one per cent — solves the cost problem and creates a worse one. The retained sample is representative, which means it mirrors the traffic: almost entirely fast successes. The slow request and the failure, which is the entire reason the telemetry exists, are kept with the same one per cent probability as everything else.

> **Representative sampling is exactly what you do not want**  
> Observability data is valuable in inverse proportion to how common the event is. A uniform sample preserves the distribution, which means it preserves the boring majority and discards the rare cases. Useful sampling is deliberately biased: keep every error, keep every slow request, and keep a small fraction of the normal traffic for baseline comparison.

**Mental model**

Sampling is a decision about what to discard. The decision can be made before the work happens, after it completes, or somewhere in between — and when it is made determines what information is available to make it.

1. **Head-based** — Decide at the start, before the outcome is known. Cheap and uninformed.
2. **Tail-based** — Decide at the end, knowing duration and outcome. Informed but requires buffering.
3. **Interest criteria** — What makes a record worth keeping: errors, latency, specific customers, specific paths.
4. **Baseline retention** — A small uniform fraction kept for comparison, since anomalies need a normal to contrast with.
5. **Consistency** — The same decision applied across services, so a sampled trace is not half-missing.

> **Inconsistent sampling across services produces fragments, not traces**  
> If each service independently samples one per cent, the probability that all ten services in a request path sampled the same trace is vanishingly small — so almost every retained trace is incomplete. The decision must be made once and propagated, so that a trace is either fully kept or fully discarded. Independent per-service sampling is one of the few observability mistakes that produces data which looks valid and is not.

**How it works**

**When the decision is made determines what it can use**

```text
HEAD-BASED
  decide at the entry point; propagate the decision
  available information: the request, and nothing
    about how it will go
  + trivial; no buffering; cost known in advance
  + the decision propagates, so traces stay whole
  - blind to outcome: errors and slow requests are
    kept at the same rate as everything else

TAIL-BASED
  buffer all spans/records; decide once complete
  available information: duration, status, errors,
    everything
  + keep 100% of errors, 100% of slow requests,
    1% of normal ones
  - must buffer every in-flight trace somewhere
  - memory and infrastructure proportional to
    throughput x trace duration
  - late-arriving spans complicate the decision

HYBRID  (what most systems actually do)
  head-based baseline at a generous rate
  FORCE sampling when something is already known
    to be interesting:
      an upstream error flag
      a debug header supplied by an operator
      a customer under active investigation
  tail-based retention where affordable
  -> most of the benefit, much less infrastructure

THE INSIGHT
  the value of a record is inversely proportional to
  how common that kind of record is
  -> so sample INVERSELY to frequency, not uniformly
```

1. **Make the decision once and propagate it** — Independent per-service sampling produces mostly incomplete traces, which is worse than fewer complete ones.
2. **Always keep errors and slow requests** — These are the records anyone will actually open, and uniform sampling discards them at the same rate as everything else.
3. **Keep a baseline of normal traffic** — An anomaly is only interpretable against a contrast, so retaining some ordinary requests is not waste.
4. **Provide a forced-sampling escape hatch** — A debug header that guarantees retention turns an unreproducible complaint into a concrete record.
5. **Sample by category, not uniformly** — High-volume health checks can be sampled aggressively while low-volume critical paths are kept in full.
6. **Record the sampling rate with the data** — Without it, counts derived from sampled telemetry are wrong by an unknown factor.

**Rate-aware analysis and its pitfalls**

```text
IF YOU SAMPLE, COUNTS MUST BE SCALED
  retained: 50 errors at a 1% sampling rate
  actual:   ~5,000 errors
  -> forgetting this understates incidents by 100x

BUT BIASED SAMPLING BREAKS NAIVE SCALING
  errors sampled at 100%, successes at 1%
  -> you CANNOT multiply everything by 100
  -> error RATE computed from the sample is wildly
     wrong: it looks like 50% when it is 0.5%
  -> each category must be scaled by ITS OWN rate

CONSEQUENCE
  do NOT compute rates or ratios from biased samples
  -> use METRICS for rates; they are unsampled
  -> use sampled traces and logs for EXAMPLES
  -> this division is the practical rule

CATEGORY-BASED RATES IN PRACTICE
  health checks           0.01%   high volume, no value
  normal successful       1%      baseline
  slow (> p99 threshold)  100%
  errors                  100%
  checkout path           10%     critical, low volume
  debug-header requests   100%

-> total volume dominated by the 1% baseline
-> total VALUE dominated by the 100% categories
```

> **Never compute rates from biased samples**  
> Once errors are retained at a hundred per cent and successes at one per cent, the retained population is not a sample of the traffic — it is a curated collection. Computing an error rate from it produces a number that is wrong by two orders of magnitude and looks alarming rather than obviously broken. Rates belong to metrics, which are unsampled aggregates; sampled telemetry is for examining individual cases.

**Worked example**

A high-volume API where full telemetry retention is unaffordable.

**Sampling policy**

```text
VOLUME  10,000 req/s
  traces: 10 spans each at ~1 KB = 100 MB/s
  unsampled = ~8.6 TB/day
  -> clearly not retainable

POLICY BY CATEGORY
  health and readiness checks      0.01%
    high volume, zero diagnostic value
  normal successful requests       1%
    baseline for comparison
  requests over 1 s                100%
  requests that errored            100%
  checkout and payment paths       10%
    low volume, high value
  requests with debug header       100%
    operator escape hatch

IMPLEMENTATION
  head-based decision at the gateway for the baseline,
    propagated so traces stay whole
  upgrade to "keep" mid-request if an error occurs
    -> requires spans buffered until completion,
       i.e. tail-based for the interesting subset
  sampling rate recorded on every retained trace

RESULT
  retained volume    ~2% of original, ~170 GB/day
  errors retained    100%
  slow requests      100%
  -> cost reduced ~50x with no loss of the records
     anyone would open

ANALYSIS DISCIPLINE
  error RATE      -> from metrics (unsampled)
  error EXAMPLES  -> from traces (fully retained)
  -> never the reverse
```

| Metric | Value | Note |
|---|---|---|
| Unsampled | 8.6 TB/day | unaffordable |
| Retained | ~170 GB/day | ~50× reduction |
| Errors | 100% | **nothing lost** |
| Rates | from metrics | never from samples |

> **Sample inversely to frequency, because value is inversely proportional to it**  
> Health checks are the highest-volume and lowest-value telemetry in most systems, while errors are the lowest-volume and highest-value. Uniform sampling treats them identically, which spends the budget on the former and discards the latter. Category-based rates — aggressive for noise, complete for failures — typically reduce volume by an order of magnitude or more while retaining every record anyone would actually open.

**When to use it**

- **High-volume services**, where full retention costs more than the service itself.
- **Tracing systems**, where per-request span volume is inherently large.
- **Debug and verbose logging**, which is valuable rarely and voluminous always.
- **Long retention windows**, where storage cost compounds over time.
- **Any telemetry whose value is concentrated in rare events**, which is most of it.

**When to avoid it**

- **Do not sample metrics**, which are already aggregated and cheap.
- **Do not sample uniformly**, which discards exactly the records worth keeping.
- **Do not sample independently per service**, which yields incomplete traces.
- **Do not compute rates from biased samples**, which produces numbers wrong by orders of magnitude.
- **Do not sample audit or compliance records**, where completeness is the requirement.

**Advantages**

- **Bounds cost** without losing the diagnostically valuable records.
- **Reduces overhead** in collection, transport and storage.
- **Focuses retention on rare events**, which is where the value is.
- **Enables longer retention** of the subset that matters.
- **Tail-based decisions are outcome-aware**, keeping failures by construction.

**Disadvantages**

- **Aggregate analysis becomes invalid** on biased samples.
- **Tail-based sampling requires buffering infrastructure** proportional to throughput.
- **Rare unsampled events are unrecoverable**, and you cannot know what you discarded.
- **Consistency across services needs coordination**, or traces fragment.
- **Sampling rates must be recorded and understood** by everyone querying the data.

**Trade-offs**

**Sampling approach trade-offs**

| Approach | Keeps what matters | Infrastructure | Cost predictability |
|---|---|---|---|
| Head-based uniform | No | Minimal | Excellent |
| Head-based by category | Partially | Minimal | Good |
| Tail-based | Yes | Buffering required | Moderate |
| Hybrid | Mostly | Some buffering | Good |
| No sampling | Yes | None | Poor — cost scales with traffic |

Head-based sampling by category captures a surprising amount of the benefit without buffering: health checks can be identified at the entry point, as can critical paths and debug headers. What it cannot do is retain requests that turn out to be slow or failing, which is why hybrid approaches upgrade those mid-flight.

**How it fails**

**Sampling failures**

| Failure | Cause | Fix |
|---|---|---|
| Failed requests have no trace | Uniform head-based sampling | Always retain errors; tail-based or upgrade mid-request |
| Traces incomplete | Independent per-service sampling | Decide once, propagate the decision |
| Error rate looks absurd | Rate computed from a biased sample | Rates from metrics only |
| Counts understated | Sampling rate not applied when scaling | Record the rate with the data; scale per category |
| Sampling infrastructure overwhelmed | Buffering all traces at high throughput | Reduce buffer window; head-based for the baseline |
| Cannot investigate a reported issue | No forced-sampling mechanism | Debug header that guarantees retention |
| Budget consumed by noise | Health checks sampled at the same rate as real traffic | Category-based rates |

**Limits**

> **Practical rates**
>
> - **Health checks**: a tiny fraction — high volume, no diagnostic value.
> - **Normal successful traffic**: 1–10% as a baseline for comparison.
> - **Errors and slow requests**: 100%, since these are what gets examined.
> - **Critical low-volume paths**: high rates, because volume is small and value is high.
> - **Overall reduction**: typically an order of magnitude or more, with no loss of examinable records.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Tail-based sampling | Retaining errors and slow traces | Buffering infrastructure |
| Head-based by category | Simple cost control | Cannot react to outcome |
| Aggregation into metrics | Rates and trends | Loses individual records |
| Shorter retention | Cost control without bias | No historical investigation |
| Dynamic sampling | Automatic rate adjustment by volume | Complexity; variable rates to track |
| No sampling | Low-volume or compliance data | Cost scales with traffic |

Dynamic sampling, where rates adjust automatically so that rare request types are kept at higher rates than common ones, generalises the category approach — it keeps volume roughly constant per category rather than per request, which automatically preserves unusual traffic without anyone enumerating what is unusual.

**In real systems**

- **Tail-based sampling collectors** buffer spans until a trace completes so the retention decision can account for errors and duration.
- **Debug headers that force retention** are standard, because they convert an unreproducible customer report into a concrete trace on demand.
- **Category-based rates** are widely used, with health checks sampled almost to nothing and critical paths retained heavily.
- **Sampling rates recorded alongside retained data** are necessary for any correct scaling, and their absence is a common source of wrong conclusions.
- **Metrics kept unsampled alongside sampled traces** preserve accurate rates while bounding the cost of per-request detail.

**Common mistakes**

- **Uniform sampling**, discarding errors at the same rate as successes.
- **Independent sampling per service**, producing fragmented traces.
- **Computing rates from biased samples**, producing wildly wrong numbers.
- **Not recording the sampling rate**, making any scaling impossible.
- **Sampling health checks at the normal rate**, spending the budget on noise.
- **No forced-sampling mechanism**, leaving reported issues uninvestigable.
- **Sampling audit records**, where completeness is a requirement.

**The staff-level view**

Sampling decisions are usually made for cost reasons and evaluated on volume, when what matters is which records survive.

- **Reject uniform sampling.** It preserves the traffic distribution, which means it preserves the boring majority and discards the failures — the records that justified collecting telemetry at all.
- **Make the decision once and propagate it.** Independent per-service sampling produces mostly incomplete traces, which is data that looks usable and is not, and that is worse than a smaller number of complete ones.
- **Separate rates from examples in team practice.** Rates come from unsampled metrics; sampled traces and logs supply individual cases. Computing an error rate from a biased sample produces a number wrong by orders of magnitude and alarming enough to be acted upon.
- **Provide a forced-sampling escape hatch.** A debug header that guarantees retention is what turns an unreproducible complaint into something investigable, and it costs almost nothing.
- **Sample health checks into near-nothing.** They are typically the largest single category by volume and contribute essentially no diagnostic value.

**Go deeper**

Full telemetry retention frequently costs more than the service generating it, so sampling is necessary. The question is which records survive. Uniform sampling is representative, which means it mirrors traffic that is overwhelmingly fast and successful, so the slow and failing requests that justified collecting traces are discarded at the same rate as everything else.

Useful sampling is deliberately biased by category: health checks kept at almost nothing since they are high volume and zero value, normal successes at a small baseline for contrast, and errors and slow requests retained in full. Head-based decisions are cheap and propagate cleanly but cannot see outcomes; tail-based decisions know duration and status but require buffering every in-flight trace. Hybrid schemes take a head-based baseline and force retention for anything already known to be interesting.

Two structural rules matter more than the rates. The decision must be made once per request and propagated, because independent per-service sampling produces mostly incomplete traces — data that looks valid and is not. And rates must never be computed from biased samples, which produce figures wrong by orders of magnitude; rates come from unsampled metrics while sampled telemetry supplies individual examples.

Telemetry sampling decides what to discard, and since the discarded data is unrecoverable, the decision determines which questions can be answered later.

**Value is inversely proportional to frequency.** Health checks are the highest-volume and lowest-value telemetry in most systems; errors are the lowest-volume and highest-value. Uniform sampling treats them identically, spending most of the retention budget on records nobody will open while discarding the ones that motivated collection. Sampling rates should therefore be assigned per category — near-zero for noise, complete for failures — which typically reduces volume by an order of magnitude or more without losing anything examinable.

**When the decision is made bounds what it can consider.** Head-based sampling happens at the entry point, before the outcome is known: cheap, requiring no buffering, and propagating naturally so traces remain whole, but blind to whether the request will fail or take four seconds. Tail-based sampling buffers all spans and decides at completion, which permits retaining every error and slow trace, at the cost of infrastructure proportional to throughput multiplied by trace duration. Most practical systems are hybrid: a head-based baseline, forced retention when something is already flagged as interesting, and tail-based handling for the subset where it is affordable.

**The decision must be made once and propagated.** If each service samples independently, the probability that every service along a ten-hop path retained the same trace is negligible, so nearly every stored trace is partial. This is a particularly awkward failure because the data looks legitimate — spans exist, timings are present — while the structure that made tracing valuable is missing. Propagating a single decision means a trace is either entirely kept or entirely dropped, which is strictly more useful than a larger number of fragments.

**Biased sampling invalidates aggregate analysis, and not obviously.** Once errors are retained at full rate and successes at one per cent, the stored population is a curated collection rather than a sample of traffic. An error rate computed from it can be two orders of magnitude too high, and the result looks alarming rather than absurd, so it may well be acted upon. Correct analysis requires scaling each category by its own rate, which is fiddly and rarely done. The robust arrangement is a division of responsibility: unsampled metrics answer questions about rates and trends, while sampled traces and logs supply individual cases for examination.

**Forced sampling is disproportionately valuable.** A header that guarantees a request is traced, or a flag marking a customer under investigation, converts an unreproducible report into a concrete record. It costs almost nothing to implement and it addresses the scenario sampling otherwise makes worst — a specific reported problem occurring in traffic that was probabilistically discarded.

**The consequences appear during incidents, not on cost dashboards.** A uniform policy looks perfectly sensible when evaluated on volume reduction, and its defect only surfaces when someone needs the trace for a particular failure and finds it absent — at which point no subsequent investment recovers it. That asymmetry, between when the decision is made and when its cost is felt, is what makes sampling policy worth deliberate design rather than treating it as a cost-control knob. Recording the sampling rate alongside retained data, and establishing an explicit organisational convention about which signal answers which kind of question, are both cheap at the outset and awkward to retrofit once dashboards and habits have formed around the wrong assumption.

**Prove it — interview questions**

1. **[Basic] Why sample telemetry at all?**

   <details><summary>Model answer</summary>

   Because full retention frequently costs more than the service producing it. A service at ten thousand requests per second generating ten spans each produces terabytes a day, and the great majority of that describes fast successful requests nobody will examine. Sampling bounds the cost. The question is not whether to sample but which records to keep, because the naive answer — a uniform fraction — throws away the ones that matter.

   </details>

2. **[Basic] What is wrong with keeping a uniform one per cent?**

   <details><summary>Model answer</summary>

   It is representative, which is exactly the problem. A uniform sample mirrors the traffic, and traffic is overwhelmingly fast successful requests, so that is what you retain. The rare slow request and the failure — the entire reason for collecting traces — are kept with the same one per cent probability as everything else, meaning ninety-nine times out of a hundred the record you want to open does not exist. Useful sampling is deliberately biased toward the uncommon.

   </details>

3. **[Senior] What is the difference between head-based and tail-based sampling?**

   <details><summary>Model answer</summary>

   When the decision is made and therefore what information it can use. Head-based decides at the entry point before anything has happened, so it is cheap, needs no buffering, and propagates naturally to keep traces whole — but it is blind to outcome, so errors and slow requests are retained at the same rate as everything else. Tail-based buffers all spans and decides once the request completes, so it can keep every error and every slow trace alongside a small baseline of normal ones. The cost is buffering infrastructure proportional to throughput and trace duration, which is why hybrid schemes — a head-based baseline plus forced retention for anything already known to be interesting — are common.

   </details>

4. **[Senior] Why can you not compute rates from sampled data?**

   <details><summary>Model answer</summary>

   Because once sampling is biased, the retained population is a curated collection rather than a sample. If errors are kept at a hundred per cent and successes at one per cent, then computing an error rate from what was retained gives something like fifty per cent when the true figure is half a per cent — wrong by two orders of magnitude, and alarming enough that someone may act on it. Each category would need scaling by its own rate, which is error-prone and rarely done correctly. The practical rule is a division of labour: rates and ratios come from metrics, which are unsampled aggregates, while sampled traces and logs supply individual examples.

   </details>

5. **[Staff] Design a sampling policy for a service handling ten thousand requests per second.**

   <details><summary>Model answer</summary>

   Category-based rates rather than a single number, because value is inversely proportional to frequency. Health and readiness checks sampled to almost nothing — they are usually the largest category by volume and contribute no diagnostic value whatsoever. Normal successful requests at around one per cent, retained as a baseline, because an anomaly is only interpretable against a contrast. Everything that errored or exceeded a latency threshold retained in full, since those are the only records anyone will open. Critical low-volume paths such as checkout retained heavily, because the volume cost is small and the value is high. And a debug header that forces retention, which is what turns an unreproducible customer report into a concrete trace. Implementation is hybrid: a head-based decision at the gateway for the baseline, propagated so traces stay whole rather than fragmenting across services, with an upgrade to retain mid-request when an error occurs, which requires buffering for that subset. The sampling rate is recorded on every retained record. Typically that gives an order-of-magnitude-plus reduction with no loss of examinable records — and I would pair it with an explicit team convention that error rates come from metrics and error examples come from traces, never the other way round.

   </details>

6. **[Principal] What makes sampling decisions consequential beyond cost?**

   <details><summary>Model answer</summary>

   That they silently determine what questions can be answered later, and the cost of a bad decision only becomes visible during an incident. A uniform sampling policy looks entirely reasonable on a cost dashboard and produces exactly the wrong retained population, so the first time anyone needs the trace for a specific failure they discover it was discarded — and no amount of subsequent investment recovers data that was never stored. The second-order problem is that biased sampling, which is the right answer, makes the retained data unsuitable for aggregate analysis in a way that is not obvious from looking at it: the data appears complete and produces plausible-looking rates that are wrong by orders of magnitude. So the two things I would insist on are structural rather than numerical. First, that the sampling decision is made once per request and propagated, because independent per-service sampling produces fragmentary traces that look valid and are not — a failure mode that is genuinely hard to notice. Second, that the organisation adopts an explicit convention about which signal answers which kind of question, because the alternative is someone confidently reporting an error rate derived from a curated collection of errors. Both of those cost nothing to establish at the start and are difficult to correct once people have built habits and dashboards on the wrong assumption.

   </details>

---

### Cardinality and label design

*Control the number of distinct time series a metric produces, because cardinality is multiplicative and unbounded labels destroy metric systems.*

**Flow:** `Metric name` → `Label dimensions` → `Series multiplication` → `Storage and query cost` → `Bounded design`

> **The 30-second version**  
> Total series is the product of all label cardinalities, so keep labels bounded, template paths, enumerate errors, keep identifiers out — and enforce ingestion limits, because one careless label can take monitoring down.

**The problem**

An engineer adds a user identifier label to a request counter so that per-user traffic can be examined. The service has two million users. Overnight the metrics backend is storing two million time series for that one metric, query performance collapses, ingestion falls behind, and the on-call engineer discovers that the monitoring system is now the outage.

The change looked trivial — one additional label on one metric — and its cost was invisible at the point it was made. That asymmetry between how easy it is to add a label and how expensive the consequence can be is what makes cardinality a recurring production hazard.

> **Cardinality multiplies, and each series has a fixed cost**  
> A metric's series count is the product of the cardinality of every label. Five endpoints, four status codes and three regions is sixty series, which is fine; adding a user identifier with a million values makes it sixty million. Each series carries its own index entry, memory footprint and storage overhead, so the multiplication translates directly into resource consumption that scales with something nobody bounded.

**Mental model**

Each unique combination of label values is a separate time series with its own storage, index entry and memory cost. The total is the product of label cardinalities, so adding a dimension multiplies rather than adds.

1. **Series** — One unique combination of metric name and label values.
2. **Cardinality** — The number of distinct values a label can take.
3. **Multiplication** — Total series equals the product of all label cardinalities.
4. **Bounded label** — One whose value set is known and finite — status code, region, endpoint template.
5. **Unbounded label** — One that grows with usage — user id, request id, URL path, error message.

> **Unbounded labels are the single most common way to destroy a metrics system**  
> User identifiers, request identifiers, full URL paths, email addresses, raw error messages and timestamps all have value sets that grow without limit. A single one of these on a single metric can generate millions of series, exhausting memory and collapsing query performance. The damage is immediate on deployment and persists until the offending series expire, so the monitoring system fails at exactly the moment it is needed to diagnose the failure.

**How it works**

**The arithmetic and where it goes wrong**

```text
SERIES = product of all label cardinalities

REASONABLE
  http_requests_total{
    endpoint (20 templates),
    method (5),
    status (8)
  }
  = 20 x 5 x 8 = 800 series
  add instance (50) -> 40,000
  -> still manageable

CATASTROPHIC
  add user_id (2,000,000)
  = 800 x 2,000,000 = 1.6 BILLION series
  -> the metrics backend dies

SUBTLER DISASTERS
  raw URL path instead of a route template
    /users/12345/orders/67890
    -> unbounded: one series per distinct path
    -> use /users/{id}/orders/{id} instead
  raw error message as a label
    "connection to 10.2.3.4:5432 timed out after 30s"
    -> unbounded: address and duration vary
    -> use error_type="connection_timeout"
  histogram buckets multiply everything
    12 buckets x 800 = 9,600 series for ONE metric
    -> histograms are where cardinality bites hardest

COST PER SERIES
  index entry + in-memory state + storage
  roughly a few KB of memory per active series
  1,000,000 series -> several GB just to hold them
```

1. **Use route templates, never raw paths** — A path with identifiers in it is unbounded by construction; the template is bounded by the number of routes.
2. **Categorise errors rather than labelling with messages** — A small enumeration of error types is bounded; message text containing addresses and durations is not.
3. **Keep identifiers out of metrics entirely** — Per-entity questions belong to logs and traces, which are built for high cardinality; metrics are for aggregates.
4. **Remember histograms multiply** — Bucket count multiplies every label combination, making histograms the most cardinality-sensitive metric type.
5. **Enforce limits at ingestion** — A cap that drops or aggregates beyond a threshold protects the system from a single careless deployment.
6. **Review label additions like schema changes** — The cost is invisible at the call site and multiplicative in effect, which is exactly the combination that needs review.

**Where high-cardinality questions belong**

```text
THE QUESTION: "what is happening for customer X?"

WRONG: add customer_id as a metric label
  -> millions of series
  -> and you still cannot see what happened, only
     counts

RIGHT: use logs or traces
  -> both are designed for high-cardinality fields
  -> structured logs filter on customer_id cheaply
  -> traces are indexed by attributes without
     multiplying stored series
  -> and they carry the detail that actually answers
     the question

THE DIVISION OF LABOUR
  metrics  low cardinality, aggregate, always-on,
           cheap, good for alerting and trends
  logs     high cardinality, per-event detail,
           expensive at volume, sampled
  traces   high cardinality, causal structure,
           sampled

IF A LABEL WOULD HAVE MANY VALUES, THE QUESTION
BELONGS IN A DIFFERENT SIGNAL.

EXCEPTION: bounded tiers
  customer_tier = {free, pro, enterprise}   fine
  customer_id                               not fine
  -> aggregate the entity into a category
```

> **Histograms are where cardinality damage is largest**  
> A histogram with twelve buckets produces twelve series for every label combination, so a metric that would have eight hundred series as a counter has nearly ten thousand as a histogram. Adding a label to a histogram is therefore roughly an order of magnitude more expensive than adding it to a counter, and histograms are precisely the metrics teams want to break down by endpoint and outcome. The bucket count needs to be in the arithmetic from the start.

**Worked example**

Designing labels for an API service's request metrics.

**Label design with the arithmetic**

```text
GOOD LABELS  (bounded, known value sets)
  endpoint   route templates, ~20 values
  method     GET/POST/PUT/DELETE/PATCH, 5
  status     2xx/4xx/5xx classes or codes, ~8
  outcome    success/failure, 2
  region     ~5

BAD LABELS  (unbounded, grow with usage)
  user_id           millions
  request_id        one per request
  raw path          unbounded
  error_message     unbounded text
  session_id        unbounded
  timestamp         unbounded by definition

SERIES ARITHMETIC
  counter: 20 x 5 x 8 = 800
  with region: 4,000
  histogram with 12 buckets: 20 x 2 x 12 = 480
    (deliberately fewer labels on the histogram,
     because buckets multiply)
  total for the service: low thousands
  -> comfortable

HIGH-CARDINALITY QUESTIONS, ROUTED ELSEWHERE
  "which customers saw errors?"
    -> structured logs, filter customer_id
  "why was this request slow?"
    -> traces, filter by trace_id
  "what error message appeared?"
    -> logs; metrics carry error_type only

PROTECTION
  ingestion-time cardinality limit per metric
  alert when a metric's series count grows sharply
  -> catches a bad deployment in minutes rather than
     after the backend falls over
```

| Metric | Value | Note |
|---|---|---|
| Counter | 800 series | bounded labels |
| Histogram | fewer labels | **buckets multiply** |
| user_id | logs, not metrics | millions of values |
| Protection | ingest limit + alert | catches deploys |

> **If a label would have many values, the question belongs in another signal**  
> The impulse to add a high-cardinality label is usually a symptom of asking a per-entity question of an aggregate tool. Logs and traces are designed for high-cardinality fields and carry the detail that actually answers such questions, while metrics exist for bounded aggregates. Recognising the misplacement — rather than trying to make metrics do it — resolves the problem and gets a better answer.

**When to use it**

- **Whenever adding a label to any metric**, since the cost is multiplicative and invisible locally.
- **Designing histograms**, where bucket count multiplies every label combination.
- **Reviewing instrumentation changes**, which should be treated like schema changes.
- **Diagnosing metrics backend problems**, where cardinality is the usual cause.
- **Deciding which signal answers a question**, since high-cardinality questions belong to logs and traces.

**When to avoid it**

- **Do not put identifiers in metric labels**, which is the classic unbounded case.
- **Do not label with raw paths or error messages**, both unbounded by construction.
- **Do not add labels to histograms casually**, where the cost is multiplied by bucket count.
- **Do not use metrics for per-entity questions**, which logs and traces answer better and cheaper.
- **Do not deploy label changes without an ingestion limit**, which is the only protection against a single bad change.

**Advantages**

- **Bounded, predictable resource usage** in the metrics backend.
- **Fast queries**, since series count drives query cost directly.
- **Long retention is affordable** when series counts are controlled.
- **Stable behaviour under traffic growth**, because bounded labels do not grow with usage.
- **Clear signal boundaries**, with high-cardinality questions routed to appropriate tools.

**Disadvantages**

- **Limits the breakdowns available** directly from metrics.
- **Requires discipline** across everyone who adds instrumentation.
- **Per-entity investigation needs another tool**, adding a step.
- **Bucket design constrains histogram labelling**, forcing trade-offs.
- **Aggregation into categories loses detail** that some questions would want.

**Trade-offs**

**Label choice trade-offs**

| Label | Cardinality | Value | Verdict |
|---|---|---|---|
| Route template | Bounded, tens | High | Use |
| Status code class | Bounded, few | High | Use |
| Customer tier | Bounded, a handful | Moderate | Use |
| Instance id | Bounded but sizeable | Moderate | Use with care |
| Customer id | Unbounded | High but misplaced | Logs or traces |
| Raw error message | Unbounded | Moderate | Categorise into error_type |

Instance identifiers occupy the interesting middle ground: bounded but potentially large, and genuinely useful for spotting a single bad host. The usual compromise is to include them on a small number of key metrics rather than everywhere, since the multiplication applies to every metric that carries them.

**How it fails**

**Cardinality failures**

| Failure | Cause | Fix |
|---|---|---|
| Metrics backend out of memory | Unbounded label deployed | Ingestion limits; remove the label; wait for expiry |
| Queries become unusably slow | Series count too high | Reduce labels; aggregate before storing |
| Monitoring fails during an incident | Cardinality explosion from a recent change | Alert on series growth; review label changes |
| Histogram cost far above expectation | Bucket count multiplying labels | Fewer labels on histograms; fewer buckets |
| Path label growing without limit | Raw URLs rather than route templates | Template the route before labelling |
| Error label unbounded | Raw message text as a label | Enumerate error types |
| Cannot answer per-customer questions | Attempted via metrics | Use structured logs or traces |

**Limits**

> **Practical figures**
>
> - **Series cost**: roughly a few kilobytes of memory per active series, so millions of series means gigabytes.
> - **Multiplication**: total series is the product of all label cardinalities, not the sum.
> - **Histograms**: multiply by bucket count, typically 10–15×.
> - **Comfortable scale**: thousands to low tens of thousands of series per service is unremarkable; millions is a problem.
> - **Blast radius**: one unbounded label on one metric can exceed the entire rest of the system's cardinality.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Bounded labels only | Metrics | Limited breakdowns |
| Logs for high cardinality | Per-entity questions | Higher cost per event; sampled |
| Traces for per-request detail | Causal investigation | Sampled; instrumentation needed |
| Aggregation into categories | Tier or class breakdowns | Loses individual detail |
| Separate high-cardinality store | Specialised analysis | Additional system to operate |
| Exemplars on metrics | Linking aggregates to examples | Requires tracing integration |

Exemplars are a neat resolution to part of this tension: rather than adding a high-cardinality label to find out which requests populated a bucket, the metric carries sample trace identifiers, so the aggregate stays cheap while the path to individual detail remains open.

**In real systems**

- **Route templating before labelling** is universal practice, because raw paths containing identifiers are unbounded by construction.
- **Ingestion-time cardinality limits** exist in metrics platforms precisely because a single careless deployment can otherwise take the system down.
- **Error classification into bounded types** replaces raw message labelling, keeping the dimension useful and finite.
- **Exemplars linking metric buckets to traces** provide a path from aggregate to individual detail without high-cardinality labels.
- **Alerting on series-count growth** catches a bad instrumentation change within minutes rather than after the backend degrades.

**Common mistakes**

- **Identifiers as metric labels**, generating millions of series.
- **Raw URL paths** rather than route templates.
- **Raw error messages** as label values.
- **Labels added to histograms** without accounting for bucket multiplication.
- **No ingestion-time cardinality limit**, leaving one deployment able to take down monitoring.
- **No alerting on series growth**, so the problem surfaces as an outage.
- **Using metrics for per-entity questions** that logs and traces answer properly.

**The staff-level view**

Cardinality is the one instrumentation mistake that can take down monitoring at the moment monitoring is needed.

- **Treat label additions as schema changes.** The cost is multiplicative and entirely invisible at the call site, which is exactly the combination that requires review rather than individual judgement.
- **Enforce ingestion-time limits.** Discipline reduces the frequency of the mistake but cannot eliminate it, and the limit is what converts a fleet-wide outage into a dropped metric.
- **Alert on series-count growth per metric.** It catches a bad deployment in minutes, before memory exhaustion turns the monitoring system into the incident.
- **Redirect high-cardinality questions explicitly.** The impulse to add a customer identifier is a per-entity question aimed at an aggregate tool; logs and traces answer it better and at appropriate cost.
- **Put bucket count into the arithmetic for histograms.** Adding a label to a histogram costs roughly an order of magnitude more than adding it to a counter, and histograms are what people most want to break down.

**Go deeper**

A metric's cost is driven by its series count, which is the product of the cardinality of every label. Twenty endpoints, five methods and eight status codes is eight hundred series and entirely comfortable; adding a user identifier with millions of values makes it billions. Since each series carries index, memory and storage overhead, that multiplication translates directly into resource exhaustion — and it happens on deployment, all at once.

The unbounded labels are predictable: identifiers, raw URL paths, session values, raw error messages, anything derived from time. The corresponding fixes are equally predictable — route templates instead of paths, enumerated error types instead of messages — and the general rule is that if a label would have many values, the question being asked is a per-entity one that belongs in logs or traces, which are designed for high cardinality and give a better answer anyway.

Histograms deserve particular care because bucket count multiplies every label combination, making a label roughly an order of magnitude more expensive there than on a counter. And since the mistake is one word at a call site with no local cost signal, discipline alone is insufficient: ingestion-time limits and alerting on series growth are what keep a single careless change from taking down monitoring during the incident it caused.

Cardinality is the number of distinct time series a metric generates, and because it multiplies across labels it is the dimension along which metrics systems most often fail.

**The arithmetic is multiplicative, and the cost per series is fixed.** Total series equals the product of all label cardinalities, so each additional dimension scales the whole rather than adding to it. With a few kilobytes of memory per active series for index and state, a metric producing millions of series consumes gigabytes on its own. This is why a single label change can exceed the entire rest of a system's cardinality, and why the failure arrives abruptly rather than gradually.

**Unbounded labels are a recognisable category.** User and session identifiers, request identifiers, raw URL paths containing ids, raw error message text, and anything timestamp-derived all have value sets that grow with usage. Each has a bounded substitute: route templates instead of paths, enumerated error types instead of messages, customer tiers instead of customer identifiers. Applying those substitutions mechanically removes most of the risk without requiring anyone to reason about the arithmetic.

**Histograms concentrate the damage.** Every bucket is a separate series per label combination, so a dozen buckets multiplies the series count for every dimension present. Adding a label to a histogram therefore costs roughly an order of magnitude more than adding it to a counter — and histograms are precisely the metrics teams most want broken down by endpoint, outcome and region. The practical consequence is that histograms should deliberately carry fewer labels than counters, and the bucket multiplier belongs in any cardinality estimate.

**High-cardinality questions are misplaced rather than unanswerable.** The urge to add a customer identifier reflects a per-entity question being asked of an aggregate tool, and even with unlimited cardinality the answer would be poor — counts per customer rather than what actually happened to them. Logs and traces handle high-cardinality fields by design and carry the detail that answers such questions properly. Reframing the impulse this way resolves the cardinality problem and improves the answer simultaneously, which makes it a more durable correction than a prohibition.

**Discipline is necessary and insufficient.** The mistake is a single word at a call site, it looks useful, it works in development where the value set is tiny, and nothing locally signals cost. The consequence appears only at production scale, all at once, and degrades the monitoring system — the worst possible blast radius and the worst possible timing, since the tool required to diagnose the problem is the one that broke. Individual vigilance does not reliably prevent a mistake whose cost is invisible where it is made.

**Structural protections are what work.** Ingestion-time limits per metric, which drop or aggregate beyond a threshold, convert a system-wide outage into one degraded metric. Alerting on series-count growth catches a bad deployment within minutes rather than after memory exhaustion. Treating label additions as schema changes in review puts a second pair of eyes on a line whose cost is multiplicative and unstated. And exemplars offer a partial resolution to the underlying tension, attaching sample trace identifiers to metric buckets so that the path from an aggregate anomaly to an individual example stays open without any high-cardinality label existing at all.

**Prove it — interview questions**

1. **[Basic] What is cardinality in metrics?**

   <details><summary>Model answer</summary>

   The number of distinct time series a metric produces, which is the product of the cardinality of each of its labels. A counter with twenty endpoint values, five methods and eight status codes produces eight hundred series. The key property is that it multiplies rather than adds — each new label multiplies the total by that label's value count — so adding one dimension with a large value set can increase series count by orders of magnitude.

   </details>

2. **[Basic] Why are user identifiers bad labels?**

   <details><summary>Model answer</summary>

   Because their value set grows without bound. A service with two million users produces two million distinct values for that label, and since cardinality multiplies, a metric that had eight hundred series now has over a billion. Each series carries index, memory and storage overhead, so this typically exhausts the metrics backend's memory and collapses query performance — meaning the monitoring system fails just when someone needs it to diagnose the problem the change caused.

   </details>

3. **[Senior] Where should high-cardinality questions go?**

   <details><summary>Model answer</summary>

   To logs and traces, which are built for them. The impulse to add a customer identifier to a metric is really a per-entity question being asked of an aggregate tool, and even if the cardinality were free it would give a poor answer — counts per customer rather than what actually happened. Structured logs filter on a customer identifier cheaply and carry the detail that answers the question; traces do the same with causal structure. Metrics exist for bounded aggregates, and recognising the misplacement resolves the cardinality problem and produces a better answer at the same time.

   </details>

4. **[Senior] Why are histograms especially cardinality-sensitive?**

   <details><summary>Model answer</summary>

   Because each bucket is a separate series for every label combination. A histogram with twelve buckets produces twelve times as many series as a counter with the same labels, so a metric at eight hundred series as a counter is nearly ten thousand as a histogram. That makes adding a label to a histogram roughly an order of magnitude more expensive than adding it to a counter — and histograms are exactly the metrics teams most want to break down by endpoint, outcome and region. The bucket count therefore has to be in the arithmetic from the outset, which usually means histograms carry deliberately fewer labels than counters do.

   </details>

5. **[Staff] How would you prevent cardinality incidents in an organisation?**

   <details><summary>Model answer</summary>

   Three layers, because discipline alone is insufficient. First, conventions that remove the common mistakes: route templates rather than raw paths, error types rather than raw messages, no identifiers in labels, and an explicit rule that per-entity questions go to logs or traces. Second, enforcement at ingestion — a per-metric series limit that drops or aggregates beyond a threshold — because a single careless deployment should degrade one metric rather than take down the monitoring system for everyone, and this is the only protection that does not depend on someone remembering. Third, alerting on series-count growth per metric, which catches a bad change within minutes rather than after memory exhaustion makes the metrics backend the incident. I would also treat label additions as schema changes in code review, since the cost is multiplicative and entirely invisible at the call site — one line that looks trivial and can generate a billion series is precisely the shape of change that needs a second pair of eyes. And for histograms specifically, I would require the arithmetic to be stated, because bucket multiplication makes them the most expensive place to add a dimension.

   </details>

6. **[Principal] Why does cardinality remain a recurring problem despite being well understood?**

   <details><summary>Model answer</summary>

   Because of an asymmetry between how the mistake is made and how it is felt. Adding a label is one word in one line of code, it looks obviously useful, it works perfectly in development where there are three users, and nothing in the local context indicates cost. The consequence appears only at production scale, arrives all at once on deployment, and manifests as the monitoring system degrading — which is both a large blast radius and the worst possible timing, since the tool needed to diagnose it is the tool that broke. That combination of low local friction and high remote cost is not something individual discipline reliably solves, no matter how well the principle is understood, because the person adding the label is usually not thinking about cardinality at all; they are thinking about a question they want answered. So the interventions that actually work are structural: ingestion limits that bound the damage, growth alerts that shorten detection, and conventions specific enough to be applied without thought — template the route, enumerate the error, identifiers go to logs. The educational framing that seems to stick best is the last one, because it reframes the impulse rather than forbidding it: if a label would have many values, you are asking a per-entity question, and there is a better tool for it.

   </details>

---

### Actionable alerting and burn rates

*Page only on user-visible impact, using multi-window burn rates so severity determines urgency and nobody is woken for something that does not need them.*

**Flow:** `Symptom detection` → `Burn rate windows` → `Severity routing` → `Page or ticket` → `Actionable response`

> **The 30-second version**  
> Page only when users are affected and someone must act now, using paired short and long burn-rate windows so urgency is computed — and delete every alert that has not led to action.

**The problem**

An on-call engineer receives two hundred alerts a week. Most require no action: a brief CPU spike, a queue that drained on its own, a replica that restarted and recovered. They learn to dismiss alerts quickly, and three months later a genuine outage alert is dismissed with the same reflex because it looked like all the others.

The opposite failure coexists with it. Alerts are configured for the failures somebody once experienced, so a novel failure — a bug that returns wrong results with a success status, a dependency degrading slowly — produces no alert at all, and the first notification is a customer complaint.

> **Alert on symptoms, because symptoms catch causes you did not predict**  
> Cause-based alerting requires enumerating failure modes in advance, so it fires for anticipated causes that may not matter and stays silent for unanticipated ones that do. Symptom-based alerting fires when users are affected, whatever the reason — which covers novel failures by construction and, equally importantly, does not fire when a predicted cause occurs without user impact.

**Mental model**

An alert is a request for a human to act. If nothing needs doing, it should not exist. Severity should follow from how quickly action is required, and that is derived from how fast the error budget is being consumed.

1. **Symptom** — User-visible impact: errors, latency, unavailability.
2. **Burn rate** — How fast the error budget is being consumed relative to the objective window.
3. **Window** — The period over which burn is measured — short windows catch severity, long windows catch erosion.
4. **Routing** — Page for what needs immediate action, ticket for what needs action eventually.
5. **Runbook** — What the responder should do, without which the alert is only a notification of distress.

> **Alert fatigue is a safety failure, not an annoyance**  
> Every alert that requires no action trains responders to dismiss alerts. That training is indiscriminate — it applies to the real one too. A team receiving many low-value pages is measurably slower to respond to genuine incidents than a team receiving few, so noisy alerting does not merely waste time; it actively degrades the response capability it exists to provide.

**How it works**

**Burn rate: severity from budget consumption**

```text
ERROR BUDGET, 99.9% over 30 days = 43 minutes

BURN RATE = how fast budget is consumed relative to
the window
  rate 1    -> budget exactly exhausted at day 30
  rate 2    -> exhausted in 15 days
  rate 14.4 -> exhausted in ~2 days
  rate 100  -> exhausted in ~7 hours

MULTI-WINDOW ALERTING
  FAST BURN   14.4x over 1 hour   -> PAGE
    budget gone in 2 days if sustained
    severe enough to need someone now
  MEDIUM      6x over 6 hours     -> PAGE
    catches slower but still serious degradation
  SLOW BURN   1x over 3 days      -> TICKET
    gradual erosion; needs work, not a wake-up

WHY SHORT AND LONG WINDOWS TOGETHER
  short window alone -> fires on brief blips
  long window alone  -> slow to detect a severe outage
  require BOTH to be breaching before paging
    -> a 5-minute spike does not page
    -> a sustained severe degradation pages quickly

WHY THIS BEATS A STATIC THRESHOLD
  "error rate > 1%" fires identically whether it is
    a 30-second blip or a 6-hour outage
  burn rate encodes DURATION and SEVERITY together
  -> urgency comes out of the arithmetic rather than
     from someone's guess
```

1. **Page only for things needing immediate human action** — If the response is to look and go back to sleep, it was not a page.
2. **Use paired short and long windows** — Requiring both to breach removes brief spikes while keeping fast detection of sustained problems.
3. **Route by urgency, not by severity of the metric** — A slow burn is a real problem that needs a ticket, not a wake-up.
4. **Attach a runbook to every page** — An alert without a documented response is an interruption, not a call to action.
5. **Delete alerts that never lead to action** — A quarterly review of which alerts produced changes is the cheapest noise reduction available.
6. **Alert on the absence of signal too** — Traffic dropping to zero means clients cannot reach you, and no error-rate alert will fire.

**What to page on, and what not to**

```text
PAGE  (immediate action required)
  user-visible errors burning budget fast
  latency breaching the objective, sustained
  complete unavailability
  traffic dropped to zero
  data loss or corruption detected
  -> all symptoms; all require someone now

TICKET  (action required, not urgently)
  slow budget erosion
  saturation trending toward a limit
  certificate expiring in two weeks
  a dependency degraded but handled by fallback
  -> real problems, no emergency

DO NOT ALERT AT ALL
  a single instance restarted and recovered
  CPU spike with no user impact
  a retry succeeded
  a queue grew and drained
  -> these belong on dashboards, not in anyone's night

THE TEST FOR EVERY ALERT
  1  is a human needed?
  2  is it needed NOW?
  3  is there something specific to do?
  all three yes -> page
  1 and 3 only  -> ticket
  otherwise     -> delete it

GRADUAL DEGRADATION IS THE HARD CASE
  latency creeping from 100 ms to 400 ms over a week
  -> no threshold crossing on any single day
  -> slow-burn alerting catches it; static thresholds
     do not
```

> **Alerting on causes misses the failures nobody anticipated**  
> Cause-based alerts are written after incidents, so the set of alerts describes the failures already experienced. A novel failure — wrong results returned with success status codes, a slow memory leak, a dependency returning stale data — matches none of them and produces silence. Symptom-based alerting covers these by construction, because any failure that matters eventually manifests as user impact.

**Worked example**

An alerting configuration for a checkout service with a 99.9 per cent objective.

**Alert definitions and routing**

```text
OBJECTIVE  99.9% of checkouts succeed within 3 s
           over 30 days -> 43 minutes of budget

PAGE - FAST BURN
  condition: burn rate > 14.4x over 1 h
             AND > 14.4x over 5 min
  meaning:   budget exhausted in ~2 days
  runbook:   check payment provider status, recent
             deploys, database saturation
  -> the paired windows mean a 2-minute spike does
     not wake anyone

PAGE - TOTAL FAILURE
  condition: checkout success rate < 50% for 5 min
  -> catastrophic; does not wait for burn arithmetic

PAGE - NO TRAFFIC
  condition: checkout requests = 0 for 5 min during
             business hours
  -> clients cannot reach us; no error alert would
     ever fire

TICKET - SLOW BURN
  condition: burn rate > 1x over 3 days
  -> budget will be exhausted this month; needs work

TICKET - SATURATION
  condition: database connection pool > 80% for 30 min
  -> leading indicator, no user impact yet

NOT ALERTED
  individual instance restarts
  transient provider retries that succeeded
  cache hit ratio changes
  -> visible on dashboards; nobody is woken

REVIEW
  quarterly: for every page fired, did it lead to
  action? alerts with no actions in a quarter are
  deleted, not tuned.
```

| Metric | Value | Note |
|---|---|---|
| Fast burn | 14.4× / 1 h | **pages** |
| Slow burn | 1× / 3 days | ticket |
| No traffic | zero for 5 min | silent outage |
| Review | quarterly | delete unactioned |

> **Alerting on zero traffic catches outages no error metric can**  
> If a load balancer misroutes, DNS fails, or a network path breaks, requests never reach the service — so error rate stays at zero, latency looks perfect, and every dashboard is green while nobody can use the product. An alert on the absence of expected traffic is the only thing that fires, and it is routinely missing because the failure mode is the inverse of what alerting usually anticipates.

**When to use it**

- **Any service with defined objectives**, where burn rate makes urgency computable.
- **On-call rotations**, where alert quality determines both response capability and sustainability.
- **Replacing threshold alerts**, which fire on brief spikes and miss gradual degradation.
- **Distinguishing urgency levels**, routing immediate action to pages and eventual action to tickets.
- **Detecting slow erosion**, which no static threshold catches.

**When to avoid it**

- **Do not page on causes**, which fire without impact and miss unanticipated failures.
- **Do not page for anything without a defined response**, which is an interruption rather than an alert.
- **Do not use single-window burn rates**, which either fire on spikes or detect too slowly.
- **Do not keep alerts that never lead to action**, since each one degrades response to the ones that do.
- **Do not alert on every anomaly**, when most anomalies are normal variance.

**Advantages**

- **Catches unanticipated failures**, because any failure that matters shows up as user impact.
- **Urgency derived arithmetically** from budget consumption rather than from guesswork.
- **Paired windows suppress spikes** while retaining fast detection of real degradation.
- **Detects gradual erosion** that static thresholds never cross.
- **Fewer, higher-quality alerts**, which measurably improves response to real ones.
- **Directly tied to the objective**, so alerting and reliability targets agree.

**Disadvantages**

- **Requires well-defined objectives and good indicators**, without which burn rate is meaningless.
- **Symptom alerts say what, not why**, so diagnosis still requires other signals.
- **Burn-rate arithmetic is less intuitive** than a threshold and needs explaining.
- **Low-traffic services** produce noisy rate calculations.
- **Some causes genuinely warrant alerts**, such as expiring certificates with no symptom until failure.

**Trade-offs**

**Alerting approach trade-offs**

| Approach | Catches novel failures | Noise | Detection speed |
|---|---|---|---|
| Static thresholds | Poorly | High | Immediate but spiky |
| Cause-based | No | High | Varies |
| Single-window burn rate | Yes | Moderate | Either spiky or slow |
| Multi-window burn rate | Yes | Low | Fast for severe, patient for minor |
| Anomaly detection | Sometimes | Often high | Variable |

Some cause-based alerts remain justified: a certificate expiring in two weeks has no symptom until it produces a total outage, so waiting for user impact is not an option. The rule is that cause alerts should be reserved for conditions where the symptom arrives too late to act on.

**How it fails**

**Alerting failures**

| Failure | Cause | Fix |
|---|---|---|
| Real alert dismissed | Fatigue from noisy alerts | Delete unactionable alerts; page only on impact |
| Novel failure produced no alert | Cause-based alerting only | Symptom-based alerts on objectives |
| Paged for a two-minute blip | Single short window | Require both short and long windows to breach |
| Severe outage detected slowly | Only long-window alerting | Add a fast-burn window |
| Gradual degradation missed | Static thresholds never crossed | Slow-burn alerting |
| Total outage with no alert | No alert on absence of traffic | Alert when traffic drops to zero |
| Responder does not know what to do | No runbook attached | Every page links to a documented response |

**Limits**

> **Burn rate reference**
>
> - **14.4× over 1 hour** exhausts a 30-day budget in about 2 days — a standard fast-burn page threshold.
> - **6× over 6 hours** catches serious but slower degradation.
> - **1× over 3 days** indicates the budget will be exhausted within the window — a ticket, not a page.
> - **Paired windows**: require both a short and a long window to breach before paging.
> - **Review cadence**: quarterly, deleting alerts that produced no action.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Multi-window burn rate | Objective-backed services | Needs good indicators |
| Static thresholds | Simple bounded conditions | Spiky; misses gradual change |
| Cause-based alerts | Conditions with no timely symptom | Misses novel failures |
| Anomaly detection | Unknown normal behaviour | False positives; hard to tune |
| Synthetic monitoring | End-to-end verification | Not real user traffic |
| Human monitoring | Nothing at scale | Does not work |

Synthetic checks complement symptom alerting usefully at low traffic, where real-traffic rates are too noisy to compute burn rates from — a steady stream of probe requests gives a reliable signal when genuine volume does not.

**In real systems**

- **Multi-window multi-burn-rate alerting** is the current standard, replacing static thresholds because it encodes severity and duration in one condition.
- **Symptom-based paging with cause-based dashboards** separates what wakes people from what explains the problem once they are awake.
- **Runbooks linked from every alert** are standard practice, since a page without a defined response is an interruption rather than a call to action.
- **Alerting on absence of traffic** catches routing, DNS and network failures that no error-rate metric can detect.
- **Periodic alert review deleting unactioned alerts** is how mature teams control fatigue, treating deletion rather than tuning as the default.

**Common mistakes**

- **Paging on causes**, firing without impact and missing novel failures.
- **Single-window thresholds**, either spiky or slow.
- **Alerts with no runbook**, leaving responders without a defined action.
- **Keeping alerts that never produce action**, training people to dismiss.
- **No slow-burn alerting**, missing gradual degradation entirely.
- **No alert on zero traffic**, so a routing failure looks perfectly healthy.
- **Tuning noisy alerts** rather than deleting them.

**The staff-level view**

Alert quality is an operational capability, and it degrades by accretion unless something actively removes alerts.

- **Treat fatigue as a safety issue.** A team receiving many low-value pages responds measurably slower to genuine incidents, so noisy alerting degrades the very capability it exists to provide.
- **Page on symptoms, dashboard the causes.** Cause alerts enumerate the failures you have already had and are silent for the ones you have not, which is exactly backwards for the purpose.
- **Require a runbook for every page.** If nobody can say what the responder should do, the alert is a notification of distress and should be a ticket or nothing.
- **Delete rather than tune.** An alert that produced no action in a quarter is not mistuned; it is measuring something nobody acts on, and deletion is the cheapest available improvement.
- **Add an alert for traffic disappearing.** It is the one outage that produces perfect-looking error and latency metrics, and it is almost always missing.

**Go deeper**

Alerting should page on symptoms rather than causes, because cause-based alerts enumerate the failures already experienced and stay silent for novel ones, while also firing for predicted causes that produce no user impact. Symptom alerts fire when users are affected regardless of the reason, covering unanticipated failures by construction.

Burn rate — how fast the error budget is being consumed relative to the objective window — encodes severity and duration in a single number, so urgency follows from arithmetic rather than from a guessed threshold. Pairing a short and a long window, and requiring both to breach, suppresses brief spikes while still detecting sustained severe degradation within minutes; a separate slow window catches gradual erosion and routes it to a ticket.

The governing constraint is fatigue, which is a safety issue rather than an annoyance: teams trained by low-value alerts to dismiss quickly respond measurably slower to genuine incidents. That makes deletion the primary tool — an alert producing no action in a quarter is not mistuned — along with requiring a runbook for every page, and adding the alert almost always missing: traffic dropping to zero, which is the outage where every other metric looks perfect.

Alerting exists to bring a human into the loop when one is needed, so every alert should correspond to an action somebody takes.

**Symptoms cover what causes cannot.** Cause-based alerting requires anticipating failure modes, which means the alert set describes failures already experienced. A novel failure — wrong results returned with success status codes, gradual staleness in a dependency, a slow leak — matches nothing and produces silence, while predicted causes that happen to have no user impact produce noise. Alerting on user-visible symptoms inverts both properties: any failure that matters eventually shows up as errors or latency, and conditions that do not affect users do not fire.

**Burn rate turns urgency into arithmetic.** Expressing consumption relative to the objective window means a single number captures both how bad a degradation is and how long it can be tolerated: a rate of one exhausts the budget exactly at the window's end, a rate of fourteen in a couple of days. A static threshold like a one per cent error rate treats a thirty-second blip and a six-hour outage identically, whereas burn rate distinguishes them without anyone having to guess what number feels serious.

**Paired windows resolve the noise-versus-speed dilemma.** A short evaluation window detects severe problems quickly and also fires on transient spikes; a long one ignores spikes and is slow when the problem is real. Requiring both to breach simultaneously before paging keeps the fast detection while discarding the noise, and configuring a second, much longer window at a low rate catches slow erosion that no threshold would ever cross — routing it to a ticket, because gradual budget consumption is work rather than an emergency.

**Fatigue degrades the capability alerting provides.** Every page that requires no action trains the responder to acknowledge and move on, and that learned response is indiscriminate: it applies to the genuine outage too. Teams with high alert volume respond measurably more slowly to real incidents, which means noisy alerting is not merely inefficient but actively harmful to incident response. Framing it this way matters, because it makes removing alerts an improvement in coverage rather than a reduction of it.

**Deletion beats tuning.** An alert that has produced no action over a quarter is not badly calibrated — it is measuring a condition nobody responds to, and adjusting its threshold merely narrows when the useless interruption occurs. A recurring audit against three questions — is a human needed, is one needed now, is there something specific to do — sorts every alert into page, ticket or delete, and typically removes a large majority. The runbook requirement performs similar work indirectly, because attempting to document the response frequently reveals that no response exists.

**The missing alert is usually absence of traffic.** When a load balancer misroutes, DNS fails, or a network path breaks, requests never arrive: error rate stays at zero, latency is perfect, saturation is low, and every dashboard is green while the product is entirely unusable. No symptom-based alert built on error or latency will fire, because there are no requests to measure. Alerting on expected traffic failing to appear is the only detection available, and it is routinely absent precisely because it inverts the assumption that failures produce bad numbers rather than no numbers.

**Prove it — interview questions**

1. **[Basic] Why alert on symptoms rather than causes?**

   <details><summary>Model answer</summary>

   Because cause-based alerts require enumerating failure modes in advance, so they cover the failures you have already experienced and stay silent for the ones you have not. They also fire when a predicted cause occurs without any user impact, which is noise. Symptom-based alerts fire when users are actually affected, whatever the reason — which catches novel failures by construction and does not fire when nothing that matters is happening.

   </details>

2. **[Basic] What is a burn rate?**

   <details><summary>Model answer</summary>

   How fast the error budget is being consumed relative to the objective window. A burn rate of one means the budget will be exactly exhausted at the end of the period; a rate of fourteen means a thirty-day budget disappears in about two days. It is useful for alerting because it encodes severity and duration in a single number, so urgency comes out of the arithmetic rather than from someone guessing what threshold feels serious.

   </details>

3. **[Senior] Why use multiple windows?**

   <details><summary>Model answer</summary>

   Because each window alone fails in a different direction. A short window fires on brief spikes that resolve themselves, producing noise; a long window is slow to detect a severe outage, so the response is late. Requiring both a short and a long window to be breaching before paging gives both properties: a two-minute spike does not satisfy the long window and so does not page, while a sustained severe degradation satisfies both quickly and pages promptly. A separate slower window catches gradual erosion and routes it to a ticket rather than a wake-up.

   </details>

4. **[Senior] Why is alert fatigue a safety issue?**

   <details><summary>Model answer</summary>

   Because the dismissal reflex it trains is indiscriminate. An engineer receiving two hundred alerts a week, most requiring no action, learns to acknowledge and move on quickly — and that learned response applies equally to the genuine outage, which looks like all the others at first glance. Teams with noisy alerting are measurably slower to respond to real incidents, so the noise is not merely wasted time; it actively degrades the response capability the alerting exists to provide. That reframing matters because it makes deleting alerts a safety improvement rather than a convenience.

   </details>

5. **[Staff] Design alerting for a checkout service with a 99.9 per cent objective.**

   <details><summary>Model answer</summary>

   Symptom-based, built on the objective. A fast-burn page when the burn rate exceeds roughly fourteen times over an hour and simultaneously over a five-minute window — the pairing is what stops a brief spike from waking anyone while still detecting severe degradation within minutes. A separate page for catastrophic failure, say success rate below half for five minutes, which should not wait for burn arithmetic. A page for checkout traffic dropping to zero during business hours, because that is the outage where error rate and latency both look perfect and nothing else would fire. A slow-burn ticket when the budget is on track to be exhausted within the month, which is real work but not a wake-up, and a saturation ticket on connection pool utilisation as a leading indicator. Deliberately not alerted: instance restarts, successful retries, cache ratio changes — those live on dashboards. Every page links to a runbook naming the first things to check, because a page without a defined response is an interruption. And a quarterly review asking, for each alert, whether it led to any action; those that did not get deleted rather than tuned, because an alert nobody acts on is training people to ignore the ones they should.

   </details>

6. **[Principal] How would you improve alerting in an organisation with severe fatigue?**

   <details><summary>Model answer</summary>

   By deleting, which is counterintuitive enough that it usually needs framing. The instinct when alerts are noisy is to tune thresholds, but an alert that has produced no action in a quarter is not mistuned — it is measuring something nobody responds to, and adjusting its threshold preserves the interruption while narrowing when it fires. So the first exercise is auditing every alert against three questions: is a human needed, is one needed now, and is there something specific to do. Anything failing the first or third is deleted outright; anything failing only the second becomes a ticket. That typically removes a large majority of alerts immediately. Then rebuild what pages from symptoms tied to objectives, using multi-window burn rates so that urgency is computed rather than asserted, and require a runbook for every remaining page — the runbook requirement alone eliminates several alerts, because the exercise of writing down what the responder should do reveals that nobody knows. The framing that makes this acceptable to people reluctant to remove monitoring is the safety argument: a team trained by noise to dismiss alerts responds slower to the real one, so reducing alert count is improving incident response rather than reducing coverage. And the practice has to be recurring, because alerting accretes — every postmortem adds an alert and nothing removes them unless a review does.

   </details>

---

### Profiling and resource attribution

*Measure where CPU, memory and time are actually consumed inside a process, so optimisation targets what dominates rather than what seems slow.*

**Flow:** `Sampled stacks` → `Aggregated profile` → `Flame graph` → `Attributed cost` → `Verified improvement`

> **The 30-second version**  
> Sample where the process actually spends CPU, time and memory, read self time rather than total, match the profile type to the symptom, and optimise strictly in order of share.

**The problem**

A service uses more CPU than expected. The team reviews the code, identifies a loop that looks inefficient, spends a week optimising it, deploys, and observes no measurable change — because that loop accounted for two per cent of CPU time and the actual cost was in serialisation happening in a library nobody had considered.

Intuition about where programs spend time is famously unreliable. Cost concentrates in unexpected places: logging, allocation, reflection, string formatting, cryptographic operations, and serialisation routinely dominate profiles in code that appears to be doing something else entirely.

> **Measure before optimising, because the bottleneck is rarely where you think**  
> A profile attributes resource consumption to actual call paths, which regularly contradicts expert intuition. Optimising without one means spending effort on code that looks expensive rather than code that is expensive — and the characteristic outcome is a week of work producing no measurable improvement, which is both wasted and demoralising.

**Mental model**

A profiler samples what the program is doing many times per second and aggregates those samples by call stack. Frequency of appearance approximates proportion of resource consumed, and the aggregate shows which paths dominate.

1. **Sample** — A snapshot of the call stack at a moment in time.
2. **Aggregation** — Combining samples so frequency approximates cost share.
3. **Self versus total** — Time in a function itself versus time in it and everything it calls.
4. **Flame graph** — A visualisation where width is cost and stacking shows call relationships.
5. **Attribution** — Connecting consumption to code paths, and ideally to the requests that triggered them.

> **Optimising without a profile usually achieves nothing measurable**  
> Amdahl's constraint is unforgiving: making a component twice as fast improves overall performance only in proportion to that component's share of the total. Halving the cost of something responsible for two per cent yields a one per cent improvement — indistinguishable from noise. Effort spent without knowing the share is effort spent at random, and the frequency with which experienced engineers guess wrong is the strongest argument for profiling first.

**How it works**

**Reading a profile correctly**

```text
SELF TIME vs TOTAL TIME
  function A calls B calls C
  A total 100%, self 2%
  B total 98%, self 5%
  C total 93%, self 93%
  -> C is where the work happens
  -> optimising A is pointless despite its 100% total

COMMON MISREADING
  looking at TOTAL and concluding the top frame is the
  problem
  -> the entry point always has 100% total time
  -> self time is what identifies the actual consumer

FLAME GRAPH
  x-axis  = proportion of samples (NOT time order)
  y-axis  = stack depth
  width   = cost
  -> look for WIDE frames, at any depth
  -> a wide plateau near the top is a hot leaf function

WHAT PROFILES ROUTINELY REVEAL
  serialisation and deserialisation      often dominant
  logging, especially string formatting  surprisingly large
  memory allocation and GC pressure      frequently top 3
  reflection in frameworks               invisible in source
  cryptographic operations               TLS, hashing
  -> none of these are where people look first

AMDAHL ARITHMETIC
  component at 40% of CPU, made 2x faster
    -> 20% total improvement
  component at 2% of CPU, made 10x faster
    -> 1.8% total improvement
  -> the SHARE matters far more than the speedup
```

1. **Profile in production, or with production-like load** — Development workloads have different data shapes, cache behaviour and concurrency, and frequently profile differently.
2. **Use continuous low-overhead profiling** — Capturing profiles only during incidents means having none when the interesting behaviour occurred.
3. **Distinguish self time from total time** — The entry point always has the highest total; self time identifies the actual consumer.
4. **Profile the right resource** — CPU profiles say nothing about a service blocked on I/O; wall-clock or off-CPU profiling is needed there.
5. **Attribute to requests or tenants where possible** — Knowing which endpoint or customer caused the consumption turns a profile into an actionable target.
6. **Verify improvements by profiling again** — An optimisation that looks better in isolation may have moved cost rather than removed it.

**Choosing what to profile**

```text
CPU PROFILE
  where compute time goes
  -> useless if the service is blocked on I/O:
     a thread waiting on a database consumes no CPU
     and appears nowhere

WALL-CLOCK / OFF-CPU PROFILE
  where elapsed time goes, including waiting
  -> shows lock contention, I/O waits, queueing
  -> usually the right choice for request latency

MEMORY / ALLOCATION PROFILE
  what is allocating and what is retained
  allocation profile -> GC pressure and churn
  heap profile       -> leaks and retention
  -> allocation rate often drives CPU cost invisibly

LOCK CONTENTION PROFILE
  where threads wait on each other
  -> explains the common case of high latency at
     modest CPU

THE DIAGNOSTIC QUESTION FIRST
  symptom: high CPU            -> CPU profile
  symptom: high latency, low CPU -> off-CPU / lock
  symptom: memory growth       -> heap profile
  symptom: GC pauses           -> allocation profile
  -> picking the wrong profile type produces a clean
     report that explains nothing
```

> **A CPU profile of an I/O-bound service explains nothing**  
> Threads blocked waiting on a database or a network call consume no CPU, so they contribute no samples and appear nowhere in a CPU profile. A service spending ninety per cent of its request latency waiting will produce a CPU profile that looks entirely healthy, because it is — the CPU is fine and the problem is elsewhere. Matching the profile type to the symptom is the first decision, and getting it wrong produces a clean report that explains nothing.

**Worked example**

Investigating a service consuming more CPU than its request volume suggests.

**Profiling investigation**

```text
SYMPTOM  CPU at 70% at 2,000 req/s; expected ~30%

STEP 1  choose the profile type
  symptom is high CPU -> CPU profile
  (if latency were high with low CPU, this would be
   an off-CPU profile instead)

STEP 2  capture under production load
  continuous profiler, 100 Hz sampling
  overhead well under 1%

STEP 3  read self time, not total

  FINDINGS (self time)
    JSON serialisation            34%
    log string formatting         18%
    TLS handshake                 12%
    business logic                 9%
    database driver                8%
    everything else               19%

STEP 4  interpret
  business logic is 9% of CPU
  -> optimising it, even perfectly, cannot help much
  serialisation + logging = 52%
  -> this is where the CPU went, and nobody suspected
     either

STEP 5  act, in order of share
  logging: the formatting ran even for DEBUG lines
    that were then discarded
    -> guard formatting behind the level check
    -> 18% -> ~1%
  serialisation: a faster library and avoiding
    re-serialising unchanged responses
    -> 34% -> 20%
  TLS: connection reuse rather than new handshakes
    -> 12% -> 4%

STEP 6  verify by re-profiling
  CPU 70% -> 34%
  -> and the new profile shows a different top
     consumer, which is expected: optimisation moves
     the bottleneck rather than removing it
```

| Metric | Value | Note |
|---|---|---|
| Suspected | business logic | 9% actual |
| Actual | serialisation + logging | **52%** |
| Result | 70% → 34% CPU | verified |
| Lesson | profile first | intuition was wrong |

> **Discarded log lines can still cost a fortune**  
> String formatting typically happens before the level check, so a debug line that will be thrown away still pays for its interpolation, allocation and concatenation. In hot paths this can consume a substantial share of CPU producing output nobody ever sees. Guarding formatting behind the level check, or using structured logging that defers formatting, is one of the most common large wins a first profile reveals.

**When to use it**

- **Before any optimisation work**, to establish where cost actually is.
- **Investigating high CPU or memory consumption**, where profiles attribute it directly.
- **Diagnosing latency with low CPU**, using off-CPU and lock profiling.
- **Capacity planning**, since knowing the cost distribution predicts scaling behaviour.
- **Verifying optimisations**, by re-profiling rather than trusting a benchmark.

**When to avoid it**

- **Do not optimise without profiling first**, which spends effort at random.
- **Do not use CPU profiles for I/O-bound problems**, where the waiting is invisible.
- **Do not profile only in development**, where data shapes and concurrency differ.
- **Do not read total time as the answer**, since the entry point always dominates it.
- **Do not profile only during incidents**, which means having no profile of normal behaviour to compare against.

**Advantages**

- **Attributes cost to actual code paths**, replacing intuition with measurement.
- **Reveals unexpected consumers**, which is the common case rather than the exception.
- **Low overhead when sampled**, making continuous production profiling viable.
- **Directs effort by share**, which is what determines achievable improvement.
- **Verifies improvements** and shows where the bottleneck moved.

**Disadvantages**

- **Sampling misses very brief events**, which can matter for latency spikes.
- **Requires choosing the right profile type**, and the wrong one explains nothing.
- **Inlining and optimisation can obscure attribution** in compiled languages.
- **Aggregate profiles hide per-request variation**, so a rare expensive path can be invisible.
- **Interpretation requires experience**, particularly distinguishing self from total time.

**Trade-offs**

**Profile type selection**

| Symptom | Profile type | Reveals |
|---|---|---|
| High CPU | CPU profile | Compute-heavy call paths |
| High latency, low CPU | Off-CPU / wall-clock | I/O waits, lock contention, queueing |
| Memory growth | Heap profile | What is retained and by whom |
| Frequent GC pauses | Allocation profile | Allocation churn sources |
| Threads idle but slow | Lock contention profile | Synchronisation bottlenecks |

Choosing the type from the symptom is the first and most consequential decision. A clean CPU profile of an I/O-bound service is not evidence that nothing is wrong; it is evidence that the wrong question was asked.

**How it fails**

**Profiling failures**

| Failure | Cause | Fix |
|---|---|---|
| Optimisation produced no improvement | Optimised a small share of total cost | Profile first; target by share |
| Profile explains nothing | Wrong profile type for the symptom | Match type to symptom |
| Production behaviour differs from profile | Profiled in development | Profile under production load |
| No profile when needed | Profiling only enabled during incidents | Continuous low-overhead profiling |
| Wrong function identified | Read total time rather than self time | Use self time to find consumers |
| Rare expensive path invisible | Aggregate profile only | Per-request or per-endpoint attribution |
| Improvement did not persist | Bottleneck moved elsewhere | Re-profile after each change |

**Limits**

> **Practical figures**
>
> - **Sampling rate**: around 100 Hz is typical, with overhead usually well under 1%.
> - **Amdahl**: improvement is bounded by the component's share — halving 2% of cost yields 1%.
> - **Profile types** must match the symptom; a CPU profile is blind to blocked threads.
> - **Continuous profiling** is what ensures a profile exists for the period that mattered.
> - **Verification**: re-profile after changes, since optimisation moves the bottleneck rather than eliminating it.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Sampling profiler | General resource attribution | Misses very brief events |
| Instrumenting profiler | Exact call counts | High overhead; distorts behaviour |
| Distributed tracing | Cross-service attribution | Not within-process detail |
| Metrics | Aggregate resource trends | No attribution to code |
| Benchmarks | Comparing implementations | Does not reflect production behaviour |
| Load testing with profiling | Pre-production investigation | Synthetic load shapes differ |

Tracing and profiling are complementary at different scales: a trace identifies which service and which span consumed the time, and a profile of that service explains what inside it did. Using them together moves an investigation from an unexplained latency figure to a specific function in a few steps.

**In real systems**

- **Continuous production profiling** has become standard practice, since profiles captured only during incidents miss the period of interest.
- **Flame graphs** are the common visualisation because width directly represents cost share, making dominant paths immediately visible.
- **Serialisation, logging and allocation** are consistently among the largest consumers in service profiles, despite rarely being suspected.
- **Off-CPU profiling** addresses the large class of latency problems where threads are waiting rather than computing.
- **Request-level attribution** connects resource consumption to specific endpoints or tenants, which turns a profile into a prioritised list.

**Common mistakes**

- **Optimising without profiling**, targeting code that looks expensive rather than is.
- **CPU profiling an I/O-bound service**, producing a report that explains nothing.
- **Profiling in development only**, where data and concurrency differ.
- **Reading total time as the answer**, pointing at entry points that consume nothing.
- **Profiling only during incidents**, so no profile exists for the period of interest.
- **Ignoring allocation cost**, which drives CPU through garbage collection invisibly.
- **Not re-profiling after a change**, missing that the bottleneck simply moved.

**The staff-level view**

Optimisation without measurement is the most reliable way to spend a week and change nothing.

- **Require a profile before approving optimisation work.** The share of total cost bounds the achievable improvement, and intuition about that share is wrong often enough that the requirement pays for itself immediately.
- **Match the profile type to the symptom.** A CPU profile of an I/O-bound service produces a clean report that explains nothing, which is worse than no report because it looks conclusive.
- **Run continuous profiling in production.** Enabling a profiler during an incident gives you data about the incident's aftermath, not about the conditions that caused it.
- **Teach self time versus total time.** Misreading total time as the answer is the most common interpretation error, and it points investigations at entry points that consume nothing.
- **Re-profile after changes.** Optimisation moves the bottleneck rather than removing it, and the new profile is what tells you whether another round is worth it.

**Go deeper**

A profiler samples the call stack many times per second and aggregates by path, so the frequency a function appears approximates its share of resource consumption. Sampling overhead is low enough to run continuously in production, which matters because a profile captured after an incident begins describes its aftermath rather than the conditions that caused it.

Two reading errors are common. Total time includes everything a function calls, so the entry point always dominates it — self time is what identifies the actual consumer. And the profile type must match the symptom: a CPU profile is blind to threads blocked on I/O, so a service spending most of its latency waiting produces a clean CPU profile that explains nothing, which is worse than no profile because it looks conclusive.

Optimisation should follow share, because achievable improvement is bounded by it — a tenfold speedup on two per cent of total cost is unmeasurable. Profiles routinely show serialisation, log string formatting, allocation and cryptographic operations dominating while business logic is a single-digit percentage, which is why measuring first reliably prevents a week of careful work producing no detectable change.

Profiling attributes resource consumption to actual code paths, replacing intuition about where a program spends its time with measurement of where it does.

**Sampling makes continuous production profiling viable.** Capturing the call stack at a modest frequency and aggregating by path gives a good approximation of cost distribution at overhead usually well under one per cent. That low cost is what permits running it always rather than enabling it during investigations — which matters, because a profiler started after an incident begins captures the aftermath, while the conditions that produced the problem have already passed.

**Self time identifies consumers; total time identifies ancestors.** Total time counts everything a function calls, so an entry point always registers near a hundred per cent while consuming nothing itself. Reading total as the answer is the most frequent interpretation error and it directs investigation at frames that merely delegate. Flame graphs make this legible by representing cost as width at every stack depth, so a wide plateau near the top of the graph is a hot leaf function regardless of what called it.

**Profile type must follow symptom.** CPU profiles account only for threads executing; a thread blocked on a database call consumes no CPU and contributes no samples. A service whose latency is dominated by waiting therefore produces a healthy-looking CPU profile, which is accurate and useless. Off-CPU and wall-clock profiling account for elapsed time including waits, exposing I/O, lock contention and queueing; allocation profiles explain garbage collection pressure; heap profiles explain retention. Choosing wrongly yields a clean report that appears conclusive, which is more damaging than having no report at all.

**Share bounds improvement, and intuition about share is unreliable.** Making a component twice as fast improves the whole in proportion to that component's contribution, so a tenfold improvement on two per cent of cost is unmeasurable. Profiles consistently show consumption concentrated in serialisation, logging, allocation churn, reflection inside frameworks and cryptographic operations — none of which are conspicuous in source code, because they are distributed thinly across every call site rather than gathered in one obviously expensive block. Meanwhile the loop that looks inefficient attracts attention precisely because it is legible as work.

**Some of the largest wins are incidental.** String formatting for log lines typically happens before the level check, so debug output that is immediately discarded still pays for interpolation, allocation and concatenation — in a hot path this can account for a substantial share of CPU producing text nobody will ever read. Connection reuse eliminating repeated handshakes, and avoiding re-serialisation of unchanged responses, are similarly unglamorous and similarly large. First profiles routinely surface at least one of these.

**Verification matters because optimisation relocates bottlenecks.** After a change, the profile will show a different dominant consumer, which is the expected outcome rather than a sign of failure — and it is the information needed to decide whether another round is worthwhile. Re-profiling also catches the case where cost was moved rather than removed, such as an optimisation that reduces CPU by increasing allocation, which shows up as garbage collection pressure elsewhere. Making measurement a precondition of optimisation work, and re-measurement a condition of declaring it finished, is a cheap procedural requirement that prevents the most demoralising outcome in performance engineering: careful work that changes nothing anyone can detect.

**Prove it — interview questions**

1. **[Basic] What does a profiler do?**

   <details><summary>Model answer</summary>

   It samples what the program is executing many times per second and aggregates those samples by call stack. Because a function appearing in more samples was running more of the time, the aggregate approximates how resource consumption is distributed across call paths. Sampling keeps overhead low enough — typically well under one per cent — that it can run continuously in production, which matters because profiles captured only during incidents describe the aftermath rather than the cause.

   </details>

2. **[Basic] Why profile before optimising?**

   <details><summary>Model answer</summary>

   Because the achievable improvement is bounded by the component's share of total cost, and intuition about that share is unreliable. Halving the time spent in something responsible for two per cent of CPU yields a one per cent improvement, which is indistinguishable from noise — so a week of careful work produces nothing measurable. Profiles routinely show that cost concentrates in serialisation, logging, allocation or cryptographic operations rather than in the business logic everyone was looking at.

   </details>

3. **[Senior] What is the difference between self time and total time?**

   <details><summary>Model answer</summary>

   Total time is everything spent in a function and everything it calls; self time is what was spent in that function's own code. The entry point therefore always has a total time near a hundred per cent, which makes total time useless for identifying the actual consumer — a request handler that immediately delegates has total time of everything and self time of nearly nothing. Self time is what points at the function actually doing the work. Misreading total as the answer is the most common interpretation error and it directs investigations at frames that consume nothing.

   </details>

4. **[Senior] Why is a CPU profile useless for an I/O-bound service?**

   <details><summary>Model answer</summary>

   Because a thread blocked waiting on a database or a network call consumes no CPU, produces no samples, and appears nowhere. A service spending ninety per cent of its request latency waiting will produce a CPU profile that looks entirely healthy — and it is healthy, from a CPU perspective. What is needed is an off-CPU or wall-clock profile that accounts for elapsed time including waiting, which reveals I/O waits, lock contention and queueing. Choosing the profile type from the symptom is the first decision, and the wrong choice produces a clean report that appears conclusive and explains nothing.

   </details>

5. **[Staff] How would you investigate a service using more CPU than expected?**

   <details><summary>Model answer</summary>

   Start by matching the profile type to the symptom — high CPU means a CPU profile, whereas high latency at low CPU would call for off-CPU or lock contention profiling instead. Capture under production load, ideally from a continuous profiler already running, because development workloads have different data shapes, cache behaviour and concurrency and frequently profile differently. Then read self time rather than total, because the entry frame always dominates total and tells you nothing. In my experience the result is usually surprising: serialisation, log string formatting, allocation churn and TLS handshakes are consistently among the top consumers while business logic is a single-digit percentage. Then act strictly in order of share, since a large speedup on a small share cannot help — and the log formatting case is often the easiest win, because formatting typically runs before the level check, so debug lines that are immediately discarded still pay for interpolation and allocation. Finally, re-profile to verify, expecting the top consumer to have changed: optimisation moves the bottleneck rather than eliminating it, and the new profile is what tells you whether another round is worthwhile.

   </details>

6. **[Principal] Why does intuition about performance fail so consistently?**

   <details><summary>Model answer</summary>

   Because the mental model people hold is of the code they wrote, and most of the cost is in code they did not. Serialisation, logging, allocation and garbage collection, reflection inside frameworks, cryptographic operations, string handling — none of these are visible as significant in the source, they are spread thinly across every call site rather than concentrated in one obviously expensive block, and they are precisely the things a reader's eye skips over as plumbing. Meanwhile the loop that looks inefficient attracts attention because it is legible as work. The compounding factor is Amdahl's constraint, which means being wrong about where the cost is does not merely reduce the improvement proportionally — it makes the work essentially worthless, because a tenfold speedup on two per cent of the total is unmeasurable. So the discipline I would enforce is procedural rather than analytical: no optimisation work without a profile showing the share, and re-profiling afterwards to verify. It is a cheap requirement that reliably prevents the most demoralising failure mode in performance work, which is a week of careful engineering that changes nothing anyone can detect.

   </details>

---
