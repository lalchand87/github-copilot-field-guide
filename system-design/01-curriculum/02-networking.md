# Curriculum · Networking

[← System Design index](../README.md)

> 8 lessons in **Networking**. Part of the Curriculum — Build the fundamentals.

## Contents

- **Networking** (8): [DNS resolution and caching](#dns-resolution-and-caching) · [TCP reliability and congestion](#tcp-reliability-and-congestion) · [TLS termination](#tls-termination) · [Layer 4 and layer 7 load balancing](#layer-4-and-layer-7-load-balancing) · [Connection pooling](#connection-pooling) · [HTTP/2 multiplexing](#http2-multiplexing) · [QUIC and HTTP/3](#quic-and-http3) · [Service discovery](#service-discovery)

## Networking

### DNS resolution and caching

*Names become addresses through a cache hierarchy you do not control — which makes DNS a fast failover tool in theory and a slow one in practice.*

**Flow:** `Client cache` → `Recursive resolver` → `Authoritative DNS` → `Address` → `Service`

> **The 30-second version**  
> DNS turns names into addresses through caches you do not control. Use it for naming and coarse steering, never for second-scale failover.

**The problem**

DNS is invisible until it breaks, and when it breaks everything breaks at once: no database connections, no service-to-service calls, no third-party APIs. It is also the layer engineers most often misunderstand, because they treat TTL as a promise when it is only a suggestion.

The specific trap: a team plans regional failover by “just updating DNS.” The TTL is 60 seconds, so they expect a minute of impact. In practice, some ISP resolvers ignore short TTLs, some JVMs cache resolutions forever by default, connection pools hold sockets to the old address, and a long tail of clients keeps hitting the dead region for hours.

> **The three caches you do not control**
>
> - **Client runtime**: the JVM historically cached successful lookups indefinitely (`networkaddress.cache.ttl`); many HTTP clients and connection pools never re-resolve at all.
> - **Recursive resolvers**: ISPs and public resolvers may clamp very low TTLs upward to reduce their own load.
> - **Operating systems and browsers**: separate caches with their own policies, plus connection reuse that bypasses resolution entirely.

**Mental model**

DNS is a globally distributed, aggressively cached, eventually consistent key-value store. Reads are cheap and stale; writes propagate on a timescale you can influence but not control.

1. **Stub resolver (the client)** — Asks a configured recursive resolver. Caches locally, and the application runtime may cache above that.
2. **Recursive resolver** — Does the actual walk on your behalf and caches results. This is where most cache hits happen.
3. **Root and TLD servers** — Delegate downward: roots know who serves `.com`, TLD servers know who is authoritative for `example.com`.
4. **Authoritative server** — Holds your records. The only place your change actually exists at the moment you make it.
5. **TTL** — Your *request* for how long each cache should hold the answer. Respected mostly, not always.

> **TTL is a trade between failover speed and query volume**  
> A 60-second TTL means every resolver re-asks every minute — more load, more exposure to authoritative outages, but fast propagation. A 24-hour TTL means near-zero query load and a day-long tail after any change. The right value depends on what you use DNS *for*: 30–60 s for anything in a failover path, hours for stable infrastructure.

**How it works**

**Resolution walk and where caching intervenes**

```text
app -> getaddrinfo()
  [runtime cache]      <- JVM/libc/Go resolver cache
  [OS cache]
     -> recursive resolver (8.8.8.8, ISP, corporate)
          [resolver cache]   <- most hits land here
             -> root server        "ask the .com servers"
             -> TLD server         "ask ns1.example.com"
             -> authoritative      "api.example.com = 10.2.3.4"
          <- cached for TTL
  <- cached for TTL (maybe)
app connects to 10.2.3.4
  [connection pool]    <- REUSES the socket; no re-resolution at all
```

1. **Record types you actually use** — `A`/`AAAA` map a name to an address. `CNAME` aliases one name to another (and cannot coexist with other records at the same name). `SRV` carries port and priority. `TXT` is used for verification and policy. `NS` delegates a zone.
2. **Health-checked DNS is coarse failover** — Providers can return only healthy endpoints, but the granularity is the TTL plus every cache above it. Treat it as minutes-scale, not seconds-scale.
3. **Anycast makes DNS itself resilient** — The same IP announced from many locations; BGP routes clients to the nearest. This is how root servers and public resolvers stay available.
4. **Negative caching is a separate policy** — `NXDOMAIN` is cached too, governed by the zone's SOA minimum. A record created after a failed lookup can be invisible for that duration.
5. **Connection pools defeat DNS entirely** — An open TCP connection does not re-resolve. Failover requires closing connections, not just changing records.

**Failover reality check**

```text
plan:    change DNS, 60s TTL, "one minute of impact"

reality:
  t+0s     authoritative updated
  t+60s    well-behaved resolvers refresh
  t+300s   resolvers that clamped the TTL refresh
  t+???    JVM with default caching NEVER refreshes
  t+???    long-lived connection pools keep using the old IP

what actually works for fast failover:
  - anycast / BGP withdrawal        (seconds, no DNS involved)
  - load balancer health checks     (seconds, within a region)
  - client-side service discovery   (seconds, explicit refresh)
  - DNS                             (minutes to hours, the tail)
```

> **Do not build second-scale failover on DNS**  
> Use DNS for *coarse* traffic steering — moving a region's share of traffic over minutes, or directing users to their nearest edge. For fast failover, use a layer that controls both ends: a load balancer that drains connections, anycast withdrawal, or a discovery mechanism whose clients you own.

**Worked example**

A service runs active-passive across two regions. Design the failover and be honest about the timeline.

**Layered failover, DNS as the outer ring**

```text
LAYER 1  within a region  (seconds)
  LB health check fails an instance -> removed from the pool
  recovery: 2-3 failed checks x 5s = ~15s

LAYER 2  across zones     (seconds)
  regional LB spans AZs; losing an AZ is a pool change
  recovery: same health-check window

LAYER 3  across regions   (minutes)
  health-checked DNS, TTL 60s, returns the healthy region
  realistic recovery: 1-5 min for most clients,
                      long tail of hours for badly behaved ones

MITIGATIONS for the tail:
  - set runtime DNS cache TTL explicitly (JVM: 30s, not -1)
  - bound connection lifetime so pools re-resolve periodically
  - client retry with a second endpoint hostname
  - anycast the front door so the IP does not need to change
```

| Metric | Value | Note |
|---|---|---|
| Instance failure | ~15 s | health checks |
| Zone failure | ~15 s | same pool |
| Region failure (median) | 1–5 min | **DNS TTL + caches** |
| Region failure (tail) | hours | misbehaving caches |

> **The design consequence**  
> Because the DNS tail is long, an active-passive region failover must assume the old region receives traffic for a while. That means the failed region must *reject* cleanly rather than serve stale data, and the new primary must be safe to write to even while stragglers still reach the old one — which is an argument for fencing at the data layer, not just at the traffic layer.

**When to use it**

- **Coarse traffic steering**: geographic routing, regional weighting, blue-green cutover at a minutes timescale.
- **Service naming and indirection**: giving stable names to things whose addresses change.
- **Third-party integration points**, where you control neither client nor server and DNS is the only shared mechanism.
- **Multi-CDN or multi-provider strategies**, switching providers at the name level.
- **Internal service discovery** in environments where a DNS-based discovery mechanism (e.g. Kubernetes service DNS) is the platform default.

**When to avoid it**

- **Do not use DNS for sub-minute failover.** The caches above you make it unreliable at that timescale.
- **Do not rely on very low TTLs** (under ~30 s) being respected; many resolvers clamp them.
- **Do not leave runtime DNS caching at its default** — the JVM's historical infinite cache is a classic production surprise.
- **Do not put a CNAME at a zone apex** — it is invalid alongside the required SOA/NS records; use provider-specific alias records instead.
- **Do not assume a DNS change drains connections.** Open sockets keep using the old address indefinitely.

**Advantages**

- **Universal**: every client speaks it, with no coordination or SDK required.
- **Free indirection**: addresses can change behind a stable name, which is the basis of most migration strategies.
- **Global and highly available** when anycast is used, with caching that absorbs enormous read volume.
- **Enables geographic and latency-based steering** without any application awareness.

**Disadvantages**

- **Propagation is unbounded in the tail**, because you do not control the caches.
- **No connection awareness** — changing a record does not move existing connections.
- **Coarse granularity**: you steer by name, not by request, user, or capacity.
- **A shared failure domain**: a DNS outage takes out everything simultaneously, including your ability to reach your own tooling.
- **Negative caching** can hide newly created records for the SOA minimum.

**Trade-offs**

**Choosing a TTL**

| TTL | Failover speed | Query load | Use for |
|---|---|---|---|
| 30–60 s | Fast (median), long tail | High | Failover endpoints, canary cutover |
| 300 s | Balanced | Moderate | General service endpoints |
| 1–24 h | Slow | Minimal | Stable infrastructure, MX, NS |
| Very low (<30 s) | No better than 30 s in practice | Very high | Rarely justified — often clamped |

A common pattern is to lower TTLs *in advance* of a planned migration — reduce to 60 s a day before, cut over, then raise again. This works because the reduction itself propagates at the old TTL.

**How it fails**

**DNS failure modes**

| Failure | Symptom | Fix |
|---|---|---|
| Authoritative outage | Everything fails once caches expire — a cliff, not a ramp | Multiple providers, or long TTLs on critical records |
| Runtime cache never expires | One client keeps hitting a dead endpoint forever | Set the runtime TTL explicitly; bound connection lifetime |
| Negative cache hides a new record | A record exists but clients get NXDOMAIN | Lower the SOA minimum; avoid querying before creation |
| Resolver clamps low TTLs | Failover slower than planned | Do not depend on sub-minute DNS; use another layer |
| Connection pool pins the old IP | Traffic continues to a drained region | Max connection age; explicit pool reset on failover |
| Split-horizon misconfiguration | Internal clients get external addresses or vice versa | Explicit views with tests in both networks |
| Expired domain or zone delegation error | Total, sudden, and embarrassing outage | Registrar lock, auto-renew, monitoring on NS and expiry |

**Limits**

> **Numbers to know**
>
> - **UDP DNS responses** were traditionally limited to 512 bytes; EDNS0 extends this, but very large responses still fall back to TCP.
> - **Typical resolution latency**: 1–5 ms on a cache hit, 20–150 ms on a full recursive walk.
> - **TTL floor in practice**: assume ~30–60 s regardless of what you publish.
> - **Negative TTL** comes from the SOA minimum — commonly 300–3600 s.
> - **JVM default**: `networkaddress.cache.ttl` historically `-1` (forever) with a security manager; set it explicitly.

**Alternatives**

| Mechanism | Failover speed | Control | Use when |
|---|---|---|---|
| DNS | Minutes to hours | None over clients | Coarse steering, third parties |
| Anycast + BGP | Seconds | Network layer | Global front doors, DNS itself |
| Load balancer health checks | Seconds | Full, within a region | Instance and zone failure |
| Client-side discovery (Consul, Eureka, xDS) | Seconds | Full, if you own clients | Internal service meshes |
| Client-side multi-endpoint retry | Immediate | Full | SDKs you control |

**In real systems**

- **Root and public resolvers (1.1.1.1, 8.8.8.8)** use anycast so the same address is served from hundreds of locations.
- **Cloud provider global load balancers** deliberately use a single anycast IP so that failover does not depend on DNS propagation at all.
- **Kubernetes** gives every service a cluster-internal DNS name, and its short TTLs plus in-cluster resolvers make it a practical discovery layer.
- **Multi-CDN strategies** switch providers at the DNS layer, accepting minutes of transition as the cost of provider independence.
- **The 2016 Dyn incident** demonstrated the shared-failure-domain problem: a DNS provider outage made many unrelated services simultaneously unreachable.

**Common mistakes**

- **Planning a 60-second regional failover** on DNS alone.
- **Leaving the JVM DNS cache at its default**, so one service never notices the change.
- **Assuming a record change drains existing connections.**
- **Placing a CNAME at the zone apex.**
- **Using a single DNS provider** for records whose outage is a total outage.
- **Querying a name before creating it**, then being blocked by negative caching.
- **Setting a 1-second TTL** and believing it.

**The staff-level view**

The Staff move is to stop treating DNS as a failover mechanism and start treating it as a naming and coarse-steering mechanism, then build real failover at a layer you control.

- **Classify each record by role**: failover-critical (short TTL), stable infrastructure (long TTL), third-party verification (irrelevant). One global TTL policy is wrong.
- **Use multiple DNS providers** for anything whose outage would be a company-level incident; a single provider is a single failure domain for everything at once.
- **Standardise runtime resolver settings** in the base image or service template. This is the fix nobody remembers and everybody needs.
- **Bound connection lifetime** fleet-wide so pools re-resolve without needing a restart.
- **Assume a long tail in failover design** and ensure the drained region rejects rather than serves — fencing at the data layer, not just traffic steering.

**Go deeper**

DNS is a globally distributed, aggressively cached, eventually consistent key-value store. A lookup walks from the client's runtime cache through the OS, a recursive resolver, and the root/TLD/authoritative hierarchy, with every layer caching for the TTL. Most lookups never leave a cache.

The TTL is a request, not a guarantee. Some resolvers clamp low values upward; some runtimes — the JVM's historical default being the classic example — cache resolutions indefinitely; and connection pools with open sockets never re-resolve at all. So a “60-second TTL means one minute of impact” failover plan is wrong: the median may follow the TTL while a tail of clients hits the dead endpoint for hours.

Use DNS for what it is good at: stable naming, geographic steering, and coarse cutovers over minutes. Build real failover at layers you control — load balancer health checks within a region, anycast for a global front door, client-side discovery where you own the client. And design regional failover assuming the drained region still receives traffic, which means fencing at the data layer rather than trusting traffic steering alone.

DNS is the layer engineers most often misunderstand because its interface is simple and its behaviour is governed almost entirely by caches outside their control.

**The hierarchy and the caches.** A resolution passes through the application runtime's cache, the OS cache, a recursive resolver's cache, and only then the root → TLD → authoritative walk. Most queries are served from the recursive resolver. Your authoritative server sees a tiny fraction of real traffic, which is why DNS scales so well — and also why changes propagate so unevenly.

**TTL as a policy lever.** A short TTL (30–60 s) means faster propagation, higher query volume, and greater exposure if your authoritative servers become unreachable. A long TTL (hours) means near-zero query load and a very long tail after any change. The right value is per-record: failover endpoints deserve short TTLs, while NS, MX and stable infrastructure records deserve long ones. Lowering a TTL in preparation for a migration works, but the reduction itself propagates at the *old* TTL, so it must be done a full TTL in advance — which makes it useless as an incident-time action.

**Why failover disappoints.** Three mechanisms defeat it. Resolvers may clamp very low TTLs upward. Application runtimes cache independently — the JVM historically cached successful lookups forever, which is a recurring production surprise. And connection pools hold open sockets that never re-resolve, so a DNS change does not move existing traffic at all; you must also bound connection lifetime or explicitly reset pools. Negative caching adds a fourth: an `NXDOMAIN` is cached for the zone's SOA minimum, so a record created after a failed lookup can be invisible for minutes.

**Layer your failover.** Within a region, load balancer health checks move traffic in seconds. Across zones, the same pool mechanism applies. Across regions, an anycast front door lets you withdraw a BGP announcement in seconds without touching DNS at all. DNS is the outer ring: minutes for the median, hours for the tail. Crucially, this means the design must assume the drained region continues to receive requests — so that region must fail closed, and the data layer must be fenced so stragglers cannot write to a demoted primary.

**DNS as a shared failure domain.** Because every service depends on it, a DNS provider outage is simultaneously an outage of your application, your internal tooling, and often your ability to reach the systems you would use to fix it. The mitigations are redundancy — two independent authoritative providers with synchronised zones and drift detection, registrar locks, monitored expiry, alerting on NS and SOA changes — and reduction of dependence, by putting the customer front door behind an anycast address and moving internal discovery onto a mechanism whose clients you own.

**Prove it — interview questions**

1. **[Basic] Walk through what happens when a client resolves `api.example.com`.**

   <details><summary>Model answer</summary>

   The application calls the resolver, which checks the runtime and OS caches first. On a miss it asks the configured recursive resolver, which checks its own cache and otherwise walks the hierarchy: a root server points it at the `.com` TLD servers, which point it at the authoritative nameservers for `example.com`, which return the A record. Each layer caches the answer for the TTL. On a warm cache this is a sub-millisecond local lookup; on a cold one it is a few network round trips totalling tens to low hundreds of milliseconds.

   </details>

2. **[Basic] What does TTL actually guarantee?**

   <details><summary>Model answer</summary>

   Very little. It is a request to caches to hold the answer for at most that long, and most resolvers honour it approximately. But some clamp very low values upward to reduce load, some application runtimes cache independently and may ignore it entirely, and open connections bypass resolution altogether. Treat TTL as influencing the median propagation time, never as bounding the tail.

   </details>

3. **[Senior] Why is DNS a poor mechanism for fast failover, and what do you use instead?**

   <details><summary>Model answer</summary>

   Because the caches that matter are not yours: resolver caches, OS caches, runtime caches and connection pools all sit between your change and the client's next request. The median might follow your TTL, but the tail runs for hours. For fast failover I use layers where I control both ends — load balancer health checks within a region (seconds), anycast with BGP withdrawal for a global front door (seconds), or client-side discovery with explicit refresh if I own the clients. DNS remains useful for coarse steering and as the outer ring of a layered strategy.

   </details>

4. **[Senior] What does a low TTL actually cost, and when is it worth paying?**

   <details><summary>Model answer</summary>

   It costs query volume and adds resolution latency to the path. A 60-second TTL means every resolver re-queries once a minute per client population it serves, which multiplies load on your authoritative servers and makes them a more attractive target and a more consequential dependency — and any client that misses a warm cache pays a resolution round trip before its request even starts. It is worth paying when the record is a failover target, because the TTL is the floor on how quickly well-behaved resolvers can follow you elsewhere, and that floor has to be set before the incident rather than during it. For records that never move, a long TTL is free reliability: fewer queries, fewer opportunities for a DNS outage to become your outage.

   </details>

5. **[Staff] Design regional failover for a service where DNS is the only cross-region mechanism available.**

   <details><summary>Model answer</summary>

   I would design assuming the old region keeps receiving traffic for hours. That means the failed region must fail closed — returning errors or redirects rather than serving stale or writable data — and the data layer must be fenced so stragglers cannot write to the demoted primary. I would keep the failover record at a 60-second TTL permanently rather than lowering it during an incident, since lowering it propagates at the old TTL and is therefore useless in the moment. I would standardise runtime resolver TTLs and maximum connection age across the fleet so pools re-resolve, and add a client-side fallback hostname so well-behaved clients can move immediately. Finally I would use two DNS providers, because a provider outage during a regional failure is exactly the correlated failure that turns an incident into a crisis.

   </details>

6. **[Principal] How do you reduce DNS as a company-wide single point of failure?**

   <details><summary>Model answer</summary>

   Treat it as tier-0 infrastructure with the same rigour as a database. That means at least two independent authoritative providers with automated zone synchronisation and drift detection, registrar locks and monitored expiry dates, and alerting on NS and SOA changes since delegation errors are silent until they are catastrophic. Beyond redundancy, I would reduce dependence: put the customer-facing front door behind an anycast address so routine failover never touches DNS at all, and move internal service discovery onto a mechanism whose clients we own, so an external DNS incident degrades ingress rather than paralysing internal traffic. The strategic goal is that a DNS provider outage becomes a partial degradation with a known blast radius, not a total loss of the ability to operate — including the ability to reach the tools we would use to fix it.

   </details>

---

### TCP reliability and congestion

*TCP turns an unreliable packet network into an ordered byte stream, and its congestion control is why your throughput depends on round-trip time.*

**Flow:** `Bytes` → `Segments` → `Network` → `Acknowledgments` → `Ordered stream`

> **The 30-second version**  
> TCP gives an ordered reliable stream over an unreliable network, at the cost of head-of-line blocking, handshake round trips, and throughput capped by window ÷ RTT.

**The problem**

Applications see a reliable, ordered byte stream. The network underneath delivers packets that may be lost, duplicated, reordered or delayed arbitrarily. TCP bridges that gap — and the mechanisms it uses to do so explain most of the latency and throughput behaviour engineers blame on their code.

The concrete consequence: a single-connection transfer between two regions 150 ms apart cannot exceed a throughput determined by the window size divided by the round-trip time, regardless of how much bandwidth you have bought. Teams discover this when a 10 Gbps link delivers 50 Mbps on one connection and conclude the network is broken.

> **Three TCP behaviours that surprise people**
>
> - **Head-of-line blocking**: one lost packet stalls *every* stream multiplexed on that connection until it is retransmitted.
> - **Slow start**: every new connection begins cautiously, so short-lived connections never reach full speed.
> - **Bandwidth-delay product**: throughput on one connection is capped at window ÷ RTT, which is why cross-region transfers need parallelism or a bigger window.

**Mental model**

TCP is two mechanisms layered on the same acknowledgement stream: **reliability** (retransmit what was lost, deliver in order) and **congestion control** (guess how fast the network can accept data, and back off when it cannot).

1. **Sequence numbers and ACKs** — Every byte is numbered. The receiver acknowledges what it has received contiguously; gaps trigger retransmission.
2. **Receive window (flow control)** — Protects the *receiver* from being overwhelmed. Advertised in every ACK.
3. **Congestion window (congestion control)** — Protects the *network*. Maintained by the sender from its own inference about loss and delay.
4. **Effective window** — min(receive window, congestion window). Throughput ≈ effective window ÷ RTT.
5. **Loss as a signal** — Classic algorithms treat packet loss as congestion and halve the window. Modern ones (BBR) model bandwidth and RTT directly.

> **The bandwidth-delay product is the number that matters**  
> To keep a pipe full you must have BDP = bandwidth × RTT bytes in flight. At 1 Gbps and 100 ms RTT that is 12.5 MB. If your window is 64 KB, you achieve 64 KB ÷ 0.1 s ≈ 5 Mbps — 0.5% of the link. Window scaling exists precisely for this, and it is why long-distance single-stream transfers are slow unless tuned.

**How it works**

**Connection lifecycle and its costs**

```text
HANDSHAKE            1 RTT  (SYN, SYN-ACK, ACK)
+ TLS 1.3            1 RTT  (or 0 RTT on resumption)
-> first byte of application data: ~2 RTT on a new connection

SLOW START
  cwnd starts ~10 segments (~14 KB)
  doubles each RTT until loss or ssthresh
  to reach 1 MB in flight: ~7 RTTs

  at 100 ms RTT that is 700 ms before full speed.
  A 100 KB response finishes before slow start ends.

CONGESTION AVOIDANCE
  additive increase (+1 segment/RTT), multiplicative decrease
  on loss (halve) -> the classic sawtooth

CLOSE
  FIN/ACK exchange, then TIME_WAIT (~60s) holds the port
```

1. **Reuse connections relentlessly** — Because handshake plus slow start dominates short transfers, connection reuse is usually the single biggest network win available. Keep-alive, HTTP/2, connection pools.
2. **Tune the window for long fat networks** — Window scaling and larger buffers are required to fill a high-bandwidth, high-latency path. Without them, throughput is RTT-bound.
3. **Use parallel connections when you cannot tune** — N connections get roughly N× the window. This is what download accelerators and multi-stream transfer tools do.
4. **Know which congestion control you are running** — Cubic (loss-based) is the common default; BBR models bandwidth and RTT and performs far better on lossy or buffer-bloated paths.
5. **Set timeouts with RTT in mind** — A retransmission timeout is derived from measured RTT and its variance; application timeouts shorter than a few RTTs will fire during ordinary recovery.

**Why one lost packet is expensive**

```text
sender:   [1][2][3][4][5][6][7][8]
network:  [1][2][X][4][5][6][7][8]      packet 3 lost

receiver has 4-8 buffered but CANNOT deliver them:
  the byte stream must be in order -> HEAD OF LINE BLOCKING

recovery:
  3 duplicate ACKs -> fast retransmit (~1 RTT)
  or RTO timeout   -> much slower, and cwnd collapses

consequence for multiplexing:
  HTTP/2 puts many streams on ONE TCP connection.
  One lost packet stalls ALL of them.
  This is exactly why QUIC moved reliability above the
  transport, giving each stream independent ordering.
```

> **Nagle plus delayed ACK is a classic 40 ms mystery**  
> Nagle's algorithm withholds small writes until the previous data is acknowledged; delayed ACK withholds acknowledgements for up to ~40 ms hoping to piggyback. Together they produce a stall on request/response patterns with small writes. `TCP_NODELAY` is the standard fix for latency-sensitive protocols, and most RPC frameworks set it.

**Worked example**

Transferring a 1 GB file between two data centres 80 ms apart on a 10 Gbps link. Why does it take so long, and what fixes it?

**RTT-bound throughput, and three fixes**

```text
BDP = 10 Gbps x 0.080 s = 100 MB in flight to fill the pipe

DEFAULT (64 KB window, no scaling):
  throughput = 65,536 B / 0.080 s = 819 KB/s = 6.5 Mbps
  1 GB takes ~21 minutes on a 10 Gbps link

FIX 1  window scaling + 16 MB buffers
  throughput = 16 MB / 0.080 s = 200 MB/s = 1.6 Gbps
  1 GB takes ~5 seconds

FIX 2  8 parallel connections at 2 MB each
  8 x 2 MB / 0.080 s = 200 MB/s        (same effect)

FIX 3  BBR instead of Cubic
  on a path with 0.1% loss, Cubic collapses;
  BBR sustains near link rate because it does not
  treat loss as the primary congestion signal
```

| Metric | Value | Note |
|---|---|---|
| Default window | 6.5 Mbps | 0.065% of link |
| Window scaling | 1.6 Gbps | **250× better** |
| Parallel streams | 1.6 Gbps | no kernel tuning |
| With 0.1% loss | Cubic collapses | BBR survives |

> **The general lesson**  
> Whenever throughput looks far below link capacity on a long path, compute window ÷ RTT before suspecting anything else. This single calculation explains the majority of “the network is slow” reports between regions, and the fix is almost always window tuning, parallelism, or moving the data closer.

**When to use it**

- **Whenever ordering and reliability are required end to end** — which is most application traffic.
- **Long-lived connections** where handshake and slow-start costs are amortised: databases, message brokers, gRPC channels.
- **Bulk transfer**, where window tuning and parallelism give large wins.
- **Anything behind TLS**, which assumes a reliable ordered stream underneath (until QUIC).

**When to avoid it**

- **Do not open a new connection per request.** Handshake plus slow start can exceed the transfer itself.
- **Do not multiplex latency-sensitive independent streams over one TCP connection** on a lossy path — head-of-line blocking couples them.
- **Do not use TCP for real-time media** where a late packet is worthless; UDP-based protocols are correct there.
- **Do not set application timeouts below a few RTTs**, or ordinary retransmission looks like failure.
- **Do not assume the default window suits a cross-region path.**

**Advantages**

- **Reliability and ordering for free**, implemented once in the kernel and correct everywhere.
- **Congestion control protects the network** as a shared resource, which is why the internet works at all.
- **Universally supported** and traverses nearly all middleboxes without special handling.
- **Mature tuning surface**: window scaling, selective acknowledgement, modern congestion control algorithms.

**Disadvantages**

- **Head-of-line blocking** couples independent streams sharing a connection.
- **Connection setup costs 1–2 RTTs** before any application data flows.
- **Slow start penalises short transfers**, which is most web traffic.
- **Loss-based congestion control misreads wireless and buffer-bloated paths**, where loss is not congestion.
- **Ossified in the kernel and middleboxes**, so improvements deploy slowly — a core motivation for QUIC.

**Trade-offs**

**Congestion control algorithms**

| Algorithm | Signal | Strength | Weakness |
|---|---|---|---|
| Reno / NewReno | Loss | Simple, historic baseline | Very slow recovery on long paths |
| Cubic (common default) | Loss | Good on wired, high-BDP paths | Collapses on lossy links; fills buffers |
| BBR | Bandwidth + RTT estimate | Excellent on lossy or bloated paths | Can be aggressive toward loss-based flows |
| Vegas / delay-based | Delay | Low queueing latency | Loses to loss-based flows sharing a link |

The practical guidance: leave Cubic alone for typical data-centre traffic, and consider BBR for long-haul, lossy, or last-mile-heavy paths such as CDN egress to mobile users.

**How it fails**

**TCP-related failures**

| Symptom | Cause | Fix |
|---|---|---|
| Throughput far below link rate | Window ÷ RTT limit | Window scaling, larger buffers, parallel streams |
| 40 ms stalls on small requests | Nagle + delayed ACK interaction | `TCP_NODELAY` |
| All HTTP/2 streams stall together | Head-of-line blocking from one lost packet | QUIC/HTTP-3, or separate connections |
| Connection storms after a restart | Every client reconnects and slow-starts simultaneously | Jittered reconnect; connection reuse |
| Ports exhausted | Many short connections held in TIME_WAIT | Reuse connections; tune port range and reuse settings |
| High latency under load with no loss | Bufferbloat — deep queues in the path | BBR or delay-based control; AQM at the bottleneck |
| Silent connection death through a NAT | Idle timeout dropped the mapping | TCP keepalives shorter than the NAT timeout |

**Limits**

> **Numbers worth carrying**
>
> - **BDP = bandwidth × RTT.** 1 Gbps × 100 ms = 12.5 MB in flight to fill the pipe.
> - **Initial congestion window** is commonly 10 segments (~14 KB).
> - **Slow start doubles per RTT**: reaching 1 MB in flight takes about 7 RTTs.
> - **Handshake cost**: 1 RTT for TCP, plus 1 for TLS 1.3 (0 on resumption).
> - **Same-AZ RTT** ~0.5 ms; cross-region 50–150 ms; the ratio is why locality dominates design.

**Alternatives**

| Transport | Gives | Costs |
|---|---|---|
| TCP | Reliable ordered stream | HOL blocking, handshake, slow start |
| UDP | No guarantees, no overhead | You implement reliability yourself |
| QUIC (HTTP/3) | Per-stream ordering, 0–1 RTT setup, connection migration | Userspace CPU cost; some networks block UDP |
| SCTP | Multi-streaming with partial reliability | Poor middlebox traversal |
| RDMA / RoCE | Kernel-bypass, microsecond latency | Data-centre only, specialised hardware |

**In real systems**

- **HTTP/2** multiplexes streams over one TCP connection and inherits head-of-line blocking, which is the primary motivation for HTTP/3.
- **Google's BBR** was deployed on YouTube traffic and reported large throughput gains on lossy last-mile paths.
- **Database drivers and connection pools** exist substantially to avoid paying handshake and slow-start costs per query.
- **Bulk transfer tools** (multi-stream copy utilities, cloud CLI parallel uploads) parallelise precisely to work around the window ÷ RTT limit.
- **CDNs reduce RTT** to shorten slow start and raise effective single-connection throughput — a large part of why they work.

**Common mistakes**

- **Opening a connection per request** and paying handshake plus slow start every time.
- **Blaming the application** for throughput that is actually window-limited.
- **Leaving default socket buffers** on long-haul transfers.
- **Multiplexing latency-sensitive streams on one connection** over a lossy path.
- **Setting sub-RTT timeouts**, which turn ordinary recovery into errors.
- **Forgetting keepalives** through NATs and load balancers with idle timeouts.
- **Assuming loss means congestion** on wireless paths, where it often does not.

**The staff-level view**

Most TCP knowledge pays off indirectly: it stops teams from misattributing latency to their code and directs effort at the real constraint.

- **Teach the window ÷ RTT calculation.** It resolves most cross-region throughput arguments in one line.
- **Standardise connection reuse** in shared clients, since it is the largest single win and is easy to get wrong per-team.
- **Choose the transport deliberately for multiplexed traffic.** If independent streams share a connection on a lossy path, HTTP/3 is a material improvement, not a fashion.
- **Set timeouts from measured RTT distributions**, not from round numbers, so retransmission is not mistaken for failure.
- **Prefer moving data closer over tuning transports.** A CDN or a regional replica beats any window tuning, because it attacks RTT itself.

**Go deeper**

TCP layers two mechanisms on one acknowledgement stream: reliability through sequence numbers and retransmission, and congestion control through a window the sender adjusts based on inferred network capacity. Effective throughput is the smaller of the receive and congestion windows divided by round-trip time — which is why a 10 Gbps cross-region link delivers only a few Mbps on a single default-configured connection.

Three behaviours dominate real systems. Connection setup costs one round trip, plus another for TLS, and slow start then takes several more RTTs to reach full speed, so short-lived connections never perform well and reuse is the biggest available win. Head-of-line blocking means one lost packet stalls every stream multiplexed on that connection, which is HTTP/2's central weakness and QUIC's central motivation. And loss-based congestion control (Cubic) misreads wireless and buffer-bloated paths, where BBR's bandwidth-and-delay model performs far better.

Practically: reuse connections, enable window scaling with adequate buffers on long paths or use parallel streams, set application timeouts from measured RTT distributions rather than round numbers, and remember that reducing RTT — via a CDN or regional replicas — improves handshake, slow start and window-limited throughput all at once, which no amount of socket tuning can match.

TCP's job is to present an ordered reliable byte stream over a network that loses, reorders and delays packets. The mechanisms it uses to do so explain most of the latency and throughput behaviour that application teams misattribute to their own code.

**Reliability.** Bytes are sequence-numbered; the receiver acknowledges the highest contiguous byte received. Gaps trigger fast retransmit after three duplicate acknowledgements, or a timer-based retransmission derived from measured RTT and its variance. Because delivery must be ordered, segments arriving after a gap are buffered but withheld from the application — head-of-line blocking. When many logical streams share one connection, as in HTTP/2, a single lost packet stalls all of them, which is precisely why QUIC relocated reliability above the transport so each stream has independent ordering.

**Congestion control and the window.** The sender maintains a congestion window representing how much it believes the network can absorb; the receiver advertises a window representing what it can buffer. Throughput is bounded by the smaller of the two divided by RTT. The bandwidth-delay product — bandwidth × RTT — is the amount of data that must be in flight to keep a path full: 12.5 MB at 1 Gbps and 100 ms. A default 64 KB window on an 80 ms path caps throughput near 6.5 Mbps no matter how fast the link is, which explains most “the network between regions is broken” reports. The remedies are window scaling with larger buffers, or parallel connections that multiply the effective window.

**Setup and slow start.** A new connection costs one RTT for the handshake and another for TLS 1.3 (zero on resumption), then begins with roughly ten segments in flight and doubles each RTT. Reaching 1 MB in flight takes about seven RTTs — 700 ms on a 100 ms path — so most web-sized responses complete before the connection ever reaches full speed. This is why connection reuse, keep-alive and pooling are usually the highest-return network optimisation available, and why per-request connections are a recurring performance bug.

**Choosing congestion control.** Cubic, the common default, treats loss as the congestion signal and performs well on wired, low-loss, high-bandwidth-delay paths, but collapses on links where loss is caused by radio conditions rather than queueing, and it tends to fill deep buffers, worsening latency. BBR estimates available bandwidth and minimum RTT directly, sustaining high throughput on lossy paths and keeping queues shorter; its trade-off is aggressiveness toward coexisting loss-based flows. For CDN egress to mobile users the difference is large; for intra-data-centre traffic it rarely matters.

**Operational consequences.** Set application timeouts from measured RTT distributions, because a timeout shorter than a few round trips turns ordinary retransmission into an error. Enable `TCP_NODELAY` on request/response protocols to avoid the Nagle-plus-delayed-ACK stall. Use keepalives shorter than any NAT or load balancer idle timeout, or connections die silently. And above all, prefer attacking RTT itself — edge caching, regional replicas, moving computation closer to data — because reducing round-trip time improves handshake cost, slow-start duration and window-limited throughput simultaneously, which no socket tuning can do.

**Prove it — interview questions**

1. **[Basic] How does TCP provide reliability?**

   <details><summary>Model answer</summary>

   Every byte is sequence-numbered, and the receiver acknowledges the highest contiguous byte it has received. The sender keeps unacknowledged data buffered and retransmits when it sees duplicate acknowledgements indicating a gap, or when a retransmission timer derived from measured round-trip time expires. Because delivery must be in order, data received after a gap is buffered but not handed to the application until the missing segment arrives — which is head-of-line blocking.

   </details>

2. **[Basic] Why is a new TCP connection slow for small responses?**

   <details><summary>Model answer</summary>

   Two reasons compound. The handshake costs a full round trip before any data flows, and TLS adds another. Then slow start begins with roughly ten segments in flight and doubles each round trip, so a connection takes several RTTs to reach full speed. A 100 KB response over a 100 ms path finishes while the connection is still ramping, meaning the transfer is dominated by round trips rather than by bandwidth. This is why connection reuse is usually the single largest available network optimisation.

   </details>

3. **[Senior] A transfer between regions gets 6 Mbps on a 10 Gbps link. Diagnose it.**

   <details><summary>Model answer</summary>

   I would compute window divided by RTT first. With a default 64 KB window and an 80 ms round trip, the ceiling is about 6.5 Mbps regardless of link capacity — which matches exactly, so the link is not the problem and the application is not the problem. The fixes are window scaling with much larger socket buffers to approach the bandwidth-delay product, or running several parallel streams to multiply the effective window. If the path also has even slight loss, Cubic will keep collapsing the window, and switching to BBR is the bigger win.

   </details>

4. **[Senior] Why does packet loss hurt throughput so disproportionately?**

   <details><summary>Model answer</summary>

   Because loss-based congestion control interprets any loss as congestion and collapses the sending window, then rebuilds it slowly. On a high-bandwidth long-latency path the rebuild takes many round trips, so a single loss event costs far more than the retransmitted packet — and at a steady low loss rate the connection never reaches a large window at all, which is why a link with a fraction of a per cent loss can deliver a small fraction of its capacity. That is also the argument for congestion control that models the path's bandwidth and latency directly rather than treating loss as the signal, since on paths where loss is caused by something other than congestion the loss-based interpretation is simply wrong.

   </details>

5. **[Staff] When would you choose QUIC over TCP, and what do you give up?**

   <details><summary>Model answer</summary>

   I would choose QUIC when independent streams share a connection over a path with meaningful loss — mobile clients, long-haul, or anything behind a lossy last mile — because QUIC moves reliability above the transport so a lost packet stalls only its own stream rather than all of them. It also gives faster connection establishment, including zero round trip on resumption, and connection migration across network changes, which matters for mobile. The costs are real: it runs in userspace, so CPU per byte is higher than kernel TCP; some restrictive networks block or throttle UDP, so a TCP fallback is mandatory; and operational tooling for debugging is less mature. For internal data-centre traffic on a low-loss path, TCP remains the better choice.

   </details>

6. **[Principal] How do you decide whether to invest in transport tuning at all?**

   <details><summary>Model answer</summary>

   By checking whether the transport is actually the binding constraint, which it usually is not. I would look at where time goes end to end: if round-trip time dominates, the highest-leverage move is reducing RTT — edge caching, regional replicas, or moving computation closer — because that improves handshake, slow start and window-limited throughput simultaneously, whereas tuning only improves the last one. Transport tuning is worth real investment in two situations: bulk data movement between fixed locations, where window sizing and parallelism yield order-of-magnitude gains for a few days of work, and serving large numbers of mobile or international users, where congestion-control choice materially changes delivered throughput. Otherwise I would standardise connection reuse and sensible defaults in shared clients, and spend the engineering effort on architecture rather than on sockets.

   </details>

---

### TLS termination

*Where you decrypt decides who can read the traffic, who owns the certificates, and what a compromised hop can do.*

**Flow:** `Client` → `TLS handshake` → `Gateway` → `Upstream TLS` → `Service`

> **The 30-second version**  
> Decide deliberately where encrypted traffic becomes plaintext. Edge termination is simplest, re-encryption keeps the internal wire safe, and mTLS replaces network-position trust with cryptographic identity.

**The problem**

Every architecture has a point where encrypted traffic becomes plaintext. Choosing that point is a security decision disguised as a performance decision, and teams usually make it by accident — terminating at whichever load balancer was easiest to configure.

The consequence shows up later: an auditor asks whether traffic is encrypted inside the VPC, and the honest answer is that it is plaintext from the load balancer onward, across shared network fabric, through a service mesh sidecar, and into a pod that any compromised neighbour could sniff.

> **What terminating early actually gives away**
>
> - **Everything downstream is readable** by anyone with network access or a packet capture on that path.
> - **Client identity is lost** unless you explicitly forward it — the backend sees the proxy, not the user.
> - **Compliance scope expands**: any component handling plaintext cardholder or health data falls inside the audit boundary.

**Mental model**

Think of TLS as an envelope. Termination is where the envelope is opened. Everything after that point sees the letter.

1. **Edge termination** — Decrypt at the CDN or load balancer, plaintext to backends. Simplest and fastest; largest plaintext blast radius.
2. **Re-encryption** — Decrypt at the edge for routing decisions, then open a new TLS connection to the backend. Costs a second handshake; keeps the wire encrypted.
3. **Passthrough** — The proxy forwards TCP without decrypting; the backend terminates. Preserves end-to-end secrecy and client certificates, but the proxy cannot route on HTTP attributes.
4. **mTLS everywhere** — Both sides present certificates on every hop, so identity is cryptographic rather than network-positional. This is what service meshes automate.

> **The question that decides it**  
> Ask: *what must the proxy be able to read?* If it must route by path, rewrite headers, cache, or apply a WAF, it must decrypt — so the choice is edge termination or re-encryption, never passthrough. If it only needs to route by SNI, passthrough keeps the envelope sealed end to end.

**How it works**

**The three topologies**

```text
EDGE TERMINATION
  client --TLS--> LB --plaintext--> service
  + one cert, one handshake, full L7 features
  - plaintext on the internal network

RE-ENCRYPTION
  client --TLS--> LB --TLS--> service
  + L7 features AND encrypted internal hop
  - two handshakes; LB needs client trust for upstream certs

PASSTHROUGH (TLS/SNI routing)
  client ----------TLS----------> service
            LB routes by SNI only
  + true end to end; client certs reach the service
  - no path routing, no header injection, no caching, no WAF
```

1. **Handshake cost is asymmetric and mostly one-time** — TLS 1.3 needs one round trip, or zero on resumption. The CPU cost is dominated by the key exchange, so session resumption and keep-alive matter far more than cipher choice.
2. **Forward the client identity explicitly** — After termination the backend sees the proxy's address. `X-Forwarded-For`, `X-Forwarded-Proto` and client-certificate headers must be injected by the proxy and **stripped from inbound requests** so they cannot be spoofed.
3. **SNI is visible even when the payload is not** — The hostname is in the clear during the handshake (unless Encrypted Client Hello is in use), which is why SNI-based routing works at all — and why hostnames leak.
4. **mTLS gives identity, not just secrecy** — A service mesh issues short-lived certificates per workload, so authorisation can be based on cryptographic identity instead of IP addresses, which is the foundation of zero-trust networking.
5. **Certificate lifecycle is the real operational cost** — Automated issuance and rotation (ACME, mesh CAs) is not optional at scale — expired certificates are one of the most common causes of self-inflicted outages.

**Header hygiene after termination**

```text
INBOUND from the internet:
  X-Forwarded-For: 1.2.3.4      <- attacker-supplied, UNTRUSTED

PROXY MUST:
  strip inbound X-Forwarded-*  (do not append to attacker input)
  set  X-Forwarded-For: <real peer address>
  set  X-Forwarded-Proto: https
  set  X-Client-Cert-Subject: <from mTLS, if used>

BACKEND MUST:
  trust these headers ONLY from known proxy addresses
  otherwise any client can claim any identity
```

> **The trusted-header trap**  
> If a backend trusts `X-Forwarded-For` unconditionally and is reachable directly — even from inside the cluster — any caller can forge a source address, bypassing IP allowlists, rate limits and audit logs. Trust these headers only from an explicitly configured set of proxy addresses, and make direct access to backends impossible at the network layer.

**Worked example**

A payments API behind a CDN and an internal gateway. Where should TLS terminate?

**Decision by requirement**

```text
REQUIREMENTS
  R1  WAF and bot protection at the edge   -> edge must read HTTP
  R2  cardholder data must not traverse
      the internal network in plaintext     -> internal hop encrypted
  R3  service-to-service authz by identity  -> mTLS internally
  R4  audit: prove encryption in transit    -> no plaintext anywhere

RESULT
  client --TLS1.3--> CDN        (terminate: R1 needs L7)
         --TLS-->     gateway   (re-encrypt: R2)
         --mTLS-->    service   (R3: identity, not IP)
         --mTLS-->    database  (R4)

COSTS ACCEPTED
  3 handshakes on a cold path (~1 RTT each, amortised by keep-alive)
  a certificate authority and rotation automation
  ~1-3% CPU overhead per hop for symmetric encryption
```

| Metric | Value | Note |
|---|---|---|
| Handshake RTTs | 1 per hop | 0 on resumption |
| CPU overhead | 1–3% | symmetric crypto is cheap |
| Cert rotation | automated | **the real cost** |
| Plaintext hops | 0 | audit satisfied |

> **The cost is operational, not computational**  
> Engineers resist mTLS believing encryption is expensive. Modern symmetric crypto with AES-NI costs a few percent of CPU; the handshake is the expensive part and it amortises over a long-lived connection. The genuine cost is certificate lifecycle — issuance, rotation, revocation, trust distribution — which is why service meshes exist and why doing mTLS by hand does not scale.

**When to use it**

- **Edge termination** when the proxy must do L7 work and the internal network is genuinely trusted and segmented — increasingly rare.
- **Re-encryption** as the sensible default for regulated or sensitive data with an L7 edge.
- **Passthrough** when the backend must see client certificates, or when the proxy must not be able to read the payload at all.
- **mTLS everywhere** for zero-trust internal architectures, multi-tenant clusters, and anywhere IP-based authorisation is inadequate.

**When to avoid it**

- **Do not terminate at the edge and call the internal network safe** without segmentation, because a single compromised workload can then read everything.
- **Do not implement mTLS by hand across many services** — certificate rotation will eventually cause an outage. Use a mesh or an automated CA.
- **Do not trust forwarded headers** from anywhere but your own proxies.
- **Do not use passthrough** when you need path routing, caching or a WAF; those require plaintext.
- **Do not pin long-lived certificates** in clients; rotation becomes a coordinated outage.

**Advantages**

- **Edge termination** is simplest, fastest, and enables all L7 features with one certificate to manage.
- **Re-encryption** keeps the internal wire encrypted while preserving routing and inspection.
- **Passthrough** gives true end-to-end confidentiality and carries client certificates to the origin.
- **mTLS** replaces network-position trust with cryptographic identity, which makes lateral movement far harder.

**Disadvantages**

- **Each termination point is a plaintext exposure** and a component inside the compliance boundary.
- **Re-encryption doubles handshakes** and requires the proxy to trust upstream certificates.
- **Passthrough removes L7 capability** entirely from the proxy.
- **mTLS adds a certificate authority, rotation, and trust distribution** — a permanent operational surface.
- **Debugging encrypted internal traffic is harder**, requiring mesh tooling rather than tcpdump.

**Trade-offs**

**Topology comparison**

|  | Edge | Re-encrypt | Passthrough | mTLS mesh |
|---|---|---|---|---|
| L7 routing / WAF | Yes | Yes | No | Yes |
| Internal plaintext | Yes | No | No | No |
| Client cert reaches origin | No | No | Yes | n/a (per-hop identity) |
| Handshakes | 1 | 2 | 1 | 1 per hop |
| Cert management | Minimal | Moderate | At origin | Automated CA required |
| Identity model | Network position | Network position | Client cert | Cryptographic per workload |

**How it fails**

**TLS failure modes**

| Failure | Symptom | Fix |
|---|---|---|
| Certificate expiry | Total outage at a precise moment | Automated renewal; alert at 30/14/7 days remaining |
| Clock skew | Certificates rejected as not-yet-valid | NTP everywhere; monitor drift |
| Missing intermediate chain | Works in browsers, fails in strict clients | Serve the full chain; test with non-browser clients |
| Spoofed forwarded headers | Rate limits and allowlists bypassed | Strip inbound, trust only from known proxies |
| SNI mismatch on passthrough | Connection resets that look like network faults | Verify SNI routing rules against certificate names |
| Handshake CPU exhaustion | Latency spikes during reconnect storms | Session resumption, keep-alive, hardware offload |
| mTLS rotation failure | A subset of services cannot talk to each other | Overlapping validity windows; staged rotation with canary |

**Limits**

> **Practical numbers**
>
> - **TLS 1.3 handshake**: 1 RTT, or 0 RTT on resumption (with replay caveats for non-idempotent requests).
> - **Symmetric encryption overhead**: roughly 1–3% CPU with AES-NI; the handshake dominates, not the bulk cipher.
> - **Public certificate lifetimes** are trending shorter (90 days and below), making automation mandatory.
> - **Mesh workload certificates** are typically valid for hours, which limits the damage from a leaked key.
> - **Cookie/header size** matters: forwarded certificate details can be large; keep them minimal.

**Alternatives**

| Approach | Protects | Cost |
|---|---|---|
| TLS / mTLS | Traffic in transit, with identity | Certificate lifecycle |
| IPsec / WireGuard tunnels | All traffic between hosts, transparently | Network-layer only; no per-workload identity |
| Application-level encryption | Specific fields end to end, even from the database | Key management; breaks queryability |
| Network segmentation alone | Limits who can reach what | No confidentiality against an in-segment attacker |

These compose: segmentation limits reach, mTLS authenticates and encrypts hops, and field-level encryption protects the few values that must stay secret even from operators and backups.

**In real systems**

- **CDNs** terminate at the edge for caching and WAF, and offer re-encryption to origin as a standard configuration for sensitive workloads.
- **Istio, Linkerd and other meshes** issue short-lived workload certificates and enforce mTLS transparently, which is the practical way to run zero-trust at scale.
- **AWS ALB/NLB** expose exactly this choice: ALB terminates and can re-encrypt; NLB can pass TLS through untouched.
- **Let's Encrypt and ACME** made automated certificate lifecycle the norm, which is what makes short lifetimes practical.
- **Payment and healthcare architectures** typically mandate no plaintext hops, forcing re-encryption or mTLS throughout.

**Common mistakes**

- **Terminating at the edge and assuming the internal network is private.**
- **Appending to an inbound `X-Forwarded-For`** instead of replacing it, letting clients forge source addresses.
- **Manual certificate renewal**, which fails exactly once and takes the service down.
- **Serving an incomplete certificate chain**, which works in browsers and breaks API clients.
- **Enabling 0-RTT resumption for non-idempotent requests**, exposing them to replay.
- **Choosing passthrough and then needing path-based routing.**
- **Rotating a CA without an overlapping trust window.**

**The staff-level view**

The Staff-level contribution is to make the termination topology an explicit, documented decision rather than a side effect of load balancer defaults.

- **Draw the plaintext map.** For each data class, mark every hop where it exists in the clear. That diagram usually ends the debate on its own.
- **Automate certificate lifecycle before mandating encryption.** Mandating mTLS without automated rotation guarantees an outage.
- **Define trusted-proxy boundaries centrally** so header spoofing is impossible by construction rather than by each team's care.
- **Prefer the mesh default over per-service configuration**, because uniform automated mTLS is far safer than heterogeneous hand-rolled TLS.
- **Track certificate expiry as a first-class SLI.** Expiry outages are self-inflicted, entirely predictable, and embarrassingly common.

**Go deeper**

TLS termination is the point where the envelope is opened, and everything downstream of it sees plaintext. Edge termination at a CDN or load balancer is simplest and enables all L7 features — routing, caching, WAF — but leaves internal traffic readable. Re-encryption adds a second handshake to keep the internal hop encrypted. Passthrough keeps the connection end to end and delivers client certificates to the origin, but the proxy can then route only on SNI.

The deciding question is what the proxy must be able to read. Anything requiring path routing, header rewriting, caching or inspection forces decryption. Once terminated, client identity must be forwarded explicitly via `X-Forwarded-*` headers that the proxy sets and strips from inbound requests, and that backends trust only from known proxy addresses — otherwise any caller can forge a source address.

mTLS everywhere replaces network-position trust with cryptographic workload identity, which is what makes zero-trust practical. Its cost is not CPU — symmetric encryption is a few percent — but certificate lifecycle: issuance, rotation, revocation and trust distribution. Automate that first, roll out in permissive mode to inventory real traffic, then enforce; mandating mTLS without automated rotation reliably produces an outage.

Where TLS terminates is a security architecture decision that most teams make implicitly, by configuring whichever load balancer was convenient. Making it explicit is the whole value of understanding it.

**The three topologies.** Edge termination decrypts at the CDN or load balancer and forwards plaintext: one certificate, one handshake, full layer-7 capability, and a plaintext blast radius covering every internal hop. Re-encryption decrypts for routing and then opens a fresh TLS connection upstream: two handshakes, but nothing on the wire is readable. Passthrough forwards TCP untouched and routes only on the SNI field, preserving end-to-end confidentiality and delivering client certificates to the origin, at the cost of all L7 features. The selection rule is simply: what must the proxy read?

**Identity after termination.** Once a proxy terminates, the backend sees the proxy's address, so client identity must be forwarded explicitly. The critical detail is that inbound `X-Forwarded-*` headers are attacker-controlled and must be stripped and replaced, never appended to — and backends must accept them only from a configured set of proxy addresses, with direct backend access blocked at the network layer. Getting this wrong turns every IP allowlist, per-client rate limit and audit log into fiction.

**mTLS and the shift in trust model.** Mutual TLS gives each hop a cryptographic identity, which means authorisation can be expressed as “service A may call service B” rather than “this IP range is trusted.” That is the substance of zero trust: a compromised workload can no longer move laterally merely because it sits inside the perimeter. Service meshes make this feasible by issuing short-lived per-workload certificates automatically, typically valid for hours, which also bounds the damage from a leaked key.

**The cost is operational.** Symmetric encryption with hardware acceleration costs a small single-digit percentage of CPU; the handshake is the expensive part and amortises across a long-lived connection, which is one more reason connection reuse matters. The genuine, permanent cost is certificate lifecycle. Public certificate lifetimes are trending shorter, mesh certificates are measured in hours, and manual renewal fails exactly once — at a precisely predictable moment — and takes the service down. Automation must precede any encryption mandate.

**Migration.** Moving from perimeter trust to zero trust safely follows a fixed order: build and prove automated certificate issuance and rotation; run mTLS in permissive mode so services accept both plaintext and mTLS while reporting which peers are still plaintext, producing a real inventory rather than an assumed one; then enforce incrementally, least critical first. Encryption is the easy half; the durable win is migrating authorisation from IP allowlists to workload identity, and the discipline that makes it safe is that every stage is observable before it is enforcing.

**Prove it — interview questions**

1. **[Basic] What is TLS termination?**

   <details><summary>Model answer</summary>

   It is the point in the request path where encrypted traffic is decrypted. Typically a load balancer or CDN holds the certificate, completes the handshake with the client, and forwards the request onward — either as plaintext, or re-encrypted over a new TLS connection to the backend. That point matters because every component after it can read the traffic.

   </details>

2. **[Senior] When would you choose passthrough over termination at the load balancer?**

   <details><summary>Model answer</summary>

   When the backend must see the client's certificate for mutual authentication, or when the proxy must be incapable of reading the payload — for example a regulated workload where the load balancer is operated by a different trust domain. The cost is that the proxy can then route only on SNI, so path-based routing, header injection, caching and WAF inspection are all unavailable. If I need any of those, the real choice is between edge termination and re-encryption.

   </details>

3. **[Senior] How do you preserve client identity after terminating TLS?**

   <details><summary>Model answer</summary>

   The proxy injects it explicitly: the real peer address in `X-Forwarded-For`, the original scheme in `X-Forwarded-Proto`, and client-certificate details in a dedicated header when mTLS is used. Crucially it must *strip* any inbound versions of those headers rather than appending to them, and the backend must trust them only when they arrive from a known proxy address. Without both halves, a client can forge its own identity and bypass allowlists, rate limits and audit trails.

   </details>

4. **[Senior] What does terminating TLS at the edge cost you?**

   <details><summary>Model answer</summary>

   Visibility at the edge in exchange for plaintext behind it. Once TLS terminates at a load balancer, everything downstream travels unencrypted unless you deliberately re-encrypt, so the internal network becomes part of your trust boundary — which is a defensible position in a controlled data centre and an untenable one in a shared or regulated environment. You also move the certificate and key material to the termination point, which concentrates both the operational burden and the blast radius of a compromise. The usual resolution is terminating at the edge for the performance and routing benefits, then re-encrypting to the backend so the plaintext segment is bounded and auditable.

   </details>

5. **[Staff] Design the TLS topology for a system handling payment data.**

   <details><summary>Model answer</summary>

   I would start from the requirement that no cardholder data exists in plaintext on any network segment, which rules out edge termination straight to plaintext backends. The edge terminates because I need a WAF and bot protection, then re-encrypts to the internal gateway. Inside the cluster I would use mesh-issued mTLS with short-lived workload certificates, so service-to-service authorisation is based on cryptographic identity rather than IP address — which also shrinks the audit story to “every hop is encrypted and authenticated.” The costs I would name explicitly: an extra handshake per hop, amortised by connection reuse; a few percent CPU; and a certificate authority with automated rotation, which is the real ongoing investment. I would not mandate mTLS until that automation is proven, because manual rotation at scale reliably causes outages.

   </details>

6. **[Principal] How do you move an organisation from perimeter trust to zero trust without an outage?**

   <details><summary>Model answer</summary>

   Incrementally, and with the automation first. Step one is a certificate authority with fully automated issuance and rotation, validated on non-critical services — without it, everything downstream eventually breaks on an expiry. Step two is running mTLS in permissive mode, where services accept both plaintext and mTLS while reporting which peers are still plaintext; that gives a real inventory instead of an assumed one. Step three is enforcing per-namespace or per-service as coverage reaches 100%, starting with the least critical and using the reports to catch stragglers. Throughout, authorisation policy migrates from IP allowlists to workload identity, which is the actual goal — encryption is the easy half. The discipline that makes this safe is that every step is observable before it is enforcing, so the organisation is never guessing about what will break.

   </details>

---

### Layer 4 and layer 7 load balancing

*L4 moves connections and is fast and opaque; L7 understands requests and can route, retry and shed — at the cost of decryption and CPU.*

**Flow:** `Client` → `Balancer` → `Health policy` → `Backend pool` → `Service`

> **The 30-second version**  
> L4 routes connections cheaply and blindly; L7 routes each request and can retry, shed and eject failing backends. Use L4 at the perimeter and L7 wherever decisions matter.

**The problem**

A load balancer that only sees TCP connections cannot tell a health check from a request, cannot retry a failed call, cannot route `/api/v2` differently from `/api/v1`, and cannot notice that one backend is returning errors as fast as it can.

A load balancer that understands HTTP can do all of those things — but it must decrypt the traffic, parse every request, and spend meaningfully more CPU per byte. Choosing between them is about what decisions you need made in the path, and where.

> **The distinction in one sentence**  
> **L4 balances connections; L7 balances requests.** With a long-lived HTTP/2 connection carrying thousands of requests, an L4 balancer makes exactly one routing decision and then every subsequent request is stuck on that backend — which is why L4 in front of multiplexed protocols produces badly skewed load.

**Mental model**

Think of L4 as a switchboard operator who connects a call and walks away, and L7 as a receptionist who listens to each request and decides where it should go.

1. **L4 (transport)** — Sees addresses, ports and TLS SNI. Chooses a backend per *connection*. Cheap, fast, protocol-agnostic, and cannot see failures.
2. **L7 (application)** — Parses HTTP. Chooses a backend per *request*. Can route by path/header/cookie, retry idempotent calls, rewrite, cache, rate limit, and eject backends that return errors.
3. **Health checking** — L4 can only test that a port accepts connections. L7 can test that the application actually works, which is a different question.
4. **Direct server return / DSR** — An L4 optimisation where responses bypass the balancer entirely — excellent for high-egress workloads, impossible at L7.

> **Health checks are the real difference**  
> An L4 check confirms a socket opens. A process that has deadlocked, lost its database connection, or is returning 500s for every request will still accept sockets. L7 checks hit a real endpoint that exercises dependencies, and L7 balancers can additionally use *passive* health signals — ejecting a backend because its live error rate spiked — which is the single most valuable feature in this topic.

**How it works**

**What each layer can decide**

```text
L4                                L7
---------------------------       ---------------------------
route by IP/port/SNI              route by path, header, cookie,
                                  method, query, JWT claim
one decision per CONNECTION       one decision per REQUEST
health = "port open"              health = "GET /healthz is 200"
no retries                        retry idempotent requests
no request visibility             observability per endpoint
DSR possible                      response flows back through
~microseconds, minimal CPU        parse + TLS: more CPU per byte
any TCP/UDP protocol              HTTP/gRPC/WebSocket aware
```

1. **Balancing algorithms matter more than people expect** — Round robin ignores backend state. Least-connections adapts to slow backends. **Least outstanding requests** (or peak-EWMA) is generally best for L7, because it naturally routes around instances whose queues are deep.
2. **Passive health and outlier ejection** — Beyond active probes, eject a backend when its observed error rate or latency deviates from its peers, then probe it back in gradually. This catches the sick-but-alive instance that active checks miss.
3. **Connection draining** — On removal, stop sending new work but let in-flight requests finish. Without draining, every deploy produces a burst of client errors.
4. **Retries need budgets** — L7 retries turn transient failures invisible — and turn an overload into a self-amplifying storm if uncapped. Budget retries as a percentage of traffic and never retry non-idempotent requests blindly.
5. **Layer them** — A common production shape is L4 (or anycast) at the edge for raw scale and DDoS absorption, then L7 inside for routing, retries and observability.

**Why L4 + HTTP/2 skews load**

```text
10 clients, each opening ONE long-lived HTTP/2 connection
carrying 1,000 requests

L4 balancer, 5 backends:
  assigns 10 connections -> 2 per backend, looks balanced
  but a heavy client's 1,000 requests all land on ONE backend
  -> per-request load can be wildly uneven and cannot rebalance

L7 balancer:
  makes 10,000 independent routing decisions
  -> even distribution, and it can shift away from a slow backend
     mid-connection
```

> **Session affinity is a liability, not a feature**  
> Cookie- or IP-based stickiness concentrates load, breaks safe scale-in, and turns any backend failure into user-visible state loss. It exists to support stateful backends; the correct fix is to externalise the state (see stateless service design) and turn stickiness off.

**Worked example**

A public API with a CDN, a regional entry point, and internal microservices. Which layer goes where?

**Layering the balancers**

```text
EDGE          anycast IP + L4
              absorbs volumetric DDoS, TLS passthrough or
              termination, routes to nearest region
              -> fast, cheap per byte, protocol-agnostic

REGION        L7 gateway
              terminates TLS, authenticates, rate limits,
              routes /v1 and /v2 to different services,
              retries idempotent GETs, emits per-endpoint metrics

INTERNAL      L7 sidecar / mesh
              per-request load balancing across pods,
              outlier ejection, mTLS, circuit breaking,
              least-outstanding-request selection

WHY NOT L7 EVERYWHERE?
  the edge handles enormous volume where per-request parsing
  is expensive and unnecessary; SNI is enough to steer.

WHY NOT L4 INTERNALLY?
  gRPC uses long-lived HTTP/2 connections. L4 would pin
  each connection to one pod and skew load badly.
```

| Metric | Value | Note |
|---|---|---|
| Edge (L4) | µs latency | DDoS absorption |
| Gateway (L7) | auth, routing | retries + metrics |
| Mesh (L7) | per-request | **outlier ejection** |
| L4 internally | avoid | skews HTTP/2 |

**When to use it**

- **L4** for very high throughput, non-HTTP protocols, TLS passthrough, DDoS absorption, and anywhere per-request parsing is wasted.
- **L4** when direct server return matters — high-egress workloads like video where responses should bypass the balancer.
- **L7** whenever you need path/header routing, retries, per-endpoint observability, rate limiting, or request-level fairness.
- **L7** in front of gRPC or HTTP/2, always, because connection-level balancing skews badly.
- **Both, layered**, in most large systems: L4 or anycast at the perimeter, L7 inside.

**When to avoid it**

- **Do not put L4 in front of long-lived multiplexed connections** and expect balanced load.
- **Do not rely on port-open health checks** for anything that has dependencies — they pass while the service is useless.
- **Do not enable retries without a budget**; you will convert a brownout into an outage.
- **Do not retry non-idempotent requests** at the load balancer unless idempotency keys make it safe.
- **Do not use session affinity as a substitute for externalising state.**

**Advantages**

|  | L4 | L7 |
|---|---|---|
| Latency added | Microseconds | Sub-millisecond to milliseconds |
| CPU per byte | Very low | Higher (parse + TLS) |
| Protocols | Any TCP/UDP | HTTP-family only |
| Routing granularity | Per connection | Per request |
| Health signal | Port reachable | Application-level + passive outlier detection |
| Retries / circuit breaking | No | Yes |
| Observability | Connections and bytes | Per-endpoint latency, status, size |
| DSR possible | Yes | No |

**Disadvantages**

- **L4** cannot see failures, cannot retry, and skews load with multiplexed protocols.
- **L4** health checks are nearly meaningless for application health.
- **L7** must decrypt, expanding the plaintext blast radius and the compliance boundary.
- **L7** costs more CPU per byte and adds a parsing step to every request.
- **L7** becomes a complex, stateful, business-logic-bearing component if you let it — routing rules and header rewrites accumulate into an unreviewable configuration.

**Trade-offs**

**Selection algorithms**

| Algorithm | Adapts to slow backends | Use when |
|---|---|---|
| Round robin | No | Uniform request cost, homogeneous backends |
| Weighted round robin | No (static) | Heterogeneous instance sizes |
| Least connections | Partially | L4, long-lived connections |
| Least outstanding requests | Yes | L7 default — routes around deep queues |
| Peak EWMA / latency-aware | Yes, strongly | Heterogeneous latency; tail-sensitive services |
| Consistent hash | No | Cache affinity where hit rate matters more than balance |

Least-outstanding-requests is the right default for L7 because it is self-correcting: a backend that slows down accumulates outstanding requests and automatically receives less traffic, without any explicit health signal.

**How it fails**

**Load balancing failures**

| Failure | Cause | Fix |
|---|---|---|
| Traffic to a dead backend | Health check too shallow or interval too long | Deep health endpoint; shorter interval; passive ejection |
| Errors on every deploy | No connection draining | Drain on removal; readiness gates before adding |
| One backend melts, others idle | L4 with multiplexed connections, or consistent-hash skew | L7 per-request balancing |
| Retry storm | Uncapped retries during a brownout | Retry budget, jitter, circuit breaker |
| Flapping backends | Health thresholds too tight; check competes with load | Hysteresis; separate health path; require N consecutive results |
| Cascading ejection | Outlier ejection removes so many backends that the rest overload | Cap the ejected fraction (e.g. max 30%) |
| Sticky sessions overload one node | Affinity plus heavy users | Remove affinity; externalise session state |

> **Health checks that share the failure**  
> If the health endpoint queries the same database the requests do, then a database outage marks every backend unhealthy, the balancer removes them all, and a degraded system becomes a completely unavailable one. Health checks should verify the process can serve, with dependency checks reported separately — so a shared dependency failure degrades responses rather than deleting the entire pool.

**Limits**

> **Operating numbers**
>
> - **Health check interval** 5–10 s with 2–3 consecutive failures gives 10–30 s detection without excessive flapping.
> - **Outlier ejection**: eject on a 5× error-rate deviation, cap ejected backends at ~30% of the pool.
> - **Retry budget**: cap retries at ~10% of request volume; always use jittered backoff.
> - **Draining period** should exceed the p99 request duration, typically 30–60 s.
> - **L7 overhead**: expect a fraction of a millisecond of added latency, and materially more CPU per byte than L4.

**Alternatives**

| Approach | Where the decision is made | Trade |
|---|---|---|
| Hardware/network L4 | In the network path | Fastest; least intelligence |
| Software L7 proxy (Envoy, nginx) | A dedicated hop | Flexible; an extra hop to operate |
| Service mesh sidecar | Next to every workload | Per-request control everywhere; sidecar cost |
| Client-side load balancing | In the caller's library | No extra hop; every client must implement it correctly |
| DNS round robin | In the client's resolver | Free; no health awareness, poor balance |

Client-side balancing (as in gRPC's built-in balancer) is attractive for internal traffic because it removes a hop and a failure domain — at the cost of needing consistent behaviour across every language's client library, which is exactly what meshes exist to avoid.

**In real systems**

- **Envoy** is the reference L7 implementation, and its outlier detection plus least-request balancing are the features most worth copying conceptually.
- **AWS NLB (L4) and ALB (L7)** make the distinction explicit as separate products with different price and capability profiles.
- **Maglev-style L4 balancers** at hyperscale use consistent hashing plus DSR to handle enormous connection volumes cheaply.
- **gRPC's client-side balancing** exists precisely because L4 balancing of long-lived HTTP/2 connections distributes load badly.
- **CDNs** combine anycast L4 steering with L7 logic at the edge PoP, which is the layered pattern in its most common form.

**Common mistakes**

- **L4 in front of gRPC**, producing severe load skew.
- **Port-open health checks** that pass while the service returns 500s.
- **Health endpoints that query shared dependencies**, so one outage removes every backend.
- **Retries with no budget**, amplifying a brownout.
- **No connection draining**, so every deploy produces client errors.
- **Session affinity** used to paper over in-process state.
- **Uncapped outlier ejection**, which can remove the whole pool.

**The staff-level view**

The load balancer is where reliability policy is actually enforced, so it deserves the same design scrutiny as application code.

- **Make health checks meaningful and independent.** Deep enough to catch a broken process, isolated enough that a shared dependency failure does not delete the pool.
- **Turn on passive outlier ejection with a cap.** It catches the failure mode active checks miss, and the cap prevents it from cascading.
- **Set retry budgets centrally**, not per team, because the failure mode is systemic amplification.
- **Use L7 per-request balancing for anything HTTP/2 or gRPC.** This is a correctness issue for load distribution, not an optimisation.
- **Resist logic accumulation in the L7 config.** Routing rules and header rewrites become unreviewable business logic in a file nobody tests.

**Go deeper**

An L4 balancer sees addresses, ports and TLS SNI, and picks a backend once per connection. An L7 balancer parses HTTP and picks a backend per request, which enables path and header routing, retries, rate limiting, circuit breaking and per-endpoint observability. The cost is decryption and meaningfully more CPU per byte.

The most consequential difference is health. L4 can only confirm a port accepts connections, which a deadlocked process still does. L7 can probe a real endpoint and, more valuably, use passive signals — ejecting a backend whose live error rate or latency deviates from its peers. The second most consequential difference is that L4 in front of multiplexed protocols like gRPC pins thousands of requests to one backend per connection, skewing load badly and preventing mid-connection avoidance of a slow instance.

Large systems layer both: anycast or L4 at the perimeter for raw volume and DDoS absorption, an L7 gateway per region for auth, routing and retries, and L7 again internally for per-request fairness. Key operational settings are least-outstanding-request selection, connection draining longer than p99 request duration, retry budgets capped near 10% of traffic, outlier ejection capped at about 30% of the pool, and health checks that do not fail merely because a shared dependency is down.

Load balancing is where reliability policy is actually enforced, which makes the L4/L7 choice more consequential than its framing as a networking detail suggests.

**What each layer can decide.** L4 sees connection-level facts — addresses, ports, SNI — and makes one decision per connection, in microseconds, for any TCP or UDP protocol. It can also support direct server return, where responses bypass the balancer entirely, which matters enormously for high-egress workloads. L7 parses the application protocol and makes a decision per request, which unlocks path and header routing, retries, rewrites, caching, rate limiting, circuit breaking and per-endpoint telemetry — at the cost of terminating TLS and spending real CPU per byte.

**Health is the sharpest difference.** A port-open check passes for a process that has deadlocked, lost its database pool, or is returning errors as fast as it can. L7 active checks can exercise a real endpoint, but the more valuable capability is passive: ejecting a backend because its observed error rate or latency deviates from its peers under real traffic. That catches the sick-but-alive instance that dominates tail latency and that no active probe notices. Two guardrails are essential — cap the fraction of the pool that may be ejected, so ejection cannot cascade into removing everything, and never let the health endpoint depend on a shared backend, or one database outage deletes the entire pool and converts degradation into total unavailability.

**Multiplexing breaks L4.** With HTTP/2 and gRPC, one long-lived connection carries thousands of requests. An L4 balancer decides once, at connection setup, so a heavy client's entire workload lands on one backend and cannot be moved even as that backend degrades. This is not an optimisation issue; it is a load-distribution correctness issue, and it is the main reason internal service traffic uses L7 proxies, sidecars, or client-side balancing.

**Algorithm choice matters.** Round robin ignores backend state entirely. Least-connections adapts partially. Least outstanding requests — or a latency-weighted variant like peak EWMA — is self-correcting: a backend that slows accumulates outstanding requests and automatically receives less traffic, with no explicit health signal required. Consistent hashing is the exception, chosen deliberately when cache locality is worth more than even distribution.

**Operational settings that matter more than the layer choice.** Connection draining longer than p99 request duration, or every deploy produces client errors. Readiness gates before a backend joins the pool. Retry budgets capped as a fraction of traffic with jittered backoff, because uncapped L7 retries convert a brownout into a self-sustaining outage. Health thresholds with hysteresis so backends do not flap. And session affinity treated as a liability to be removed rather than a feature, since it concentrates load, breaks scale-in, and makes any backend failure user-visible.

**The layering pattern.** Most large systems run anycast or L4 at the perimeter where volume is enormous and SNI-level steering suffices, an L7 gateway per region for authentication, versioned routing, rate limiting and telemetry, and L7 per-request balancing internally via sidecar or client library. Each tier exists because it has a different scarce resource. The discipline worth enforcing is that the L7 configuration owns traffic policy and not business meaning — once tenant-specific rewrites and entitlement logic accumulate there, the most business-critical artifact in the system is an untested configuration file that nobody owns.

**Prove it — interview questions**

1. **[Basic] What is the difference between L4 and L7 load balancing?**

   <details><summary>Model answer</summary>

   L4 operates on connections: it sees addresses, ports and TLS SNI, picks a backend when the connection opens, and forwards bytes without understanding them. L7 parses the application protocol, usually HTTP, and picks a backend per request, which lets it route by path or header, retry, rate limit and report per-endpoint metrics. L4 is cheaper and protocol-agnostic; L7 is more capable but must decrypt and parse.

   </details>

2. **[Senior] Why does L4 balancing behave badly in front of gRPC?**

   <details><summary>Model answer</summary>

   Because gRPC multiplexes many requests over a single long-lived HTTP/2 connection. An L4 balancer makes one routing decision when that connection is established, so every subsequent request from that client lands on the same backend for the life of the connection. A heavy client's thousands of requests all hit one pod while others sit idle, and the balancer cannot shift traffic away even if that pod becomes slow. L7 balancing makes an independent decision per request, which both distributes evenly and allows mid-connection avoidance of a degraded backend.

   </details>

3. **[Senior] How should a health check be designed?**

   <details><summary>Model answer</summary>

   It should verify that this process can serve requests, without failing because a shared dependency is down. A liveness-style check confirms the process is not deadlocked; a readiness check confirms local initialisation is complete and it is willing to accept traffic. Dependency health belongs in a separate, reported signal rather than in the check that controls pool membership — because if every backend queries the same database in its health check, a database outage removes the entire pool and turns a degraded system into a completely unavailable one. I would pair that with passive outlier ejection so backends returning errors are removed based on real traffic, which catches failures active probes cannot see.

   </details>

4. **[Senior] Why does layer 7 balancing cost more than layer 4, and when is it worth it?**

   <details><summary>Model answer</summary>

   Because it has to terminate the connection and parse the request before it can decide anything, so it holds connection state, does the TLS work, and spends CPU per request rather than per connection. Layer 4 forwards packets on the basis of addresses and ports and can therefore move enormous volume very cheaply. The cost buys decisions that require knowing what the request is: routing by path or header, retrying an idempotent request on another backend, rewriting, per-endpoint rate limiting and meaningful request-level observability. So the useful arrangement is usually both — layer 4 at the outer edge for volume and for absorbing floods, layer 7 behind it where the routing intelligence is actually needed.

   </details>

5. **[Staff] Design the load balancing layers for a large public API.**

   <details><summary>Model answer</summary>

   Anycast plus L4 at the perimeter, because that tier must absorb enormous volume and volumetric attacks where per-request parsing is wasted, and SNI is enough to steer to the right region. An L7 gateway per region that terminates TLS, authenticates, applies rate limits, routes by version and path, retries idempotent requests within a budget, and produces per-endpoint telemetry. Then L7 again internally — sidecar or client-side — for per-request balancing across pods with least-outstanding-request selection, outlier ejection capped at around 30% of the pool, and circuit breaking. The reason for three tiers rather than one is that each has a different scarce resource: the edge is bandwidth-bound, the gateway is policy-bound, and the internal tier is latency- and fairness-bound.

   </details>

6. **[Principal] Where do you draw the line on what logic belongs in the load balancer?**

   <details><summary>Model answer</summary>

   I let it own traffic policy and nothing else: routing by stable attributes, retries, timeouts, rate limits, circuit breaking, health and observability. The moment it starts making decisions that depend on business meaning — pricing tiers, feature entitlements, tenant-specific rewrites — the configuration becomes untested, unreviewable business logic in a file that no engineer owns and no test exercises. The practical control is to require that every routing rule be expressible in terms the platform team can reason about, and to push anything else into the application where it can be unit-tested and versioned with the code. The failure mode I am guarding against is the gateway config becoming the most business-critical and least understood artifact in the organisation, which is a place many companies arrive at without noticing.

   </details>

---

### Connection pooling

*Reuse expensive connections instead of creating them per request — and use the pool size as your primary concurrency limit against the database.*

**Flow:** `Request` → `Pool wait` → `Borrow connection` → `Database` → `Release`

> **The 30-second version**  
> Reuse connections to avoid handshake cost — and treat the pool size as your concurrency limit against the database, because a bigger pool during an overload makes it worse, not better.

**The problem**

Opening a database connection costs a TCP handshake, a TLS handshake, authentication, and often session setup — typically 5–50 ms and a meaningful amount of server-side memory. Doing that per request wastes most of your latency budget on setup.

But the deeper problem is the one teams discover during an incident: the pool is not just a performance optimisation, it is a **concurrency limit**. Its size determines how many requests can be in flight against the database at once, and therefore whether a slow database degrades your service or destroys it.

> **The pool-exhaustion cascade**  
> A query slows from 10 ms to 200 ms. By Little's law, in-flight work grows 20×. The pool exhausts. New requests block waiting for a connection, so their latency grows too. Clients time out and retry, adding load. The database, now receiving more concurrent work than before, slows further. **Nothing changed except one query's latency**, and the service is down.

Which is why the instinctive remedy — make the pool bigger — usually makes things worse: it pushes *more* concurrent work onto a database that is already struggling.

**Mental model**

A pool is a bounded set of pre-established connections plus a queue of borrowers. Its size is a statement about how much concurrency the downstream system should ever see from this service.

1. **Borrow** — A request takes a connection. If none is free, it waits in the acquisition queue (up to a timeout).
2. **Use** — The connection carries one query or transaction. Hold time is what matters — Little's law uses it directly.
3. **Release** — Returned to the pool, ideally with session state reset so the next borrower is not surprised.
4. **Bound** — Max size caps concurrency; acquisition timeout caps how long a request waits; max lifetime forces periodic re-establishment.

> **Pool size is Little's law with a safety property**  
> Required connections = request rate × average hold time. At 500 rps holding a connection for 20 ms, that is 10. But the pool's *value* is that it refuses to grow: when hold time rises to 200 ms and demand jumps to 100, a pool of 25 makes 75 requests wait or fail fast — which protects the database instead of drowning it.

**How it works**

**Pool sizing and the parameters that matter**

```text
STEADY STATE
  connections = rps x hold_time
  500 rps x 0.020 s = 10 connections

SIZE FOR VARIANCE, THEN CAP
  pool_max = 2-3x steady state = 25
  implied latency ceiling before queueing:
    25 / 500 = 50 ms of average hold time

PARAMETERS
  max_size            concurrency cap -> the important one
  min_idle            keeps warm connections for bursts
  acquisition_timeout SHORT (100-500 ms). Fail fast, do not
                      let callers queue behind a sick database
  max_lifetime        recycle (e.g. 30 min) so DNS changes and
                      load balancer rebalancing take effect
  idle_timeout        release unused connections
  validation query    cheap liveness test on borrow
```

1. **Size against the database's capacity, not your app's ambition** — Total connections = pool size × number of instances. Twenty app servers with a pool of 50 means 1,000 connections — which many databases cannot handle, and each one costs server memory.
2. **Keep the acquisition timeout short** — A long acquisition timeout converts a database slowdown into a full-service stall. A short one converts it into fast, visible rejections that a circuit breaker can act on.
3. **Bound connection lifetime** — Without a max lifetime, connections pin themselves to whichever database replica they first resolved, so failovers and DNS changes never take effect.
4. **Separate pools by workload** — Reporting queries and interactive queries should not share a pool; otherwise one slow report starves the checkout path. This is a bulkhead.
5. **Reset session state on release** — Temporary tables, session variables, prepared statement caches and search paths leak between users if not cleared — a correctness and security issue, not a performance one.
6. **Consider an external pooler at scale** — PgBouncer-style poolers multiplex many client connections onto few server connections, which is how you serve thousands of app instances from a database that supports hundreds of connections.

**Why a bigger pool makes an overload worse**

```text
database can serve 100 concurrent queries efficiently

pool = 100 per app x 10 apps = 1000 concurrent
  -> database context-switches, lock contention rises,
     each query gets SLOWER
  -> hold time rises -> more concurrency needed -> spiral

pool = 20 per app x 10 apps = 200 concurrent
  -> database stays in its efficient regime
  -> excess requests wait briefly or fail fast
  -> total THROUGHPUT is higher, latency is bounded

Less concurrency, more throughput. This is counter-intuitive
and it is the single most useful fact about pool sizing.
```

> **The classic sizing heuristic**  
> For CPU-bound OLTP databases, a well-known starting point is roughly `cores × 2 + effective spindle count` for the *total* connections across all clients — often a far smaller number than teams expect, frequently under 50. Start small, measure, and only increase if the database is demonstrably idle while requests queue.

**Worked example**

A service at 800 rps, mean query time 15 ms, running 12 instances against one PostgreSQL primary.

**Sizing the pool correctly**

```text
PER-INSTANCE DEMAND
  800 rps / 12 instances     = 67 rps per instance
  67 x 0.015 s               = 1 connection steady state (!)

SIZE WITH HEADROOM
  pool_max = 8 per instance
  total to database = 8 x 12 = 96 connections

SANITY CHECK AGAINST THE DATABASE
  8 cores -> efficient concurrency ~16-32 queries
  96 connections is fine as a CAP because they are not
  all busy simultaneously; but if they ever were, the
  database would thrash.
  -> consider PgBouncer in transaction mode:
     12 apps x 8 = 96 client connections
     -> 24 server connections, multiplexed

IMPLIED PROTECTION
  acquisition_timeout = 250 ms
  if the DB slows to 100 ms/query, each instance can serve
  8 / 0.1 = 80 rps; above that, requests fail fast in 250 ms
  instead of piling up for 30 seconds.
```

| Metric | Value | Note |
|---|---|---|
| Steady state | 1 conn | per instance |
| Pool max | 8 | 3× headroom, capped |
| Total to DB | 96 | or 24 via pooler |
| Acquire timeout | 250 ms | **fail fast** |

> **What the numbers reveal**  
> Steady-state demand is a single connection per instance, yet the default pool size in most frameworks is 10–20. The default is not a sizing decision, it is an arbitrary number — and it is usually far larger than needed while still being the thing that fails first. Compute it; do not inherit it.

**When to use it**

- **Any expensive-to-establish connection**: databases, message brokers, gRPC channels, HTTP clients to internal services.
- **High request rates** where handshake cost would otherwise dominate latency.
- **Whenever you need a concurrency limit against a downstream** — the pool is the simplest and most reliable one available.
- **Multi-tenant or mixed workloads**, using separate pools per workload class as bulkheads.

**When to avoid it**

- **Do not share one pool across unrelated workloads.** A slow analytics query will starve your checkout path.
- **Do not set a large acquisition timeout.** It converts a downstream slowdown into a service-wide stall.
- **Do not increase pool size to fix an overload** — it adds concurrency to a struggling dependency.
- **Do not leave `max_lifetime` unbounded**, or failover and rebalancing will never take effect.
- **Do not use transaction-mode external poolers with session-scoped features** (session variables, advisory locks, some prepared statements) without understanding what breaks.

**Advantages**

- **Removes handshake cost** from the request path, often the single largest latency win for database-backed services.
- **Bounds downstream concurrency**, which is the primary protection against cascading failure.
- **Enables fast failure** during a downstream slowdown, which a circuit breaker can then act on.
- **Amortises TLS and authentication**, which are expensive per connection and free per reuse.
- **Provides a natural bulkhead boundary** when pools are split per workload.

**Disadvantages**

- **Pool exhaustion is a real and common failure mode**, and its symptoms (timeouts, high latency) look like many other problems.
- **Session state leaks between borrowers** unless carefully reset.
- **Connections pin to a backend**, so load balancer and failover changes need explicit lifetime bounds.
- **Total connection count multiplies across instances**, which is easy to overlook until the database refuses connections.
- **Sizing requires measurement**; defaults are almost always wrong in one direction or the other.

**Trade-offs**

**Pool size trade-offs**

| Pool size | Effect on throughput | Effect on latency | Risk |
|---|---|---|---|
| Too small | Throughput capped by pool/hold-time | Queueing at acquisition | Under-using an idle database |
| Right-sized | Database in its efficient regime | Bounded and predictable | Requires measurement |
| Too large | **Lower** — database thrashes | Unbounded under load | Cascading failure; connection limits hit |
| Unbounded | Collapses under load | Unbounded | Guaranteed outage under a slowdown |

The non-obvious row is “too large”. Beyond the database's efficient concurrency, additional simultaneous queries reduce total throughput through context switching and lock contention. Less concurrency genuinely yields more work done.

> **The sentence that shows understanding**  
> “Steady state needs one connection per instance, so I'd cap the pool at eight with a 250 ms acquisition timeout. The cap is the point: if the database slows, we fail fast and trip a circuit breaker rather than queueing every request behind a sick dependency.”

**How it fails**

**Pool failure modes**

| Symptom | Cause | Fix |
|---|---|---|
| Timeouts waiting for a connection | Hold time rose; pool exhausted | Cap concurrency, shorten acquisition timeout, circuit-break |
| Database refuses connections | pool × instances exceeded the server limit | Reduce pool; add an external pooler |
| Leaked connections, pool drains over time | Missing release on an error path | Always release in a `finally`/`using` block; add leak detection |
| Queries fail after a failover | Connections pinned to the old primary | Bound `max_lifetime`; validate on borrow |
| Mysterious cross-request state | Session variables or temp tables not reset | Reset on release; avoid session-scoped features with poolers |
| One slow report blocks checkout | Shared pool across workloads | Separate pools as bulkheads |
| Latency spike after a restart | Cold pool, all connections establishing at once | `min_idle` warm connections; stagger startup |

**Limits**

> **Sizing numbers**
>
> - **Connections needed = rps × hold time.** Usually far smaller than the default.
> - **Efficient database concurrency** is often near `cores × 2`; more simultaneous queries reduce throughput.
> - **Acquisition timeout** 100–500 ms. Longer converts a dependency slowdown into a service stall.
> - **Max lifetime** 15–60 min, so failovers and DNS changes take effect without a restart.
> - **Total connections = pool × instances.** Check this against the database's configured limit *before* scaling out.

**Alternatives**

| Approach | When | Trade |
|---|---|---|
| In-process pool | Default for every client | Per-instance sizing; total multiplies |
| External pooler (PgBouncer, ProxySQL) | Many app instances, limited server connections | Extra hop; transaction-mode restrictions |
| Serverless data proxy | Function-based workloads with unpredictable instance counts | Managed cost; provider coupling |
| Per-request connections | Very low volume, or scripts | Handshake cost per request |
| Async drivers with multiplexing | Protocols that support concurrent requests per connection | Fewer connections, but no natural concurrency cap — add one explicitly |

The last row is a trap worth naming: async multiplexed drivers remove the connection bottleneck and therefore remove your accidental concurrency limit. You must then add an explicit limiter, or an overload has nothing to push back against.

**In real systems**

- **HikariCP** popularised the “small pools are faster” result, with sizing guidance far below what most teams assume.
- **PgBouncer in transaction mode** is the standard way to serve thousands of application instances from a PostgreSQL primary that supports hundreds of connections.
- **Serverless runtimes** forced the invention of managed data proxies, because per-invocation connections overwhelm databases at scale.
- **gRPC channels** pool and multiplex HTTP/2 connections, which is why gRPC clients need explicit concurrency limits rather than relying on connection scarcity.
- **Most production database incidents attributed to “the database”** are actually pool exhaustion in the application tier.

**Common mistakes**

- **Inheriting the framework default** instead of computing the size.
- **Enlarging the pool during an incident**, adding load to a struggling database.
- **A long acquisition timeout**, turning a slow dependency into a stalled service.
- **Forgetting that total connections multiply** by instance count when scaling out.
- **Sharing one pool across interactive and batch workloads.**
- **Unbounded connection lifetime**, so failover never takes effect.
- **Leaking connections on error paths** without a `finally` release.

**The staff-level view**

Pool configuration is one of the highest-leverage, least-reviewed settings in a typical system. Treat it as a reliability control, not a performance tweak.

- **Standardise pool defaults in the shared service template** — small max size, short acquisition timeout, bounded lifetime — so every team inherits safe behaviour.
- **Publish the total connection budget per database** and make teams account against it before scaling out instance counts.
- **Require separate pools per workload class** so a reporting query can never starve a transactional path.
- **Alert on pool utilisation and acquisition wait time**, which rise before latency and errors do.
- **Teach the counter-intuitive rule**: during an overload, reduce concurrency. The instinct to enlarge the pool is the single most common way engineers deepen an incident.

**Go deeper**

Establishing a connection costs a TCP handshake, a TLS handshake and authentication — typically 5–50 ms plus server memory — so pooling removes that from the request path. The more important role is as a bound: pool size caps how many operations can be in flight against the downstream at once.

Size it with Little's law: connections = request rate × hold time, which is usually far smaller than framework defaults suggest. Then cap it, and remember the total the database sees is pool size × instance count. Beyond a database's efficient concurrency — often near twice its core count — more simultaneous queries reduce total throughput through context switching and lock contention, which is why enlarging a pool during an overload deepens the incident rather than relieving it.

Key settings: a short acquisition timeout (100–500 ms) so a slow dependency produces fast rejections instead of a service-wide stall; a bounded max lifetime (15–60 min) so failovers and DNS changes actually take effect; separate pools per workload class as bulkheads; and session state reset on release. At large instance counts, an external pooler like PgBouncer in transaction mode decouples client connections from server connections entirely.

A connection pool looks like a performance optimisation and behaves like a reliability control. Its sizing decision is one of the least-reviewed and most consequential numbers in a typical service.

**The performance half.** Establishing a connection requires a TCP handshake, a TLS handshake, authentication, and often session initialisation — 5 to 50 milliseconds, plus meaningful server-side memory per connection. Reusing connections removes all of that from the request path and is frequently the single largest latency improvement available to a database-backed service.

**The reliability half.** Pool size caps concurrency against the downstream. This matters because of Little's law: in-flight work equals arrival rate times hold time, so when a query slows from 10 ms to 200 ms, required concurrency grows twentyfold at unchanged traffic. A pool refuses that growth. Requests either wait briefly or fail fast, and the database continues operating in a regime where it is efficient. Without a bound, the application floods a struggling dependency with more concurrent work, hold time rises further, and the slowdown becomes self-sustaining.

**Why bigger is worse.** Beyond a database's efficient concurrency — often around twice its core count — additional simultaneous queries reduce total throughput through context switching, buffer contention and lock waits. Ten application servers with pools of 100 present a thousand concurrent queries to a database that performs best at a few dozen; the same fleet with pools of 20 presents 200, stays in the efficient regime, and completes more work with bounded latency. Less concurrency, more throughput. This is counter-intuitive, and it is why the instinct to enlarge a pool during an incident so reliably deepens it.

**Parameters that matter.** Maximum size is the concurrency cap and the number to compute rather than inherit. Acquisition timeout should be short — 100 to 500 ms — so a downstream slowdown produces visible fast failures that a circuit breaker can act on, rather than a silent stall where every request queues. Maximum connection lifetime must be bounded, because an open connection never re-resolves DNS and never reconsiders its backend, so without recycling a failover or rebalance simply never takes effect. Minimum idle connections avoid a cold-start latency spike. And session state must be reset on release, since temporary tables, session variables and search paths otherwise leak between unrelated requests — a correctness and security issue, not a performance one.

**Scale changes the topology.** Total connections equal pool size times instance count, so scaling out a deployment silently multiplies pressure on a shared database. Past a few dozen instances the right answer is an external pooler in transaction mode, which lets each application hold a small client pool while multiplexing onto a small number of server connections sized to the database's efficient concurrency. The caveat is that transaction-mode pooling breaks session-scoped features, so the application must be checked for session variables, advisory locks and certain prepared-statement patterns.

**One modern trap.** Async drivers that multiplex many queries over one connection remove the pool bottleneck — and with it, the accidental concurrency limit that was protecting the database. The limiter must then be reintroduced explicitly, sized from the same calculation, or overload arrives as a latency cliff with nothing pushing back.

**Prove it — interview questions**

1. **[Basic] Why use a connection pool?**

   <details><summary>Model answer</summary>

   Because establishing a connection is expensive — TCP handshake, TLS handshake, authentication, session setup — typically 5–50 ms plus server-side memory. Pooling amortises that across many requests. Equally important, the pool bounds how many concurrent operations can hit the downstream at once, which is what protects the database when something slows down.

   </details>

2. **[Basic] How do you calculate the right pool size?**

   <details><summary>Model answer</summary>

   Little's law: connections = request rate × average hold time. At 500 requests per second holding a connection for 20 ms, that is 10 connections. I would size the maximum at two to three times that for variance, then treat it as a hard cap. I would also multiply by instance count and check the total against the database's connection limit, because that product is what the database actually sees.

   </details>

3. **[Senior] Your pool is exhausted during an incident. Should you increase it?**

   <details><summary>Model answer</summary>

   Almost never. Exhaustion means hold time rose, so in-flight work grew — and enlarging the pool sends more concurrent work to a dependency that is already struggling, which raises hold time further. Beyond the database's efficient concurrency, extra simultaneous queries reduce total throughput through context switching and lock contention. The correct response is to shorten the acquisition timeout so requests fail fast, trip a circuit breaker, shed load, and fix the underlying slowdown. Enlarging the pool is justified only if the database is measurably idle while requests queue, which is rare.

   </details>

4. **[Senior] Why bound connection lifetime?**

   <details><summary>Model answer</summary>

   Because an open connection never re-resolves DNS and never reconsiders which backend it is attached to. Without a maximum lifetime, connections pin to whichever replica or primary they first reached, so a failover, a DNS change, or a load balancer rebalance has no effect until the application restarts. A 15–60 minute lifetime forces gradual re-establishment, which also spreads the reconnection cost rather than concentrating it.

   </details>

5. **[Staff] Design connection management for 200 application instances against one PostgreSQL primary.**

   <details><summary>Model answer</summary>

   The core constraint is that PostgreSQL allocates real memory per connection and performs best at a concurrency near a small multiple of its core count, so 200 instances with even a modest pool each would be far beyond that. I would put PgBouncer in transaction mode between them, letting each instance hold a small client pool while the pooler multiplexes onto a few dozen server connections sized to the database's efficient concurrency. I would then verify that the application does not rely on session-scoped features that transaction pooling breaks — session variables, advisory locks, some prepared statement forms — and adjust where it does. Separate pooler endpoints would isolate transactional traffic from reporting, and I would alert on both pooler queue depth and server-side active query count, since those are the two places saturation shows up first.

   </details>

6. **[Staff] You move to an async driver that multiplexes many queries per connection. What changes?**

   <details><summary>Model answer</summary>

   The accidental concurrency limit disappears. Previously the pool size capped how many queries could be outstanding; with multiplexing, one connection can carry hundreds, so the application will happily send far more concurrent work than the database can efficiently handle — and the first sign will be a latency cliff rather than clean queueing. I would add an explicit concurrency limiter in front of the driver, sized from the same Little's law calculation, with a short acquisition timeout so overload produces fast rejections. Removing the bottleneck also removes the backpressure, so it has to be reintroduced deliberately.

   </details>

7. **[Principal] How do you prevent connection-limit incidents across a large organisation?**

   <details><summary>Model answer</summary>

   By making the database's connection budget a governed, visible resource rather than something each team discovers by exhausting it. I would publish a per-database connection budget, require services to declare their pool size and expected instance count, and enforce the total in a platform check so scaling out a deployment cannot silently overrun the database. The service template ships with a small pool, short acquisition timeout and bounded lifetime as defaults, plus pool utilisation and acquisition wait exported as standard metrics with default alerts, because those rise before latency does. And I would put an external pooler in front of shared databases as the default topology, so the number of application instances is decoupled from the number of server connections — which is the structural fix rather than a per-team discipline.

   </details>

---

### HTTP/2 multiplexing

*Many concurrent requests share one connection with binary framing and header compression — which removes HTTP head-of-line blocking but not TCP's.*

**Flow:** `HTTP requests` → `Stream frames` → `TCP connection` → `Demultiplexer` → `Handlers`

> **The 30-second version**  
> HTTP/2 interleaves many streams over one connection with binary frames and header compression. It removes HTTP-layer head-of-line blocking but inherits TCP's, and it breaks connection-level load balancing.

**The problem**

HTTP/1.1 allows one outstanding request per connection. Browsers worked around this by opening six connections per host, and engineers worked around it by sharding assets across domains, inlining resources, and concatenating files — an entire generation of “performance best practices” that existed purely to defeat a protocol limitation.

Worse, each of those connections paid its own handshake and its own slow start, and each sent the same verbose headers — cookies, user agent, accept headers — on every single request.

> **What HTTP/2 actually changed**  
> **Multiplexing**: many concurrent streams on one connection, so request count stops being the constraint. **Binary framing**: messages become length-prefixed frames instead of text, so interleaving is possible. **HPACK header compression**: repeated headers cost a few bytes instead of a kilobyte each. **Server push** (now largely deprecated in favour of early hints).

But one thing it did not change: HTTP/2 still runs over TCP, and TCP still requires in-order delivery. A single lost packet stalls every stream sharing that connection — which is the exact problem HTTP/2 was supposed to eliminate, relocated one layer down.

**Mental model**

Think of HTTP/1.1 as a single-lane road where each vehicle must clear before the next may enter, and HTTP/2 as a multi-lane road built on the same single bridge. More lanes help enormously — until the bridge is blocked, at which point every lane stops together.

1. **Stream** — An independent bidirectional sequence of frames with its own identifier. Requests and responses are streams.
2. **Frame** — The unit on the wire: HEADERS, DATA, SETTINGS, WINDOW_UPDATE, RST_STREAM. Frames from different streams interleave freely.
3. **HPACK** — A shared, connection-scoped compression table. Repeated header values are sent as small indexes.
4. **Flow control** — Per-stream and per-connection windows, independent of TCP's own window.
5. **Concurrency limit** — `SETTINGS_MAX_CONCURRENT_STREAMS` caps how many streams may be open at once — typically 100–250.

> **The layered head-of-line problem**  
> HTTP/1.1 blocks at the *request* layer: one response must finish before the next begins. HTTP/2 removes that but inherits TCP's blocking at the *byte-stream* layer: a lost packet withholds all subsequent bytes until retransmission, stalling every multiplexed stream. On a clean network this rarely matters; on a lossy mobile path it can make HTTP/2 slower than several HTTP/1.1 connections.

**How it works**

**Framing and interleaving**

```text
HTTP/1.1, one connection:
  [--- request A ---][--- response A ---][--- request B ---]...
  strictly serial

HTTP/2, one connection:
  HEADERS(s1) HEADERS(s3) DATA(s1) DATA(s3) DATA(s1) HEADERS(s5)
  streams 1, 3 and 5 progress simultaneously

FRAME TYPES that matter
  HEADERS        request/response metadata (HPACK-compressed)
  DATA           body bytes
  SETTINGS       negotiated limits (max streams, window size)
  WINDOW_UPDATE  flow control credit
  RST_STREAM     cancel ONE stream without closing the connection
  GOAWAY         graceful connection shutdown
```

1. **Stop optimising for HTTP/1.1** — Domain sharding, spriting and file concatenation actively hurt under HTTP/2: sharding defeats multiplexing and HPACK by forcing multiple connections, and bundling defeats fine-grained caching.
2. **Cancellation becomes cheap** — `RST_STREAM` cancels one request without tearing down the connection. This is what makes gRPC deadlines and client cancellation actually free resources.
3. **Flow control is yours to tune** — The default 65 KB per-stream window is small for high-bandwidth-delay paths; servers and clients should raise it for bulk transfer, or throughput will be window-limited just as in TCP.
4. **Watch the concurrency limit** — If a server advertises 100 max concurrent streams and a client wants 500, the excess queues invisibly at the client — which looks like server slowness but is client-side queueing.
5. **Load balance per request, not per connection** — Because one connection carries all traffic from a client, an L4 balancer pins everything to one backend. This is the single most common HTTP/2 operational mistake.

**Why HTTP/2 can be slower on a lossy path**

```text
6 HTTP/1.1 connections, 1% packet loss:
  a lost packet stalls ONE connection
  the other 5 continue -> 1/6 of work affected

1 HTTP/2 connection, 1% packet loss:
  a lost packet stalls the TCP stream
  ALL multiplexed streams wait for retransmission
  -> 100% of work affected for ~1 RTT

Consequence: on high-loss mobile networks, HTTP/2's single
connection can underperform HTTP/1.1's parallel ones.
QUIC fixes this by giving each stream independent ordering.
```

> **Priorities largely did not work**  
> HTTP/2's original priority scheme (dependency trees and weights) was complex, inconsistently implemented, and frequently ignored or overridden by servers. Do not design around it. HTTP/3 and the Extensible Prioritization scheme replaced it with a much simpler urgency/incremental model.

**Worked example**

A gRPC service where each client opens one HTTP/2 channel. What goes wrong, and what to configure.

**gRPC over HTTP/2: the operational reality**

```text
SETUP
  50 clients, each ONE HTTP/2 connection
  20 backend pods behind an L4 load balancer

PROBLEM 1  load skew
  L4 assigns 50 connections -> ~2-3 per pod
  but a heavy client sends 10,000 rps on its single connection
  -> that pod is saturated, others idle
  FIX: L7 (per-request) balancing, or client-side balancing
       with multiple subchannels

PROBLEM 2  invisible client-side queueing
  server advertises MAX_CONCURRENT_STREAMS = 100
  client issues 400 concurrent calls
  -> 300 queue in the client, latency rises,
     server metrics look healthy
  FIX: raise the limit, or open multiple connections,
       or add an explicit client concurrency limit

PROBLEM 3  connections never rebalance
  a long-lived connection survives pod scale-out forever
  FIX: server sends GOAWAY periodically (max connection age),
       forcing clients to reconnect and rebalance
```

| Metric | Value | Note |
|---|---|---|
| Max streams | 100–250 | typical default |
| Stream window | 64 KB | **raise for bulk** |
| Max conn age | 30–60 min | forces rebalance |
| L4 balancing | avoid | severe skew |

> **The rebalancing trick worth remembering**  
> Setting a maximum connection age on the server, so it periodically sends `GOAWAY`, is how long-lived HTTP/2 clients ever notice that new backends exist. Without it, a scale-out event adds capacity that receives no traffic, because every existing connection stays where it is. Add jitter so all clients do not reconnect simultaneously.

**When to use it**

- **Any modern HTTP API or web application**, where it is effectively free and strictly better than HTTP/1.1 on a clean network.
- **gRPC**, which requires HTTP/2 for streaming, cancellation and bidirectional flow.
- **Many small resources**, where multiplexing and HPACK produce the largest wins.
- **Long-lived client sessions** where one connection amortises handshake and slow start.

**When to avoid it**

- **Do not multiplex latency-sensitive traffic over one connection on a lossy path** — use HTTP/3, or separate connections.
- **Do not keep HTTP/1.1-era optimisations**: domain sharding, spriting and aggressive bundling become counterproductive.
- **Do not use L4 load balancing** in front of HTTP/2 — it pins all of a client's requests to one backend.
- **Do not rely on HTTP/2 priorities** to control ordering; implementations vary too much.
- **Do not leave the default flow-control window** on high-bandwidth-delay paths for bulk transfer.

**Advantages**

- **Concurrency without connection count**, removing the six-connection browser limit and the workarounds it inspired.
- **HPACK eliminates repeated header overhead**, which is substantial for cookie-heavy APIs with small payloads.
- **One connection means one handshake and one slow start**, amortised across all requests.
- **Per-stream cancellation** frees server resources immediately, which is what makes deadlines meaningful.
- **Binary framing** is unambiguous and cheaper to parse than text.

**Disadvantages**

- **TCP head-of-line blocking** couples all streams; one lost packet stalls everything.
- **A single connection is a single point of routing**, causing load skew with L4 balancers and preventing rebalancing.
- **Priorities are unreliable** across implementations.
- **HPACK is stateful**, so proxies must maintain per-connection compression context — a memory and complexity cost.
- **Debugging is harder** than plain-text HTTP/1.1, requiring protocol-aware tooling.

**Trade-offs**

**Protocol comparison**

|  | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Concurrency | ~6 connections | Many streams, 1 connection | Many streams, 1 connection |
| Header overhead | Full headers per request | HPACK compressed | QPACK compressed |
| HOL blocking | At request layer | At TCP layer | None (per-stream ordering) |
| Setup cost | Per connection | Once | 0–1 RTT, connection migration |
| Under packet loss | Degrades one connection | Degrades all streams | Degrades one stream |
| Operational tooling | Excellent | Good | Improving |

The practical rule: HTTP/2 is a clear win over HTTP/1.1 for internal and wired traffic. HTTP/3's advantage appears specifically where loss is non-trivial — mobile, international, congested last miles — which is exactly where consumer traffic lives.

**How it fails**

**HTTP/2 operational failures**

| Symptom | Cause | Fix |
|---|---|---|
| Severe backend load skew | L4 balancing pins a connection to one backend | L7 or client-side per-request balancing |
| New pods receive no traffic | Long-lived connections never rebalance | Max connection age with jittered `GOAWAY` |
| High client latency, healthy server metrics | Client queueing behind `MAX_CONCURRENT_STREAMS` | Raise the limit or add explicit client concurrency control |
| Throughput below link capacity | Default 64 KB stream window on a high-BDP path | Increase stream and connection windows |
| All requests stall briefly together | TCP head-of-line blocking from a lost packet | HTTP/3, or separate connections for critical paths |
| Proxy memory growth | Per-connection HPACK tables across many connections | Bound table size; limit concurrent connections |
| Intermittent stream resets | Concurrency limit exceeded or idle timeouts | Tune limits; keepalive pings |

**Limits**

> **Defaults worth knowing**
>
> - **`SETTINGS_MAX_CONCURRENT_STREAMS`** commonly 100–250; excess requests queue at the client.
> - **Initial flow-control window** 65,535 bytes per stream — small for bulk transfer on long paths.
> - **HPACK dynamic table** default 4 KB per direction per connection.
> - **Max connection age** of 30–60 minutes with jitter is the standard way to force rebalancing.
> - **Practical gain over HTTP/1.1** is largest with many small resources and high header overhead; negligible for a single large download.

**Alternatives**

| Option | Best for | Cost |
|---|---|---|
| HTTP/1.1 with keep-alive | Simple APIs, maximum tooling compatibility | Limited concurrency, header overhead |
| HTTP/2 | Internal services, gRPC, wired clients | TCP HOL blocking; routing skew |
| HTTP/3 / QUIC | Mobile, lossy, international clients | Userspace CPU; UDP blocked on some networks |
| WebSocket | Bidirectional push with arbitrary framing | No HTTP semantics; separate infrastructure |
| Multiple HTTP/2 connections | Isolating critical from bulk traffic | Loses some HPACK and connection efficiency |

**In real systems**

- **gRPC** is built directly on HTTP/2, relying on streams for bidirectional streaming and on `RST_STREAM` for deadline-driven cancellation.
- **Browsers** dropped domain sharding guidance once HTTP/2 became common, because sharding actively defeats multiplexing.
- **Envoy and other L7 proxies** exist partly to provide per-request balancing for HTTP/2, which L4 balancers cannot do.
- **Server push was effectively abandoned** and removed from Chrome, replaced by `103 Early Hints`, which achieves the intent without the cache-correctness problems.
- **CDNs** widely offer HTTP/3 precisely because their traffic is dominated by lossy last-mile paths where TCP head-of-line blocking is costly.

**Common mistakes**

- **L4 load balancing in front of gRPC**, producing severe skew.
- **No maximum connection age**, so new capacity never receives traffic.
- **Keeping domain sharding and sprites** after migrating.
- **Leaving default flow-control windows** for bulk transfer.
- **Designing around HTTP/2 priorities**, which are unreliable.
- **Blaming the server** for latency caused by client-side stream queueing.
- **Assuming HTTP/2 fixes head-of-line blocking** — it only moves it to TCP.

**The staff-level view**

Most HTTP/2 value is captured by default; most HTTP/2 *problems* come from operating it as though it were HTTP/1.1.

- **Mandate L7 or client-side balancing for HTTP/2 and gRPC traffic.** This is the difference between even load and one saturated pod.
- **Set a jittered maximum connection age fleet-wide**, or autoscaling silently fails to receive traffic.
- **Remove legacy HTTP/1.1 optimisations** during migration — sharding and heavy bundling become net negatives.
- **Expose client-side stream queueing as a metric.** Latency that originates in the client's own queue is invisible in server dashboards and causes long, wrong investigations.
- **Choose HTTP/3 by audience, not fashion.** For mobile and international users the loss characteristics justify it; for internal wired traffic HTTP/2 is fine.

**Go deeper**

HTTP/2 splits messages into binary frames tagged with a stream id, so many requests progress concurrently over a single connection. HPACK compresses repeated headers to a few bytes, one handshake and one slow start serve all traffic, and `RST_STREAM` cancels a single request without closing the connection — which is what makes gRPC deadlines free resources promptly.

Two limitations dominate operations. First, streams still share a TCP connection, and TCP delivers in order, so one lost packet stalls every stream — on lossy mobile paths a single HTTP/2 connection can underperform six HTTP/1.1 ones, which is precisely why QUIC exists. Second, one long-lived connection means an L4 balancer makes exactly one routing decision, pinning all of a client's requests to one backend and skewing load badly; per-request L7 or client-side balancing is required, plus a jittered maximum connection age so new backends ever receive traffic.

Two configuration details catch teams out: the negotiated maximum concurrent streams (often 100) causes invisible client-side queueing when exceeded, and the default 64 KB per-stream flow-control window limits bulk throughput on long paths. Also delete HTTP/1.1-era workarounds — domain sharding and heavy bundling actively hurt under HTTP/2.

HTTP/2 replaced a text protocol with one outstanding request per connection by a binary framing layer that interleaves many independent streams. The gains are real and mostly automatic; the surprises are operational.

**What it provides.** Streams are independent bidirectional frame sequences identified by number, so requests and responses interleave freely on one connection. HPACK maintains a connection-scoped compression table, turning repeated headers — cookies, user agent, accept — from kilobytes into small indexes, which is a large win for chatty APIs with small bodies. One connection means one handshake and one slow start amortised over everything. `RST_STREAM` cancels a single request without tearing down the connection, which is what makes deadlines and client cancellation actually release server resources.

**What it does not fix.** Streams share a TCP connection, and TCP guarantees in-order byte delivery. A lost packet withholds every subsequent byte until retransmission, so all multiplexed streams stall together for roughly a round trip. Six HTTP/1.1 connections lose one-sixth of their progress to the same event. On low-loss wired networks this is negligible; on lossy mobile or international paths it can make HTTP/2 slower than what it replaced. QUIC's central design decision — moving reliability above the transport so each stream has independent ordering — exists to solve exactly this.

**The load balancing consequence.** With one long-lived connection carrying all of a client's traffic, an L4 balancer makes a single routing decision at connection setup. A heavy client's entire workload lands on one backend permanently, producing severe skew that no health signal corrects. This is a correctness problem for load distribution, not a tuning issue, and it is why internal gRPC traffic uses L7 proxies, sidecars, or client-side subchannel balancing. A related and equally common failure: long-lived connections never discover new backends, so scaling out adds capacity that receives no traffic. The remedy is a server-side maximum connection age with jitter, sending `GOAWAY` so clients reconnect and redistribute.

**Two settings that cause misdiagnosis.** `SETTINGS_MAX_CONCURRENT_STREAMS`, typically 100 to 250, caps open streams; calls beyond it queue inside the client, where no server metric can see them — producing high client latency alongside a healthy-looking server. And the default per-stream flow-control window of 65,535 bytes is small for bulk transfer on high-bandwidth-delay paths, capping throughput the same way an untuned TCP window does. Exposing client-side outstanding-stream count as a standard metric prevents long, wrong investigations.

**Migration hygiene.** Domain sharding, sprite sheets and aggressive bundling all existed to work around HTTP/1.1's connection limit and are net negatives under HTTP/2: sharding forces multiple connections, defeating multiplexing and HPACK, while bundling defeats fine-grained caching. Server push was a third idea that did not survive contact with cache correctness and has been superseded by `103 Early Hints`. And HTTP/2's priority scheme was implemented inconsistently enough that designing around it is unwise; the simpler urgency-based scheme in HTTP/3 replaced it.

**Prove it — interview questions**

1. **[Basic] What does HTTP/2 multiplexing do?**

   <details><summary>Model answer</summary>

   It lets many requests and responses proceed concurrently over one TCP connection by splitting messages into binary frames tagged with a stream identifier, which are then interleaved on the wire. That removes HTTP/1.1's limit of one outstanding request per connection, so browsers no longer need six connections per host and applications no longer need domain sharding or aggressive file bundling.

   </details>

2. **[Basic] Does HTTP/2 eliminate head-of-line blocking?**

   <details><summary>Model answer</summary>

   Only at the HTTP layer. Streams no longer wait for each other in the protocol, but they still share a TCP connection, and TCP delivers bytes strictly in order. A single lost packet withholds all subsequent bytes until it is retransmitted, which stalls every multiplexed stream simultaneously. On a clean network that is rare; on a lossy mobile path, one HTTP/2 connection can underperform several HTTP/1.1 connections.

   </details>

3. **[Senior] Why is L4 load balancing a problem for HTTP/2?**

   <details><summary>Model answer</summary>

   Because an L4 balancer decides once per connection, and an HTTP/2 client typically opens one long-lived connection that then carries all of its requests. Every request from that client lands on the same backend for the connection's lifetime, so a heavy client saturates one pod while others sit idle, and the balancer cannot move traffic even if that pod degrades. The fixes are per-request L7 balancing, client-side balancing across multiple subchannels, or both — plus a maximum connection age so connections are periodically re-established and rebalanced.

   </details>

4. **[Senior] A gRPC client shows high latency but the server looks healthy. What do you check?**

   <details><summary>Model answer</summary>

   Client-side stream queueing first. The server advertises a maximum concurrent stream count, commonly around 100, and calls beyond that queue inside the client where no server metric can see them. I would check the client's outstanding-call count against the negotiated limit. Other candidates are flow-control windows too small for the payload size, causing throughput limits that look like latency, and connection-level contention where a single connection's TCP behaviour is the bottleneck. The general lesson is that with multiplexed protocols, a meaningful amount of latency can originate inside the client.

   </details>

5. **[Staff] When would you move a service from HTTP/2 to HTTP/3?**

   <details><summary>Model answer</summary>

   When the client population sits on lossy or variable networks — mobile, international, congested last miles — because that is exactly where TCP head-of-line blocking hurts and where QUIC's per-stream ordering pays off. Connection migration is a second real benefit for mobile clients that change networks mid-session, and 0-RTT resumption helps short sessions. For internal, wired, low-loss data-centre traffic I would stay on HTTP/2, because the loss characteristics that motivate QUIC are absent while its costs — higher userspace CPU per byte and less mature debugging tooling — are present. I would also require a TCP fallback regardless, since some networks block or throttle UDP.

   </details>

6. **[Principal] How do you roll HTTP/2 out across an organisation without operational surprises?**

   <details><summary>Model answer</summary>

   The protocol change is easy; the operational assumptions are what break. I would sequence it as: first ensure every HTTP/2 path is behind L7 or client-side per-request balancing, because connection-level balancing is the failure that produces the most confusing symptoms. Second, set a jittered maximum connection age as a platform default, so autoscaling actually receives traffic — without it, new capacity is invisible to existing clients and the scaling system appears broken. Third, add client-side outstanding-stream count to the standard metric set, so latency originating in client queues is diagnosable rather than mysterious. Fourth, run a cleanup pass removing HTTP/1.1-era workarounds, which are now net negatives. Each of those is a small change, and each of them is something a team discovers painfully if the platform does not provide it.

   </details>

---

### QUIC and HTTP/3

*Reliability and encryption move above UDP, giving each stream independent ordering, 0–1 RTT setup, and connections that survive network changes.*

**Flow:** `HTTP messages` → `QUIC streams` → `UDP packets` → `Path migration` → `Server`

> **The 30-second version**  
> QUIC rebuilds reliability, ordering and TLS above UDP in userspace: independent per-stream ordering, 1-RTT or 0-RTT setup, and connections that survive network changes.

**The problem**

HTTP/2 solved concurrency at the HTTP layer but left the real bottleneck in place: TCP delivers bytes strictly in order, so one lost packet stalls every multiplexed stream. On a wired data-centre link that is a rounding error. On a mobile network with 1–2% loss, it means all your parallel work stops together, repeatedly.

TCP also cannot be fixed. It lives in operating system kernels and in middleboxes — NATs, firewalls, accelerators — that inspect and rewrite headers. Any change requires simultaneous deployment across devices nobody controls. TCP is **ossified**: improvements that are well understood cannot ship.

> **QUIC's core design decision**  
> Build the transport in **userspace on top of UDP**, and encrypt almost everything including the transport headers. That gives independent per-stream ordering, removes handshake round trips by integrating TLS, allows connections to survive IP changes, and — crucially — makes the protocol upgradable, because it ships with the application rather than with the kernel.

**Mental model**

QUIC is “TCP + TLS + HTTP/2's stream layer, rebuilt as one protocol in userspace.” HTTP/3 is simply HTTP mapped onto QUIC streams.

1. **UDP as a substrate** — UDP provides only addressing and ports. QUIC implements everything TCP did — reliability, ordering, congestion control, flow control — on top.
2. **Independent streams** — Each stream has its own sequence space, so a packet loss affecting stream 3 does not withhold stream 5's data.
3. **Integrated TLS 1.3** — The cryptographic and transport handshakes are one exchange: 1 RTT for a new connection, 0 RTT on resumption.
4. **Connection IDs** — A connection is identified by an opaque ID, not by the 4-tuple, so it survives a change of IP address or port.
5. **Encrypted transport headers** — Packet numbers and most metadata are encrypted, which prevents middlebox interference — and therefore prevents ossification.

> **What QUIC does not fix**  
> Congestion control still exists, and a shared bottleneck link still limits you. QUIC does not make the network faster; it removes *artificial* coupling between streams and *artificial* round trips. On a clean, low-latency, low-loss path, HTTP/2 and HTTP/3 perform nearly identically.

**How it works**

**Handshake and stream independence**

```text
CONNECTION SETUP
  TCP + TLS1.3   : 1 RTT (TCP) + 1 RTT (TLS)   = 2 RTT
  QUIC           : 1 RTT combined              = 1 RTT
  QUIC resumed   : 0 RTT (data with first flight)

LOSS BEHAVIOUR
  TCP:   [s1][s2][X][s2][s3]  -> s1, s2, s3 ALL wait
  QUIC:  [s1][s2][X][s2][s3]  -> only s2 waits;
                                 s1 and s3 deliver immediately

CONNECTION MIGRATION
  client on WiFi   : conn_id = 0xAB, 4-tuple A
  switches to LTE  : conn_id = 0xAB, 4-tuple B
  -> server recognises the connection ID, validates the new
     path, and continues. No reconnect, no TLS handshake,
     no lost application state.
```

1. **0-RTT has a replay caveat** — Data sent in the first flight on resumption can be replayed by an attacker. Only idempotent requests are safe; servers should reject non-idempotent operations at 0-RTT.
2. **Expect higher CPU per byte** — Userspace processing, per-packet encryption and lack of mature offload mean QUIC typically costs more CPU than kernel TCP. Hardware and kernel offload are improving this.
3. **Always keep a TCP fallback** — Some enterprise and mobile networks block or heavily throttle UDP. HTTP/3 is advertised via `Alt-Svc` (or DNS HTTPS records) and clients fall back automatically — but only if you still serve HTTP/2.
4. **Load balancing must be connection-ID aware** — A 4-tuple hash breaks the moment a client migrates. Balancers must route on the connection ID, which is why QUIC-aware load balancing is a distinct capability.
5. **Observability changes** — Encrypted transport headers mean packet captures reveal far less. Debugging shifts to endpoint logging (`qlog`) and application telemetry.

**When HTTP/3 actually wins**

```text
PATH CHARACTERISTICS         HTTP/2 vs HTTP/3
------------------------     --------------------------------
data centre, <1ms, 0% loss   identical; HTTP/2 cheaper on CPU
wired broadband, 0.1% loss   HTTP/3 slightly better
mobile 4G, 1% loss           HTTP/3 clearly better
congested mobile, 3% loss    HTTP/3 dramatically better
changing networks            HTTP/3 only (migration)
short sessions, repeat visit HTTP/3 (0-RTT saves a full RTT)

RULE: the benefit scales with (loss rate x number of streams)
      and with (RTT x number of connections established).
```

> **Advertise, do not force**  
> Serve HTTP/2 over TCP and advertise HTTP/3 via an `Alt-Svc` header or a DNS HTTPS record. Clients that can use QUIC will upgrade on a subsequent connection; clients behind UDP-blocking networks keep working. This makes adoption risk-free and incremental.

**Worked example**

A media app serving mobile users internationally. What does moving to HTTP/3 change?

**Measured effect on a lossy mobile path**

```text
BASELINE (HTTP/2 over TCP)
  RTT 120 ms, loss 1.5%, page needs 40 resources

  connection setup      2 RTT          = 240 ms
  HOL stalls: with 1.5% loss over ~300 packets,
    ~4-5 loss events, each stalling ALL streams ~1 RTT
                                       = ~500-600 ms
  total overhead                       ~ 800 ms

HTTP/3
  connection setup      1 RTT          = 120 ms
    (0 RTT on repeat visit             = 0 ms)
  loss events stall only their own stream
    other 39 resources continue        = ~0 ms added
  total overhead                       ~ 120 ms

NETWORK SWITCH (WiFi -> cellular mid-session)
  HTTP/2: connection dies, full reconnect + TLS + slow start
  HTTP/3: connection ID survives; playback does not stall
```

| Metric | Value | Note |
|---|---|---|
| Setup saving | 1 RTT | 2 on resumption |
| HOL stalls | eliminated | **biggest win** |
| Network switch | seamless | migration |
| CPU cost | higher | userspace crypto |

> **The benefit is proportional to loss × parallelism**  
> This is the calculation that decides whether HTTP/3 is worth it for a given audience. A service with few resources per page and a wired user base gains almost nothing. A service with dozens of parallel fetches and a mobile user base gains a large fraction of its perceived load time. Measure your audience's loss rate before deciding.

**When to use it**

- **Mobile and international clients**, where loss rates make TCP head-of-line blocking expensive.
- **Highly parallel page loads**, where the cost of a stall is multiplied across many streams.
- **Sessions that survive network changes** — mobile apps, streaming, long-lived realtime connections.
- **Short repeat sessions**, where 0-RTT resumption removes a full round trip from the critical path.
- **Anywhere you already use a CDN**, since enabling HTTP/3 there is typically a configuration flag.

**When to avoid it**

- **Do not use it for internal data-centre traffic** where loss is near zero — you pay CPU for a benefit that does not exist.
- **Do not drop TCP support.** UDP is blocked or throttled on enough networks that a fallback is mandatory.
- **Do not enable 0-RTT for non-idempotent requests** — the first flight is replayable.
- **Do not assume existing L4 load balancers work.** They hash on the 4-tuple, which migration invalidates.
- **Do not expect packet captures to help you debug** — transport headers are encrypted; instrument endpoints instead.

**Advantages**

- **Per-stream independent ordering**, eliminating the coupling that limits HTTP/2 on lossy paths.
- **Faster establishment**: 1 RTT for new connections, 0 RTT on resumption.
- **Connection migration** across IP and network changes, which matters enormously for mobile.
- **Upgradable protocol**, because it ships in userspace with the application rather than in kernels and middleboxes.
- **Encryption by default**, including most transport metadata, which also prevents middlebox tampering.

**Disadvantages**

- **Higher CPU per byte** than kernel TCP, though the gap is narrowing with offload support.
- **UDP is blocked or deprioritised** on some enterprise and mobile networks, requiring fallback.
- **Reduced network-level observability**, since transport headers are encrypted.
- **Load balancers and DDoS appliances need QUIC awareness**, which is not universal.
- **0-RTT replay risk** requires careful server-side handling.
- **Tooling maturity** still trails TCP by a wide margin for debugging and packet analysis.

**Trade-offs**

**HTTP/2 versus HTTP/3 by dimension**

| Dimension | HTTP/2 (TCP) | HTTP/3 (QUIC) |
|---|---|---|
| Stream independence | No — TCP couples them | Yes |
| Connection setup | 2 RTT (with TLS) | 1 RTT, 0 on resumption |
| Network change | Connection dies | Migrates via connection ID |
| CPU per byte | Lower (kernel, offloaded) | Higher (userspace crypto) |
| Middlebox traversal | Universal | UDP sometimes blocked |
| Network debuggability | Good (tcpdump) | Poor (encrypted); use qlog |
| Protocol evolution | Ossified | Ships with the application |

**How it fails**

**HTTP/3 deployment failures**

| Symptom | Cause | Fix |
|---|---|---|
| Some clients never use HTTP/3 | UDP blocked on their network | Expected — ensure TCP fallback works and is monitored |
| Connections break when users change networks | Load balancer hashing on the 4-tuple | Connection-ID-aware routing |
| Higher server CPU after enabling | Userspace crypto without offload | Enable kernel/NIC offload; size capacity accordingly |
| Duplicate side effects on repeat visits | 0-RTT replay of a non-idempotent request | Reject non-idempotent operations at 0-RTT |
| No visibility into a transport problem | Encrypted headers defeat packet capture | Enable qlog; instrument at endpoints |
| Amplification abuse reports | Server responding excessively to spoofed handshakes | Enforce the anti-amplification limit before path validation |
| Intermittent failures behind a firewall | UDP rate limiting rather than blocking | Detect and pin those clients to TCP |

**Limits**

> **Numbers and constraints**
>
> - **Setup**: 1 RTT new, 0 RTT resumed (idempotent requests only).
> - **Anti-amplification**: a server may send at most ~3× the bytes received before validating the client's path.
> - **CPU**: historically several times TCP's per-byte cost; modern implementations with offload narrow this considerably.
> - **Benefit scales with loss × stream count**; near zero on a clean path.
> - **Advertisement**: `Alt-Svc` header or DNS HTTPS record; adoption is client-driven and gradual.

**Alternatives**

| Option | Best for | Trade |
|---|---|---|
| HTTP/2 over TCP | Internal, wired, low-loss traffic | HOL blocking on lossy paths |
| HTTP/3 / QUIC | Mobile, international, parallel-heavy | CPU, UDP blocking, tooling |
| Both, with Alt-Svc | Public services — the practical default | Two stacks to operate |
| WebTransport | Bidirectional, unreliable-tolerant app streams | Newer, narrower support |
| Reduce RTT instead (CDN/edge) | Any audience | Often a larger win than protocol choice |

The last row deserves weight: moving content closer reduces RTT, which improves handshake cost, slow start and loss-recovery time simultaneously. A CDN is usually a bigger improvement than a protocol change, and the two compose.

**In real systems**

- **Google** developed QUIC and deployed it across Search and YouTube before standardisation, reporting notable latency improvements on high-loss paths.
- **Cloudflare, Fastly and other CDNs** enable HTTP/3 with a configuration toggle, which is how most sites adopt it.
- **Chrome, Firefox and Safari** all support HTTP/3 and upgrade automatically when advertised, with silent TCP fallback.
- **Mobile applications** benefit most from connection migration, since network handoffs would otherwise break long-lived sessions.
- **Enterprise networks** remain the main obstacle, with UDP restrictions common enough that fallback is a permanent requirement.

**Common mistakes**

- **Enabling HTTP/3 for internal traffic** and paying CPU for no benefit.
- **Removing TCP support**, breaking clients on UDP-restricted networks.
- **Allowing 0-RTT for non-idempotent requests**, creating replay exposure.
- **Keeping 4-tuple load balancer hashing**, which breaks connection migration.
- **Expecting packet captures to debug it**, then having no telemetry at all.
- **Deploying without capacity headroom** for the higher per-byte CPU cost.
- **Treating it as a universal upgrade** rather than an audience-specific optimisation.

**The staff-level view**

HTTP/3 is a genuine improvement with a narrow qualifying condition. The Staff skill is deciding whether your audience meets it.

- **Measure your users' loss rate and RTT distribution before deciding.** The benefit is proportional to loss × parallelism; without those numbers it is a fashion decision.
- **Adopt via the CDN first.** It is a configuration change with automatic fallback, and it covers the user population that benefits most.
- **Keep TCP forever.** Treat HTTP/3 as an optimisation layer, never as a replacement, because UDP restrictions are not going away.
- **Plan for reduced network observability.** Enable endpoint-level transport logging before you need it, because tcpdump will not help.
- **Recognise the strategic point.** Userspace transport means congestion control and loss recovery can now evolve on your deployment cadence rather than the kernel's — that, more than today's latency numbers, is the long-term value.

**Go deeper**

HTTP/2's remaining bottleneck is TCP's in-order delivery: one lost packet stalls every multiplexed stream. QUIC moves reliability and ordering above UDP, giving each stream its own sequence space so a loss delays only its own stream. It also integrates TLS 1.3 into the transport handshake — one round trip for a new connection, zero on resumption — and identifies connections by an opaque ID, so a client switching from WiFi to cellular keeps the same connection rather than re-establishing everything.

The benefit scales with loss rate multiplied by stream count, and with RTT multiplied by connection setups. On a clean data-centre path HTTP/2 and HTTP/3 perform nearly identically and HTTP/2 costs less CPU. On a 1–2% loss mobile path with dozens of parallel fetches, HTTP/3 removes a large fraction of perceived load time.

Costs and caveats: higher CPU per byte because crypto runs in userspace; UDP is blocked or throttled on some enterprise networks so a TCP fallback advertised via `Alt-Svc` is mandatory; 0-RTT data is replayable so only idempotent requests may use it; load balancers must route on connection ID rather than the 4-tuple or migration breaks; and encrypted transport headers mean debugging shifts from packet capture to endpoint logging.

QUIC is the transport layer rebuilt in userspace on UDP, and HTTP/3 is HTTP mapped onto it. Understanding why it exists explains more than the feature list does.

**Two motivations.** The technical one is TCP head-of-line blocking: HTTP/2 multiplexes streams, but TCP delivers bytes in strict order, so one lost packet withholds all subsequent data and stalls every stream for a round trip. The structural one is ossification: TCP lives in kernels and in middleboxes that inspect and rewrite its headers, so well-understood improvements to congestion control and loss recovery cannot be deployed. Building on UDP and encrypting the transport headers solves both — streams get independent sequence spaces, and middleboxes cannot come to depend on a format they cannot read.

**What you get.** Per-stream ordering, so loss affects one stream rather than all. A combined transport-and-TLS handshake costing one round trip, or zero on resumption. Connection identity by opaque connection ID rather than by address-port tuple, so a session survives a WiFi-to-cellular handoff, a NAT rebinding, or an IP change without re-establishing the connection, the TLS session, or congestion state. And a protocol that ships with the application, so it can evolve at release cadence.

**Where the benefit is real.** It scales roughly as loss rate × number of concurrent streams, plus RTT × number of connections established. On a sub-millisecond, zero-loss data-centre link, HTTP/2 and HTTP/3 are indistinguishable and HTTP/2 costs less CPU. On a 120 ms mobile path with 1.5% loss fetching forty resources, TCP's coupled stalls can add most of a second, while QUIC's independent streams add essentially nothing — and the saved handshake round trip compounds it. The decision should be made from a measurement of the real client population's loss and RTT distribution, not from a preference.

**What it costs.** Higher CPU per byte, because packet processing and encryption run in userspace without the maturity of kernel and NIC offload — though this gap is closing. UDP blocking or throttling on some enterprise and mobile networks, which makes a permanent TCP fallback mandatory; advertisement via `Alt-Svc` or a DNS HTTPS record keeps adoption client-driven and safe. Reduced network-level observability, since encrypted transport headers make packet captures nearly useless and push debugging to endpoint logging such as qlog. Load balancers and DDoS appliances that hash on the 4-tuple break connection migration and must be made connection-ID aware. And 0-RTT data is replayable by design, so it must be restricted to idempotent requests, with servers rejecting anything state-changing until the connection is fully established.

**Practical adoption.** Enable it at the CDN, where it is a configuration change with automatic fallback and where it covers the mobile and international users who benefit most. Keep TCP indefinitely. Add transport-level endpoint telemetry before you need it. And remember that reducing RTT itself — by moving content closer — improves handshake cost, slow start and recovery time simultaneously, which is frequently a larger win than the protocol change and composes with it.

**Prove it — interview questions**

1. **[Basic] What problem does QUIC solve that HTTP/2 does not?**

   <details><summary>Model answer</summary>

   TCP head-of-line blocking. HTTP/2 multiplexes streams but runs over TCP, which delivers bytes strictly in order, so a single lost packet stalls every stream until it is retransmitted. QUIC implements reliability per stream on top of UDP, so a loss affecting one stream does not delay the others. It also merges the transport and TLS handshakes into a single round trip, and identifies connections by an opaque ID so they survive a change of IP address.

   </details>

2. **[Basic] Why is QUIC built on UDP rather than being a new transport protocol?**

   <details><summary>Model answer</summary>

   Because a genuinely new IP protocol would not traverse the internet. NATs, firewalls and middleboxes only reliably pass TCP and UDP, and deploying anything else would require updating devices nobody controls. UDP provides just addressing and ports, leaving QUIC free to implement everything else in userspace — which also means the protocol can evolve at application release cadence instead of waiting for kernel and middlebox upgrades.

   </details>

3. **[Senior] What is connection migration and why does it matter?**

   <details><summary>Model answer</summary>

   A QUIC connection is identified by an opaque connection ID rather than by the source and destination address-port tuple. When a client's network changes — WiFi to cellular, or a NAT rebinding — it keeps the same connection ID, the server validates the new path, and the session continues without a new handshake or lost state. With TCP the connection simply dies and everything must be re-established, including TLS and congestion-control state. For mobile apps and streaming this is the difference between a seamless handoff and a visible stall.

   </details>

4. **[Senior] What are the risks of 0-RTT, and how do you handle them?**

   <details><summary>Model answer</summary>

   Data sent in the first flight on a resumed connection is not protected against replay: an attacker who captures it can send it again and the server cannot distinguish the copy. That is harmless for idempotent reads and dangerous for anything that changes state. The handling is to restrict 0-RTT to safe, idempotent requests and have the server reject others, deferring them to the fully established connection one round trip later. Application-level idempotency keys give a second layer of protection where a state-changing request must go early.

   </details>

5. **[Staff] How would you decide whether to adopt HTTP/3 for a given service?**

   <details><summary>Model answer</summary>

   By measuring the audience. The benefit scales roughly with loss rate multiplied by the number of parallel streams, plus RTT multiplied by how often connections are established. So I would look at the loss and RTT distribution of real clients and the number of resources or calls per session. A service with mobile and international users fetching dozens of resources per page gains a large share of its perceived load time; an internal service on a wired data-centre network gains nothing and pays higher CPU per byte. If it qualifies, I would adopt it at the CDN first, since that is a configuration change with automatic fallback and covers exactly the clients who benefit. And I would keep TCP permanently, because UDP restrictions on enterprise networks are a durable fact rather than a transitional one.

   </details>

6. **[Principal] What is the longer-term significance of moving the transport into userspace?**

   <details><summary>Model answer</summary>

   It ends protocol ossification, which is the more important story than any current latency number. TCP's congestion control and loss recovery have known improvements that cannot ship because they require simultaneous change across kernels and middleboxes that nobody controls; QUIC encrypts its transport headers specifically so middleboxes cannot depend on their format, and it ships with the application, so a congestion-control change can roll out on a normal release cadence. Strategically that means transport behaviour becomes something an organisation can tune for its own traffic — different algorithms for bulk versus interactive, experiments run as A/B tests — rather than a fixed property of the operating systems in the path. The cost is that we now own a complex piece of systems software that used to be somebody else's problem, which is a real operational commitment and the main reason to adopt it through a CDN rather than by running it yourself.

   </details>

---

### Service discovery

*Instances register, clients look them up, and the hard part is what happens in the gap between an instance dying and the registry noticing.*

**Flow:** `Instance` → `Registration` → `Directory` → `Client cache` → `Routing`

> **The 30-second version**  
> Instances register, clients cache the list, and the design work is all in the gap between an instance failing and clients noticing — which retries and outlier ejection must cover.

**The problem**

In a dynamic environment, instances appear and disappear constantly — autoscaling, deploys, preemptions, failures. Hard-coded addresses stop working immediately. Something must answer the question “where can I reach service X right now?”

The naive answer is a registry: instances register themselves, clients query it. But that registry is now in the critical path of every call, and the interesting engineering is in the failure modes it introduces.

> **The three hard problems discovery creates**
>
> - **Staleness window**: an instance dies, but clients keep sending to it until the registry notices and clients refresh. That window is where errors live.
> - **Registry availability**: if lookup fails, can anything talk to anything? A registry outage that stops all traffic is worse than the problem it solved.
> - **Thundering herd on change**: every client refreshing simultaneously after a deploy can overwhelm the registry or the newly registered instances.

**Mental model**

Discovery is a cache-coherence problem in disguise. The registry holds authoritative membership; every client holds a stale copy; the design question is how stale, and what happens when a client acts on stale data.

1. **Registration** — Self-registration (the instance announces itself) or third-party registration (the orchestrator registers it, which is what Kubernetes does).
2. **Health** — Liveness via heartbeat or lease, plus readiness to distinguish “running” from “willing to serve”.
3. **Lookup** — Server-side (a load balancer or proxy resolves) or client-side (the caller resolves and chooses).
4. **Propagation** — How a membership change reaches clients: polling, watch/streaming, or DNS TTL expiry.
5. **Failure handling** — What a client does when the registry is unreachable — the most important design decision here.

> **Stale-but-available beats fresh-but-unavailable**  
> The correct default when the registry is unreachable is for clients to **keep using their last known membership** indefinitely, rather than failing. A stale list is usually mostly correct, and combining it with client-side retries and outlier ejection makes staleness survivable. Systems that fail closed on registry unavailability convert a control-plane outage into a data-plane outage — the single worst design error in this topic.

**How it works**

**The discovery loop**

```text
INSTANCE                REGISTRY               CLIENT
--------                --------               ------
start
readiness OK
register(addr, meta) -> [addr list]
heartbeat every 5s  -> renew lease
                          |
                          |<- watch / poll <-  cache list
                          |                    pick instance
                          |                    call directly

INSTANCE DIES (no graceful shutdown)
  heartbeat stops
  lease expires after ~15-30s   <- STALENESS WINDOW
  registry removes it
  clients notice on next watch/poll
  -> during the window, clients send to a dead address
     and must rely on RETRIES + outlier ejection

GRACEFUL SHUTDOWN
  deregister FIRST, then drain, then exit
  -> staleness window ~0 for planned changes
```

1. **Deregister before draining** — The order matters. Deregistering first means the registry stops advertising you while you finish in-flight work. Shutting down first guarantees a window of errors.
2. **Separate liveness from readiness** — An instance that is starting up, warming caches, or shedding load is alive but not ready. Only readiness should control registry membership.
3. **Make clients tolerant, not dependent** — Cache the membership list, keep serving from it if the registry is unreachable, and pair it with retries to another instance plus passive outlier ejection.
4. **Prefer watch/streaming over polling** — Polling either wastes requests or propagates slowly. A streaming watch gives sub-second propagation with low overhead — this is what xDS does in service meshes.
5. **Bound the staleness explicitly** — Lease TTL plus propagation delay is your error window for unplanned failures. Write it down; it determines how aggressive retries must be.
6. **Avoid registry-side health checking at scale** — A central registry probing thousands of instances creates enormous load and gives a worse signal than the instance's own heartbeat plus client-observed errors.

**Client-side versus server-side discovery**

```text
SERVER-SIDE (load balancer / proxy)
  client -> LB -> instance
  + clients are trivial; one place to change policy
  + language-agnostic
  - an extra hop and an extra failure domain
  - LB must itself be discovered (usually via DNS)

CLIENT-SIDE (library or sidecar)
  client (resolves + balances) -> instance
  + no extra hop, per-request balancing, locality awareness
  - every language needs a correct implementation
  - upgrading balancing policy means redeploying every client

SIDECAR / MESH (the common compromise)
  client -> localhost sidecar -> instance
  + client-side benefits, language-agnostic
  + policy upgrades ship with the sidecar
  - one more process per workload to operate
```

> **The registry must not be a hard dependency**  
> Design the failure mode explicitly: if the registry is down, clients continue with their cached list and new instances simply cannot join until it returns. That degrades elasticity, not availability. The alternative — failing lookups and therefore failing requests — means a single control-plane component can stop the entire system.

**Worked example**

A 200-service platform. Compare DNS-based discovery with a mesh, and quantify the staleness window.

**Staleness budget under two designs**

```text
DNS-BASED (Kubernetes Service DNS)
  pod dies without graceful shutdown
    kubelet notices                    ~5-10 s
    endpoints object updated           ~1 s
    kube-proxy / DNS updated           ~1 s
    client DNS cache TTL (30 s)        up to 30 s
    client connection pool pinned      until max lifetime
  realistic error window: 10-40 s, longer with pooled conns

MESH / xDS STREAMING
  pod dies without graceful shutdown
    readiness probe fails              ~2-5 s
    control plane pushes update        <1 s
    sidecars apply immediately         <1 s
  realistic error window: 3-7 s
  PLUS passive outlier ejection removes it on first errors
       (often <1 s, before the control plane even reacts)

GRACEFUL CASE (both designs)
  deregister -> drain -> exit
  error window ~ 0
```

| Metric | Value | Note |
|---|---|---|
| DNS window | 10–40 s | plus pooling |
| Mesh window | 3–7 s | **+ outlier ejection** |
| Graceful | ~0 s | if ordered correctly |
| Retry need | always | staleness is inevitable |

> **Retries are not optional in a discovered system**  
> No discovery mechanism eliminates the staleness window for unplanned failures. Therefore every client must retry an idempotent call against a different instance. Discovery narrows the window; retries make the remaining window invisible to users. Teams that treat retries as optional discover the window as user-facing errors during every node failure.

**When to use it**

- **Any dynamic environment** where instance addresses change — containers, autoscaling groups, spot capacity.
- **Microservice architectures**, where the number of caller/callee pairs makes static configuration unmanageable.
- **Multi-region or multi-zone routing**, where locality preference needs membership metadata.
- **Blue-green and canary deployments**, which are membership manipulations at heart.

**When to avoid it**

- **Do not make the registry a hard dependency** — cache and degrade rather than failing lookups.
- **Do not rely on DNS TTLs alone** for fast membership change; client and pool caching defeat them.
- **Do not use registry-side active health checks at large scale** — the probe load is enormous and the signal is worse than client-observed errors.
- **Do not shut down before deregistering**, or every deploy produces errors.
- **Do not skip retries and outlier ejection** on the assumption that discovery is fast enough. It never is for unplanned failures.

**Advantages**

- **Instances become fungible**, which is what makes autoscaling, rolling deploys and spot capacity practical.
- **Routing policy becomes centralised and dynamic** — locality preference, canary weights, failover — without redeploying callers.
- **Metadata travels with membership**, enabling version-aware and zone-aware routing.
- **Replaces brittle configuration** that otherwise drifts and breaks silently.

**Disadvantages**

- **Adds a control-plane component** that must be highly available and carefully degraded.
- **A staleness window is unavoidable** for unplanned failures, which forces retries and idempotency.
- **Propagation storms** can overwhelm the registry or newly registered instances after a mass change.
- **Client-side implementations multiply** across languages, which is the problem meshes exist to solve.
- **Debugging routing becomes indirect** — the answer to “where did my request go?” now involves a distributed cache.

**Trade-offs**

**Discovery mechanisms**

| Mechanism | Propagation | Client complexity | Failure mode |
|---|---|---|---|
| Static config / DNS A records | Minutes to hours | None | Stale until redeployed |
| Kubernetes Service DNS | Seconds to tens of seconds | None | Cached TTLs; pooled connections pin |
| Registry with polling (Consul, Eureka) | Poll interval | Moderate | Depends on client caching policy |
| Streaming watch / xDS | Sub-second | In the sidecar | Serves last known state if control plane is down |
| Client-side library balancing | Sub-second | High, per language | Depends on library quality |

The dominant industry answer is a sidecar receiving streamed membership, because it gives client-side benefits — per-request balancing, locality, outlier ejection — without requiring a correct implementation in every language.

**How it fails**

**Discovery failures**

| Failure | Cause | Fix |
|---|---|---|
| Errors during every deploy | Shutdown before deregistration | Deregister, wait for propagation, then drain |
| Traffic to dead instances | Staleness window | Retries to another instance; passive outlier ejection |
| Registry outage stops all traffic | Clients fail closed on lookup failure | Serve from cached membership; fail open |
| New instances overwhelmed | All traffic shifts to a freshly registered instance | Slow start / gradual weight ramp |
| Registry overload after a mass restart | Thundering herd of registrations and watches | Jittered registration; rate limits; incremental updates |
| Split brain in the registry | Registry partition creates divergent membership | Consensus-backed registry; prefer stale over divergent |
| Zombie instances | Registered but not serving; heartbeat succeeds while the app is broken | Readiness reflects real dependencies; client-observed ejection |

> **The registry as a correlated failure**  
> A registry that every service depends on synchronously is a component whose failure is a total outage — and it will fail during exactly the events that stress it most, like a mass restart. The discipline is that every client must be able to run indefinitely on its last known membership. Test this by taking the registry down in a game day and confirming that traffic continues.

**Limits**

> **Operating numbers**
>
> - **Heartbeat/lease**: 5 s heartbeat with a 15–30 s TTL is a common balance between detection speed and flapping.
> - **Staleness window** for unplanned failure: seconds with streaming, tens of seconds with DNS caching.
> - **Registration storms**: jitter registration and watch reconnects, or a mass restart self-DDoSes the control plane.
> - **Slow start**: ramp traffic to a new instance over 15–60 s so it is not overwhelmed while cold.
> - **Connection max age** matters as much as TTL — pooled connections ignore membership changes entirely.

**Alternatives**

| Approach | When | Trade |
|---|---|---|
| Static configuration | Few, stable services | Manual, drifts, breaks silently |
| DNS-based | Platform default (Kubernetes) | Caching and pooling defeat fast change |
| Dedicated registry (Consul, etcd) | Heterogeneous, non-container workloads | Another HA system to operate |
| Service mesh with xDS | Many services, many languages | Sidecar cost and operational complexity |
| Load balancer per service | Simple architectures | Extra hop; per-service configuration |

**In real systems**

- **Kubernetes** uses third-party registration: the control plane maintains Endpoints/EndpointSlices from readiness probes, and Service DNS exposes them — which is why deregistration ordering during pod shutdown is such a common source of deploy errors.
- **Envoy's xDS** streams membership and configuration to sidecars, giving sub-second propagation and the ability to serve last-known state during a control-plane outage.
- **Consul and etcd** provide consensus-backed registries for mixed workloads, trading some write availability for consistent membership.
- **Netflix's Eureka** deliberately chose availability over consistency, preferring to serve possibly-stale membership rather than fail — an explicit statement of the stale-but-available principle.
- **gRPC's name resolver plus load balancer interface** is the canonical client-side discovery design, and its per-language implementation burden is exactly what motivated meshes.

**Common mistakes**

- **Failing requests when the registry is unreachable** instead of using cached membership.
- **Shutting down before deregistering**, producing errors on every deploy.
- **Assuming DNS TTL bounds propagation** while connection pools pin to old addresses.
- **Registry-side active health checks at scale**, which create massive probe load.
- **No jitter on registration or watch reconnect**, so a mass restart overwhelms the control plane.
- **Sending full traffic to a cold instance** the moment it registers.
- **Treating retries as optional**, leaving the staleness window user-visible.

**The staff-level view**

The interesting decisions in discovery are all about failure behaviour, not about lookup.

- **Specify the registry's failure mode as a requirement**: clients run indefinitely on cached membership, and new registrations are what degrade. Then test it by taking the registry down deliberately.
- **Own the shutdown sequence platform-wide.** Deregister, wait for propagation, drain, exit — encoded in the platform's lifecycle hooks rather than left to each service.
- **Pair discovery with retries, idempotency and outlier ejection as a package.** Any one of them alone leaves user-visible errors during ordinary node failures.
- **Publish the staleness window** so teams size their retry and timeout budgets against a real number.
- **Prefer a sidecar over per-language libraries** once you have more than two or three languages; the cost of subtly different behaviour across implementations exceeds the sidecar's overhead.

**Go deeper**

Service discovery makes instances fungible: they register (themselves, or via an orchestrator), prove liveness with a heartbeat or lease, and clients look up or subscribe to the resulting membership list. That is what enables autoscaling, rolling deploys and spot capacity.

The engineering is in the failure modes. For planned shutdowns, deregistering *before* draining reduces the error window to near zero — the reverse order produces errors on every deploy. For unplanned failures a staleness window is unavoidable: roughly 3–7 seconds with streaming discovery, 10–40 seconds with DNS caching, and longer still where connection pools pin to old addresses. That window must be covered by retries to a different instance, idempotency, passive outlier ejection, and bounded connection lifetimes.

The most important design rule is that the registry must not be a hard dependency. When it is unreachable, clients should keep serving from their last known list indefinitely, so a control-plane outage degrades elasticity rather than availability. Systems that fail lookups — and therefore fail requests — convert one component's failure into a total outage, and the property must be verified by deliberately taking the registry down while traffic continues.

Service discovery is a cache-coherence problem wearing a directory's clothing. The registry holds authoritative membership; every client holds a stale copy; and all the interesting design is about how stale, and what a client does when acting on stale data.

**The moving parts.** Registration is either self-announced by the instance or performed by the orchestrator — Kubernetes uses the latter, deriving Endpoints from readiness probes. Liveness is proven by heartbeat or lease renewal, and readiness must be distinguished from it so that a warming or load-shedding instance is alive but not advertised. Lookup is either server-side, through a load balancer or proxy, or client-side, where the caller resolves and chooses. Propagation happens by polling, by streaming watch, or by DNS TTL expiry — and streaming is the only one that gives sub-second change without wasteful query volume.

**The staleness window is the product.** For planned changes it can be driven to zero by ordering: deregister, wait for propagation, drain in-flight work, then exit. Teams that shut down first produce a burst of client errors on every deploy and usually misdiagnose it as a load balancer problem. For unplanned failures the window is irreducible — detection time plus propagation plus client cache — and lands around 3–7 seconds with streaming xDS or 10–40 seconds with DNS-based discovery, with connection pooling extending it arbitrarily because an open socket never reconsiders its destination. Consequently retries to an alternate instance, idempotent operations, passive outlier ejection and bounded connection lifetimes are not optional extras; they are what makes discovery usable.

**Fail open, always.** The single worst error in this area is making the registry a synchronous hard dependency, so that a control-plane outage stops all data-plane traffic. The correct behaviour is for clients to serve indefinitely from their last known membership, with the degradation being that new instances cannot join and routing policy cannot change. Netflix's Eureka made this trade explicitly, preferring possibly-stale membership over unavailability. Because a fail-closed path tends to lurk somewhere regardless of intent, the property has to be exercised — take the registry down in a game day and confirm traffic continues.

**Scale changes the mechanism.** Registry-side active health checking does not survive thousands of instances: the probe load is enormous and the signal is worse than the instance's own heartbeat combined with client-observed errors. Mass restarts produce registration and watch-reconnect storms that can overwhelm the control plane precisely when it is needed, so jitter and rate limiting are required. And newly registered instances need a traffic ramp — slow start over tens of seconds — or they are overwhelmed while their caches and JIT are cold.

**Client-side, server-side, or sidecar.** Server-side discovery keeps clients trivial and centralises policy at the cost of an extra hop and failure domain. Client-side removes the hop and enables per-request balancing and locality awareness, but requires a correct implementation in every language and a redeploy to change policy. Past two or three languages the divergence between implementations costs more than a sidecar does, which is why the industry converged on a sidecar receiving streamed membership: client-side behaviour, language independence, and policy that ships with the proxy rather than with the application.

**Prove it — interview questions**

1. **[Basic] What does a service registry do?**

   <details><summary>Model answer</summary>

   It holds the current set of healthy instances for each service, so callers can find them without hard-coded addresses. Instances register themselves or are registered by an orchestrator, they renew a lease or heartbeat to prove liveness, and clients look up or subscribe to the list. The registry is what makes instances fungible, which is the prerequisite for autoscaling, rolling deploys and spot capacity.

   </details>

2. **[Basic] Why must you deregister before shutting down?**

   <details><summary>Model answer</summary>

   Because the registry keeps advertising an instance until it learns otherwise, and callers act on their cached copy of that list. If the process exits first, clients continue sending requests to a dead address for the full staleness window, producing errors on every planned deploy. Deregistering first stops new traffic while in-flight work finishes, which makes planned shutdowns invisible — the remaining error window belongs only to unplanned failures.

   </details>

3. **[Senior] What should a client do when the registry is unreachable?**

   <details><summary>Model answer</summary>

   Keep using its last known membership list, indefinitely. A stale list is usually mostly correct, and combining it with retries to alternate instances and passive outlier ejection makes the inaccuracy survivable. Failing lookups instead would turn a control-plane outage into a complete data-plane outage, which is strictly worse than the staleness it avoids. The degradation should be that new instances cannot join and routing policy cannot change — elasticity, not availability.

   </details>

4. **[Senior] Compare client-side and server-side discovery.**

   <details><summary>Model answer</summary>

   Server-side puts a load balancer or proxy between caller and callee: clients stay trivial and policy lives in one place, but you pay an extra hop and an extra failure domain, and the balancer itself must be discoverable. Client-side has the caller resolve and choose directly: no extra hop, per-request balancing, and locality awareness — but every language needs a correct implementation, and changing balancing policy means redeploying every client. The common compromise is a sidecar, which gives client-side behaviour with language independence and lets policy upgrades ship with the sidecar rather than with each application.

   </details>

5. **[Staff] Quantify the staleness window for an unplanned instance failure and design around it.**

   <details><summary>Model answer</summary>

   With streaming discovery it is roughly the readiness detection time plus push and apply latency, so three to seven seconds. With DNS it is detection plus endpoint propagation plus the client's DNS cache TTL, commonly ten to forty seconds — and longer still if connection pools pin to the old address, which TTL does not affect at all. Since that window cannot be eliminated, I design around it: idempotent operations with retries to a different instance, passive outlier ejection so a failing instance is dropped on observed errors rather than waiting for the control plane, bounded connection lifetimes so pools re-resolve, and a published window figure so teams size timeouts and retry budgets against a real number rather than a guess.

   </details>

6. **[Principal] How do you prevent the discovery control plane from becoming a company-wide single point of failure?**

   <details><summary>Model answer</summary>

   By making data-plane operation independent of it by design and by proof. Every client must be able to run indefinitely on cached membership, with control-plane unavailability degrading only elasticity and policy changes — and that property has to be verified in regular game days where the registry is deliberately taken down while traffic continues, because a fail-closed path always exists somewhere until you test for it. Structurally, I would keep the control plane out of the request path entirely, using streaming push to sidecars rather than synchronous lookup, and make registration and watch reconnects jittered so a mass restart does not self-DDoS the very component needed to recover. And I would treat the registry's own availability target as strictly higher than any service that depends on it, with its blast radius scoped per cluster or region rather than global, so a single control plane failure cannot be simultaneous everywhere.

   </details>

---
