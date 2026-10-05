# Curriculum · Realtime

[← System Design index](../README.md)

> 8 lessons in **Realtime**. Part of the Curriculum — Build the fundamentals.

## Contents

- **Realtime** (8): [WebSocket connections](#websocket-connections) · [Server-sent events](#server-sent-events) · [Long polling](#long-polling) · [Presence and heartbeats](#presence-and-heartbeats) · [Realtime fan-out routing](#realtime-fan-out-routing) · [Ordering, acknowledgments and resume](#ordering-acknowledgments-and-resume) · [Offline-first synchronization](#offline-first-synchronization) · [Operational transformation](#operational-transformation)

## Realtime

### WebSocket connections

*Upgrade an HTTP request into a persistent bidirectional channel, trading statelessness for the ability to push — and inheriting connection state as a scaling problem.*

**Flow:** `HTTP upgrade` → `Persistent socket` → `Connection registry` → `Message routing` → `Reconnect`

> **The 30-second version**  
> Upgrade HTTP into a persistent bidirectional channel for low-latency push — accepting connection affinity, a routing registry, and reconnection with replay as the cost.

**The problem**

A collaborative document, a chat application and a trading dashboard all need the server to send data the client did not ask for, within milliseconds. HTTP gives no mechanism for this: the client must ask, and asking repeatedly is either wasteful when nothing has changed or slow when something has.

Polling every second for a change that happens twice an hour means thousands of empty requests per client per day, each with connection overhead, authentication and a response — and still up to a second of delay when the change finally arrives.

> **The trade is statelessness for push**  
> A WebSocket keeps a connection open so either side can send at any time, which removes both the latency and the waste of polling. What it costs is the property that made HTTP servers easy to scale: any server could handle any request, because no server remembered anything. With persistent connections, a specific client is attached to a specific process, and every scaling, deployment and routing decision changes as a result.

**Mental model**

A WebSocket begins as an ordinary HTTP request that asks to be upgraded. Once upgraded, the connection is a bidirectional message stream that persists until one side closes it — and the server must now track which client is on which connection on which process.

1. **Upgrade** — An HTTP request with an upgrade header; the server accepts and the same TCP connection becomes a message channel.
2. **Persistent socket** — Full-duplex frames in both directions, with no request-response pairing required.
3. **Connection registry** — Which user is connected to which process — the state that makes routing possible and scaling hard.
4. **Routing** — Delivering a message to a user means finding their connection, which may be on another process entirely.
5. **Reconnect** — Connections drop constantly; the client must reconnect and the system must handle what was missed.

> **Connection state is affinity, and affinity breaks everything HTTP made easy**  
> A user's socket lives on one process. Sending them a message requires knowing which one, so a registry is needed. Restarting that process disconnects every client on it. Load balancing cannot move an existing connection. Autoscaling down kills live sessions. None of these are problems for stateless HTTP, and all of them arrive together the moment connections persist.

**How it works**

**From upgrade to routed message**

```text
1  UPGRADE
   GET /ws  Upgrade: websocket
   -> 101 Switching Protocols
   -> the TCP connection is now a message channel
   -> AUTHENTICATE HERE. After the upgrade there are
      no more HTTP requests to authenticate.

2  REGISTER
   registry[userId] = {process, connectionId}
   -> so other processes can find this connection
   -> typically in Redis or a similar shared store

3  ROUTING A MESSAGE TO A USER
   look up the user's process
   if it is this process:  write to the socket
   else:                   forward via pub/sub or RPC
   -> this indirection is the price of horizontal scale

4  DISCONNECT
   remove from the registry
   -> but a process that CRASHES removes nothing
   -> stale entries accumulate and route messages
      into the void
   -> hence heartbeats and TTLs on registry entries

WHY THE REGISTRY IS THE HARD PART
  it must be fast (every message consults it)
  it must be accurate (stale entries silently lose
    messages)
  it must survive process crashes (which never clean up
    after themselves)
```

1. **Authenticate during the upgrade** — It is the last HTTP request. After it, there is no per-message authentication unless you build one, and a long-lived connection outlives token expiry.
2. **Keep a shared connection registry with TTLs** — Routing across processes requires knowing where each user is, and crashed processes never deregister themselves.
3. **Send application-level heartbeats** — TCP can hold a connection open long after the peer has gone; only application traffic reveals a dead connection promptly.
4. **Expect and plan for reconnection** — Mobile networks, proxies and idle timeouts drop connections routinely; reconnection is the normal case, not the exception.
5. **Handle deploys explicitly** — Every restart disconnects everyone on that process, so staggered restarts and reconnect backoff are required to avoid a stampede.
6. **Bound per-connection memory** — Each connection holds buffers and state; a hundred thousand connections at a few hundred kilobytes each is tens of gigabytes.

**What connections actually cost**

```text
PER CONNECTION ON THE SERVER
  socket + kernel buffers        ~10-40 KB
  application state, buffers     ~10-100 KB
  TLS session state              ~20 KB
  -> roughly 50-150 KB per idle connection

SO
  100,000 connections  = 5-15 GB just to hold them
  1,000,000 connections = spread across many processes

FILE DESCRIPTORS
  one per connection; default limits are far too low
  -> raise ulimit, or fail at a few thousand

CONNECTION CHURN IS THE REAL COST
  establishing a WebSocket = TCP + TLS + upgrade
  -> expensive relative to holding one open
  -> a client reconnecting every 30 s is far more
     expensive than one connected for hours
  -> which is why a reconnect STAMPEDE after a deploy
     can be worse than the outage that caused it

RECONNECT STAMPEDE
  restart a process holding 50,000 connections
  -> all 50,000 reconnect within seconds
  -> the replacement process handles 50,000
     TLS handshakes at once
  -> fix: exponential backoff WITH JITTER on the
     client, staggered restarts on the server
```

> **Authentication happens once but the connection lives for hours**  
> The upgrade request is the last point at which normal HTTP authentication applies. A connection established with a token valid for fifteen minutes may remain open for eight hours, long after the token expired, the user's permissions changed, or their session was revoked. Systems that need authorisation to remain current must re-validate periodically over the connection and close it when entitlement ends — otherwise revocation simply does not take effect.

**Worked example**

A collaborative editor, showing where each piece of machinery earns its place.

**Architecture and its consequences**

```text
CONNECT
  client opens WebSocket with an auth token
  server validates, registers:
    registry[userId] = {process: "ws-7", ttl: 60 s}
  client joins document rooms

SENDING AN EDIT TO COLLABORATORS
  edit arrives on ws-7
  look up who else is in this document
  some are on ws-7      -> write directly
  others on ws-2, ws-9  -> publish to a channel;
                           those processes deliver
  -> pub/sub decouples routing from connection location

HEARTBEAT
  ping every 30 s, expect pong
  no pong in 60 s -> close and deregister
  registry TTL 60 s, refreshed by heartbeat
  -> a crashed process's entries expire on their own,
     which is the only thing that cleans them up

RECONNECT
  client reconnects with backoff + jitter
  sends the last sequence number it saw
  server replays anything missed, or tells it to resync
  -> WITHOUT this, a 3-second disconnect silently
     loses edits and the document diverges

DEPLOY
  drain one process at a time
  send a "reconnect soon" message before closing
  clients reconnect with jitter to other processes
  -> restarting everything at once = a stampede of
     handshakes against a cold fleet

CAPACITY
  50,000 connections per process at ~100 KB = ~5 GB
  20 processes = 1,000,000 concurrent users
```

| Metric | Value | Note |
|---|---|---|
| Per connection | 50–150 KB | memory floor |
| Heartbeat | 30 s ping | detects dead peers |
| Registry TTL | 60 s | **survives crashes** |
| Reconnect | backoff + jitter | prevents stampede |

> **Reconnection is where correctness is won or lost**  
> The connection dropping is routine; what matters is what happens to messages sent while it was down. A client that reconnects without telling the server what it last saw has a gap it does not know about, and the application silently diverges. Sequence numbers on messages and a replay-or-resync decision on reconnect are what turn an unreliable transport into a reliable application — and they are usually the part left until last.

**When to use it**

- **Bidirectional communication**, where the client also sends frequently — chat, collaboration, games.
- **Low-latency push**, where sub-second delivery of server-initiated events matters.
- **High message frequency**, where per-message HTTP overhead would dominate.
- **Long-lived sessions**, where connection setup cost is amortised over hours.
- **Interactive applications** whose state must track the server continuously.

**When to avoid it**

- **Do not use WebSockets for server-to-client-only streams**, where server-sent events are simpler and survive proxies better.
- **Do not use them for infrequent updates**, where polling costs less than maintaining connections.
- **Do not use them where HTTP semantics are valuable** — caching, standard status codes, ordinary observability.
- **Do not adopt them without a reconnection and replay design**, which is where the correctness problems live.
- **Do not use them behind infrastructure that does not support upgrades**, which some proxies and gateways still do not.

**Advantages**

- **True bidirectional messaging** with no request-response pairing.
- **Very low latency**, with no per-message connection or header overhead.
- **Efficient for high-frequency traffic**, where HTTP overhead would dominate.
- **Server-initiated push** without polling.
- **Binary and text frames**, so protocols can be compact.

**Disadvantages**

- **Connection state destroys statelessness**, complicating scaling, routing and deployment.
- **Memory per connection** sets a hard ceiling per process.
- **Reconnection and missed-message handling** must be built, and are easy to get subtly wrong.
- **Authentication is a point-in-time check** on a long-lived channel.
- **Weaker infrastructure support** — some proxies, gateways and corporate networks interfere.
- **Harder to observe**, since a message stream does not map onto request-based tooling.

**Trade-offs**

**Transport comparison**

| Transport | Direction | Overhead | Best for |
|---|---|---|---|
| WebSocket | Bidirectional | Low per message; high per connection | Chat, collaboration, games |
| Server-sent events | Server to client | Low; plain HTTP | Feeds, notifications, progress |
| Long polling | Server to client | High per message | Legacy compatibility |
| Short polling | Client pulls | Wasteful when idle | Infrequent, tolerant updates |
| HTTP/2 or /3 streams | Bidirectional within a request | Low | Streaming RPC between services |

The most common mistake is reaching for WebSockets when the traffic is entirely server-to-client. Server-sent events give the same push with ordinary HTTP semantics, automatic reconnection with a last-event identifier built into the browser, and far fewer infrastructure surprises.

**How it fails**

**WebSocket failure modes**

| Failure | Cause | Fix |
|---|---|---|
| Messages silently lost after a blip | Reconnect without replay | Sequence numbers; replay or resync on reconnect |
| Fleet overwhelmed after a deploy | Reconnect stampede | Backoff with jitter; staggered draining |
| Messages routed to nothing | Stale registry entries from crashed processes | TTLs refreshed by heartbeat |
| Connections appear alive but are dead | TCP holds the socket open | Application-level ping and pong |
| Revoked users stay connected | Auth checked only at upgrade | Periodic re-validation; close on revocation |
| Process memory exhausted | Unbounded per-connection buffers | Cap buffers; shed slow consumers |
| Connections fail for some users | Proxies or networks blocking upgrades | Fallback transport; use SSE where one-way suffices |

**Limits**

> **Capacity figures**
>
> - **Memory**: roughly 50–150 KB per idle connection including TLS state.
> - **Per process**: tens of thousands of connections is typical; beyond that, memory and event-loop pressure dominate.
> - **File descriptors**: one per connection, so default limits must be raised.
> - **Heartbeat**: 30 s ping with a 60 s timeout is a common starting point.
> - **Connection setup** costs far more than holding one open, which makes churn and reconnect stampedes the dominant load risk.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| WebSocket | Bidirectional interactive apps | Connection state; infrastructure friction |
| Server-sent events | One-way push | No client-to-server channel |
| Long polling | Universal compatibility | Request overhead per message |
| Polling | Infrequent updates | Latency or waste |
| WebTransport | Modern low-latency, unreliable streams | Limited support |
| Managed realtime service | Avoiding the operational burden | Cost; vendor dependency |

Managed realtime services are worth serious consideration: connection state, routing, presence and reconnection are genuinely difficult to operate at scale, and the burden of running them is often larger than the feature they support.

**In real systems**

- **Chat and collaboration products** use WebSockets because traffic is genuinely bidirectional and latency-sensitive.
- **Connection registries in a shared store with TTLs** are the standard pattern, because crashed processes never deregister themselves.
- **Pub/sub between processes** decouples message routing from which process holds a connection, and is what allows horizontal scaling.
- **Reconnect with backoff and jitter** is universal client practice, because synchronised reconnection after a deploy is a self-inflicted load spike.
- **Sequence numbers with replay on reconnect** are what make delivery reliable across the disconnections that certainly occur.

**Common mistakes**

- **Using WebSockets for one-way push**, where SSE is simpler and more robust.
- **No reconnection replay**, silently losing messages sent during a blip.
- **Reconnect without jitter**, turning every deploy into a stampede.
- **Registry entries without TTLs**, so crashed processes leave permanent stale routes.
- **Relying on TCP to detect dead peers**, which it does slowly or never.
- **Authenticating only at upgrade**, so revocation has no effect on live connections.
- **Unbounded per-connection buffers**, letting one slow consumer exhaust process memory.

**The staff-level view**

WebSockets are usually adopted for the push capability and then owned for the connection-state consequences.

- **Ask whether the traffic is bidirectional.** A great many WebSocket deployments are one-way and would be simpler, more robust and better supported by infrastructure as server-sent events.
- **Design reconnection and replay before shipping.** Disconnection is routine; the question is whether messages sent during it are lost, and answering that after launch means debugging silent divergence.
- **Treat deploys as a load event.** Every restart disconnects everyone on that process, and synchronised reconnection is expensive precisely when the fleet is least ready — staggered draining and jittered backoff are not optional.
- **Make the registry self-healing.** Crashed processes never clean up, so entries must expire on their own or routing quietly delivers into the void.
- **Re-validate authorisation over the connection.** A point-in-time check at upgrade means revocation does not take effect for hours, which is a security property nobody intended.

**Go deeper**

A WebSocket begins as an HTTP request asking to be upgraded; afterwards the connection carries messages in both directions at any time. That eliminates polling's waste and latency, which is why chat, collaboration and live dashboards use it. The cost is statelessness: a client is now attached to a specific process, so routing a message to a user requires a registry, and every scaling and deployment operation becomes a connection-management problem.

The machinery that makes it work is mostly about failure. Application-level heartbeats detect dead peers that TCP will happily hold open; registry entries carry TTLs because crashed processes never deregister; cross-process pub/sub decouples routing from connection location; and clients reconnect with backoff and jitter, because synchronised reconnection after a deploy is a load spike arriving precisely when the fleet is coldest.

Correctness is decided at reconnection. Disconnections are routine, so what matters is whether messages sent during one are lost — which means sequence numbers and a replay-or-resync decision when the client returns. Two further points are commonly missed: authentication happens once at upgrade while the connection may live for hours, so revocation requires periodic re-validation; and a great many WebSocket deployments carry only server-to-client traffic, where server-sent events would be simpler and better supported.

WebSockets exist because HTTP's request-response model cannot express server-initiated delivery, and the workarounds — polling frequently or waiting long — trade waste against latency with no good setting.

**Statelessness is the price.** The upgrade converts a TCP connection into a message channel, which is exactly the capability wanted, but the connection now belongs to one process. Delivering a message to a user requires knowing which process holds them, so a shared registry appears; delivering across processes requires pub/sub or RPC; restarting a process disconnects everyone on it; load balancers cannot relocate live connections; scaling down terminates sessions. Every one of these is trivial for stateless HTTP and all of them arrive together.

**Connection cost is memory and churn, not throughput.** Each idle connection holds socket buffers, TLS state and application structures totalling roughly fifty to a hundred and fifty kilobytes, so capacity per process is bounded by memory long before CPU matters, and file descriptor limits bite earlier still. More importantly, establishing a connection — TCP handshake, TLS negotiation, upgrade — costs far more than holding one, which makes churn the dominant load risk. A deploy that restarts a process holding fifty thousand connections produces fifty thousand simultaneous handshakes against a cold replacement, which is why jittered client backoff and staggered server draining are structural requirements rather than polish.

**Liveness must be checked at the application layer.** TCP will hold a connection open long after the peer has vanished, particularly across mobile networks and stateful middleboxes, so a socket that appears healthy may be delivering into nothing. Periodic ping and pong reveals this promptly, and the same heartbeat is what refreshes the registry TTL — which matters because a crashed process deregisters nothing, and stale registry entries route messages into the void without producing an error anywhere.

**Reconnection is where the application's correctness is determined.** The transport is unreliable; disconnection is the normal case. What distinguishes a working system is whether messages sent during the gap are recovered. That requires sequence numbers on messages, the client reporting its last seen position on reconnect, and the server deciding between replaying the gap and instructing a full resync when the gap is too large. Without it, a brief network blip silently loses updates and the client's view diverges from the server's — a failure that produces no error and is discovered as inexplicable inconsistency.

**Authentication is a point-in-time check on a long-lived channel.** The upgrade request is the final moment at which ordinary HTTP authentication applies, yet the resulting connection may live for hours — well past token expiry, permission changes or session revocation. Systems where entitlement can change must re-validate periodically over the connection and close it when access ends; otherwise revocation is nominal, and the security model differs from the one everybody assumes.

**The question worth asking first is whether the traffic is bidirectional.** A substantial share of WebSocket deployments carry only server-to-client messages, for which server-sent events provide the same push over ordinary HTTP — traversing proxies more reliably, remaining visible to standard observability tooling, and with browser-native reconnection carrying a last-event identifier. Choosing WebSockets there buys connection-state complexity for a capability that is never exercised. And where a bidirectional channel genuinely is needed, the operational burden of connection registries, routing, presence and deployment choreography is large enough that a managed realtime service deserves honest evaluation against building and owning it.

**Prove it — interview questions**

1. **[Basic] What does a WebSocket give you that HTTP does not?**

   <details><summary>Model answer</summary>

   The ability for the server to send without being asked. HTTP is request-response, so a client learns about changes only by polling — which is wasteful when nothing has changed and slow when something has. A WebSocket starts as an HTTP request that asks to be upgraded, after which the same connection carries messages in both directions at any time. That removes both the polling waste and the polling latency, at the cost of the connection now being stateful.

   </details>

2. **[Basic] When should you use server-sent events instead?**

   <details><summary>Model answer</summary>

   Whenever the traffic is server-to-client only — notifications, feeds, progress updates, live figures. SSE is ordinary HTTP, so it works through proxies and infrastructure that sometimes interfere with upgrades, it is observable with normal HTTP tooling, and browsers implement reconnection with a last-event identifier for you. WebSockets are the right choice when the client also sends frequently; using them for one-way push buys connection-state complexity for a capability that is not being used.

   </details>

3. **[Senior] What does persistent connection state do to scaling?**

   <details><summary>Model answer</summary>

   It reintroduces affinity into a stack built to avoid it. A user's socket lives on one specific process, so delivering a message to them requires knowing which — hence a shared connection registry, plus pub/sub or RPC to forward messages between processes. Restarting a process disconnects everyone on it, load balancers cannot relocate an existing connection, and scaling down terminates live sessions. Each of these is a non-problem for stateless HTTP and all of them arrive together the moment connections persist.

   </details>

4. **[Senior] Why do registry entries need TTLs?**

   <details><summary>Model answer</summary>

   Because a process that crashes does not deregister its connections. A clean shutdown can remove entries; a crash, an out-of-memory kill or a network partition leaves them behind, and messages routed to those entries disappear silently — which is a particularly unpleasant failure because nothing errors. Giving entries a short TTL refreshed by the connection's heartbeat means stale routes expire on their own, making the registry self-healing rather than dependent on cleanup that by definition will not happen in the cases that matter.

   </details>

5. **[Staff] Design the connection layer for a collaborative editor with a million concurrent users.**

   <details><summary>Model answer</summary>

   Around twenty processes at fifty thousand connections each, which at roughly a hundred kilobytes per connection is about five gigabytes per process — memory is the binding constraint, so per-connection buffers must be capped and slow consumers shed rather than buffered. Authentication at upgrade, since that is the last HTTP request, with periodic re-validation over the connection so that revocation actually takes effect rather than waiting hours for the socket to close. A shared registry mapping user to process with a sixty-second TTL refreshed by a thirty-second heartbeat, so crashed processes stop receiving routes without anyone cleaning up. Cross-process delivery via pub/sub so routing does not care which process holds a connection. On the client, reconnection with exponential backoff and jitter, sending the last sequence number it saw so the server can replay or instruct a full resync — this is the part that determines whether a three-second network blip silently loses edits. And deploys drain one process at a time with an advance notice message, because fifty thousand simultaneous TLS handshakes against a cold process is a self-inflicted outage.

   </details>

6. **[Principal] What makes realtime connection layers disproportionately expensive to own?**

   <details><summary>Model answer</summary>

   That they invert the assumptions the rest of the stack is built on, and the costs appear in operations rather than in development. Stateless HTTP made deployment, scaling and routing trivial; persistent connections make each of them a design problem, so a deploy becomes a load event, autoscaling down becomes a user-visible disruption, and every process restart is a coordinated reconnection that must be paced. The correctness surface is equally unusual: the transport is unreliable in a way the application must compensate for, so sequence numbers, replay and resync are load-bearing rather than nice-to-have, and getting them wrong produces silent divergence rather than errors. On top of that, ordinary observability assumes requests, so a message stream is nearly invisible to standard tooling until custom instrumentation exists. My practical conclusion is twofold: first, confirm the traffic is genuinely bidirectional, because a large fraction of WebSocket deployments would be better as SSE; and second, seriously price a managed realtime service against building it, because the operational burden frequently exceeds the value of the feature it enables.

   </details>

---

### Server-sent events

*A one-way stream of events over an ordinary HTTP response, with browser-native reconnection and resume — the right default whenever the client does not need to push.*

**Flow:** `HTTP request` → `Streaming response` → `Event IDs` → `Auto-reconnect` → `Resume from last ID`

> **The 30-second version**  
> Stream events over a never-ending HTTP response, with event ids giving browser-native reconnection and exact resume — the right default for any push that does not need a client channel.

**The problem**

A dashboard shows live figures, a notification bell must light up, an export job reports progress. All are server-to-client only, all need updates within a second or two, and none require the client to send anything on the same channel.

Polling for these wastes requests and adds latency. WebSockets solve it but bring a bidirectional channel that is never used in one direction, along with connection-state complexity, proxy friction and observability gaps — a heavyweight answer to a one-way question.

> **SSE is push without leaving HTTP**  
> A server-sent event stream is an ordinary HTTP response that simply does not end. It traverses proxies and infrastructure that mishandle upgrades, it is visible to standard HTTP tooling, and the browser implements reconnection with resume for free. For one-way delivery it gives almost everything a WebSocket would while remaining, architecturally, a long HTTP request.

**Mental model**

The client issues a normal GET. The server responds with a streaming content type and writes events as they occur, keeping the response open indefinitely. Each event may carry an identifier, which the browser remembers and sends back automatically if the connection drops.

1. **Request** — An ordinary GET with an event-stream accept header — cookies, headers and auth all behave normally.
2. **Streaming response** — Events written as text in a simple line-based format, flushed as they occur.
3. **Event id** — An identifier per event, remembered by the browser.
4. **Auto-reconnect** — The browser reconnects on failure without any client code.
5. **Resume** — The last seen id is sent back on reconnect, so the server can replay the gap.

> **Resume is built into the protocol, not bolted on**  
> The browser stores the last event id it received and sends it as a header when reconnecting. A server that honours that header and replays subsequent events gives exact recovery from disconnection with no client code at all — which is precisely the part teams most often fail to build when rolling their own over WebSockets.

**How it works**

**The wire format and what each field does**

```text
RESPONSE
  Content-Type: text/event-stream
  Cache-Control: no-cache
  Connection: keep-alive

EVENTS  (text, line-based, blank line terminates)

  id: 1042
  event: price
  data: {"symbol":"ABC","price":41.20}

  id: 1043
  event: notification
  data: {"text":"Job complete"}

  : this is a comment - used as a keepalive

ON RECONNECT the browser automatically sends
  Last-Event-ID: 1043
-> the server replays from 1044 onward
-> EXACT recovery, with zero client code

KEEPALIVE
  idle proxies close connections after 30-120 s
  -> send a comment line every ~20 s
  -> costs almost nothing, prevents mysterious drops

WHAT THE CLIENT WRITES
  const es = new EventSource("/stream");
  es.addEventListener("price", e => update(e.data));
  -> reconnection, backoff and resume: all built in
```

1. **Always send event ids** — They are what make resume work. Without them a reconnect silently starts from the present and the gap is lost.
2. **Honour the Last-Event-ID header** — The browser sends it for free; a server that ignores it discards the protocol's best feature.
3. **Send periodic keepalive comments** — Idle intermediaries close quiet connections, producing drops that look like network faults.
4. **Disable response buffering along the path** — Proxies and frameworks that buffer responses will hold events until the buffer fills, converting a live stream into a batch delivery.
5. **Bound how far back you can replay** — Replay requires retained history; beyond the retention window the server must tell the client to resynchronise.
6. **Watch connection limits on HTTP/1.1** — Browsers cap connections per origin, and a long-lived stream occupies one; HTTP/2 removes this.

**The two failure modes that surprise people**

```text
1  BUFFERING BREAKS STREAMING SILENTLY
   a proxy, gateway or framework that buffers the
   response holds events until the buffer fills
   -> the client receives nothing for a long time,
      then everything at once
   -> the stream "works" in tests and fails in
      production behind a different proxy
   -> fix: disable buffering explicitly at every hop
      and flush after each event

2  HTTP/1.1 CONNECTION LIMIT
   browsers allow ~6 connections per origin
   an SSE stream holds one open permanently
   -> open the app in 6 tabs and the 7th hangs,
      including ordinary API calls
   -> this presents as "the app randomly stops
      working with several tabs open"
   -> fix: HTTP/2 (no practical limit), or share one
      stream across tabs via a shared worker

NEITHER is a problem with SSE itself; both are
environment interactions, and both are hard to
diagnose because they do not look like stream bugs.
```

> **Replay depth is a retention decision**  
> Honouring Last-Event-ID means the server must still have the events after that id. That requires retaining recent history per stream, and the retention window bounds how long a disconnection can last before exact resume is impossible. Beyond it the only correct response is to tell the client to discard its state and resynchronise from scratch — which the application must support, or a long disconnection produces silent divergence.

**Worked example**

A notification and live-metrics stream, with the choices that make it robust.

**Design**

```text
ENDPOINT  GET /stream  (ordinary auth: cookies/headers)
  -> auth works exactly as for any other request,
     unlike a WebSocket upgrade

SERVER
  subscribe this user to their channels
  write each event with a monotonic id
  flush after every event
  comment keepalive every 20 s
  retain the last 5 minutes of events per user
    -> bounds memory and defines the resume window

CLIENT
  new EventSource("/stream")
  that is the whole reconnection implementation

DISCONNECTION
  network blip of 8 s
  browser reconnects automatically
  sends Last-Event-ID: 4471
  server replays 4472..current
  -> no gap, no client code

DISCONNECTION LONGER THAN RETENTION
  client sends Last-Event-ID: 2100
  server no longer has it
  -> respond with a resync event
  -> client refetches full state, then resumes
  -> this path MUST exist or long disconnections
     silently diverge

SCALE
  one connection per user, ~30 KB each
  200,000 users = ~6 GB across the fleet
  cheaper than WebSockets: no bidirectional buffers,
  no connection registry for inbound routing
```

| Metric | Value | Note |
|---|---|---|
| Client code | one line | reconnect included |
| Keepalive | 20 s comment | beats proxy timeouts |
| Retention | 5 minutes | **resume window** |
| Fallback | resync event | beyond retention |

> **The client-side simplicity is the real argument**  
> Constructing an EventSource gives reconnection, exponential backoff and resume-from-last-id with no code. The equivalent over WebSockets is several hundred lines of client logic that every team writes slightly differently and most get subtly wrong — particularly the resume path. For one-way delivery, that difference in what can go wrong usually outweighs any protocol-level comparison.

**When to use it**

- **Server-to-client only traffic** — notifications, feeds, live figures, progress.
- **When reconnection and resume matter** and you would rather not implement them.
- **Where HTTP semantics are valuable** — existing auth, proxies, observability, standard tooling.
- **Streaming incremental results**, such as token-by-token responses.
- **Environments hostile to upgrades**, such as corporate proxies that break WebSockets.

**When to avoid it**

- **Do not use SSE when the client must also send frequently** on the same channel — that is a WebSocket.
- **Do not use it for binary data**, since the format is text-based.
- **Do not use it over HTTP/1.1 in multi-tab applications** without addressing the per-origin connection limit.
- **Do not use it where very high message rates** make per-event text framing wasteful.
- **Do not deploy it without checking for response buffering** anywhere along the path.

**Advantages**

- **Ordinary HTTP**, so auth, proxies, observability and tooling all work unchanged.
- **Built-in reconnection** with exponential backoff, implemented by the browser.
- **Built-in resume** via Last-Event-ID, which is exact recovery for free.
- **Trivial client code** — a single constructor.
- **Cheaper per connection** than WebSockets, with no inbound routing registry needed.
- **Passes through infrastructure** that mishandles protocol upgrades.

**Disadvantages**

- **One-way only**, so client messages need a separate mechanism.
- **Text only**, making binary payloads awkward.
- **Per-origin connection limits** on HTTP/1.1 cause hard-to-diagnose multi-tab failures.
- **Response buffering** anywhere on the path silently breaks streaming.
- **Replay depth is bounded** by whatever history the server retains.
- **Higher per-message overhead** than a binary frame at very high rates.

**Trade-offs**

**SSE against the alternatives**

| Aspect | SSE | WebSocket | Polling |
|---|---|---|---|
| Direction | Server to client | Bidirectional | Client pulls |
| Client code | One line | Hundreds of lines | Simple |
| Reconnect and resume | Built in | Must be built | Not applicable |
| Infrastructure friction | Low — plain HTTP | Moderate — upgrades | None |
| Latency | Low | Lowest | Up to the interval |
| Per-connection cost | Lower | Higher | None between polls |

The decision is almost entirely about direction. If the client needs to send on the same channel, use a WebSocket; otherwise SSE gives the same push with less to build and less to go wrong. Client-to-server messages can always travel as ordinary POST requests, which for most applications is perfectly adequate.

**How it fails**

**SSE failure modes**

| Failure | Cause | Fix |
|---|---|---|
| Events arrive in bursts, not live | Response buffering in a proxy or framework | Disable buffering at every hop; flush per event |
| Connection drops every minute | Idle timeout on an intermediary | Comment keepalive every ~20 s |
| App hangs with several tabs open | HTTP/1.1 per-origin connection limit | Serve over HTTP/2; or share a stream via a worker |
| Gap in events after a reconnect | Server ignores Last-Event-ID | Honour the header and replay |
| Silent divergence after a long outage | Disconnection exceeded retention with no fallback | Send a resync event; client refetches state |
| Memory growth on the server | Unbounded per-stream event retention | Bound the retention window explicitly |
| Events lost on server restart | Retention held only in process memory | Shared retention, or resync on reconnect |

**Limits**

> **Practical parameters**
>
> - **Keepalive**: a comment every 15–30 s, comfortably inside typical proxy idle timeouts.
> - **Retention**: a few minutes of events per stream is a common resume window; longer costs memory.
> - **HTTP/1.1**: around 6 connections per origin, one of which a stream consumes permanently.
> - **Per connection**: lighter than a WebSocket — roughly tens of kilobytes.
> - **Message rate**: text framing makes very high rates wasteful relative to binary frames.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Server-sent events | One-way push over HTTP | No client channel; text only |
| WebSocket | Bidirectional interaction | Build reconnect and resume yourself |
| Long polling | Very old clients | Request overhead per message |
| HTTP/2 server push | Preloading resources | Not a general eventing mechanism |
| Polling | Infrequent updates | Latency or waste |
| Managed realtime service | Avoiding operations | Cost; vendor dependency |

SSE combined with ordinary POST requests for client-to-server messages covers a surprising proportion of what people reach for WebSockets to do, with far less machinery — the asymmetry in traffic usually matches the asymmetry in the design.

**In real systems**

- **Streaming language-model responses** commonly use SSE, since token delivery is one-way and incremental.
- **Notification and activity feeds** in web applications use it because ordinary HTTP auth and proxies work unchanged.
- **Progress reporting for long-running jobs** is a natural fit — one-way, low rate, resumable.
- **Last-Event-ID resume** is relied upon in production precisely because it is the recovery path teams most often fail to build themselves.
- **HTTP/2 deployment** removes the per-origin connection limit that otherwise makes SSE awkward in multi-tab applications.

**Common mistakes**

- **Choosing WebSockets for one-way traffic**, buying complexity that is never used.
- **Omitting event ids**, which disables resume entirely.
- **Ignoring Last-Event-ID**, discarding the protocol's best feature.
- **No keepalive**, so idle proxies close connections mysteriously.
- **Response buffering left enabled**, converting a live stream into batches.
- **HTTP/1.1 in a multi-tab app**, causing hangs that look unrelated.
- **No resync path** for disconnections longer than retention.

**The staff-level view**

SSE is consistently underused because WebSockets are better known, and the difference shows up as avoidable client complexity.

- **Make SSE the default for one-way push.** The reconnection and resume logic that teams write over WebSockets is the part most often subtly wrong, and SSE provides it in the protocol.
- **Verify buffering end to end before launch.** A stream that works locally and batches in production behind a different proxy is a confusing failure that has nothing to do with the code.
- **Serve it over HTTP/2.** The per-origin connection limit produces a bizarre multi-tab failure that is extremely hard to attribute to a streaming endpoint.
- **Define the retention window and the resync path together.** Resume is exact only within retention, and the fallback for exceeding it must exist or long disconnections diverge silently.
- **Pair SSE with ordinary POST for client messages.** The traffic is usually asymmetric, and matching the design to that asymmetry avoids a bidirectional channel that is used in one direction.

**Go deeper**

Server-sent events deliver push over an ordinary HTTP response that stays open. Because it is plain HTTP, authentication, proxies and observability behave normally, and it traverses infrastructure that mishandles protocol upgrades. In the browser, the EventSource API provides reconnection with exponential backoff and resume from the last event id with no application code at all.

Resume is the standout feature: each event carries an id, the browser sends the last one it saw as a header on reconnect, and a server that honours it replays the gap exactly. This requires retaining recent history per stream, and that retention window bounds how long a disconnection can last — beyond it the server must instruct the client to resynchronise, a path that must exist or long outages diverge silently.

Two environment interactions cause most trouble and neither looks like a stream bug. Response buffering anywhere along the path holds events until the buffer fills, converting a live stream into batch delivery that works in development and fails in production. And on HTTP/1.1 the per-origin connection limit means a permanently open stream plus several tabs eventually hangs all requests to that origin — serving over HTTP/2 removes it.

Server-sent events provide server-to-client push without leaving HTTP: the response is an ordinary one that simply never completes.

**Staying inside HTTP is the point.** Authentication works through existing cookies and headers rather than requiring a separate path at upgrade time; proxies, gateways and corporate networks that interfere with protocol upgrades pass it through as a normal response; and standard HTTP observability tooling still sees a request. That last property matters more than it sounds, because a WebSocket message stream is largely invisible to request-oriented monitoring until custom instrumentation is built.

**Reconnection and resume come from the protocol.** The browser reconnects automatically with backoff, and because each event may carry an id which the browser retains, it sends Last-Event-ID on reconnection. A server honouring that header replays everything after that point, giving exact recovery from a disconnection with no client code. This is precisely the mechanism teams building on WebSockets must implement themselves, and it is the piece most often left incomplete — so brief network interruptions silently drop events and the client's state diverges without any error surfacing.

**Resume depth is a retention decision.** Replaying after an id requires still holding the events following it, so the server retains a window of recent history per stream. That window bounds how long a client can be absent and still resume exactly, and beyond it the only correct behaviour is to send a resynchronisation instruction so the client discards its state and refetches. Applications that do not implement that fallback appear to work — right up to the first long disconnection, after which the client is quietly wrong.

**Two environment interactions cause most production incidents.** Any intermediary or framework that buffers the response will hold events until its buffer fills, so the client sees nothing and then a burst; this passes local testing and fails behind different infrastructure, and it presents as a latency mystery rather than a configuration problem. Separately, HTTP/1.1's per-origin connection limit of roughly six means a permanently open stream occupies one slot per tab, so a user with several tabs open eventually finds that ordinary API requests hang — a symptom almost nobody attributes to a streaming endpoint. Serving over HTTP/2 eliminates the second entirely; the first requires explicitly disabling buffering at every hop and flushing after each event.

**The limitation is genuinely one-way, and usually acceptable.** SSE carries no client-to-server channel and no binary frames, so bidirectional or high-rate binary protocols belong on WebSockets. But most realtime requirements are asymmetric — the server has much to say and the client occasionally acts — and pairing an event stream with ordinary POST requests matches that shape closely while avoiding connection registries, upgrade-time authentication and custom reconnection logic.

**The strongest argument is what cannot go wrong.** Constructing an EventSource yields reconnection, backoff and resume; the equivalent over WebSockets is substantial client code that every team writes differently and most get subtly wrong in the recovery path. For one-way delivery, the reduction in what has to be built and maintained typically outweighs any protocol-level performance comparison, which is why the first question about any realtime feature should be whether the client actually needs to send on the same channel.

**Prove it — interview questions**

1. **[Basic] What is a server-sent event stream?**

   <details><summary>Model answer</summary>

   An ordinary HTTP response that never ends. The client makes a normal GET, the server replies with an event-stream content type and writes events as they occur, flushing each one. Because it is plain HTTP, authentication, cookies, proxies and observability tooling all behave exactly as they do for any other request — and in the browser, the EventSource API handles reconnection and resume without any application code.

   </details>

2. **[Basic] When would you choose SSE over WebSockets?**

   <details><summary>Model answer</summary>

   Whenever the client does not need to send on the same channel. Notifications, live figures, feeds, progress updates and streamed responses are all one-way, and for those SSE gives the same push with far less to build: reconnection, backoff and resume-from-last-event are part of the protocol. Client-to-server messages can travel as ordinary POST requests. WebSockets earn their complexity only when traffic is genuinely bidirectional and frequent in both directions.

   </details>

3. **[Senior] How does resume work, and what does it require of the server?**

   <details><summary>Model answer</summary>

   Each event carries an id, and the browser remembers the last one it received. On reconnection it automatically sends that value in a Last-Event-ID header, and a server that honours it replays everything after that point — exact recovery with no client code. What it requires is that the server still has those events, which means retaining recent history per stream. That retention window bounds how long a disconnection can be before exact resume becomes impossible, and beyond it the server must send a resync instruction so the client refetches full state rather than continuing with a silent gap.

   </details>

4. **[Senior] What are the two environment problems that catch people out?**

   <details><summary>Model answer</summary>

   Response buffering and connection limits. Any proxy, gateway or framework that buffers the response will hold events until its buffer fills, so the client receives nothing for a long time and then everything at once — the stream appears to work in development and batches in production behind different infrastructure, and it does not look like a streaming bug. Separately, HTTP/1.1 browsers allow about six connections per origin and a stream permanently occupies one, so opening the application in several tabs eventually hangs everything including ordinary API calls. Serving over HTTP/2 removes the second problem entirely; the first requires explicitly disabling buffering at every hop and flushing after each event.

   </details>

5. **[Staff] Design a live notification system for a web application.**

   <details><summary>Model answer</summary>

   SSE over HTTP/2, because the traffic is entirely server-to-client and HTTP/2 removes the per-origin connection limit that otherwise breaks multi-tab use. The endpoint is an ordinary authenticated GET, so existing session handling applies unchanged — no separate auth path as an upgrade would need. The server subscribes the user to their channels, writes each event with a monotonic id, flushes immediately, and sends a comment keepalive every twenty seconds to stay inside proxy idle timeouts. It retains about five minutes of events per user, which bounds memory and defines the resume window: a reconnect within that window replays exactly from Last-Event-ID with no client code, and one beyond it receives a resync event telling the client to refetch state. Client-to-server actions — marking a notification read — go as normal POST requests, matching the asymmetry of the traffic. The whole client implementation is constructing an EventSource and attaching listeners, which is the strongest argument for the design: there is almost nothing to get wrong.

   </details>

6. **[Principal] Why is SSE underused relative to how well it fits?**

   <details><summary>Model answer</summary>

   Mostly visibility. WebSockets are the well-known answer to realtime, so they get chosen before the question of direction is examined, and once chosen they bring a bidirectional channel used in one direction plus several hundred lines of client reconnection logic that the protocol would have provided. The consequences are not dramatic but they accumulate: the resume path is the piece teams most often implement incompletely, so brief disconnections silently lose events; connection state forces a registry and deployment choreography that a one-way stream over HTTP largely avoids; and observability tooling that understands requests stops working on a message stream. The corrective I would push for is procedural rather than technical — make the first question about any realtime requirement whether the client needs to send on the same channel, because the honest answer is usually no, and SSE with ordinary POST requests then matches the shape of the traffic with substantially less machinery to own and less to get subtly wrong.

   </details>

---

### Long polling

*Hold a request open until data arrives or a timeout expires, giving push latency over plain request-response — the compatibility fallback, not the first choice.*

**Flow:** `Client request` → `Server holds` → `Data or timeout` → `Response` → `Immediate re-request`

> **The 30-second version**  
> Hold the request until there is something to say, then respond and let the client immediately ask again — near-push latency at slow-polling cost, as a compatibility fallback beneath SSE.

**The problem**

Before persistent connections were widely supported, and still today in environments that block them, the only available primitive is request-response. Polling on a fixed interval forces a bad choice: poll every second and generate a vast number of empty requests, or poll every thirty and accept thirty seconds of latency.

The waste is severe. A client polling once per second for an event that occurs twice an hour produces over eighty-six thousand requests per day, of which two return anything — each carrying connection setup, authentication, headers and a response.

> **Hold the request instead of repeating it**  
> If the server does not answer immediately but waits until there is something to say, the client gets the data as soon as it exists rather than at the next poll. The request-response shape is preserved — so every proxy, gateway and client library works — while latency becomes push-like. The cost is a held connection and a server that must park requests without consuming a thread each.

**Mental model**

The client issues a request; the server does not respond until either data is available or a timeout is reached; the client immediately issues another. From the client's perspective it is ordinary HTTP; from the system's perspective there is almost always an outstanding request per client.

1. **Request** — An ordinary HTTP request carrying the client's last known position.
2. **Park** — The server registers interest and holds the request without replying, consuming no thread if done correctly.
3. **Wake** — Data arrives, or the timeout fires, and the response is written.
4. **Re-request** — The client immediately issues the next request, closing the window.
5. **Cursor** — The position the client reports, which is what prevents the gap between responses from losing events.

> **The gap between responses is where events are lost**  
> Between the server responding and the client's next request arriving, there is a window — small, but real — in which new events can occur. A design that only delivers events happening while a request is parked loses anything falling in that gap. The fix is a cursor: the client reports what it last saw, and the server delivers everything after it, whether it arrived during the wait or between requests.

**How it works**

**The cycle, and where it goes wrong**

```text
CLIENT                          SERVER

GET /poll?since=1042  ------->  no new data yet
                                PARK the request
                                (register interest,
                                 release the thread)
                  ...waiting...
                                event 1043 occurs
                <-------------  respond with 1043
GET /poll?since=1043  ------->  ...
      ^
      |
THE GAP: between receiving the response and the next
request arriving, events can occur.
-> the since cursor is what covers it: the server returns
   everything after 1043, whether it happened during
   the wait or in the gap.
-> without a cursor, those events are simply lost.

TIMEOUT PATH
  no data for 30 s -> respond with an empty result
  client immediately re-requests
  -> the timeout exists to beat intermediaries that
     kill quiet connections, and to let the client
     detect a dead server

THREAD MODEL - THE CRITICAL DETAIL
  blocking:     one thread per parked request
                10,000 clients = 10,000 threads
                -> memory exhaustion, scheduler thrash
  async:        parked requests hold no thread
                10,000 clients = 10,000 small objects
                -> the only viable model
```

1. **Always carry a cursor** — Without a position, the gap between responses loses events, and the loss is silent.
2. **Park requests asynchronously** — A thread per waiting client makes long polling more expensive than the polling it replaced.
3. **Time out below intermediary limits** — Proxies and load balancers kill quiet requests; responding first with an empty result keeps the cycle predictable.
4. **Batch events in one response** — If several occurred, return them together rather than one per cycle, or a burst becomes a request storm.
5. **Add jitter to client retries** — Otherwise clients synchronise after any server restart and arrive together.
6. **Bound how far back a cursor may reach** — Old cursors require retained history; beyond the window the client must resynchronise.

**Cost compared with the alternatives**

```text
ONE CLIENT, ONE EVENT PER HOUR, 8-HOUR SESSION

SHORT POLL every 1 s
  28,800 requests, 28,792 empty
  latency: up to 1 s
  -> enormous waste

SHORT POLL every 30 s
  960 requests
  latency: up to 30 s
  -> tolerable cost, poor latency

LONG POLL, 30 s timeout
  ~960 requests (mostly timeouts) + 8 data responses
  latency: milliseconds
  -> SAME request count as 30 s short polling,
     but near-zero latency
  -> this is the whole argument for it

SSE / WEBSOCKET
  1 connection for 8 hours
  latency: milliseconds
  -> strictly better, IF the environment allows it

SO WHEN IS LONG POLLING RIGHT?
  when persistent connections are unavailable or
  unreliable: old clients, restrictive proxies,
  infrastructure that terminates streams
  -> it is a COMPATIBILITY choice, not a performance one
```

> **Long polling still holds a connection — it just pretends not to**  
> The appeal is that it looks like ordinary HTTP, but a parked request occupies a socket, a file descriptor and intermediary state exactly as a streaming response would. The per-origin browser connection limit applies too. What it genuinely avoids is protocol upgrades and unusual content types; what it does not avoid is the resource cost of many simultaneously open connections.

**Worked example**

A notification channel that must work behind restrictive corporate proxies.

**Design**

```text
ENDPOINT  GET /poll?since={cursor}

SERVER
  if events exist after the since cursor:
      respond immediately with all of them
  else:
      park the request asynchronously
      wake on a new event, or after 25 s
  retain 10 minutes of events per user
    -> defines how stale a cursor may be

TIMEOUT 25 s
  chosen below the 30 s idle limit of the proxies in
  question -> the server always responds first, so
  the client never sees an ambiguous connection reset

CLIENT
  loop:
    response = GET /poll?since=cursor
    apply events; cursor = last id
    immediately re-request
  on error: backoff with jitter

BURST HANDLING
  50 events arrive at once
  -> ONE response containing all 50
  -> not 50 cycles; otherwise a burst becomes a
     request storm against the same server

STALE CURSOR
  cursor older than retention
  -> respond with a resync directive
  -> client refetches state and resumes

CAPACITY
  10,000 concurrent clients
  parked asynchronously: ~10,000 lightweight objects
  with a thread-per-request model this would be
  10,000 threads - which is why the async model is
  not optional
```

| Metric | Value | Note |
|---|---|---|
| Timeout | 25 s | under proxy limit |
| Requests | ~960 / 8 h | same as 30 s polling |
| Latency | milliseconds | **the whole point** |
| Parking | async | no thread per client |

> **Long polling buys latency at the request cost of slow polling**  
> Compared with polling every thirty seconds, long polling with a thirty-second timeout generates roughly the same number of requests but delivers events in milliseconds instead of up to thirty seconds. That is the entire case for it: identical cost, vastly better latency. Against SSE or WebSockets it loses on every axis, which is why it belongs in the design only when persistent connections are not dependable.

**When to use it**

- **Environments that block or break persistent connections**, such as some corporate proxies.
- **Legacy clients** without EventSource or WebSocket support.
- **As a fallback tier** beneath SSE or WebSockets for the minority of clients that cannot use them.
- **Low-frequency events** where the cycle overhead is amortised over long waits.
- **When ordinary request-response semantics** are required by surrounding infrastructure.

**When to avoid it**

- **Do not use it as a first choice** where SSE or WebSockets work — they are better on every axis.
- **Do not use it for high-frequency events**, where each event costs a full request cycle.
- **Do not implement it with a thread per parked request**, which is worse than the polling it replaces.
- **Do not use it without a cursor**, which silently loses events in the gap between requests.
- **Do not use it for bidirectional traffic**, where the client also sends frequently.

**Advantages**

- **Works everywhere**, since it is plain request-response.
- **Near-push latency** at the request cost of infrequent polling.
- **No protocol upgrade**, so hostile intermediaries do not interfere.
- **Ordinary auth and observability**, as every cycle is a normal request.
- **Simple client logic**, needing no streaming parser.

**Disadvantages**

- **Still holds a connection per client**, with the same resource cost as streaming.
- **A full request cycle per event**, which is wasteful at high rates.
- **Requires asynchronous request parking**, which not every framework makes easy.
- **Events can be lost in the gap** without a cursor.
- **Higher latency and overhead than SSE** for the same one-way delivery.
- **Bursts turn into request storms** unless responses batch events.

**Trade-offs**

**Polling strategies compared**

| Strategy | Latency | Requests | Best for |
|---|---|---|---|
| Short poll, 1 s | Up to 1 s | Very high | Almost never |
| Short poll, 30 s | Up to 30 s | Low | Tolerant, infrequent updates |
| Long poll, 30 s timeout | Milliseconds | Same as 30 s short poll | Compatibility-constrained push |
| Server-sent events | Milliseconds | One connection | One-way push where supported |
| WebSocket | Milliseconds | One connection | Bidirectional traffic |

The table makes the positioning clear: long polling dominates short polling outright, and is dominated by SSE outright. It is therefore a fallback tier rather than a design choice — worth implementing only for the clients that cannot use something better.

**How it fails**

**Long polling failures**

| Failure | Cause | Fix |
|---|---|---|
| Events lost intermittently | No cursor; the gap between requests | Client reports last seen position |
| Server exhausts threads | Blocking model, one thread per parked request | Asynchronous parking |
| Connections reset unpredictably | Timeout longer than an intermediary's idle limit | Time out below the shortest limit on the path |
| Request storm during a burst | One event per response cycle | Batch all pending events into one response |
| Synchronised client stampede | All clients re-request together after a restart | Jittered backoff on errors |
| Client stuck after a long outage | Cursor older than retained history | Resync directive; client refetches state |
| Duplicate event delivery | Cursor not advanced before re-request | Advance the cursor from the response before re-requesting |

**Limits**

> **Typical parameters**
>
> - **Timeout**: 20–30 s, chosen below the shortest idle limit on the network path.
> - **Request count**: roughly one per timeout interval per client, plus one per event.
> - **Retention**: minutes of history, bounding how stale a cursor may be.
> - **Connections**: one held per client, with the same resource profile as a streaming connection.
> - **Threading**: asynchronous parking is required; thread-per-request collapses in the thousands.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Long polling | Compatibility-constrained push | Request cycle per event |
| Server-sent events | One-way push | Needs streaming support on the path |
| WebSocket | Bidirectional | Upgrade may be blocked |
| Short polling | Infrequent, tolerant updates | Latency or waste |
| Webhooks | Server-to-server delivery | Requires a reachable endpoint |
| Managed realtime service | Avoiding the whole problem | Cost; vendor dependency |

For server-to-server delivery the equivalent question is answered by webhooks rather than any polling scheme, since the receiver can expose an endpoint — polling exists because browsers and mobile clients generally cannot.

**In real systems**

- **Fallback tiers in realtime libraries** commonly degrade from WebSocket to SSE to long polling depending on what the environment permits.
- **Corporate network environments** remain the main reason long polling is still implemented, since some proxies terminate streaming responses.
- **Asynchronous request parking** is standard in modern server frameworks precisely because the thread-per-request model made long polling unaffordable.
- **Cursor-based delivery** is universal, because the gap between responses would otherwise lose events silently.
- **Batched responses during bursts** prevent a spike of events from becoming a spike of requests against the same server.

**Common mistakes**

- **Choosing it over SSE** without an environment constraint requiring it.
- **No cursor**, losing events in the gap between requests.
- **Thread per parked request**, exhausting the server in the thousands.
- **Timeout above the proxy idle limit**, producing unexplained resets.
- **One event per response**, turning bursts into request storms.
- **No jitter on retry**, synchronising clients after every restart.
- **Unbounded cursor age**, with no resync path for long absences.

**The staff-level view**

Long polling is a compatibility mechanism whose main risk is being chosen for the wrong reason.

- **Position it as a fallback, not a design.** It is strictly worse than SSE wherever SSE works, so its presence should be justified by specific clients or networks that cannot support streaming.
- **Insist on cursors.** Without a reported position, events falling between a response and the next request vanish, and the loss produces no error anywhere.
- **Check the threading model before adopting it.** A blocking implementation makes long polling more expensive than the frequent polling it was meant to replace, which inverts the entire rationale.
- **Set the timeout from the network path.** Responding just before the shortest intermediary idle limit turns an ambiguous connection reset into a clean, predictable cycle.
- **Batch during bursts.** One event per cycle means a spike of activity becomes a spike of requests, concentrated on exactly the server already under load.

**Go deeper**

Long polling holds a client's request open until data arrives or a timeout expires, then responds; the client immediately issues another. The shape remains request-response, so every proxy, gateway and client library works unchanged, while latency becomes push-like. Against polling every thirty seconds it generates a similar number of requests and reduces latency from tens of seconds to milliseconds — that comparison is the entire argument for it.

Three implementation details decide whether it works. Requests must be parked asynchronously, because a thread per waiting client makes it more expensive than the polling it replaced. Clients must carry a cursor, because events occurring between a response and the next request would otherwise be lost silently. And timeouts must sit below the shortest idle limit on the network path, so the server always responds first rather than the client interpreting an ambiguous reset.

Its position is as a fallback. Server-sent events beat it on latency, overhead and client simplicity wherever streaming responses survive the path, so long polling should appear only for clients and networks that cannot support streaming. The risk is adopting it because it superficially looks simpler — it is not, since cursors, async parking, timeout tuning and burst batching are all places to get it subtly wrong.

Long polling adapts request-response to deliver events promptly: the server withholds its reply until it has something to say.

**The comparison that justifies it.** Frequent short polling generates enormous numbers of empty requests to achieve modest latency; infrequent polling is cheap and slow. Long polling with a thirty-second timeout produces roughly the same request volume as thirty-second short polling — mostly empty timeout responses — while delivering events in milliseconds. It therefore dominates short polling outright, and that is the only comparison in which it wins.

**Asynchronous parking is not optional.** A held request that occupies a thread makes the technique more expensive than what it replaced: ten thousand waiting clients become ten thousand stacks, exhausting memory and the scheduler well before that count. Registering interest, releasing the thread, and resuming on an event or timeout reduces each waiting client to a small object. Teams adopting long polling on a blocking framework typically discover this at a few thousand concurrent users, having inverted the rationale for the change.

**Cursors close a gap that is otherwise invisible.** Between the server writing a response and the client's next request arriving, events can occur. A design that delivers only what happens while a request is parked loses those, silently and without error, at a rate proportional to event frequency. Having the client report its last seen position means the server returns everything after it regardless of timing, which also makes recovery from a failed request and suppression of duplicates fall out naturally.

**Timeouts are a network-path decision.** Proxies and load balancers terminate quiet requests on their own schedules, and a server timeout longer than the shortest such limit means clients experience connection resets rather than clean empty responses — ambiguous events that are hard to distinguish from real failures. Choosing a timeout comfortably below that limit makes the cycle deterministic: the server always speaks first.

**Bursts must be batched.** If each response carries one event, a spike of fifty events becomes fifty complete request cycles from every affected client, directed at the server already handling the spike. Returning all pending events in a single response converts a self-amplifying load pattern into a single exchange, which matters most precisely when the system is busiest.

**Its honest position is a fallback tier.** Server-sent events provide the same one-way delivery with one connection instead of a cycle per event, with browser-native reconnection and resume, and with less to implement incorrectly. Long polling remains worth having only because some corporate proxies, intermediaries and older clients still terminate or buffer streaming responses, and for those users the alternative is not SSE but poor latency. The trap is that it looks like ordinary HTTP and therefore feels simpler than it is — cursors, asynchronous parking, path-aware timeouts and burst batching are four distinct opportunities for subtle error, every one of which disappears when a streaming transport is viable.

**Prove it — interview questions**

1. **[Basic] How does long polling differ from ordinary polling?**

   <details><summary>Model answer</summary>

   The server does not answer immediately when there is nothing to say — it holds the request open until data arrives or a timeout fires. The client then immediately issues another request. The shape is still request-response, so all existing infrastructure works, but the client learns about an event as soon as it happens rather than at the next scheduled poll. Compared with polling every thirty seconds, a thirty-second long poll generates roughly the same number of requests and reduces latency from up to thirty seconds to milliseconds.

   </details>

2. **[Basic] When is long polling the right choice?**

   <details><summary>Model answer</summary>

   When persistent connections are unavailable or unreliable — older clients, or networks and proxies that terminate streaming responses. It is a compatibility mechanism rather than a performance one: wherever server-sent events work, they are better on every axis. The reasonable pattern is a fallback tier, using SSE or WebSockets by default and long polling only for the clients or environments that cannot support them.

   </details>

3. **[Senior] Why is a cursor essential?**

   <details><summary>Model answer</summary>

   Because there is a window between the server sending a response and the client's next request arriving, and events occurring in that window would otherwise be missed. If the server only delivers what happens while a request is parked, anything falling in the gap is lost silently — no error, just missing data. Having the client report the position it last saw means the server returns everything after that point regardless of when it occurred, which closes the gap. The same cursor also makes duplicate suppression and recovery after a failed request straightforward.

   </details>

4. **[Senior] Why does the threading model matter so much?**

   <details><summary>Model answer</summary>

   Because a parked request that occupies a thread makes long polling more expensive than the polling it replaced. Ten thousand waiting clients would mean ten thousand threads, each with its own stack, causing memory exhaustion and scheduler pressure long before that. Asynchronous parking — registering interest and releasing the thread, resuming when an event or timeout occurs — reduces each waiting client to a small object, which is the only model in which the technique makes sense. Adopting long polling on a blocking framework inverts its entire rationale.

   </details>

5. **[Staff] Design a notification channel that must work behind restrictive corporate proxies.**

   <details><summary>Model answer</summary>

   Long polling, since the constraint is that streaming responses get terminated. The endpoint takes a cursor and, if events exist after it, responds immediately with all of them batched — batching matters because otherwise a burst of fifty events becomes fifty request cycles aimed at the server already handling the burst. If nothing is pending, the request parks asynchronously and wakes on an event or after about twenty-five seconds, chosen deliberately below the proxies' thirty-second idle limit so the server always responds first and the client never has to interpret an ambiguous reset. The server retains around ten minutes of events per user, which bounds how stale a cursor may be; older cursors get a resync directive so the client refetches state rather than continuing with an invisible gap. Client-side, the loop advances the cursor from each response before re-requesting, and errors back off with jitter so a server restart does not bring every client back simultaneously. I would still serve SSE to clients whose networks permit it and treat this as the fallback tier, because it is strictly worse wherever streaming works.

   </details>

6. **[Principal] Long polling is widely considered obsolete. Is it?**

   <details><summary>Model answer</summary>

   As a first choice, yes — it is dominated by server-sent events on latency, overhead and client simplicity wherever streaming responses survive the network path. But the qualification is doing real work in that sentence. Restrictive corporate proxies, some mobile network intermediaries and older embedded clients still terminate or buffer streams, and for those users the alternative to long polling is not SSE but thirty-second latency or nothing at all. So the mature position is that it belongs in the fallback tier of a realtime stack rather than in its design, and the risk to watch is teams adopting it for the wrong reason: because it looks like ordinary HTTP and therefore feels simpler. It is not simpler — it needs cursors to avoid silently losing events, asynchronous parking to avoid costing more than the polling it replaced, timeouts tuned to the network path, and batching so bursts do not become storms. Every one of those is a place to get it subtly wrong, and all of them are avoided by using a streaming transport where one is available.

   </details>

---

### Presence and heartbeats

*Infer who is online from periodic liveness signals with expiring state — accepting that presence is always a guess with a bounded staleness window.*

**Flow:** `Client heartbeat` → `Presence store with TTL` → `Expiry` → `Status change event` → `Fan-out`

> **The 30-second version**  
> Treat presence as a short lease refreshed by heartbeats and expiring on its own — self-healing across crashes, always stale by up to the TTL, and usually dominated by fan-out cost.

**The problem**

A chat application shows which contacts are online, a collaborative document shows who is viewing it, a game lobby shows available players. In each case the system must answer a question it cannot directly observe: is this user still there?

The difficulty is that absence produces no signal. A user who closes their laptop lid, loses network coverage, or has their process killed sends nothing — and a TCP connection may remain apparently open for minutes afterwards. The only way to distinguish present from gone is to require continuous evidence of presence and treat its absence as departure.

> **Presence is expiring state, not stored state**  
> The correct model is a lease: a client's presence record has a short expiry, refreshed by a periodic heartbeat, and vanishes on its own if the heartbeat stops. This makes crashes self-healing, since a dead client cannot refresh, and it makes explicit cleanup unnecessary — which matters because the failure cases that most need cleanup are exactly the ones where no cleanup code runs.

**Mental model**

Each online user holds a short-lived lease in a presence store. Heartbeats renew it. If renewal stops, the lease expires and the user is considered offline. Every part of the system reads presence from that store rather than from connection state.

1. **Heartbeat** — A periodic signal from the client, or from the server on the client's behalf, proving liveness.
2. **Lease** — A presence entry with a TTL a few multiples of the heartbeat interval.
3. **Expiry** — The absence of renewal, which is the only reliable evidence of departure.
4. **Transition** — Going from present to absent or back, which is what observers actually care about.
5. **Fan-out** — Notifying interested parties of transitions, which is usually the expensive part.

> **Presence is always stale, and the staleness is the TTL**  
> There is no configuration in which presence is accurate. A user who vanishes is shown as online until their lease expires, so the displayed state lags reality by up to the TTL. Shortening the TTL reduces the lag and increases heartbeat traffic and false departures on flaky networks. This is not an implementation weakness to be fixed but a property to be chosen, and the product must tolerate the window.

**How it works**

**The lease cycle and its parameters**

```text
CLIENT                      PRESENCE STORE

heartbeat every 20 s  --->  SET user:123 online
                            EXPIRE 60 s

...20 s...            --->  refresh, EXPIRE 60 s
...20 s...            --->  refresh, EXPIRE 60 s

client disappears (crash, lid closed, no signal)
                            no refresh arrives
...60 s...                  key EXPIRES
                            -> user is offline

CHOOSING THE NUMBERS
  TTL = 3x heartbeat interval
  -> survives two lost heartbeats before declaring
     departure
  -> on mobile networks, single lost heartbeats are
     routine; declaring offline on one produces
     constant flapping

THE TRADE
  heartbeat 5 s / TTL 15 s
    detection within 15 s
    12 heartbeats per minute per user
    flaps on brief mobile interruptions
  heartbeat 30 s / TTL 90 s
    detection within 90 s
    2 heartbeats per minute per user
    stable, but "online" can be 90 s wrong

COST AT SCALE
  1,000,000 online users, 20 s heartbeat
  = 50,000 writes per second to the presence store
  -> presence is a WRITE-HEAVY workload, which is
     unusual and often unanticipated
```

1. **Set the TTL to several heartbeat intervals** — One lost heartbeat is normal on mobile networks; declaring departure on it produces constant flapping.
2. **Use a store with native expiry** — Expiry must happen without anyone running cleanup code, because the cases needing cleanup are the ones where nothing runs.
3. **Publish transitions, not state** — Observers care about someone arriving or leaving; republishing unchanged state multiplies traffic for no information.
4. **Debounce brief absences** — A short gap followed by a return should not surface as a departure and an arrival, which is visual noise and wasted fan-out.
5. **Bound the fan-out** — Notifying every contact of every transition is quadratic in the worst case; large groups need aggregate counts rather than per-member events.
6. **Consider coarse statuses** — Exact online state is rarely needed; active, idle and offline convey more while tolerating a longer TTL.

**Fan-out is usually the real cost**

```text
NAIVE: notify every contact of every transition

  user with 500 contacts goes online
    -> 500 notifications
  1,000,000 users, average 200 contacts,
  each transitioning 10 times a day
    -> 2,000,000,000 notifications per day
    -> presence traffic exceeds the application's
       actual messaging traffic

WHY IT EXPLODES
  transitions are frequent (every network blip,
  every tab close)
  contact lists are large
  cost = users x contacts x transitions

MITIGATIONS
  1  notify only ACTIVE observers
     someone must be looking at the contact list for
     the update to matter
     -> collapses fan-out to who is actually watching
  2  debounce
     a 5-second absence should not produce two events
  3  aggregate for large groups
     "1,248 online" instead of 1,248 individual states
  4  pull instead of push
     client asks for presence of the 20 contacts
     currently on screen
     -> bounded by screen size, not contact count

THE PULL MODEL IS UNDERRATED: presence is only
interesting for what the user can see.
```

> **Presence is a privacy surface, not just a feature**  
> Online status reveals behavioural patterns: when someone sleeps, works, or is at their computer. Aggregated over time it reveals a great deal about a person's life, and it is often exposed to anyone who can add them as a contact. Treating presence as ordinary feature data rather than as personal information — with visibility controls and an option to disable it — is a recurring and avoidable mistake.

**Worked example**

A messaging application's presence system, sized and bounded.

**Design with costs**

```text
HEARTBEAT
  clients send every 20 s over the existing connection
  -> no separate transport; it doubles as connection
     liveness detection
  server writes presence with a 60 s TTL

STORE
  in-memory key-value store with native expiry
  key: presence:{userId} -> {status, lastSeen, device}
  1,000,000 online users at 20 s = 50,000 writes/s
  -> sized as a write-heavy workload from the start

STATUSES
  online  - heartbeat within TTL, recent interaction
  idle    - heartbeat present, no interaction for 5 min
  offline - lease expired
  -> idle carries more information than a binary state
     and tolerates a longer TTL

DELIVERY - PULL, NOT PUSH
  client requests presence for the contacts currently
  visible (typically 20-50)
  refreshes while that view is open
  -> bounded by screen, not by contact list size
  -> avoids the billions of notifications a push model
     would generate

TRANSITIONS
  debounce 10 s: a brief drop does not surface
  only genuine changes are emitted

PRIVACY
  per-user visibility setting: everyone / contacts /
  nobody
  -> "last seen" is behavioural data and treated as such

RESULT
  presence lag: up to 60 s
  heartbeat cost: 50,000 writes/s
  fan-out cost: bounded by visible contacts
```

| Metric | Value | Note |
|---|---|---|
| Heartbeat | 20 s | over existing socket |
| TTL | 60 s | survives 2 losses |
| Writes | 50,000/s | **write-heavy** |
| Fan-out | pull, visible only | bounded |

> **Pulling presence for what is visible collapses the hardest cost**  
> Push-based presence scales as users times contacts times transitions, which reaches billions of events daily for a large messaging application. But presence only matters for contacts the user can actually see — typically a few dozen on screen. Fetching presence for the visible set, refreshed while that view is open, bounds the cost by screen size rather than social graph size, and users cannot perceive the difference.

**When to use it**

- **Showing who is online** in chat, collaboration or gaming.
- **Detecting dead connections**, where heartbeats serve double duty as liveness checks.
- **Collaborative editing**, showing active participants in a document.
- **Service discovery and membership**, where the same lease pattern tracks live instances.
- **Any state that should disappear when its owner stops maintaining it.**

**When to avoid it**

- **Do not use presence where accuracy is required**, since it is always stale by up to the TTL.
- **Do not push every transition to every contact**, which grows as users times contacts times transitions.
- **Do not set the TTL near the heartbeat interval**, which makes flapping constant on mobile networks.
- **Do not rely on explicit offline signals**, since crashes and network loss produce none.
- **Do not expose presence without visibility controls**, as it reveals behavioural patterns.

**Advantages**

- **Self-healing**, since dead clients cannot refresh and their state expires without cleanup.
- **Simple model** — a lease with a TTL, refreshed periodically.
- **Doubles as connection liveness**, detecting sockets that are open but dead.
- **Tunable**, trading detection latency against heartbeat cost explicitly.
- **Generalises**, as the same pattern underlies service membership and leader leases.

**Disadvantages**

- **Always stale** by up to the TTL, with no configuration that removes the lag.
- **Write-heavy at scale**, which is an unusual and often unplanned load profile.
- **Fan-out can dominate**, exceeding the application's real traffic.
- **Flapping on unreliable networks**, requiring debouncing.
- **Privacy-sensitive**, exposing behavioural patterns unless controlled.
- **Continuous cost even when idle**, since heartbeats never stop.

**Trade-offs**

**Heartbeat interval trade-offs**

| Interval / TTL | Detection lag | Write load | Best for |
|---|---|---|---|
| 5 s / 15 s | Up to 15 s | Very high | Small user counts; games |
| 20 s / 60 s | Up to 60 s | Moderate | General messaging |
| 60 s / 180 s | Up to 3 min | Low | Large scale; coarse status |
| Push transitions | Immediate for observers | Fan-out heavy | Small contact graphs |
| Pull for visible set | Refresh interval | Bounded by screen | Large contact graphs |

The interval choice is genuinely a product decision disguised as a technical one: users rarely notice whether offline detection takes twenty seconds or ninety, but they do notice a contact flickering between states on a train — which argues for longer TTLs than intuition suggests.

**How it fails**

**Presence failure modes**

| Failure | Cause | Fix |
|---|---|---|
| Users shown online long after leaving | TTL too long, or no expiry at all | Short TTL with native expiry in the store |
| Status flickers constantly | TTL too close to the heartbeat interval | TTL of several intervals; debounce transitions |
| Presence traffic exceeds message traffic | Push fan-out across large contact lists | Pull for the visible set; aggregate for groups |
| Ghost users after a server crash | Presence cleaned only on graceful disconnect | Expiry as the sole mechanism, never explicit cleanup |
| Presence store saturated | Write load underestimated | Size for writes; shard; lengthen the interval |
| Battery drain on mobile | Frequent heartbeats waking the radio | Longer intervals; align with existing traffic |
| Behavioural data exposed | Presence and last-seen public by default | Per-user visibility controls |

**Limits**

> **Sizing figures**
>
> - **Heartbeat**: 15–30 s is typical; shorter costs battery and writes, longer increases lag.
> - **TTL**: roughly 3× the heartbeat interval, so two consecutive losses do not cause a false departure.
> - **Write load**: online users divided by heartbeat interval — a million users at 20 s is 50,000 writes per second.
> - **Fan-out**: push scales as users × contacts × transitions, which is why pull for visible contacts is preferred at scale.
> - **Staleness**: bounded by the TTL and never zero.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Heartbeat leases with TTL | General presence | Stale by up to the TTL |
| Connection state as presence | Single-process systems | Breaks with multiple processes; stale on crash |
| Explicit online/offline messages | Controlled clients | Crashes and network loss send nothing |
| Last-activity timestamps | Coarse recency display | Not realtime |
| Pull presence for visible set | Large contact graphs | Refresh-interval latency |
| Managed presence service | Avoiding the write load | Cost; vendor dependency |

Using raw connection state as presence is the tempting shortcut and fails in exactly the important cases: a crashed server's connections are gone with no notification, a TCP socket can stay open long after the client is unreachable, and multiple processes each know only their own connections.

**In real systems**

- **Messaging applications** show coarse statuses like online, idle and last-seen rather than exact presence, because the imprecision is unavoidable and the coarser display is more honest.
- **In-memory stores with native key expiry** are the standard presence backend, since expiry must occur without any process running cleanup.
- **Service registries** use exactly the same lease-and-heartbeat pattern to track live instances, which is why an instance that crashes disappears on its own.
- **Pull-based presence for visible contacts** is common at large scale, because push fan-out across big social graphs exceeds the application's real traffic.
- **Per-user presence visibility settings** are standard in consumer messaging because online history reveals daily routines.

**Common mistakes**

- **Relying on explicit disconnect messages**, which crashes never send.
- **TTL too close to the heartbeat interval**, causing constant flapping.
- **Pushing every transition to every contact**, generating more traffic than messaging.
- **Underestimating the write rate**, saturating the presence store.
- **Using connection state as presence**, which is wrong across processes and after crashes.
- **Frequent heartbeats on mobile**, draining battery by waking the radio.
- **Exposing presence by default**, publishing behavioural patterns without consent.

**The staff-level view**

Presence looks like a small feature and is frequently one of the heaviest write workloads in a messaging system.

- **Size the write load before building it.** Online users divided by heartbeat interval gives the write rate, and it is routinely an order of magnitude beyond what teams expect from a status indicator.
- **Make expiry the only cleanup mechanism.** The scenarios requiring cleanup — crashes, network loss, killed processes — are precisely the ones in which no cleanup code executes.
- **Choose pull over push for large graphs.** Push scales with the social graph; pull scales with what is on screen, and users cannot tell the difference.
- **Set the TTL for the worst network, not the best.** Flapping on a train is far more visible to users than a slightly delayed offline transition.
- **Treat presence as personal data.** Online patterns reveal sleep and work schedules, so visibility controls and an off switch belong in the initial design rather than a later privacy review.

**Go deeper**

Presence cannot be observed directly, because a client that crashes or loses coverage sends nothing and its connection may stay apparently open for minutes. The working model is a lease: a short-lived presence entry refreshed by periodic heartbeats, which expires by itself when the heartbeats stop. That makes it self-healing across exactly the failures where no cleanup code would run.

Two parameters define the behaviour. The heartbeat interval sets the write load — online users divided by interval, which for a million users at twenty seconds is fifty thousand writes per second, an unusually heavy profile for what appears to be a minor feature. The TTL, at roughly three intervals, sets both the staleness window and the resistance to flapping; single lost heartbeats are routine on mobile networks, so a tight TTL makes contacts flicker.

At scale the dominant cost is usually fan-out, which grows as users times contacts times transitions and can exceed the application's real messaging traffic. The effective response is to notice that presence only matters for contacts currently on screen, and to pull for that visible set rather than pushing every transition to every contact. Separately, presence is behavioural data — it reveals sleep and work patterns — and deserves visibility controls from the start.

Presence answers a question the system cannot observe: whether a user is still there. Departure generates no signal, so presence must be inferred from continuous evidence of its opposite.

**Leases rather than stored state.** Each online user holds a presence entry with a short expiry, renewed by heartbeats. When the heartbeats stop — crash, lid closed, coverage lost, process killed — the entry expires unaided. This is the essential property, because every scenario that most requires cleanup is one in which no cleanup code executes; relying on explicit disconnect messages means ghost users precisely in the cases that matter. It also means presence survives server crashes, since the state lives in a shared store rather than in a process's connection table.

**The two parameters are a product decision.** The heartbeat interval determines write load directly: online users divided by interval, which makes presence one of the heaviest continuous write workloads in a large messaging system — a fact that routinely surprises teams treating it as a minor feature. The TTL, conventionally around three intervals, sets both the staleness window and the tolerance for lost heartbeats. Mobile networks drop individual heartbeats routinely, so a TTL close to the interval produces contacts flickering between states, which users notice far more than a slightly delayed offline transition. This argues for longer TTLs than intuition suggests.

**Staleness is inherent and must be accepted.** There is no configuration in which presence is accurate; a departed user appears online until their lease expires. Any requirement expressed as accurate online status is asking for something the mechanism cannot provide, and the honest response is to name the window and, often, to display coarser statuses — active, idle, last seen — which convey more useful information while tolerating the imprecision rather than pretending it away.

**Fan-out usually dominates the cost.** Pushing every transition to every contact scales as the product of users, contact list size and transition frequency, which for a large social graph produces billions of events daily and can exceed the application's genuine messaging traffic. The reframing that collapses this is that presence matters only for contacts the user can currently see: fetching presence for the visible set and refreshing while that view is open bounds cost by screen size rather than graph size, and is perceptually identical. Debouncing brief absences and aggregating large groups into counts remove most of what remains.

**The heartbeat does double duty.** Carried over an existing realtime connection, it simultaneously refreshes presence and proves the connection is alive — which matters because TCP will hold a socket open long after the peer is unreachable. Sharing one signal for both purposes avoids a second transport and a second set of intervals to reason about, and aligns naturally with mobile radio behaviour, where fewer separate wake-ups means less battery drain.

**Presence is personal information.** Online and last-seen history reveals when someone sleeps, works and is at their desk, and aggregated over weeks it describes a person's routine in considerable detail. It is commonly exposed to anyone able to add the user as a contact, and it is almost always designed as a feature rather than as behavioural data. Visibility controls and the ability to disable it entirely are inexpensive at design time and awkward to introduce once users depend on seeing each other — which makes this the point worth raising first, before the write load and the fan-out that will otherwise dominate the discussion.

**Prove it — interview questions**

1. **[Basic] Why can't presence just be tracked by connection state?**

   <details><summary>Model answer</summary>

   Because disappearance produces no signal. A client that crashes, closes a laptop lid, or loses coverage sends nothing, and the TCP connection can remain apparently open for minutes afterwards — so connection state says online when the user is long gone. It also does not survive server crashes, since the process that held those connections takes its knowledge with it, and it does not work across multiple processes, each of which knows only its own connections. Presence has to be inferred from continuous positive evidence rather than from the absence of a goodbye.

   </details>

2. **[Basic] How does the heartbeat lease work?**

   <details><summary>Model answer</summary>

   Each online user has an entry in a presence store with a short expiry, and a periodic heartbeat refreshes it. If the heartbeats stop for any reason — crash, network loss, process kill — the entry expires by itself and the user is considered offline. The key property is that it is self-healing: nothing has to run cleanup code, which matters because the failures that most need cleaning up are exactly the ones where no cleanup code executes.

   </details>

3. **[Senior] How do you choose the heartbeat interval and TTL?**

   <details><summary>Model answer</summary>

   The TTL should be roughly three times the heartbeat interval so that two consecutive lost heartbeats do not trigger a false departure — single losses are routine on mobile networks, and declaring someone offline on one produces visible flickering. The interval itself trades detection lag against write load and battery: a million online users heartbeating every twenty seconds is fifty thousand writes per second, which makes presence a write-heavy workload that has to be sized deliberately. In practice users barely notice whether offline detection takes twenty seconds or ninety, but they do notice a contact flickering on a train, which argues for longer TTLs than intuition suggests.

   </details>

4. **[Senior] Why is fan-out often the dominant cost?**

   <details><summary>Model answer</summary>

   Because it scales as the product of users, contact list size and transition frequency. A million users with two hundred contacts each, transitioning ten times a day as networks blip and tabs close, produces billions of notifications daily — which can exceed the application's actual messaging traffic for a feature that is a coloured dot. The effective mitigation is to observe that presence only matters for contacts the user can see: fetching presence for the few dozen contacts currently on screen, refreshed while that view is open, bounds the cost by screen size rather than social graph size, and is indistinguishable to the user.

   </details>

5. **[Staff] Design presence for a messaging application with a million concurrent users.**

   <details><summary>Model answer</summary>

   Heartbeats every twenty seconds carried over the existing connection so there is no separate transport and they double as connection liveness detection, writing a presence entry with a sixty-second TTL to an in-memory store with native expiry — expiry rather than cleanup, because crashes never run cleanup. That is fifty thousand writes per second, so I would size and shard it as a write-heavy workload from the outset rather than discovering that later. Three statuses instead of two — online, idle after five minutes without interaction, and offline — because idle carries more information and tolerates the lag better than a binary state does. Delivery by pull rather than push: clients request presence for the contacts currently visible and refresh while that view is open, which bounds fan-out by screen size instead of the social graph and avoids the billions of daily notifications a push model would create. Transitions debounced by about ten seconds so a brief drop does not surface as leaving and returning. And per-user visibility controls, because online and last-seen data reveals when someone sleeps and works, which is personal information rather than a feature detail.

   </details>

6. **[Principal] What do people consistently underestimate about presence?**

   <details><summary>Model answer</summary>

   Three things, in increasing order of how late they are discovered. First, the write load: heartbeats from every online user make presence one of the heaviest continuous write workloads in a messaging system, which is a surprising thing to learn about a coloured dot, and it arrives as a capacity problem rather than a feature problem. Second, the fan-out: push-based presence scales as users times contacts times transitions and can exceed the real messaging traffic, which is only avoidable by recognising that presence matters solely for what is currently on screen — a reframing that collapses the cost entirely and that nobody reaches for until the bill appears. Third, and the one I would raise earliest, presence is behavioural data: online history reveals sleep schedules, working hours and routines, it is frequently exposed to anyone who can add you as a contact, and it is almost always designed as a feature rather than as personal information. The privacy controls are cheap to add at the start and awkward to retrofit once users have come to rely on seeing each other. Underlying all three is the point worth stating plainly in any design review: presence is a guess with a bounded staleness window, and any requirement phrased as accurate online status is asking for something the mechanism cannot provide.

   </details>

---

### Realtime fan-out routing

*Deliver one message to many connected recipients spread across many processes, deciding where the multiplication happens and how large it is allowed to get.*

**Flow:** `Publisher` → `Subscription index` → `Fan-out strategy` → `Process routing` → `Connection write`

> **The 30-second version**  
> Decide where one message becomes many — topic routing for small channels, read-time fetch for large ones — and back it with a durable log so push is an accelerator rather than a guarantee.

**The problem**

A message is posted to a channel with fifty thousand members. Their connections are distributed across thirty processes, and each member must receive it within a few hundred milliseconds. The system must determine who should receive it, where each recipient's connection lives, and how to get the message there without every process receiving every message.

The difficulty is multiplicative. One published message becomes fifty thousand writes, and a busy channel with several messages per second becomes hundreds of thousands of writes per second from a single source — so the design question is not how to deliver a message but where the multiplication happens and what bounds it.

> **Fan-out is a placement decision: where does one become many?**  
> The message can be multiplied at the publisher, at a routing tier, at the process holding connections, or at read time when the client asks. Each choice moves the cost to a different place and fails differently. Choosing deliberately — rather than letting it happen wherever the code happens to loop — is the whole of fan-out design.

**Mental model**

Between a publisher and thousands of sockets sit three questions: who is subscribed, which process holds each subscriber, and how the message reaches those processes. The answers determine the cost profile.

1. **Subscription index** — Which users are interested in which channels — read on every publish, so it must be fast.
2. **Connection registry** — Which process holds each user's connection, which changes constantly as clients connect and drop.
3. **Transport** — How a message reaches the processes that need it — typically pub/sub, sometimes direct RPC.
4. **Local fan-out** — Each process writing to its own sockets, which is the only place the actual writes can happen.
5. **Back pressure** — What happens when a recipient cannot keep up, which decides whether one slow client harms others.

> **Broadcasting every message to every process does not scale, but it is the default**  
> The simplest design publishes each message to a channel every process subscribes to, letting each filter for its own connections. It works beautifully until the message rate times the process count exceeds what the pub/sub layer or the processes can handle — at which point every process is doing full message-rate work regardless of how few relevant connections it holds. The fix is topic-based routing so a process receives only messages for channels it actually serves.

**How it works**

**Four places to multiply, and what each costs**

```text
1  BROADCAST TO ALL PROCESSES  (simplest)
   publish once; every process receives every message;
   each filters for its own connections
   cost: message_rate x process_count on every process
   -> fine at 10 processes and 100 msg/s
   -> hopeless at 100 processes and 10,000 msg/s

2  TOPIC-BASED ROUTING  (the usual answer)
   each process subscribes only to channels it holds
   connections for
   cost: a process sees only its relevant traffic
   -> requires the pub/sub layer to handle many topics
   -> requires subscribe/unsubscribe as clients move

3  DIRECTED ROUTING  (registry lookup)
   look up which processes hold members of this channel,
   send only to those
   cost: a registry read per publish, plus N sends
   -> most precise, most moving parts
   -> the registry read is on the hot path

4  READ-TIME FAN-OUT  (no push multiplication)
   store the message once; clients fetch on connect
   and poll or subscribe to a cursor
   cost: moved to read time, bounded by active readers
   -> the answer for very large channels

THE ASYMMETRY THAT DECIDES IT
  small channels, many of them   -> push fan-out
  huge channels, few of them     -> read-time fan-out
  -> and most systems have BOTH, so most systems need
     both strategies with a threshold between them
```

1. **Route by topic, not by broadcast** — A process should receive only messages for channels it holds connections for, or its load grows with the whole system's message rate.
2. **Keep the subscription index fast** — It is read on every publish, so its latency is multiplied by the message rate.
3. **Split strategy by channel size** — Push fan-out is right for many small channels and catastrophic for one enormous one; a threshold switching to read-time fan-out handles both.
4. **Apply back pressure per connection** — A slow client must not cause unbounded buffering that harms every other client on the same process.
5. **Bound the write burst** — Fifty thousand socket writes from one message is a latency spike for everything else on that process unless it is paced.
6. **Handle the case where a recipient is offline** — Realtime delivery only reaches connected clients; anything durable needs a separate store-and-fetch path.

**What one message actually costs**

```text
CHANNEL WITH 50,000 MEMBERS, 30 PROCESSES

per message:
  1 publish
  ~30 process deliveries (or fewer with directed routing)
  50,000 socket writes
  -> the socket writes dominate everything else

AT 10 MESSAGES PER SECOND IN THAT CHANNEL
  500,000 socket writes per second
  spread over 30 processes = ~17,000 writes/s each
  -> feasible, but this is ONE channel

WHY LARGE CHANNELS BREAK PUSH FAN-OUT
  the cost is members x message_rate
  a 1,000,000-member channel at 10 msg/s
    = 10,000,000 writes/s
  -> no amount of process count fixes this, because
     every member genuinely needs the bytes
  -> the only real answer is to stop pushing:
     store once, let active clients read

THE ACTIVE-READER INSIGHT
  of 1,000,000 members, how many have the channel
  open right now?
  -> often a tiny fraction
  -> read-time fan-out costs active_readers, not members
  -> push fan-out costs members regardless
```

> **One slow consumer can degrade every client on a process**  
> When a socket's send buffer fills, writes either block or queue in application memory. Unbounded queueing means one client on a poor connection consumes memory until the process degrades for everyone it serves. The choices are to drop messages for that client, to disconnect it, or to bound its queue and let it resynchronise — but doing nothing is a choice too, and it is the one that turns an individual's bad network into a shared outage.

**Worked example**

A team messaging application with channels ranging from three members to a hundred thousand.

**Two strategies with a threshold**

```text
SMALL CHANNELS  (< 1,000 members)  -- push fan-out
  publish to a per-channel topic
  processes subscribe to topics for channels they hold
    connections for
  each process writes to its local sockets
  -> latency ~50 ms
  -> cost bounded by members, which is small

LARGE CHANNELS  (>= 1,000 members)  -- read-time
  write the message once to the channel log
  push only a lightweight "new activity" signal
  clients with the channel OPEN fetch from their cursor
  -> cost bounded by ACTIVE viewers, not members
  -> a 100,000-member channel with 200 people looking
     costs 200 reads, not 100,000 writes

THE THRESHOLD IS THE DESIGN
  below it, push is cheap and simplest
  above it, push cost grows with a number that has
    nothing to do with who is watching
  -> the switch must be automatic, because channels
     grow past the threshold without anyone noticing

CONNECTION REGISTRY
  user -> process, TTL 60 s, refreshed by heartbeat
  consulted for directed routing and direct messages

BACK PRESSURE
  per-connection queue capped at 100 messages
  exceeded -> drop the client to a resync path
  -> it refetches from its cursor on reconnect
  -> one bad network costs that user a resync, not
     the process its stability

OFFLINE MEMBERS
  realtime delivery reaches connected clients only
  durable history lives in the channel log
  -> the log is the source of truth; push is an
     accelerator
```

| Metric | Value | Note |
|---|---|---|
| Small channels | push fan-out | cost = members |
| Large channels | read-time | **cost = viewers** |
| Registry | 60 s TTL | self-healing |
| Back pressure | cap then resync | protects others |

> **Push is an accelerator over a durable log, not the delivery mechanism**  
> If the message is written to a log that clients can read from a cursor, then push delivery becomes an optimisation for latency rather than the thing correctness depends on. A dropped message, a slow consumer, or a disconnection all degrade to a slightly slower fetch instead of permanent loss — which removes an entire category of hard problems from the realtime layer.

**When to use it**

- **Chat and team messaging**, with many channels of varying size.
- **Collaborative editing**, fanning changes to everyone viewing a document.
- **Live dashboards and feeds**, where many clients watch the same data.
- **Multiplayer and gaming**, delivering state to everyone in a session.
- **Notification delivery** to users' active sessions across devices.

**When to avoid it**

- **Do not push fan-out to very large audiences**, where cost scales with members rather than viewers.
- **Do not broadcast every message to every process**, which makes each process do whole-system work.
- **Do not rely on push alone for durability**, since offline recipients receive nothing.
- **Do not buffer slow consumers without bound**, which converts one bad connection into a shared failure.
- **Do not use realtime fan-out where a periodic fetch suffices**, which is far cheaper and simpler.

**Advantages**

- **Low latency**, with messages reaching clients in tens of milliseconds.
- **Scales horizontally**, since each process handles only its own connections.
- **Topic routing bounds per-process load** to relevant traffic.
- **Read-time fan-out bounds cost by attention**, not by membership.
- **Backed by a log, it degrades gracefully**, with delivery failures becoming slower fetches.

**Disadvantages**

- **Cost multiplies** — one message becomes many writes.
- **Requires a connection registry** that changes constantly and must be kept accurate.
- **Large channels need a different strategy**, so two mechanisms must coexist.
- **Slow consumers threaten shared resources** without explicit back pressure.
- **Offline delivery is a separate problem**, which push does not solve.
- **Hard to observe**, since one logical message becomes thousands of physical writes.

**Trade-offs**

**Fan-out strategy trade-offs**

| Strategy | Cost scales with | Complexity | Best for |
|---|---|---|---|
| Broadcast to all processes | Message rate × processes | Lowest | Small deployments |
| Topic-based routing | Relevant traffic per process | Moderate | General case |
| Directed routing via registry | Exact recipients | Highest | Precise delivery; direct messages |
| Read-time fan-out | Active viewers | Moderate | Very large channels |
| Hybrid with a threshold | Both, appropriately | Highest | Real systems |

Most production systems end up hybrid, because channel sizes span several orders of magnitude and no single strategy is appropriate across that range. The important part is that the switch between strategies is automatic — channels cross the threshold as they grow, and nobody is watching when they do.

**How it fails**

**Fan-out failure modes**

| Failure | Cause | Fix |
|---|---|---|
| Every process saturated regardless of load | Broadcasting all messages to all processes | Topic-based subscription |
| Latency spikes for unrelated clients | Unpaced burst of thousands of socket writes | Pace large fan-outs; prioritise fairness |
| One process degraded by one user | Unbounded queueing for a slow consumer | Cap per-connection queues; drop to resync |
| Messages delivered to nothing | Stale connection registry entries | TTLs refreshed by heartbeat |
| Offline users never receive messages | Push treated as the delivery mechanism | Durable log as the source of truth |
| Large channel collapses the cluster | Push fan-out scaling with membership | Read-time fan-out above a size threshold |
| Duplicate delivery after reconnection | No cursor or idempotency | Per-client cursor; deduplicate by message id |

**Limits**

> **Sizing reference**
>
> - **Socket writes dominate**: one message to N members costs N writes regardless of routing cleverness.
> - **Threshold**: pushing works well for channels up to roughly a thousand members; beyond that read-time fan-out is usually better.
> - **Active fraction**: for large channels, concurrent viewers are typically a small percentage of membership — which is the entire argument for read-time delivery.
> - **Per-connection queue**: bound it explicitly, since unbounded buffering converts one slow client into a process-wide problem.
> - **Registry TTL**: tens of seconds, refreshed by heartbeat so crashed processes stop attracting routes.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Push fan-out with topic routing | Many small channels | Cost scales with membership |
| Read-time fan-out from a log | Very large channels | Slightly higher latency |
| Direct per-user routing | Direct messages, notifications | Registry on the hot path |
| Broadcast to all processes | Small deployments | Does not scale with process count |
| Managed pub/sub service | Avoiding routing operations | Cost; less control |
| Periodic polling | Low-frequency updates | Latency; wasted requests |

The read-time approach deserves more consideration than it usually gets. Because it costs in proportion to who is actually looking rather than who is a member, it handles the largest channels effortlessly — and it composes naturally with push as a lightweight activity signal that tells active clients when to fetch.

**In real systems**

- **Team messaging platforms** use different strategies by channel size, because membership spans several orders of magnitude within one product.
- **Pub/sub with per-channel topics** is the standard routing layer, so a process receives only traffic for channels it serves.
- **Connection registries with TTLs** are universal, since processes that crash never deregister their connections.
- **Durable channel logs with client cursors** make push an accelerator rather than the delivery guarantee, which is what allows graceful degradation.
- **Per-connection queue limits with forced resynchronisation** are standard practice, because unbounded buffering for slow clients is a shared-resource hazard.

**Common mistakes**

- **Broadcasting every message to every process**, making per-process load track total system load.
- **One fan-out strategy for all channel sizes**, which fails at the large end.
- **Treating push as the delivery guarantee**, leaving offline users with nothing.
- **Unbounded queues for slow consumers**, turning one bad network into shared degradation.
- **No cursor on reconnect**, causing gaps or duplicates.
- **Registry without TTL**, so crashed processes keep attracting routes.
- **Monitoring logical message rate** while the real load is the write multiplier.

**The staff-level view**

Fan-out designs usually work at launch and fail as a distribution of channel sizes develops.

- **Decide explicitly where multiplication happens.** Left implicit, it happens wherever the code loops, which is rarely the cheapest place and is invisible until the load arrives.
- **Plan for two strategies from the start.** Channel sizes span orders of magnitude, and the threshold switch must be automatic because channels grow past it without anyone noticing.
- **Back the realtime layer with a durable log.** It converts delivery failures, slow consumers and disconnections from correctness problems into latency problems, which removes most of the hard cases.
- **Bound per-connection buffering as a policy.** Without it, one user's poor network becomes a process-wide degradation, and the causality is extremely hard to see in metrics.
- **Measure physical writes, not logical messages.** A dashboard showing message rate hides the fan-out multiplier entirely, which is where the actual load lives.

**Go deeper**

Fan-out turns one published message into one write per connected recipient, so the design question is where that multiplication happens and what bounds it. Broadcasting every message to every process is simplest and makes per-process load track total system traffic; topic-based routing, where a process subscribes only to channels it holds connections for, restores the property that adding processes divides the work.

Channel size demands two strategies. Push fan-out costs one write per member per message, which is cheap for small channels and impossible for very large ones — and horizontal scaling cannot help, because every member genuinely needs the bytes. For large channels, writing the message once to a log and letting clients with it open fetch from a cursor costs in proportion to active viewers, typically a small fraction of membership. The threshold switch must be automatic, since channels grow past it unnoticed.

The decision that removes most difficulty is backing the realtime layer with a durable log. When the log is the source of truth, push becomes a latency optimisation rather than a guarantee, so dropped messages, slow consumers and disconnections all degrade to a slower fetch instead of permanent loss. Per-connection queues must still be bounded explicitly, because unbounded buffering converts one client's poor network into degradation for everyone on that process.

Realtime fan-out is the problem of turning one message into many deliveries across many processes without the multiplication becoming the system's limiting factor.

**Where multiplication happens is the design.** A message can be multiplied at the publisher, at a routing tier, at the process holding the sockets, or deferred to read time when a client asks. Each placement moves cost somewhere different and fails differently, and left unchosen it simply occurs wherever the code loops — which is rarely the cheapest place and is invisible until the load arrives. Making the choice explicit is most of what fan-out design consists of.

**Topic routing is what makes horizontal scaling work.** The simplest structure publishes every message to a channel that all processes consume, letting each filter for its own connections. This is fine at small scale and pathological beyond it, because per-process work then tracks total system message rate rather than local connection count, so adding processes increases aggregate work. Subscribing each process only to the channels it actually holds connections for restores the intended property, at the cost of managing subscriptions as clients connect and disconnect.

**Channel size splits the problem in two.** Push fan-out costs one write per member per message, which no amount of horizontal scaling reduces, because each member genuinely requires the bytes. For a channel with a hundred thousand members this is untenable at any meaningful message rate. The observation that resolves it is that membership and attention are different numbers: most members of a large channel are not currently looking at it. Writing the message once and letting clients with the channel open read from a cursor costs in proportion to active viewers, which is typically a small percentage of membership. Real systems therefore need both strategies and an automatic threshold, because channels cross it as they grow and nobody is watching when they do.

**A durable log removes most of the hard cases.** When the log is the source of truth and push is an accelerator, delivery failures stop being correctness problems: a dropped message, a disconnected client or a consumer that fell behind all resolve to fetching from a cursor slightly later. Without that backing, the realtime layer must itself guarantee delivery, which drags in buffering for slow consumers, reconciliation after disconnection, and a wholly separate path for offline recipients — a set of guarantees a connection-oriented push layer is badly suited to provide.

**Back pressure is a policy, and its absence is also a policy.** When a client's socket cannot keep up, messages either block or accumulate in application memory. Unbounded accumulation means one user on a poor connection consumes resources until the process serving thousands of others degrades — a shared failure with an individual cause, and one whose causality is very hard to read in aggregate metrics. Capping per-connection queues and forcing exceeded clients into a resynchronisation path converts it into a private cost borne by the affected user.

**Measure physical writes, not logical messages.** A dashboard reporting a few thousand messages per second can correspond to millions of socket writes, and it is the writes that determine capacity, latency spikes and failure. Similarly, a single large fan-out performed as an unpaced burst introduces latency for every unrelated client on that process. Both are invisible at the level people naturally instrument, which is why fan-out systems commonly appear healthy right up to the point where they are not.

**Prove it — interview questions**

1. **[Basic] What makes fan-out hard?**

   <details><summary>Model answer</summary>

   The multiplication. One published message becomes as many socket writes as there are connected recipients, so a channel with fifty thousand members turns a single publish into fifty thousand writes, and a busy channel does that several times a second. The delivery of any individual message is trivial; the design problem is deciding where the multiplication happens and what bounds its size, because left unconsidered it happens wherever the code loops and scales with something nobody chose.

   </details>

2. **[Basic] Why not broadcast every message to every process?**

   <details><summary>Model answer</summary>

   Because each process then does work proportional to the whole system's message rate rather than to its own connections. With ten processes and modest traffic it is fine and it is much the simplest thing to build. With a hundred processes and high traffic, every process is receiving and filtering every message even if it holds no relevant connections at all — so adding processes increases total work instead of dividing it. Topic-based subscription, where a process listens only for channels it actually serves, restores the property that scaling out reduces per-process load.

   </details>

3. **[Senior] When should you switch from push to read-time fan-out?**

   <details><summary>Model answer</summary>

   When cost starts scaling with membership rather than with attention. Push fan-out costs one write per member per message, which is fine for a channel of fifty and ruinous for a channel of a hundred thousand — and no amount of horizontal scaling fixes it, because every member genuinely needs the bytes. The observation that saves you is that most members of a large channel are not looking at it: writing the message once to a log and having clients with the channel open fetch from their cursor costs in proportion to active viewers, which is typically a tiny fraction of membership. The threshold is usually around a thousand members, and the switch must be automatic because channels cross it as they grow and nobody notices.

   </details>

4. **[Senior] How do you handle a slow consumer?**

   <details><summary>Model answer</summary>

   Deliberately, because the default is bad. When a client's socket cannot absorb writes, messages either block or accumulate in application memory, and unbounded accumulation means one person on a poor connection consumes memory until the process degrades for every client it serves — a shared outage caused by an individual's network. The options are to bound the queue and drop the client into a resynchronisation path, to drop messages for them, or to disconnect them. Bounding and forcing a resync is usually best when there is a durable log, because the client refetches from its cursor on reconnect and loses nothing permanently.

   </details>

5. **[Staff] Design fan-out for a messaging product with channels from three to a hundred thousand members.**

   <details><summary>Model answer</summary>

   Two strategies with an automatic threshold, because no single approach spans that range. Below about a thousand members, push fan-out over per-channel topics: processes subscribe only to channels they hold connections for, so per-process load tracks their own connections rather than total traffic, and latency is tens of milliseconds. Above the threshold, read-time fan-out: the message is written once to the channel log, and only a lightweight activity signal is pushed, with clients that have the channel open fetching from their cursor — so a hundred-thousand-member channel with two hundred people looking costs two hundred reads rather than a hundred thousand writes. Underneath both, the channel log is the source of truth, which is the decision that matters most: it makes push an accelerator rather than a guarantee, so dropped messages, slow consumers and disconnections all degrade to a slightly slower fetch instead of permanent loss. A connection registry with a short heartbeat-refreshed TTL handles direct routing and self-heals after process crashes. And per-connection queues are capped, with exceeding clients dropped to resync, so one bad network costs that user a refetch rather than costing everyone on that process their latency.

   </details>

6. **[Principal] What do teams get wrong about realtime fan-out at scale?**

   <details><summary>Model answer</summary>

   They treat push as the delivery mechanism rather than as an optimisation, and everything difficult follows from that. If correctness depends on the push succeeding, then slow consumers must be buffered, disconnections must be reconciled, offline users need a separate parallel path, and every delivery failure is a data-loss bug — so the realtime layer accumulates guarantees it is poorly suited to provide. Backing it with a durable log that clients read from a cursor inverts this: push becomes a latency optimisation, and every one of those failures becomes a slightly slower fetch. The second recurring error is assuming one fan-out strategy across a distribution of channel sizes spanning several orders of magnitude, when push cost scales with membership and read-time cost scales with attention — two different regimes needing two different answers and an automatic switch between them. And the third, which hides the other two, is monitoring logical message rate rather than physical writes: the dashboard shows a few thousand messages a second while the system is performing millions of socket writes, so the load nobody is measuring is exactly the load that determines capacity.

   </details>

---

### Ordering, acknowledgments and resume

*Give messages sequence numbers, have clients acknowledge and report position, and replay the gap on reconnect — turning an unreliable channel into a reliable stream.*

**Flow:** `Sequence numbers` → `Client cursor` → `Acknowledgment` → `Retention buffer` → `Replay or resync`

> **The 30-second version**  
> Number every message, advance the client cursor only after durable application, acknowledge periodically, and replay the gap on reconnect — with a resync fallback beyond the retention window.

**The problem**

A realtime connection drops for three seconds and comes back. During that window the server sent four messages. The client reconnects, receives whatever comes next, and has no idea that four messages are missing — its view of the world is now silently wrong, and nothing anywhere produced an error.

Ordering has the same character. Messages that arrive out of order can leave state inconsistent in ways that look like application bugs: an update applied before the creation it depends on, a deletion applied before the edit that preceded it.

> **Sequence numbers turn an unreliable channel into a reliable stream**  
> If every message carries a monotonically increasing number and the client remembers the highest it has processed, then gaps are detectable, duplicates are identifiable, order is verifiable, and resumption is exact. One small addition to the message format supplies the foundation for all four properties, which is why it should be present from the first version rather than retrofitted after the first silent divergence.

**Mental model**

The server maintains an ordered stream per client or per channel. The client tracks the highest position it has processed. On reconnection it reports that position and the server sends what follows — or, if it no longer has that history, tells the client to start over.

1. **Sequence** — A monotonically increasing number per stream, assigned by the server.
2. **Cursor** — The highest sequence the client has durably processed.
3. **Acknowledgment** — The client reporting its cursor, which lets the server release retained history.
4. **Retention** — How much history the server keeps, which bounds how long an absence can be.
5. **Resync** — The fallback when the gap exceeds retention: discard local state and refetch.

> **Silent gaps are worse than errors**  
> A client that misses messages without knowing produces no alert, no exception and no failed request. It simply displays incorrect data, applies later updates to a stale base, and diverges further over time. The bug surfaces days later as unexplained inconsistency, by which point reconstructing what happened is nearly impossible. Detectability is the first thing sequence numbers buy, and it matters more than recovery.

**How it works**

**The four properties from one field**

```text
EVERY MESSAGE CARRIES seq

1  GAP DETECTION
   client has 1041, receives 1043
   -> it KNOWS 1042 is missing
   -> without seq, it never finds out

2  DUPLICATE SUPPRESSION
   client has 1043, receives 1043 again
   -> discard
   -> retries and reconnect overlap become harmless

3  ORDERING
   client has 1041, receives 1043 then 1042
   -> buffer 1043, apply 1042, then 1043
   -> or request replay from 1042

4  EXACT RESUME
   reconnect with cursor 1041
   -> server sends 1042 onward

ACKNOWLEDGMENT closes the loop
  the client periodically reports its cursor
  -> the server can release history below it
  -> without acks, retention is a guess based on time
     rather than on what clients actually have

RETENTION DEFINES THE RESUME WINDOW
  keep 5 minutes -> absences under 5 minutes resume
    exactly
  longer absence -> the history is gone
  -> the server MUST have a resync answer, or a long
     disconnection silently diverges
```

1. **Assign sequence numbers server-side** — Client-assigned numbers cannot be ordered across clients and are not trustworthy.
2. **Have the client report its cursor on reconnect** — This is what makes resumption exact rather than approximate.
3. **Acknowledge periodically, not per message** — Per-message acks double the traffic; periodic acks bound retention adequately.
4. **Advance the cursor only after durable processing** — A cursor advanced on receipt loses messages if the client crashes before applying them.
5. **Define retention and the resync fallback together** — Exact resume exists only within the window, and the path beyond it must be implemented or long outages diverge.
6. **Make handlers idempotent** — At-least-once delivery plus reconnection overlap means duplicates are certain, not hypothetical.

**Where cursors go wrong**

```text
ADVANCING TOO EARLY
  receive 1042 -> cursor = 1042 -> apply it -> CRASH
  reconnect with cursor 1042
  -> 1042 was never applied, and never will be
  -> SILENT LOSS
  fix: advance the cursor only after the effect is
       durable

ADVANCING TOO LATE (safe, but noisy)
  cursor advanced only every 30 s
  -> a crash replays up to 30 s of messages
  -> harmless IF handlers are idempotent
  -> this is the correct direction to err

THE RULE
  prefer replaying a message you already applied
  over skipping one you did not
  -> which only works if applying twice is safe
  -> so idempotency is a precondition for safe
     cursor handling, not an optional extra

DELIVERY SEMANTICS IN PRACTICE
  at most once   - cursor advanced before processing
                   -> loses messages on crash
  at least once  - cursor advanced after processing
                   -> duplicates on crash; needs
                      idempotent handlers
  exactly once   - not achievable at the transport;
                   approximated by at-least-once
                   delivery plus idempotent effects
```

> **Per-client retention is a memory cost that scales with clients**  
> Keeping replay history for every connected client means memory proportional to clients multiplied by retention window multiplied by message rate. For a large deployment this is substantial, and it is why retention windows are measured in minutes rather than hours, why acknowledgments matter for releasing history early, and why the resync path is not optional — it is the pressure valve that makes a bounded window acceptable.

**Worked example**

A collaborative application's realtime channel, with the recovery paths spelled out.

**Protocol and recovery**

```text
MESSAGE
  {seq: 4471, type: "edit", payload: {...}}

CLIENT
  applies the edit, then sets cursor = 4471
  (in that order - the reverse loses messages on crash)
  sends an ack with its cursor every 10 s

SERVER
  retains messages above the minimum acked cursor,
  up to 5 minutes
  releases history below what all clients have acked

RECONNECT WITHIN THE WINDOW
  client: "resume from 4471"
  server: sends 4472..4510
  -> exact recovery; the user sees a brief pause

RECONNECT BEYOND THE WINDOW
  client: "resume from 2100"
  server: "resync"  (history no longer held)
  client: discards local state, fetches a snapshot,
          resumes from the snapshot's sequence
  -> slower, but correct

OUT-OF-ORDER ARRIVAL
  client has 4471, receives 4473
  -> buffer 4473, request 4472
  -> apply in order once the gap is filled
  -> a bounded buffer; if the gap is not filled
     quickly, resync rather than buffer indefinitely

DUPLICATES
  client has 4473, receives 4472 again -> discard
  handlers idempotent regardless, because overlap
  during reconnection is routine

WHAT THIS BUYS
  a 3-second network blip costs a 3-second pause,
  not a silently divergent document
```

| Metric | Value | Note |
|---|---|---|
| Cursor | after apply | **never before** |
| Ack | every 10 s | releases history |
| Retention | 5 minutes | resume window |
| Beyond | resync | snapshot + resume |

> **Exactly-once delivery is achieved with at-least-once plus idempotence**  
> No transport can guarantee exactly-once delivery, because the acknowledgment can always be lost after the message was processed. What is achievable is at-least-once delivery combined with handlers whose repeated application has no additional effect — sequence numbers make duplicates identifiable, and idempotent handling makes them harmless. The combination gives the property people want from exactly-once without requiring something impossible.

**When to use it**

- **Any realtime channel where missed messages matter**, which is nearly all of them.
- **Collaborative editing**, where an out-of-order or missing operation corrupts the document.
- **Event streams feeding client state**, where the client's view must track the server's.
- **Mobile clients**, which disconnect constantly and must resume cleanly.
- **Anywhere silent divergence would be discovered late**, which is the usual case.

**When to avoid it**

- **Do not add sequencing where messages are independent and disposable**, such as ephemeral presence pings.
- **Do not retain unbounded history per client**, which costs memory proportional to clients and rate.
- **Do not buffer out-of-order messages indefinitely**, which delays indefinitely on a permanently missing message.
- **Do not acknowledge every message individually** at high rates, which doubles traffic for little gain.
- **Do not implement resume without the resync fallback**, which leaves long absences silently wrong.

**Advantages**

- **Gaps become detectable**, which converts silent divergence into a handled case.
- **Duplicates become identifiable**, making retries safe.
- **Ordering is verifiable**, so dependent updates apply correctly.
- **Resumption is exact** within the retention window.
- **Delivery failures degrade to latency**, not to data loss.
- **One field supplies all of it**, so the cost is minimal.

**Disadvantages**

- **Retention costs memory** proportional to clients, rate and window.
- **The resync path must be built**, and it is genuinely more work than resume.
- **Cursor handling is subtle**, with the ordering of apply and advance deciding correctness.
- **Idempotent handlers are a precondition**, which constrains application design.
- **Acknowledgment traffic** adds overhead.
- **Per-channel ordering does not give global ordering**, which is a common misunderstanding.

**Trade-offs**

**Delivery semantics trade-offs**

| Semantics | Cursor timing | Failure result | Requires |
|---|---|---|---|
| At most once | Advance before processing | Messages lost on crash | Nothing |
| At least once | Advance after processing | Duplicates on crash | Idempotent handlers |
| Effective exactly once | Advance after; deduplicate | Neither loss nor visible duplication | Idempotence plus sequence numbers |
| Short retention | — | Frequent resyncs | Cheap memory |
| Long retention | — | Rare resyncs | Memory per client |

The right default is at-least-once with idempotent handlers, because the alternative failure — silently losing a message — is undetectable, while a duplicate applied to an idempotent handler is invisible. Erring toward replay is always the safer direction.

**How it fails**

**Ordering and resume failures**

| Failure | Cause | Fix |
|---|---|---|
| Client state silently diverges | No sequence numbers; gaps undetectable | Sequence every message |
| Messages lost on client crash | Cursor advanced before processing | Advance only after the effect is durable |
| Duplicate effects after reconnect | Non-idempotent handlers | Deduplicate by sequence; make handlers idempotent |
| Client stuck after a long outage | Resume beyond retention with no fallback | Resync path with a snapshot |
| Server memory growth | Unbounded per-client retention | Bound the window; release on acknowledgment |
| Client hangs waiting for a gap | Indefinite buffering of out-of-order messages | Bounded wait, then request replay or resync |
| Inconsistent state across channels | Assuming per-channel order implies global order | Design for per-stream ordering only |

**Limits**

> **Practical parameters**
>
> - **Retention**: minutes, not hours — memory scales with clients × rate × window.
> - **Acknowledgment**: every few seconds or every N messages, rather than per message.
> - **Out-of-order buffering**: bounded; escalate to replay or resync rather than waiting indefinitely.
> - **Cursor durability**: advance only after the effect is durable, accepting duplicates over loss.
> - **Ordering scope**: guaranteed per stream only; cross-stream ordering requires a different mechanism.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Sequence numbers with resume | Realtime channels | Retention cost; resync path needed |
| Full resync on every reconnect | Small state; infrequent drops | Expensive and slow for frequent reconnects |
| Version vectors | Multi-writer replication | More complex; larger metadata |
| Timestamp ordering | Loose ordering needs | Clock skew makes it unreliable |
| No ordering guarantees | Independent, disposable events | Silent divergence when it matters |
| Log-backed cursors | Durable event streams | Requires a durable log |

Full resynchronisation on every reconnect is a legitimate choice when client state is small and disconnections are rare — it removes retention, cursors and replay entirely. It becomes untenable for mobile clients, which reconnect frequently enough that a full refetch each time is both slow and expensive.

**In real systems**

- **Server-sent events** build resume into the protocol via event ids and the Last-Event-ID header, which is the same mechanism standardised.
- **Message brokers** expose consumer offsets that are precisely this pattern applied to durable logs.
- **Collaborative editors** sequence operations so that a client rejoining can replay exactly what it missed, without which documents diverge.
- **Mobile sync protocols** combine a retention window with snapshot resynchronisation, because long offline periods are normal rather than exceptional.
- **At-least-once delivery with idempotent handlers** is the standard industry position, since exactly-once at the transport is not achievable.

**Common mistakes**

- **No sequence numbers**, making gaps undetectable and divergence silent.
- **Advancing the cursor before processing**, losing messages on crash.
- **Non-idempotent handlers**, so certain duplicates cause real effects.
- **Resume without a resync fallback**, stranding clients after long absences.
- **Unbounded retention per client**, growing memory with clients and rate.
- **Indefinite out-of-order buffering**, hanging on a message that will never arrive.
- **Assuming per-channel order implies global order**, producing cross-channel inconsistency.

**The staff-level view**

This is the part of realtime systems most often deferred and most expensive to add later.

- **Put sequence numbers in the first version.** Retrofitting them means every client, every handler and every stored cursor changes, and the reason for retrofitting is usually an incident nobody could diagnose.
- **Insist that cursors advance after durable application.** The reverse ordering loses messages on crash and produces exactly the silent divergence the mechanism exists to prevent.
- **Require idempotent handlers as a design constraint.** Duplicates are certain under at-least-once delivery with reconnection overlap, so this is a precondition rather than defensive coding.
- **Implement the resync path alongside resume.** Resume works within the retention window; without the fallback, a long disconnection leaves a client quietly and permanently wrong.
- **Be precise about ordering scope.** Per-stream ordering is what these mechanisms provide, and teams routinely assume it implies global ordering across channels, which it does not.

**Go deeper**

A realtime channel drops messages, reorders them and delivers duplicates, and without sequence numbers none of that is detectable — a client reconnecting after a brief interruption simply has a silently incorrect view. Attaching a monotonic sequence to every message and having the client track the highest it has processed makes gaps visible, duplicates identifiable, order verifiable and resumption exact.

Cursor handling decides the failure mode. Advancing the cursor before applying a message means a crash in between skips it permanently and undetectably; advancing after means a crash replays it, which is harmless if handlers are idempotent. That makes idempotence a precondition rather than a nicety, and it is why at-least-once delivery with idempotent effects is the standard position — exactly-once at the transport is not achievable, since an acknowledgment can always be lost after processing.

Resume is exact only within a retention window, which is bounded by memory scaling with clients, message rate and window length. Periodic acknowledgments let the server release history clients have confirmed. Beyond the window the server must instruct the client to resynchronise from a snapshot — a path that must exist, or a long disconnection leaves the client quietly and permanently wrong.

Realtime transports lose, reorder and duplicate messages. This layer converts that unreliable channel into a stream an application can depend on.

**One field, four properties.** A monotonically increasing sequence on every message, paired with a client that remembers the highest it has processed, yields gap detection, duplicate suppression, order verification and exact resumption. The most valuable of these is the first: without sequencing, a client that misses messages produces no error of any kind and simply proceeds with incorrect state, applying subsequent updates to a wrong base so the divergence compounds. Detectability matters more than recovery, because an undetected inconsistency surfaces days later with no evidence remaining.

**Cursor ordering determines the failure mode.** Advancing the cursor on receipt and then applying means a crash between the two permanently skips a message the cursor claims was handled — silent loss. Applying first and advancing after means a crash causes replay, which is harmless if the handler is idempotent. The safe direction is always to prefer replaying something already applied over skipping something that was not, and that preference is only available when repeated application has no additional effect. Idempotence is therefore a precondition of correct cursor handling, not an optional robustness measure.

**Exactly-once is unavailable, and the substitute is adequate.** An acknowledgment can be lost after the message has been processed, leaving the sender unable to distinguish a lost message from a lost confirmation; it must either resend and risk duplication or not resend and risk loss. No transport escapes this. At-least-once delivery combined with sequence-based deduplication and idempotent handlers produces the property applications actually want — each message takes effect once — without requiring a guarantee that cannot exist.

**Retention bounds resumption, and acknowledgments bound retention.** Replaying from a client's cursor requires still holding the messages after it, so memory scales with connected clients multiplied by message rate multiplied by window length. That is why windows are measured in minutes. Periodic acknowledgments — every few seconds rather than per message, to avoid doubling traffic — let the server release history that all clients have confirmed, which keeps the cost tractable.

**The resync path is not optional.** Exact resume exists only within the retention window; beyond it the history is gone and the only correct response is to tell the client to discard its state and refetch a snapshot, then continue from the snapshot's sequence. Systems that implement resume without this fallback appear to work until the first disconnection longer than the window, after which the affected client is quietly and permanently wrong — the precise failure the mechanism was introduced to prevent. Out-of-order buffering needs a similar escape: waiting indefinitely for a missing message hangs on one that may never arrive, so the wait must be bounded and escalate to replay or resync.

**Ordering scope is routinely misunderstood.** These mechanisms provide ordering within a stream, not across streams. Two channels each internally ordered say nothing about the relative order of their messages, and applications that assume otherwise produce cross-channel inconsistencies that look like ordering bugs but are actually scope errors. Where genuine cross-entity ordering is required it must come from a shared stream or an explicit causal mechanism — and recognising which guarantee is actually needed, before building on the weaker one, avoids a class of defect that is extremely difficult to diagnose after the fact.

**Prove it — interview questions**

1. **[Basic] Why do realtime messages need sequence numbers?**

   <details><summary>Model answer</summary>

   So that gaps are detectable. A client that reconnects after a brief drop receives whatever comes next and has no way to know that four messages were sent while it was away — its state is now wrong, and nothing errored. With a monotonic sequence on every message and a client that remembers the highest it processed, a gap is immediately visible, duplicates are identifiable, order is verifiable, and resumption is exact. One field supplies all four properties, which is why it belongs in the first version of any realtime protocol.

   </details>

2. **[Basic] What is the resume flow on reconnection?**

   <details><summary>Model answer</summary>

   The client reports the highest sequence it has processed, and the server sends everything after it. That gives exact recovery — a three-second network interruption costs a three-second pause rather than a silently divergent view. The caveat is that the server must still hold that history, which means a retention window; if the client has been away longer than that, the server responds with a resynchronisation instruction and the client discards its state, fetches a snapshot, and resumes from the snapshot's position.

   </details>

3. **[Senior] Why does the order of applying and advancing the cursor matter?**

   <details><summary>Model answer</summary>

   Because it determines whether failures lose messages or duplicate them. Advancing the cursor on receipt and then applying means a crash in between leaves the message unapplied and permanently skipped, since the cursor claims it was handled — silent loss, which is undetectable. Applying first and advancing after means a crash replays the message on reconnect, which is harmless provided the handler is idempotent. The rule is to prefer replaying something you already applied over skipping something you did not, and that preference is only safe when handlers can be applied twice without additional effect.

   </details>

4. **[Senior] Why is exactly-once delivery not achievable?**

   <details><summary>Model answer</summary>

   Because the acknowledgment can be lost after the message has been processed. The sender then cannot distinguish between a message that never arrived and one that arrived and was handled but whose confirmation vanished, so it must either resend — risking duplication — or not resend, risking loss. There is no third option at the transport level. What is achievable is at-least-once delivery combined with idempotent handlers and sequence-based deduplication, which produces the effect people actually want: every message takes effect exactly once, even though it may be delivered more than once.

   </details>

5. **[Staff] Design the reliability layer for a collaborative editing channel.**

   <details><summary>Model answer</summary>

   Every message carries a server-assigned monotonic sequence. The client applies the operation and only then advances its cursor, in that order, because the reverse loses operations on crash — and since that ordering guarantees occasional replay, handlers must be idempotent, which I would treat as a design constraint on the operation model rather than as defensive coding. Clients acknowledge their cursor every few seconds, which lets the server release history below what all connected clients have confirmed, and the server retains roughly five minutes beyond that. A reconnection inside the window resumes exactly from the reported cursor; one outside it receives a resync directive, and the client fetches a snapshot and continues from the snapshot's sequence — that fallback is not optional, because without it a long disconnection leaves a document quietly divergent. Out-of-order arrivals are buffered but only briefly: if the gap is not filled quickly, the client requests replay or resyncs rather than waiting indefinitely for a message that may never come. The overall effect I am buying is that every failure mode — drop, duplicate, reorder, long absence — resolves to a pause or a refetch rather than to a document that is subtly wrong in a way nobody will notice until much later.

   </details>

6. **[Principal] Why is this layer so often missing, and what does its absence cost?**

   <details><summary>Model answer</summary>

   It is missing because the happy path works without it. A realtime channel delivers messages correctly in development, in testing, and in production whenever the network behaves, so the reliability layer looks like overhead for a case that does not seem to occur. It occurs constantly — mobile networks, deploys, proxy timeouts — but the failure is silent, which is what makes it so expensive. A client that misses four messages produces no error, no alert and no failed request; it simply displays state that is slightly wrong and then applies subsequent updates to that wrong base, so the divergence compounds and surfaces days later as inconsistency nobody can reproduce or explain. By then the evidence is gone. The cost of adding sequence numbers at the start is a single field and a cursor; the cost of adding them after the first such incident is changing every client, every handler and every persisted cursor, plus the incident itself. So my position in any realtime design review is that sequencing, cursors and a resync path are part of the minimum viable protocol rather than a hardening phase — not because failures are likely, but because when they happen they produce no signal at all.

   </details>

---

### Offline-first synchronization

*Treat the local store as the primary one and the server as a replica to reconcile with, accepting that conflicts are inevitable and must be resolved by policy.*

**Flow:** `Local store` → `Pending mutations` → `Sync engine` → `Server state` → `Conflict resolution`

> **The 30-second version**  
> Make the local store primary and the server a replica to reconcile with — every write succeeds immediately, mutations queue durably, and conflicts are resolved by explicit per-type policy.

**The problem**

A note-taking application on a phone in a tunnel, a field service app in a warehouse with no signal, a document editor on a flaky conference network. In each case the user expects to keep working, and expects their changes to be there when connectivity returns.

The conventional architecture cannot deliver this. If every action requires a round trip, then no connectivity means no functionality, and intermittent connectivity means a spinner on every interaction. Retrofitting offline support onto such a design is close to a rewrite, because the assumption that the server is the source of truth is embedded in every operation.

> **Invert the relationship: local is primary, the server is a replica**  
> In an offline-first design the application reads and writes only local storage and always succeeds immediately. A background process reconciles local state with the server when it can. This makes the network an optimisation rather than a dependency, and as a side effect makes the application feel instantaneous even when connectivity is perfect — because no interaction ever waits for a round trip.

**Mental model**

There are two copies of the data that drift apart and are periodically reconciled. The application only ever touches the local one; a sync engine handles the reconciliation, including deciding what happens when both sides changed.

1. **Local store** — The application's only read and write target, so every operation succeeds immediately.
2. **Pending queue** — Mutations made locally that the server has not yet accepted.
3. **Sync engine** — Pushes pending mutations, pulls remote changes, and reconciles the two.
4. **Conflict policy** — What happens when the same thing changed in both places — the part that cannot be avoided.
5. **Convergence** — The guarantee that all replicas eventually reach the same state.

> **Conflicts are not an edge case; they are the defining problem**  
> Any system permitting offline writes will have two devices modify the same data while partitioned. There is no design that prevents this — only designs that decide what to do about it. Choosing a policy deliberately, and choosing one whose failure mode is acceptable, is the substance of offline-first work; everything else is mechanics. Teams that treat conflicts as rare discover that they are common and that the default resolution silently destroys user data.

**How it works**

**The sync cycle**

```text
WRITE PATH  (always succeeds, never waits)
  user edits a note
  -> write to local store immediately
  -> append a mutation to the pending queue
  -> UI updates from local state
  -> no network involved

SYNC  (when connectivity allows)
  1  PUSH pending mutations
       server accepts, or rejects with a conflict
  2  PULL changes since the last sync cursor
  3  RECONCILE
       no overlap        -> apply cleanly
       same item changed -> resolve by policy
  4  advance the sync cursor

OFFLINE PERIOD
  mutations accumulate in the queue
  reads continue from local state
  the app is fully functional

RECONNECT
  the queue drains in order
  conflicts surface for the items touched on both sides

WHAT MAKES THIS WORK
  the UI never awaits the network
  the pending queue is durable across app restarts
  the sync cursor makes pulls incremental
  the conflict policy is explicit
```

1. **Make every local operation succeed immediately** — The moment a write can fail or block on the network, the offline-first property is gone.
2. **Persist the pending queue durably** — An app killed by the operating system must not lose mutations that were never synced — this is the most common data-loss bug in offline apps.
3. **Sync incrementally with a cursor** — Full resynchronisation on every connection is slow, expensive, and impractical on mobile data.
4. **Choose a conflict policy per data type** — Different data has different correct answers; a single global policy is wrong somewhere.
5. **Prefer mergeable operations over overwrites** — Recording intent — append this, increment that — merges far better than recording final values.
6. **Surface sync state honestly** — Users need to know whether their work has reached the server, especially before switching devices.

**Conflict policies and what each destroys**

```text
LAST-WRITE-WINS
  keep the version with the newer timestamp
  + trivial to implement
  - SILENTLY DISCARDS the other edit
  - depends on clocks that disagree between devices
  -> acceptable for: user preferences, single-device data
  -> dangerous for: anything the user typed

FIRST-WRITE-WINS
  reject later conflicting writes
  + preserves the earliest intent
  - the user who was offline loses their work
  -> rarely what anyone wants

MERGE BY FIELD
  both devices edited different fields -> take both
  + no loss when edits do not overlap
  - needs field-level change tracking
  -> good for structured records

OPERATION-BASED MERGE
  sync INTENT, not values
  "append X", "increment by 1", "set title to Y"
  + appends and increments merge perfectly
  + the common cases stop being conflicts at all
  - requires modelling changes as operations
  -> the best general answer

CRDTs
  data types that merge deterministically by
  construction
  + convergence guaranteed, no coordination
  - metadata overhead; constrained data model
  -> excellent for text; heavy for everything else

ASK THE USER
  present both versions
  + never loses data
  - interrupts; unusable if conflicts are frequent
  -> reserve for genuinely irreconcilable cases
```

> **Last-write-wins with device clocks loses data unpredictably**  
> Two devices with clocks differing by a few minutes will resolve conflicts by whose clock is ahead rather than by who edited last, so a user's newer edit can be discarded in favour of an older one from another device. Combined with silent discarding, this produces the worst class of bug in offline applications: work disappears, no error appears, and the pattern is impossible to reproduce. Logical clocks or version vectors avoid the clock dependency; nothing avoids the silent discard except a different policy.

**Worked example**

A field service application used in warehouses and basements with no connectivity.

**Design with policies chosen per data type**

```text
LOCAL STORE
  embedded database on the device
  every read and write is local and immediate
  pending mutations in a durable queue that survives
    app termination

DATA TYPES AND THEIR POLICIES
  job status (assigned -> in progress -> complete)
    -> operation-based: state transitions merge by
       taking the furthest-advanced state
    -> never regresses, no conflict in practice
  notes on a job (free text)
    -> conflicts surfaced to the user with both
       versions
    -> because silently discarding typed text is
       unacceptable
  photos attached
    -> append-only; both devices' photos are kept
    -> appends never conflict
  inventory counts
    -> increment operations, not absolute values
    -> "-3 units" merges; "now 17" does not

SYNC
  push the queue in order on connectivity
  pull with a cursor since the last sync
  exponential backoff on failure

UI
  per-item sync state: local only / syncing / synced
  a clear indicator before a shift ends, since the
    worker needs to know their work has left the device

RESULT
  fully usable with no signal for a full shift
  conflicts rare because most data is modelled as
    operations
  the remaining conflicts are text, where asking is
    the right answer
```

| Metric | Value | Note |
|---|---|---|
| Writes | always local | never blocked |
| Queue | durable | **survives app kill** |
| Counts | increments | merge cleanly |
| Text | ask the user | never discard |

> **Modelling changes as operations makes most conflicts disappear**  
> A conflict is only a conflict if the system records final values. If it records intent — add three units, append this photo, advance to the next state — then two offline devices produce operations that compose without contradiction. Most of what appears to be a hard conflict-resolution problem is an artefact of syncing values instead of operations, and reframing the data model removes it rather than solving it.

**When to use it**

- **Mobile applications** used where connectivity is unreliable or absent.
- **Field work** — warehouses, basements, remote sites, aircraft.
- **Any application where instant interaction matters**, since local-first is also the fastest design.
- **Note-taking, task and personal data apps**, where users expect to work anywhere.
- **Collaborative tools** that must tolerate a network dropping mid-session.

**When to avoid it**

- **Do not use it where the server must authorise every action**, such as financial transactions.
- **Do not use it where stale data is unsafe**, such as inventory that must not be oversold.
- **Do not use it for data too large to hold locally**, unless a subset can be selected.
- **Do not adopt it without choosing conflict policies**, since the default silently destroys work.
- **Do not retrofit it**, since the assumption of server authority is embedded throughout an online-first codebase.

**Advantages**

- **Works with no connectivity**, which is the point.
- **Instant interaction**, since nothing waits for a round trip.
- **Resilient to flaky networks**, where an online-first app degrades badly.
- **Lower server load**, as sync is batched rather than per action.
- **Better battery behaviour**, with fewer radio wake-ups than per-action requests.

**Disadvantages**

- **Conflicts are unavoidable** and resolution is genuinely difficult.
- **Substantially more complex**, with two stores, a queue and a sync engine.
- **Local storage limits** constrain how much data a device can hold.
- **Security is harder**, since data rests on devices you do not control.
- **Stale data is visible**, which some domains cannot tolerate.
- **Hard to test**, because the interesting cases involve partition timing.

**Trade-offs**

**Conflict policy trade-offs**

| Policy | Data loss | Complexity | Best for |
|---|---|---|---|
| Last-write-wins | Silent, unpredictable | Trivial | Preferences; single-device data |
| Field-level merge | Only on same-field edits | Moderate | Structured records |
| Operation-based merge | Rare by construction | Moderate | Counters, lists, state machines |
| CRDTs | None | High; metadata overhead | Collaborative text |
| Ask the user | None | Low code, high friction | Free text; irreconcilable cases |

The practical approach is per-type rather than global: operations for counters and state, append semantics for lists, user prompts for free text, and last-write-wins only for data whose loss nobody would notice. A single policy across all data is guaranteed to be wrong somewhere.

**How it fails**

**Offline sync failures**

| Failure | Cause | Fix |
|---|---|---|
| User's work silently disappears | Last-write-wins on text with skewed clocks | Per-type policy; surface conflicts on text |
| Mutations lost after app termination | Pending queue held only in memory | Durable queue persisted on every mutation |
| Sync takes minutes on reconnect | Full resynchronisation rather than incremental | Cursor-based incremental sync |
| Duplicate records after retry | Non-idempotent mutation application | Client-generated ids; idempotent server handling |
| Conflicts on every sync | Syncing values instead of operations | Model changes as operations |
| User unaware work is unsynced | No visible sync state | Per-item indicators; warn before device switch |
| Device storage exhausted | Unbounded local retention | Bound local data; evict synced history |

**Limits**

> **Practical considerations**
>
> - **Local storage**: device quotas are finite, so large datasets need a selected working subset.
> - **Queue durability**: mutations must be persisted at the moment they are made, not at sync time.
> - **Clock skew**: device clocks differ by minutes, which makes timestamp-based resolution unreliable.
> - **Sync cost**: incremental with a cursor, since full resynchronisation is impractical on mobile data.
> - **Conflict frequency**: falls dramatically when changes are modelled as operations rather than values.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Offline-first with sync | Unreliable connectivity | Conflicts; complexity |
| Online-first with cache | Mostly-connected use | Degrades badly offline |
| Read-only offline | Reference data | No offline writes; no conflicts |
| CRDT-based sync | Collaborative text | Metadata overhead; constrained model |
| Operational transformation | Realtime collaborative editing | Requires a central server |
| Optimistic UI only | Brief interruptions | Not true offline support |

Read-only offline is an underrated middle ground: caching data for offline viewing while requiring connectivity for writes removes the conflict problem entirely, and for many applications — reference material, dashboards, catalogues — it delivers most of the perceived benefit at a fraction of the cost.

**In real systems**

- **Note and task applications** are offline-first by default, because users expect to capture thoughts without checking for signal.
- **Field service and logistics apps** rely on it, since warehouses and remote sites routinely have no coverage.
- **Embedded device databases with sync layers** are the standard building block, providing local storage plus reconciliation.
- **CRDT-based collaborative editors** achieve convergence without a central arbiter, at the cost of metadata that grows with edit history.
- **Client-generated identifiers** are universal in offline systems, because records must exist locally before the server has seen them.

**Common mistakes**

- **Last-write-wins on user-authored text**, silently destroying edits.
- **Timestamp-based resolution** across devices with skewed clocks.
- **Pending queue held in memory**, losing mutations when the app is killed.
- **Full resynchronisation** on every reconnect, which is unusable on mobile data.
- **Server-generated ids**, which cannot work for records created offline.
- **Syncing values rather than operations**, manufacturing conflicts that need not exist.
- **No visible sync state**, so users switch devices believing work is saved.

**The staff-level view**

Offline-first is an architectural stance, not a feature, and it must be taken at the beginning.

- **Decide conflict policy per data type at design time.** A single global policy is wrong somewhere, and the wrong place is usually free text, where silent discarding destroys work users care about most.
- **Model changes as operations wherever possible.** Most conflicts are artefacts of syncing values; syncing intent makes counters, lists and state transitions merge without contradiction.
- **Treat queue durability as a correctness requirement.** Losing unsynced mutations when the operating system kills the app is the most common serious bug in offline applications, and it is invisible in testing.
- **Never resolve conflicts by device timestamp.** Clocks differ by minutes across devices, so resolution becomes arbitrary and unreproducible; logical versions remove the dependency.
- **Ask whether read-only offline suffices.** For many applications it provides most of the benefit and removes the conflict problem entirely, which is worth establishing before committing to full bidirectional sync.

**Go deeper**

Offline-first inverts the usual relationship: the application reads and writes only local storage, always succeeding immediately, while a sync engine reconciles with the server when connectivity permits. This makes the network an optimisation rather than a dependency, and incidentally makes the application feel instantaneous even when connected, since nothing waits for a round trip.

Conflicts are the defining problem, not an edge case. Two devices writing while partitioned will eventually modify the same data, and no design prevents it — only policies decide what happens. Last-write-wins is the tempting default and the dangerous one: it discards the losing version silently, and because it usually relies on device clocks that differ by minutes, the winner is arbitrary. For user-authored text this means work vanishing with no error and no reproducibility.

The most effective technique is to model changes as operations rather than values. Syncing intent — decrement by three, append this item, advance to the next state — lets two offline devices produce changes that compose correctly, so most conflicts stop existing rather than needing resolution. What remains, typically free text, is best surfaced to the user. Alongside that, the pending queue must be persisted at the moment of each mutation, since losing unsynced work when the operating system kills the app is the most common serious bug in offline applications.

Offline-first synchronisation treats local storage as authoritative for the application and the server as a replica to be reconciled with, rather than as the source of truth every operation must consult.

**The inversion is architectural, not incremental.** Because every read and write completes locally and immediately, the network becomes an optimisation. This cannot be retrofitted meaningfully: an online-first codebase embeds server authority in every operation, in error handling, in identifier generation and in the shape of its state management. The corollary benefit is that the design is also the fastest possible one — interactions never await a round trip — which is why local-first approaches are increasingly chosen for responsiveness even where connectivity is reliable.

**Conflicts are the substance of the problem.** Permitting offline writes guarantees that two devices will eventually modify the same data while partitioned, and no architecture prevents that; the only question is what happens next. The tempting default, keeping whichever version carries the later timestamp, fails twice over: it discards the other version silently, and it depends on device clocks that differ by minutes, so the surviving edit is determined by whose clock runs fast rather than by who edited last. For user-authored content this produces work that disappears without error and cannot be reproduced — the worst failure mode available.

**Modelling changes as operations eliminates most conflicts.** A conflict exists because the system records final values, which can contradict. Recording intent instead — decrement by three, append this photo, advance to the next state — produces changes that compose: both devices' operations apply and the result is correct without arbitration. Counters, lists and state machines all become conflict-free under this framing, which means a large share of what appears to be a difficult resolution problem is really a data-modelling choice made earlier and badly.

**Policy belongs per data type.** Different data has different correct answers: furthest-advanced wins for state transitions, append semantics for collections, increments for counters, and explicit user resolution for free text where silent discarding is unacceptable. Last-write-wins is defensible only for data whose loss nobody would notice, such as interface preferences. A single global policy is therefore guaranteed to be wrong somewhere, and it will be wrong in the place that matters most, because text is where users invest effort and where automatic resolution is least defensible.

**Durability of the pending queue is a correctness requirement.** Mutations must be persisted at the moment they are made rather than at sync time, because mobile operating systems terminate background applications without warning. A queue held in memory loses a user's entire offline session, silently, and the failure never appears in testing because test devices are not killed under memory pressure. Related mechanics follow from offline creation: identifiers must be generated on the client, since records exist before the server has seen them, and server handling must be idempotent because retries of queued mutations are routine.

**The strongest recommendation is often a smaller commitment.** Read-only offline — caching for viewing while requiring connectivity for writes — removes the conflict problem entirely and delivers most of the perceived benefit for reference material, catalogues and dashboards. Full bidirectional sync is a permanent architectural commitment with a problem that has no general solution, only per-type policies whose failure modes must each be individually acceptable. It is worth adopting when the domain genuinely requires offline writing, and worth declining when it was chosen because offline support sounded desirable.

**Prove it — interview questions**

1. **[Basic] What makes an application offline-first?**

   <details><summary>Model answer</summary>

   That the local store is the primary one and the server is a replica to reconcile with. The application reads and writes locally and every operation succeeds immediately, with a background sync engine pushing pending changes and pulling remote ones when connectivity allows. The network becomes an optimisation rather than a dependency — and a useful side effect is that the application feels instantaneous even on a perfect connection, because no interaction ever waits for a round trip.

   </details>

2. **[Basic] Why are conflicts unavoidable?**

   <details><summary>Model answer</summary>

   Because if two devices can both write while partitioned, they can both modify the same data before either sees the other's change. No design prevents this; designs only decide what happens afterwards. Treating conflicts as a rare edge case is the characteristic mistake, because they turn out to be common, and the default resolution — keeping whichever version has the later timestamp — silently discards the other, which for user-authored content means work vanishing with no error.

   </details>

3. **[Senior] Why is last-write-wins dangerous?**

   <details><summary>Model answer</summary>

   Two reasons that compound. It discards the losing version silently, so a user's edit disappears with no indication anything happened — which for typed text is destroying exactly the data people care most about. And it usually relies on device clocks, which differ by minutes, so the winner is whoever's clock runs fast rather than whoever edited last; a newer edit can lose to an older one. The combination produces work that vanishes unpredictably and cannot be reproduced. Logical versions remove the clock dependency, but only a different policy removes the silent discard.

   </details>

4. **[Senior] How does modelling changes as operations help?**

   <details><summary>Model answer</summary>

   It removes most conflicts rather than resolving them. If the client syncs a final value — inventory is now seventeen — then two offline devices produce contradictory statements that must be arbitrated. If it syncs intent — decrement by three, decrement by two — then both operations apply and the result is correct with no conflict at all. The same holds for appending to a list, or advancing a state machine, where taking the furthest-advanced state is unambiguous. A large fraction of what looks like a hard conflict-resolution problem is an artefact of recording values instead of intent.

   </details>

5. **[Staff] Design offline support for a field service app used in warehouses with no signal.**

   <details><summary>Model answer</summary>

   Local embedded database as the only thing the application touches, so every read and write is immediate and nothing ever blocks on the network, plus a pending mutation queue persisted at the moment each mutation is made — not at sync time, because an app killed by the operating system must not lose a shift's work, and that is the most common serious bug in this class of application. Sync pushes the queue in order and pulls incrementally with a cursor, since full resynchronisation over mobile data is unusable. The conflict policy is chosen per data type rather than globally: job state transitions merge by taking the furthest-advanced state so they never conflict in practice; inventory changes are modelled as increments rather than absolute values so two devices compose correctly; attached photos are append-only, and appends never conflict; and free-text notes surface both versions to the user, because silently discarding something a worker typed is not acceptable. Record identifiers are generated on the client, since records must exist before the server has seen them. And the interface shows per-item sync state with a clear indicator before a shift ends, because the worker needs to know their work has actually left the device.

   </details>

6. **[Principal] When would you argue against offline-first?**

   <details><summary>Model answer</summary>

   Whenever server authority is genuinely required, and whenever a cheaper option delivers most of the value. Financial transactions, inventory that must not be oversold, and anything requiring a real-time authorisation decision cannot be made offline-first without accepting that some accepted operations will later be rejected — which is a product problem, not a technical one, and usually a worse experience than an honest failure at the time. Beyond that, I would push hard on whether read-only offline suffices: caching data for offline viewing while requiring connectivity for writes eliminates the entire conflict problem, and for reference material, dashboards and catalogues it delivers nearly all the perceived benefit at a small fraction of the cost and risk. The reason to be firm about this is that offline-first is an architectural stance rather than a feature — it must be adopted at the beginning because server authority is embedded in every operation of an online-first codebase, and it brings a conflict-resolution problem that has no general solution, only per-type policies whose failure modes must each be acceptable. That is a large, permanent commitment, and it should be made because the domain requires it rather than because offline support sounded desirable.

   </details>

---

### Operational transformation

*Transform concurrent edits against one another so that every replica applies them in its own order and still converges on identical text.*

**Flow:** `Concurrent operations` → `Transform function` → `Server ordering` → `Convergent state` → `Intention preservation`

> **The 30-second version**  
> Rewrite concurrent operations against one another so every replica converges on identical text — with a central server for canonical order, a constrained operation set, and a tested library rather than a bespoke one.

**The problem**

Two people edit the same sentence simultaneously. One inserts a word at position five; the other deletes a character at position three. Each operation was computed against the document as its author saw it, but by the time it reaches the other replica, that document has changed — so applying it literally puts text in the wrong place.

Naive approaches all fail. Locking makes collaboration feel like taking turns. Last-write-wins discards one person's typing. Sending whole documents means the last save overwrites everything in between. What is needed is for both edits to apply, in either order, and produce the same sensible result.

> **Transform the operation, not the document**  
> If an operation arrives that was computed against an older state, adjust it to account for the changes that happened in between — shift an insertion position because a preceding character was deleted, for instance. Each replica transforms incoming operations against the ones it has already applied, and provided the transform function is correct, all replicas converge on identical content despite applying operations in different orders.

**Mental model**

Every edit is an operation with a position. When two operations are concurrent, each must be rewritten to account for the other's effect before being applied. A central server usually assigns a canonical order so that every client transforms against the same sequence.

1. **Operation** — An insert or delete at a position, plus the document version it was computed against.
2. **Concurrency** — Two operations are concurrent when neither author had seen the other.
3. **Transform** — Rewriting one operation to be correct after the other has been applied.
4. **Server order** — A canonical sequence that all clients agree on, which makes convergence tractable.
5. **Intention preservation** — The result should reflect what each author meant, not merely be identical everywhere.

> **Convergence is not the same as correctness**  
> All replicas ending up with identical text is necessary but insufficient — they could converge on nonsense. Intention preservation, the harder property, requires that each author's edit means what they meant after transformation. Transform functions that converge but mangle intent are a real and subtle class of bug, and they are hard to find because the system looks consistent while producing text nobody typed.

**How it works**

**Transformation by example**

```text
DOCUMENT  "HELLO"

Alice: insert "X" at position 5   -> "HELLOX"
Bob:   delete 1 char at position 1 -> "HLLO"
(both computed against "HELLO")

NAIVE APPLICATION, WRONG
  Alice applies Bob's delete:  "HELLOX" -> "HLLOX"  ok
  Bob applies Alice's insert at 5: "HLLO" has length 4
    -> position 5 is out of range, or lands wrongly
  -> replicas disagree

WITH TRANSFORMATION
  Bob transforms Alice's insert against his own delete
  the delete removed a character BEFORE position 5
  -> shift the insert position down by 1 -> position 4
  Bob applies insert "X" at 4: "HLLO" -> "HLLOX"
  Alice applies Bob's delete at 1: "HELLOX" -> "HLLOX"
  -> BOTH REPLICAS: "HLLOX"   converged

THE TRANSFORM RULE, INFORMALLY
  insert vs insert before  -> shift position
  insert vs delete before  -> shift position down
  delete vs delete same    -> one becomes a no-op
  equal positions          -> tie-break consistently
                              (e.g. by site id)

WHY THE TIE-BREAK MATTERS
  two inserts at the SAME position must be ordered
  the same way on every replica, or they converge
  differently
  -> the rule must be deterministic and global
```

1. **Route operations through a server for canonical order** — Peer-to-peer OT requires transform functions satisfying extra properties that are notoriously difficult to get right.
2. **Transform against everything applied since the operation's base version** — An operation is only correct relative to the state it was computed against.
3. **Tie-break deterministically** — Two operations at the same position must be ordered identically everywhere, or replicas diverge.
4. **Keep operations minimal and well-defined** — Rich operations multiply the number of transform pairs that must be correct.
5. **Test convergence exhaustively** — Property-based testing over random concurrent operation sequences finds the cases reasoning misses.
6. **Apply locally first, then send** — The author must see their edit immediately; the transform machinery runs behind that.

**Why OT is hard, and what CRDTs trade for it**

```text
THE TRANSFORM MATRIX
  for N operation types you need N x N transform
  functions
  insert/insert, insert/delete, delete/insert,
  delete/delete ... plus formatting, attributes,
  structural ops
  -> adding an operation type is quadratic work
  -> each pair must be correct under every ordering

THE CORRECTNESS CONDITIONS
  TP1: applying op1 then transformed-op2 equals
       applying op2 then transformed-op1
       -> required for two concurrent ops
  TP2: transformation is consistent regardless of the
       order operations are transformed against
       -> required for PEER-TO-PEER OT
       -> extremely hard to satisfy; several published
          algorithms were later shown incorrect

WHY A CENTRAL SERVER HELPS ENORMOUSLY
  the server imposes a total order
  -> only TP1 is needed, not TP2
  -> this is why almost every production OT system
     has a central server

CRDTs: THE ALTERNATIVE TRADE
  data structures that merge by construction, with no
  transform functions at all
  + no TP1/TP2 reasoning; peer-to-peer safe
  + convergence guaranteed
  - per-character metadata (unique ids, tombstones)
  - document size grows with edit history unless
    compacted
  -> OT: compact data, hard algorithm
     CRDT: simple algorithm, heavier data
```

> **Published OT algorithms have been wrong for years before anyone noticed**  
> The correctness conditions are subtle enough that several widely cited transformation algorithms were later shown to violate them in specific concurrent scenarios. That history is the strongest practical argument against implementing OT from scratch: use a well-tested library, keep a central server so the harder condition is unnecessary, and test convergence with randomised concurrent sequences rather than reasoning alone.

**Worked example**

A collaborative text editor, and where the design effort actually goes.

**Architecture**

```text
CLIENT
  apply the local edit IMMEDIATELY (no waiting)
  send the operation with its base version
  hold it as pending until acknowledged
  incoming operations:
    transform against pending local operations
    apply

SERVER
  receives an operation with base version V
  transforms it against everything applied since V
  applies it, assigns the next version
  broadcasts the transformed operation to all clients
  -> the server order is the canonical order

WHY THE LOCAL-FIRST STEP MATTERS
  typing must never wait for a round trip
  so the client is always slightly ahead of the server
  -> which is exactly why incoming operations must be
     transformed against the client's pending ones

CONVERGENCE CHECK
  periodically compare a document checksum across
  clients
  -> divergence is a serious bug and silent without
     this check
  -> on mismatch: resynchronise from the server

SCOPE CONTROL
  plain text insert/delete: small transform matrix
  add formatting, tables, embedded objects
    -> the matrix grows quadratically
    -> this is where OT implementations become
       unmaintainable

PRACTICAL ADVICE
  use a mature library
  keep the central server
  constrain the operation set deliberately
```

| Metric | Value | Note |
|---|---|---|
| Local edit | immediate | never waits |
| Server | canonical order | only TP1 needed |
| Transform pairs | N × N | **grows quadratically** |
| Checksum | periodic | detects divergence |

> **The operation set is the real design decision**  
> Transformation complexity grows quadratically with the number of operation types, so every addition — formatting, tables, embedded objects, structural moves — multiplies the pairs that must be correct under all orderings. OT implementations become unmaintainable through operation-set growth rather than through any single hard function, which makes deliberately constraining what counts as an operation the most consequential early choice.

**When to use it**

- **Collaborative text editing**, which is what OT was designed for and where it remains strong.
- **Applications with a central server**, since that removes the hardest correctness condition.
- **Where document size matters**, as OT carries no per-character metadata.
- **Character-level concurrent editing**, where coarse merging would be unacceptable.
- **Mature ecosystems with tested libraries**, rather than from-scratch implementations.

**When to avoid it**

- **Do not implement OT from scratch**, given the history of published algorithms proving incorrect.
- **Do not use OT peer-to-peer**, where the harder correctness condition applies.
- **Do not apply it to non-sequential data**, where CRDTs or simpler merging fit better.
- **Do not let the operation set grow unchecked**, since transform pairs grow quadratically.
- **Do not use it where coarse-grained conflict resolution suffices**, which is far simpler.

**Advantages**

- **Compact data**, with no per-character identifiers or tombstones.
- **Character-level concurrency**, so simultaneous typing works naturally.
- **Immediate local application**, so typing never waits for the network.
- **Mature and proven** in widely used collaborative editors.
- **Intention preservation** when transforms are correct, giving results authors expect.

**Disadvantages**

- **Transform functions are difficult to get right**, with a documented history of subtle errors.
- **Quadratic growth** in transform pairs as operation types increase.
- **Central server effectively required**, since peer-to-peer OT needs the harder condition.
- **Divergence is silent** without explicit checksum comparison.
- **Poor fit outside sequential data**, such as trees or sets.

**Trade-offs**

**OT against CRDTs**

| Aspect | Operational transformation | CRDTs |
|---|---|---|
| Algorithm complexity | High — transform matrix | Lower — merge by construction |
| Data overhead | Minimal | Per-character metadata and tombstones |
| Peer-to-peer | Very hard (needs TP2) | Natural |
| Central server | Effectively required | Optional |
| Document growth | Bounded by content | Grows with edit history unless compacted |
| Maturity for text | Long track record | Strong and improving |

The honest summary is that OT trades a hard algorithm for compact data, and CRDTs trade heavier data for a simpler algorithm. With a central server already in the design, that choice is closer than it appears, and library maturity and team familiarity usually matter more than the theoretical comparison.

**How it fails**

**OT failure modes**

| Failure | Cause | Fix |
|---|---|---|
| Replicas hold different text | Transform function violates TP1 | Property-based convergence testing; use a tested library |
| Divergence goes unnoticed | No consistency verification | Periodic document checksums; resync on mismatch |
| Text converges but is nonsense | Transform preserves convergence, not intent | Test intention preservation explicitly |
| Inconsistent results at equal positions | Non-deterministic tie-breaking | Deterministic global tie-break, e.g. by site id |
| Implementation becomes unmaintainable | Operation set grew; quadratic transform pairs | Constrain operation types deliberately |
| Peer-to-peer mode diverges | TP2 not satisfied | Route through a server for canonical ordering |
| Typing feels laggy | Waiting for server acknowledgment before applying | Apply locally first; transform incoming against pending |

**Limits**

> **Complexity reference**
>
> - **Transform pairs**: N × N for N operation types — the dominant maintenance cost.
> - **TP1** is required for any OT; **TP2** additionally for peer-to-peer, and is notoriously hard to satisfy.
> - **Data overhead**: minimal, unlike CRDTs which carry per-character metadata.
> - **Latency**: local application is immediate; the server round trip only affects other participants.
> - **Verification**: convergence must be checked explicitly, since divergence produces no error.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Operational transformation | Collaborative text with a server | Hard algorithm; central server |
| CRDTs | Peer-to-peer; offline-capable collaboration | Metadata overhead |
| Differential sync | Simpler collaboration needs | Coarser; less precise |
| Locking or turn-taking | Low concurrency | Poor experience |
| Last-write-wins on whole documents | Rarely concurrent editing | Silently loses work |
| Field-level merge | Structured records, not text | Not character-level |

For new projects the practical choice is usually between a mature OT library and a mature CRDT library rather than between the algorithms in the abstract — and where offline editing or peer-to-peer operation is required, CRDTs are the more natural fit regardless of the data-size comparison.

**In real systems**

- **Collaborative document editors** popularised OT, and the largest deployments still use it with a central server imposing canonical order.
- **Rich-text OT implementations** constrain their operation sets deliberately, because transform pairs grow quadratically with operation types.
- **CRDT-based editors** have become a credible alternative, trading per-character metadata for a far simpler correctness argument.
- **Periodic checksum comparison** is standard practice, since divergence between replicas produces no error on its own.
- **Published OT algorithms shown incorrect after years of use** are the standard cautionary example against bespoke implementations.

**Common mistakes**

- **Implementing transform functions from scratch**, where subtle errors are the norm.
- **Attempting peer-to-peer OT**, which requires the condition most implementations fail.
- **Non-deterministic tie-breaking** at equal positions, causing divergence.
- **Testing convergence only by reasoning**, rather than with randomised concurrent sequences.
- **Letting the operation set grow**, making the transform matrix unmaintainable.
- **No divergence detection**, so inconsistency is discovered by users.
- **Waiting for acknowledgment before applying local edits**, making typing feel laggy.

**The staff-level view**

OT is one of the few areas where the strongest engineering advice is not to implement it yourself.

- **Use a mature library.** The history of published algorithms later shown incorrect is not a reflection on the teams involved; it is evidence that the correctness conditions exceed what careful reasoning reliably delivers.
- **Keep the central server.** It reduces the requirement from two correctness conditions to one, and the harder of the two is where the documented failures live.
- **Constrain the operation set aggressively.** Maintenance cost grows quadratically with operation types, so every addition should be justified against the transform pairs it creates.
- **Verify convergence continuously.** Divergence is silent by construction — without periodic checksum comparison between replicas, the first evidence is a user reporting text they did not write.
- **Evaluate CRDTs honestly.** If peer-to-peer or offline editing is required, they are the better fit, and even with a server the simpler correctness argument often outweighs the metadata overhead.

**Go deeper**

Operational transformation resolves concurrent edits computed against different document versions. An insert at position five and a delete at position three are each correct for the state their author saw, but applying them literally elsewhere misplaces text. OT adjusts each incoming operation to account for what has been applied since its base version, so both edits take effect and all replicas converge.

Correctness is subtle. Convergence alone is insufficient — replicas can agree on text nobody typed — so intention preservation must be verified separately. Two operations at the same position require a deterministic global tie-break. And while a central server imposing canonical order reduces the requirement to the easier of the two transformation properties, peer-to-peer OT needs the harder one, which several published algorithms were later shown not to satisfy.

The practical advice follows from that history: use a mature library rather than implementing transforms, keep the central server, constrain the operation set because transform pairs grow quadratically with operation types, and run periodic checksum comparison since divergence is otherwise silent. Where offline or peer-to-peer editing is required, CRDTs are the better fit — trading per-character metadata for a much simpler correctness argument.

Operational transformation allows concurrent edits to a shared sequence to be applied in different orders on different replicas while producing identical, sensible results.

**Transform the operation, not the document.** Each edit is expressed as an operation with a position and the version it was computed against. When it arrives at a replica that has since applied other operations, it is rewritten — an insertion shifts because a preceding character was deleted, a delete becomes a no-op because the same character was already removed. Provided the transform function is correct, every replica converges regardless of the order in which operations arrive.

**Convergence and intention preservation are different properties.** Convergence means all replicas hold the same content; intention preservation means that content reflects what each author meant. A transform function can be convergent and still systematically misplace text, which produces the worst kind of failure: every replica agrees, so nothing looks wrong, while documents accumulate content nobody wrote. Testing must cover both, and only the first is checkable by comparing replicas.

**The correctness conditions are genuinely hard.** The first requires that applying two concurrent operations in either order yields the same result. The second, needed only for peer-to-peer operation, requires consistency regardless of the sequence an operation is transformed against — and it is difficult enough that several widely cited algorithms were later shown to violate it. That history is the decisive practical argument: these conditions exceed what careful reasoning reliably delivers, which is why production systems use tested libraries and a central server that makes the second condition unnecessary.

**The operation set governs maintainability.** Transform functions are needed for every ordered pair of operation types, so complexity grows quadratically. Plain insert and delete is tractable; adding formatting, attributes, tables, embedded objects and structural moves multiplies the pairs that must be correct under all orderings. OT implementations become unmaintainable through operation-set growth rather than through any individual difficult function, making deliberate constraint of what counts as an operation the most consequential early decision.

**Local-first application is required, and creates the pending-operation problem.** Typing must never wait for a server round trip, so the client applies its own edits immediately and is therefore always slightly ahead of the canonical state. That is precisely why incoming operations must be transformed against the client's own unacknowledged operations as well as against the server sequence — a step that is easy to overlook and produces divergence only under concurrent typing, which is to say only in production.

**CRDTs are the alternative trade, and the choice is closer than it looks.** They merge by construction, eliminating transform functions and working peer-to-peer naturally, at the cost of per-character identifiers and tombstones that grow documents with edit history unless compacted. OT keeps data compact and pays in algorithmic difficulty. Where a central server already exists, OT's hardest condition disappears and the comparison turns on data size against implementation simplicity; where offline or peer-to-peer editing is needed, CRDTs win regardless. In both cases the same operational discipline applies — periodic checksum comparison between replicas with automatic resynchronisation on mismatch — because divergence generates no error, and without that check the first evidence is a user reporting text they never wrote.

**Prove it — interview questions**

1. **[Basic] What problem does operational transformation solve?**

   <details><summary>Model answer</summary>

   Concurrent edits computed against different versions of a document. If one person inserts at position five and another deletes at position three, each operation was correct for the document its author was looking at, but by the time it reaches the other replica that document has changed — so applying it literally puts text in the wrong place. OT rewrites each incoming operation to account for the changes applied since its base version, so both edits take effect and every replica ends up with the same content.

   </details>

2. **[Basic] What does transformation actually do?**

   <details><summary>Model answer</summary>

   It adjusts an operation so it remains correct after another operation has been applied. The canonical example is an insert at position five arriving at a replica that has already deleted a character before that point: the insert is shifted to position four, so it lands where its author intended. The rules cover each pair of operation types — insert against insert, insert against delete, and so on — with a deterministic tie-break when two operations target the same position, since every replica must order them identically or they will diverge.

   </details>

3. **[Senior] Why do production OT systems almost always have a central server?**

   <details><summary>Model answer</summary>

   Because it reduces the correctness requirement from two conditions to one. With a canonical order imposed by a server, only the first transformation property is needed — that applying two concurrent operations in either order gives the same result. Peer-to-peer OT additionally requires the second property, concerning consistency regardless of what an operation is transformed against, which is notoriously difficult to satisfy; several published algorithms claiming to do so were later shown incorrect. A server is cheap insurance against the class of bug that has repeatedly caught expert implementers.

   </details>

4. **[Senior] Why is convergence insufficient as a correctness criterion?**

   <details><summary>Model answer</summary>

   Because replicas can agree on the wrong thing. Convergence only says every replica holds identical content; it says nothing about whether that content reflects what the authors meant. A transform function can be perfectly convergent while systematically placing insertions in the wrong position, producing text nobody typed but which every replica agrees on — so the system looks healthy while corrupting documents. Intention preservation is the harder property and needs testing in its own right, not just equality checks between replicas.

   </details>

5. **[Staff] How would you approach building a collaborative editor today?**

   <details><summary>Model answer</summary>

   By not implementing the algorithm. I would use a mature OT or CRDT library, because the documented history of published transformation algorithms later proven incorrect is strong evidence that these correctness conditions exceed what careful reasoning reliably produces. Architecturally: clients apply edits locally and immediately so typing never waits for the network, which necessarily leaves the client ahead of the server and is exactly why incoming operations must be transformed against the client's own pending ones. A central server assigns canonical order, which reduces the correctness requirement to the easier of the two conditions. I would constrain the operation set deliberately, since transform pairs grow quadratically and that growth — not any single function — is what makes these implementations unmaintainable. And I would run periodic document checksum comparison between clients with automatic resynchronisation on mismatch, because divergence produces no error and the alternative first signal is a user reporting text they did not write. If offline editing or peer-to-peer operation were required, I would choose CRDTs instead, accepting the metadata overhead for a substantially simpler correctness argument.

   </details>

6. **[Principal] How do you weigh OT against CRDTs?**

   <details><summary>Model answer</summary>

   The trade is a hard algorithm with compact data against a simpler algorithm with heavier data. OT carries no per-character metadata, so documents stay the size of their content, but the transform matrix must be correct for every operation pair under every ordering, and it grows quadratically as the operation set expands. CRDTs merge by construction, which removes the transform reasoning entirely and works peer-to-peer naturally, at the cost of unique identifiers and tombstones per character, so documents grow with edit history unless compacted. My practical position is that the theoretical comparison matters less than three concrete questions. Is there a central server? If yes, OT's hardest correctness condition disappears and the gap narrows considerably. Is offline or peer-to-peer editing required? If yes, CRDTs are the natural fit regardless of size. And what libraries and team familiarity exist? Because for either choice the right answer is to use a mature implementation, and the maturity available to a particular team is a more reliable predictor of success than the algorithmic properties. What I would not accept in either case is a bespoke implementation, given that this is a domain where expert practitioners have shipped incorrect algorithms that went undetected for years.

   </details>

---
