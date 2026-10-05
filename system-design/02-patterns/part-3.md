# Pattern Gym · Part 3 of 3

[← System Design index](../README.md)

> Realtime Delivery · Media Delivery · Media Storage · Storage Internals · Search · Scheduling (17 patterns). Part of the Pattern Gym — Learn to hear the pattern in the problem.

## Contents

- **Realtime Delivery** (3): [WebSocket](#websocket) · [Long Polling](#long-polling) · [Server Sent Events](#server-sent-events)
- **Media Delivery** (2): [CDN](#cdn) · [Adaptive Bitrate Streaming](#adaptive-bitrate-streaming)
- **Media Storage** (3): [Object Storage](#object-storage) · [Multipart Upload](#multipart-upload) · [Presigned URL](#presigned-url)
- **Storage Internals** (4): [Write Ahead Log](#write-ahead-log) · [MVCC](#mvcc) · [Bloom Filter](#bloom-filter) · [LSM Tree](#lsm-tree)
- **Search** (2): [Inverted Index](#inverted-index) · [Spatial Index](#spatial-index)
- **Scheduling** (3): [Priority Queue](#priority-queue) · [Delay Queue](#delay-queue) · [Durable Workflow](#durable-workflow)

## Realtime Delivery

### WebSocket

*Hold a persistent bidirectional connection so either side can send at any time, with no polling and no per-message request overhead.*

> **When you hear…** Bidirectional messaging · sub-second updates in both directions · high message rates where per-request overhead matters

**Flow:** `HTTP upgrade` → `Connection persists` → `Either side sends` → `Low per-message cost` → `Reconnect on drop`

**The problem**

A chat application built on polling asks the server for new messages every two seconds. Most requests return nothing, each carries full request and response headers, and a message still takes up to two seconds to appear.

Reducing the interval makes the waste worse without making it fast. At half-second polling the empty-request rate quadruples and messages still arrive up to half a second late, while the server handles four times the load to deliver the same content.

> **Pay for the connection once instead of for every message**  
> A connection that stays open removes both problems at once: the server pushes the moment something happens, so there is no interval, and each message carries only a few bytes of framing rather than a full set of headers. The cost moves from per-request overhead to holding state for every connected client, which is a much better trade wherever messages are frequent or latency matters.

**Mental model**

An HTTP request that is upgraded into a long-lived duplex channel, after which both ends send framed messages independently.

1. **Upgrade** — An ordinary HTTP request negotiates the protocol switch.
2. **Persist** — The connection remains open, held by both ends.
3. **Exchange** — Either side sends messages at any time, with minimal framing.
4. **Keep alive** — Pings detect dead connections that would otherwise linger.
5. **Reconnect** — The client re-establishes and recovers whatever it missed.

> **Connections are state, and state is what makes scaling hard**  
> Each connected client occupies memory, a file descriptor and a place in the server's connection table, so a server holds a bounded number of them and a deploy disconnects all of them at once. Every property that makes the protocol efficient per message makes the fleet harder to operate, and the difficulties are about connection management rather than message delivery.

**How it works**

**Connection cost and reconnection**

```text
PER-MESSAGE COST
  polling:    ~500-800 B of headers per request,
              both ways, plus TCP and TLS work
  websocket:  2-14 B of framing per message
  -> at high message rates the difference is orders
     of magnitude

PER-CONNECTION COST
  ~10-50 KB of memory per connection
  1 socket, 1 file descriptor
  100,000 connections -> 1-5 GB, plus tuned limits
  -> connection count, not message rate, is usually
     the binding constraint

RECONNECTION IS THE HARD PART
  networks drop connections constantly: mobile
  handovers, proxies, timeouts, deploys

  client must:
    reconnect with exponential backoff and JITTER
      without jitter, a deploy reconnects everyone
      simultaneously and the fleet is overwhelmed
    resume from a known position
      send the last received message id; the server
      replays what was missed
    handle duplicates on resume

HEARTBEATS
  ping every 30 s, expect a pong
  -> detects half-open connections that TCP will not
  -> also keeps intermediaries from idling the
     connection out
```

1. **Always send heartbeats** — Half-open connections look alive and deliver nothing; only pings reveal them.
2. **Reconnect with backoff and jitter** — Every client reconnecting at once after a deploy is a self-inflicted overload.
3. **Support resumption from a message position** — Without it, every reconnect loses whatever arrived during the gap.
4. **Plan for connection count, not message rate** — Memory and descriptors bound the fleet long before throughput does.
5. **Externalise routing state** — Which server holds which client must be shared, or messages cannot be delivered.
6. **Drain connections gradually on deploy** — Disconnecting everyone simultaneously produces a reconnection storm.

**Routing messages to the right server**

```text
THE PROBLEM
  user A is connected to server 3
  user B on server 7 sends A a message
  -> server 7 has no connection to A

OPTIONS
  1  SHARED PUB/SUB
       every server subscribes to channels for its
       connected users
       sender publishes; the holding server delivers
       -> simple, scales well, one extra hop

  2  CONNECTION REGISTRY
       a store maps user -> server
       sender looks up and forwards directly
       -> fewer messages, registry must be accurate
          and fast

  3  BROADCAST TO ALL SERVERS
       each checks whether it holds the user
       -> only viable with few servers

DEPLOYS
  rolling restart disconnects every client on each
  instance
  -> stagger, and drain slowly
  -> clients reconnect with jitter
  -> otherwise every deploy is a traffic spike

SCALING SHAPE
  connections spread across many instances
  -> adding capacity means more instances, and
     existing connections do not move
```

| Metric | Value | Note |
|---|---|---|
| Per message | 2-14 B | **vs ~600 B** |
| Per connection | 10-50 KB | the real limit |
| Hard part | reconnection | not messaging |
| Deploys | drain slowly | or storm |

> **Reconnection without jitter turns every deploy into a self-inflicted outage**  
> When a server restarts, every client it held reconnects — and if they all use the same backoff schedule they arrive together, overwhelming the remaining instances and causing more disconnections. The storm is entirely self-sustaining. Randomised backoff spreads the return across seconds or minutes and is the single most important client-side behaviour in any persistent-connection system.

**Technologies**

| Layer | Options | Note |
|---|---|---|
| Server implementations | Event-loop servers and frameworks | Connection-per-thread models do not scale here |
| Managed services | Hosted WebSocket gateways | Connection management as a service |
| Message routing | Redis pub/sub, a broker | Connects servers that hold different clients |
| Load balancers | Must support upgrade and long-lived connections | Idle timeouts are a common trap |
| Server-sent events | One-way alternative | Far simpler when only the server sends |
| Long polling | Fallback | Works through hostile intermediaries |

Load balancer idle timeouts cause a distinctive failure: connections silently closed after a fixed period of inactivity, which looks like an application bug and is configuration. Heartbeats at an interval shorter than that timeout prevent it, which is one more reason they are not optional.

**Trade-offs**

**Real-time delivery mechanisms**

| Mechanism | Direction | Overhead per message | Complexity |
|---|---|---|---|
| Polling | Client pulls | Full headers | Lowest |
| Long polling | Server pushes on a held request | Full headers per message | Low |
| Server-sent events | Server to client only | Minimal | Low |
| WebSocket | Bidirectional | Minimal | High |
| WebRTC data channels | Peer to peer | Minimal | Highest |

Server-sent events deserve consideration before WebSockets in most designs: if only the server sends, they provide the same push latency over ordinary HTTP, with automatic reconnection and resumption built into the protocol rather than implemented by hand.

> **Ask before choosing it**  
> Does the client actually need to send over the same channel? Notifications, live prices and progress updates are one-directional, and server-sent events deliver them with considerably less operational weight — the bidirectional capability is worth its cost only when it is genuinely used.

**How it fails**

**How WebSocket systems go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Reconnection storm after a deploy | Synchronised backoff | Jittered exponential backoff |
| Silent delivery failure | Half-open connection undetected | Heartbeats with a pong timeout |
| Connections dropped periodically | Load balancer idle timeout | Ping more often than the timeout |
| Messages lost across a reconnect | No resumption mechanism | Client sends its last received id; server replays |
| Server memory exhausted | Connection count exceeding capacity | Capacity planning by connections, not requests |
| Messages not delivered across servers | No cross-instance routing | Shared pub/sub or a connection registry |
| Backpressure ignored | Writing faster than the client reads | Bound the per-connection send buffer; disconnect slow clients |

> **A client that reads slower than the server writes will exhaust the server's memory**  
> Messages queued for a connection accumulate in a send buffer, and a client on a poor connection or one that has stopped reading lets that buffer grow without limit. A few thousand such clients consume the server's memory even though nothing appears to be failing. Bounding the buffer and disconnecting clients that exceed it is the only stable policy, and it must be in place before it is needed.

**Where it is used**

- **Chat and messaging platforms**, the archetypal bidirectional real-time use.
- **Collaborative editors**, where both sides send continuously and latency is felt directly.
- **Trading and live-price interfaces**, needing sub-second updates with client actions on the same channel.
- **Multiplayer games**, exchanging state in both directions many times per second.
- **Live dashboards with controls**, where the client both receives updates and issues commands.

**In the interview**

Spend the time on connection management; the protocol itself is the easy half.

- **Contrast per-message and per-connection costs** with numbers to show what the protocol actually changes.
- **Say reconnection is the hard part**, and require jittered backoff plus resumption from a position.
- **Insist on heartbeats**, because half-open connections appear healthy while delivering nothing.
- **Solve cross-server routing explicitly** with pub/sub or a registry; it is the question most designs omit.
- **Offer server-sent events** when the flow is one-directional, since the operational saving is substantial.

**Practice drill**

Design the connection layer for a chat system with 500,000 concurrent users across multiple servers. Compute the memory required and the number of instances. Describe how a message reaches a user connected to a different server. Then walk through a rolling deploy: what happens to the connections, what clients do, and exactly what prevents the reconnection from overwhelming the remaining instances.

**Go deeper**

A WebSocket is a persistent bidirectional connection established by upgrading an HTTP request, allowing either side to send at any time.

**It changes where the cost sits rather than removing it.** Polling pays full request overhead for every check, most of which return nothing; a persistent connection pays almost nothing per message and instead holds memory, a socket and a descriptor for every connected client. That trade is strongly favourable when messages are frequent or latency matters, and unfavourable for a client that needs an update every few minutes.

**Connection count, not message throughput, is the binding constraint.** Tens of kilobytes per connection means a server holds tens or hundreds of thousands, and capacity planning is about how many clients can be connected rather than how many messages flow. This reverses the usual instinct: a system can be entirely idle in message terms and still be at its limit.

**Reconnection is where the real engineering lives.** Networks drop connections constantly — mobile handovers, proxy timeouts, deploys — so a persistent-connection system spends most of its complexity on re-establishing rather than on messaging. Jittered backoff prevents clients returning in unison, and resumption from a known message position prevents every reconnect from silently losing whatever arrived during the gap. Neither is optional at scale.

**Half-open connections are invisible without heartbeats.** A connection whose peer has vanished can remain open indefinitely from the server's perspective, consuming resources and appearing healthy while delivering nothing. Application-level pings are what reveal this, and they simultaneously prevent intermediaries from closing idle connections — which is a distinct failure that presents identically and is entirely a configuration problem.

**Cross-server routing is the design question most omissions leave out.** With clients spread across many instances, a message from one user to another must reach whichever server holds the recipient's connection, which requires either a shared pub/sub layer or a registry mapping users to servers. Neither is difficult, and a design that describes the protocol without addressing this has not yet described a system that works with more than one server.

**Slow clients threaten the server's memory, not their own experience.** Messages queued for a connection accumulate in a send buffer, and a client that reads slowly lets that buffer grow while nothing appears to be wrong. Bounding it and disconnecting clients that exceed the bound is the only stable policy — and because the symptom is server memory growth rather than a delivery error, it is usually discovered during an incident rather than during design.

**Related patterns:** Server Sent Events · Long Polling · Work Queue · Backpressure

---

### Long Polling

*Hold a request open until data is available or a timeout expires, giving push-like latency using only ordinary HTTP requests.*

> **When you hear…** Push needed through restrictive networks · infrastructure that only understands request-response · a fallback for persistent connections

**Flow:** `Client requests` → `Server holds it` → `Data arrives` → `Respond immediately` → `Client requests again`

**The problem**

Push is needed, but the environment does not cooperate: a corporate proxy blocks protocol upgrades, an old load balancer terminates long-lived connections, or the client is a system that only knows how to make requests.

Regular polling is the fallback everyone reaches for, and it forces a choice between latency and waste — short intervals produce mostly empty responses, long intervals produce stale data, and neither is satisfactory.

> **A request that waits is a push in disguise**  
> Instead of the server answering immediately with nothing, it holds the request until there is something to say. The client still made an ordinary request and receives an ordinary response, so every proxy, gateway and firewall treats it as normal traffic — but the data arrives the instant it exists, which is what push latency means.

**Mental model**

A request/response cycle where the response is deferred. The client asks, the server waits, and the loop repeats immediately after each answer.

1. **Request** — The client asks for anything newer than what it already has.
2. **Wait** — The server holds the request rather than answering emptily.
3. **Deliver** — It responds as soon as data appears.
4. **Time out** — If nothing arrives within the limit, it responds empty.
5. **Repeat** — The client immediately issues the next request.

> **The gap between responses is a window in which events can be missed**  
> Between receiving a response and issuing the next request, the client is not connected. Anything published in that window is lost unless the server tracks per-client position and the client tells it where it left off — which makes cursors part of the protocol rather than an enhancement.

**How it works**

**The loop and the gap**

```text
CLIENT
  loop:
    GET /poll?since=<cursor>
    on response:
      process events
      advance cursor
    immediately repeat

SERVER
  on request:
    if events newer than cursor exist:
      return them now
    else:
      hold the request
      wait up to 30 s for a new event
      return it, or return empty at the timeout

THE GAP
  response sent  -----> client processes ----->
  next request arrives
  events published in this window are not delivered
  by that connection
  -> the CURSOR is what recovers them: the next
     request asks for everything since the last
     known position
  -> without a cursor, long polling loses messages
     routinely

TIMEOUT CHOICE
  too long   intermediaries kill the connection
  too short  request overhead dominates
  20-30 s is the usual compromise, chosen to sit
  below typical proxy idle limits

HOLDING COSTS A CONNECTION
  same resource profile as a persistent connection
  plus a full request cycle per message
```

1. **Make cursors part of the protocol** — The gap between requests loses events without them.
2. **Choose a timeout below proxy idle limits** — Otherwise intermediaries close the request and the client sees an error.
3. **Do not block a thread per held request** — Asynchronous handling is required; a thread per waiting client does not scale.
4. **Return immediately when data already exists** — Waiting when there is something to say adds pointless latency.
5. **Back off after errors, with jitter** — Reconnection storms apply here exactly as to any persistent mechanism.
6. **Treat it as a fallback** — Where WebSockets or event streams work, they are better.

**Cost comparison**

```text
REGULAR POLLING, every 2 s
  1,800 requests/hour per client
  almost all empty
  latency: up to 2 s

LONG POLLING, 30 s timeout, 1 message/minute
  ~60 requests/hour per client (one per message,
  plus timeouts)
  latency: near zero
  -> 30x fewer requests AND better latency

LONG POLLING, high message rate (10/s)
  each message ends a request and starts a new one
  -> 36,000 requests/hour per client
  -> full request overhead per message
  -> WORSE than polling in efficiency terms
  -> this is where a persistent connection wins
     decisively

SERVER-SIDE
  held requests occupy connections just like
  WebSockets
  -> must be handled asynchronously
  -> a thread per held request exhausts the pool at
     a few thousand clients
```

| Metric | Value | Note |
|---|---|---|
| Latency | near zero | like push |
| Low rate | 30× fewer requests | **than polling** |
| High rate | worse than polling | overhead per message |
| Requirement | async handling | not a thread each |

> **A thread held per waiting request exhausts the server at a few thousand clients**  
> The mechanism depends on keeping many requests open simultaneously, and a server that dedicates a thread to each one runs out at a few thousand — far below the connection count the same hardware could otherwise sustain. Asynchronous request handling, where a waiting request consumes only its connection, is a prerequisite rather than an optimisation.

**Technologies**

| Context | Use | Note |
|---|---|---|
| Fallback layers | When upgrades are blocked | Libraries often degrade to this automatically |
| Async servers | Holding many requests cheaply | Essential; synchronous models do not work |
| API-based notifications | Systems that only make requests | No client library needed |
| WebSocket | The preferred alternative | Better for bidirectional or high-rate traffic |
| Server-sent events | The preferred one-directional alternative | Similar simplicity, no gap between responses |
| Regular polling | The simpler fallback | Predictable, wasteful, higher latency |

Real-time libraries commonly implement a negotiation that prefers a persistent connection and falls back to long polling where it fails, which is why the pattern remains widely deployed even in systems whose engineers never chose it explicitly.

**Trade-offs**

**Delivery mechanisms compared**

| Mechanism | Latency | Overhead at low rates | Overhead at high rates |
|---|---|---|---|
| Polling | Up to the interval | High | Moderate |
| Long polling | Near zero | Low | High |
| Server-sent events | Near zero | Low | Low |
| WebSocket | Near zero | Low | Lowest |

The two right-hand columns explain where long polling belongs: excellent for infrequent updates, poor for frequent ones, because every message costs a complete request cycle. Message rate, more than anything else, decides whether it is the right fallback or the wrong choice.

> **Ask before choosing it**  
> Why can a persistent connection not be used here? If the answer is a genuine infrastructure constraint, long polling is the correct response; if it is unfamiliarity with WebSockets or event streams, the constraint is worth removing instead.

**How it fails**

**How long polling goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Events lost intermittently | No cursor across the gap | Client sends its position; server replays from it |
| Requests killed by intermediaries | Timeout longer than proxy idle limits | Hold for 20-30 seconds at most |
| Server exhausted at low client counts | A thread per held request | Asynchronous request handling |
| Overhead worse than polling | High message rate | Switch to a persistent connection |
| Reconnection storm | All clients retrying together after an error | Jittered backoff |
| Duplicate processing | Cursor not advanced atomically with handling | Advance only after successful processing |
| Latency higher than expected | Server waiting even when data exists | Return immediately if there is anything to send |

> **Without a cursor, the gap between responses silently drops messages**  
> The client is disconnected between receiving a response and sending the next request, and anything published during that interval is not delivered. Because the window is short and the loss is intermittent, it typically presents as occasional missing notifications that nobody can reproduce — and the fix is not a shorter gap, which cannot be eliminated, but a position the next request carries so the server can replay what was missed.

**Where it is used**

- **Real-time libraries**, which negotiate down to long polling where upgrades are blocked.
- **Corporate and restricted networks**, where proxies interfere with anything unusual.
- **API-based notification delivery**, for clients that can only make requests.
- **Legacy infrastructure**, where load balancers do not support persistent connections.
- **Simple in-house implementations**, where the whole mechanism is a loop and a cursor.

**In the interview**

Present it as the compatibility fallback, and be precise about the gap and the message-rate limit.

- **Explain the deferred response** and why it produces push latency over ordinary request-response.
- **Name the gap between responses** and require cursors, since that is where messages are lost.
- **Give the message-rate crossover**: excellent at low rates, worse than polling at high ones.
- **Require asynchronous handling**, because a thread per held request exhausts the server early.
- **Choose a timeout below proxy idle limits**, which is why 20 to 30 seconds is conventional.

**Practice drill**

Deliver notifications to 10,000 clients behind a corporate proxy that blocks protocol upgrades. Design the long-polling loop on both sides, including the cursor, the timeout and the error backoff. Compute the request volume at one message per minute per client and at ten per second, and say at which point you would insist on a different mechanism. Then describe precisely how a message published during the gap still reaches the client.

**Go deeper**

Long polling delivers push-like latency using ordinary HTTP by holding each request open until there is something to return.

**Its value is compatibility rather than efficiency.** Every proxy, gateway and firewall understands a request and a response, so the mechanism works in environments that block protocol upgrades or terminate long-lived connections. That is the entire reason it persists alongside better-performing alternatives — it works where they do not, which for some client populations is decisive.

**The gap between responses is a structural property, not an implementation flaw.** After the server answers, the client is disconnected until it issues the next request, and anything published in that interval belongs to no connection. Cursors are therefore part of the protocol: the client states what it last received and the server replays from there. Implementations that omit this lose messages intermittently, in a way that is very difficult to reproduce from a report.

**Message rate determines whether it is efficient or wasteful.** At one message a minute it produces far fewer requests than interval polling and delivers with near-zero latency, which is an unambiguous improvement. At ten messages a second, each one terminates a request and begins another, so the full HTTP overhead is paid per message and the mechanism performs worse than the polling it replaced. The crossover is the main thing to reason about when choosing it.

**Server-side it costs as much as a persistent connection, plus request overhead.** Held requests occupy connections exactly as WebSockets do, so capacity planning is the same — and the model only works at all with asynchronous request handling, because dedicating a thread to each waiting client exhausts the pool at a small fraction of the achievable connection count. Synchronous frameworks fail at this early and for reasons that look like a capacity problem rather than an architectural one.

**Timeout selection is bounded by infrastructure rather than by preference.** Holding a request longer reduces overhead and increases the chance an intermediary closes it, at which point the client sees an error rather than an empty response. Twenty to thirty seconds is conventional because it sits comfortably below common proxy idle limits, and the value is really a statement about the network the clients are on rather than about the application.

**It should be chosen as a fallback, not as a default.** Where persistent connections work, both WebSockets and event streams deliver the same latency with less overhead and no gap to compensate for. The right reason to implement long polling is a specific environment that rejects the alternatives — and the right implementation is usually a negotiated degradation, so that clients which can do better are not held to the weakest option.

**Related patterns:** WebSocket · Server Sent Events · Retry With Jitter · Work Queue

---

### Server Sent Events

*Stream updates from server to client over one long-lived HTTP response, with reconnection and resumption handled by the browser.*

> **When you hear…** One-directional server updates · push without the weight of WebSockets · standard HTTP infrastructure preferred

**Flow:** `Client opens stream` → `Server holds response` → `Events pushed as text` → `Browser auto-reconnects` → `Last-Event-ID resumes`

**The problem**

A dashboard needs live updates from the server. The client never sends anything over that channel — it only receives — and adopting a bidirectional protocol brings connection management, reconnection logic and a separate infrastructure path for a capability that will not be used.

Polling remains wasteful, and the reconnection logic that any persistent connection requires is exactly the code that most implementations get wrong.

> **For one-directional push, the existing protocol already suffices**  
> An HTTP response that is never closed is a stream, and the browser's built-in client handles reconnection and resumption automatically. The push latency matches any other persistent connection, everything in the HTTP path continues to work unchanged, and the reconnection code that is the main source of bugs elsewhere is provided by the platform rather than written by hand.

**Mental model**

A normal HTTP GET whose response never ends, delivering a stream of text events that the browser parses and dispatches.

1. **Open** — The client requests a stream endpoint like any other resource.
2. **Hold** — The server keeps the response open.
3. **Send** — Events are written as plain text with optional identifiers.
4. **Reconnect** — The browser reconnects automatically if the connection drops.
5. **Resume** — It sends the last event identifier so the server can replay what was missed.

> **It is one-directional; anything the client sends is a separate request**  
> Client-to-server communication requires an ordinary HTTP request, so an interactive feature involves two mechanisms with independent lifecycles. That is perfectly reasonable when client sends are occasional, and it becomes awkward when the interaction is genuinely conversational — at which point a bidirectional protocol is the better fit.

**How it works**

**The wire format and what it provides**

```text
REQUEST
  GET /events
  Accept: text/event-stream

RESPONSE  (never closed)
  Content-Type: text/event-stream
  Cache-Control: no-cache

  id: 1042
  event: price
  data: {"symbol":"ABC","price":41.2}

  id: 1043
  event: price
  data: {"symbol":"XYZ","price":9.8}

  : this is a comment, used as a keepalive

BUILT IN, NO CODE REQUIRED
  automatic reconnection on drop
  Last-Event-ID header sent on reconnect
    -> the server replays from that point
  retry: <ms> to control the backoff
  -> the two things WebSocket clients must implement
     by hand are provided here

SERVER OBLIGATIONS
  replay from Last-Event-ID
    requires a buffer of recent events per stream
  send periodic comments as keepalives
    prevents proxies idling the connection out
  disable response buffering
    a buffering proxy holds events until the buffer
    fills, which destroys the latency benefit
```

1. **Assign an identifier to every event** — Resumption depends entirely on it.
2. **Keep a replay buffer of recent events** — The browser will ask to resume; the server must be able to.
3. **Send periodic comment lines as keepalives** — Intermediaries close connections that appear idle.
4. **Disable buffering in every proxy on the path** — A buffering proxy silently converts streaming into batching.
5. **Use the retry field to control reconnection** — It is the only backoff control the protocol offers.
6. **Account for browser connection limits** — Older HTTP/1.1 limits constrain how many streams one origin can hold.

**Where it fits, and its limits**

```text
GOOD FIT
  notifications and alerts
  live dashboards and metrics
  progress updates for long operations
  price and score feeds
  activity streams
  -> all one-directional

POOR FIT
  chat and collaboration     bidirectional
  binary payloads            text-only format
  very high message rates    text framing overhead
  low-latency gaming         wrong tool entirely

HTTP/1.1 CONNECTION LIMIT
  browsers allow ~6 connections per origin
  each open stream consumes one
  -> a few streams starve ordinary requests
  -> HTTP/2 multiplexes, and removes this entirely
  -> with HTTP/1.1, use ONE stream and multiplex
     event types over it

SERVER COST
  similar to WebSocket: one held connection per
  client
  -> same capacity planning, same connection limits
  -> the saving is in client code and infrastructure
     compatibility, not in server resources
```

| Metric | Value | Note |
|---|---|---|
| Direction | server to client | **one way** |
| Reconnection | built in | no client code |
| Resumption | Last-Event-ID | server must buffer |
| Server cost | same as WebSocket | one connection each |

> **Any buffering proxy on the path converts a stream into a batch**  
> Reverse proxies frequently buffer responses by default, which means events accumulate until the buffer fills or the connection closes rather than being delivered as they are written. The feature appears to work in development and behaves as slow batching in production, with no error anywhere — and it is one of the most common reasons a correctly implemented stream feels broken.

**Technologies**

| Piece | Options | Note |
|---|---|---|
| Client | Native EventSource | Reconnection and resumption included |
| Server | Any HTTP server that supports streaming | Response buffering must be off |
| Proxies | Must be configured not to buffer | The most common deployment problem |
| HTTP/2 | Multiplexed streams | Removes the per-origin connection limit |
| WebSocket | Bidirectional alternative | More capability, more client code |
| Long polling | Fallback | Works where streaming is blocked entirely |

HTTP/2 materially changes the calculus: without the six-connection limit, several independent streams per origin become practical, and the main historical argument for choosing WebSockets over this for one-directional data disappears.

**Trade-offs**

**Server-sent events against WebSocket**

| Property | SSE | WebSocket |
|---|---|---|
| Direction | Server to client | Bidirectional |
| Client reconnection | Automatic | Implemented by hand |
| Resumption | Built into the protocol | Implemented by hand |
| Payload | Text only | Text or binary |
| Infrastructure compatibility | Ordinary HTTP | Requires upgrade support |
| Server connection cost | One per client | One per client |

The two reconnection rows are the practical argument. Hand-written reconnection with backoff and resumption is where persistent-connection bugs concentrate, and the protocol providing it correctly removes an entire class of defects.

> **Ask before choosing it**  
> Does the client need to send on this channel, and is the payload binary? Two noes make this the simpler choice; either yes points to WebSockets, and hedging by adopting the heavier protocol for a one-directional feature is a common over-engineering.

**How it fails**

**How server-sent events go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Events arrive in batches | A proxy buffering the response | Disable buffering on every hop |
| Connection dropped every minute | Proxy idle timeout | Send keepalive comments regularly |
| Events lost across reconnects | No identifiers or no replay buffer | Assign ids; buffer recent events |
| Other requests to the origin stall | HTTP/1.1 connection limit consumed | One multiplexed stream, or HTTP/2 |
| Reconnection storm | Default backoff identical for all clients | Set retry with jitter server-side |
| Server memory exhausted | Connection count underestimated | Plan capacity by connections |
| Binary data corrupted | Encoding binary into a text protocol | Use WebSocket, or encode explicitly |

> **Resumption is only as good as the server's replay buffer**  
> The browser faithfully sends the last event identifier it saw, and a server that keeps no history simply resumes from the present — so the client silently misses everything that occurred during the disconnection. The protocol provides the mechanism and the server must provide the data, and implementations that ignore the header have automatic reconnection without automatic recovery, which is easy to mistake for correctness.

**Where it is used**

- **Live dashboards and monitoring**, where updates flow one way and payloads are small.
- **Notification delivery**, pushing alerts to open browser sessions.
- **Progress reporting**, streaming the status of long-running operations.
- **Price and score feeds**, updating continuously with no client input.
- **Streaming responses from language models**, delivering tokens as they are produced.

**In the interview**

Position it as the simpler choice for one-directional push, and name the deployment traps.

- **Say the protocol provides reconnection and resumption**, which are the two things WebSocket clients implement by hand and get wrong.
- **Note that the server must maintain a replay buffer** for Last-Event-ID to mean anything.
- **Raise proxy buffering** as the deployment failure that turns streaming into batching with no error.
- **Mention the HTTP/1.1 connection limit** and that HTTP/2 removes it.
- **Be clear that server cost is the same** — one held connection per client — so the saving is in complexity, not resources.

**Practice drill**

A dashboard needs live metric updates for 50,000 concurrent users. Design the stream: the event format, identifiers, keepalives and the replay buffer. State what the client does on disconnection and what the server must do to honour it. Then list every place a proxy could break the stream and how you would verify, before launch, that none of them does.

**Go deeper**

Server-sent events deliver a stream of updates over one long-lived HTTP response, with reconnection and resumption defined by the protocol.

**It fits the one-directional case exactly, which is most real-time features.** Notifications, dashboards, progress reports and price feeds flow only from server to client, and adopting a bidirectional protocol for them brings connection management and client-side reconnection logic for a capability that goes unused. Matching the mechanism to the actual direction of data removes work rather than capability.

**The protocol provides precisely the parts that are usually implemented badly.** Automatic reconnection and resumption via a last-event identifier are exactly what hand-written persistent-connection clients get wrong — missing jitter, losing messages across gaps, reconnecting in unison. Having them specified and implemented by the browser eliminates a class of bugs that otherwise has to be rediscovered in every project.

**Resumption is a shared responsibility that is half-implemented surprisingly often.** The client sends the identifier of the last event it received, and the server must maintain enough recent history to replay from that point. A server ignoring the header still appears to work — the connection re-establishes and events resume — while silently dropping everything that occurred during the gap, which is easy to mistake for correct behaviour until someone notices missing data.

**Its most common deployment failure is invisible and infrastructural.** Reverse proxies that buffer responses accumulate events instead of forwarding them, converting a stream into periodic batches with no error reported anywhere. The feature works in development, where no such proxy exists, and feels sluggish in production for reasons that look like application behaviour — so verifying the full path before launch is part of implementing the pattern.

**Server resource cost is the same as any persistent connection.** One held connection per client means the same memory, descriptor and capacity-planning considerations as WebSockets, so the saving is in client code and infrastructure compatibility rather than in server load. Describing it as the lightweight option is accurate about complexity and misleading about capacity.

**Its historical limitation has largely disappeared.** The six-connection-per-origin limit in HTTP/1.1 meant open streams could starve ordinary requests, which pushed many teams towards WebSockets for reasons unrelated to direction. With HTTP/2 multiplexing, that constraint is gone, and the choice reduces to the genuine question — whether the client needs to send on the same channel, and whether the payload is binary.

**Related patterns:** WebSocket · Long Polling · CDN · Backpressure

---

## Media Delivery

### CDN

*Serve content from caches near users, so most requests never reach the origin and distance stops dominating latency.*

> **When you hear…** Users far from the origin · static assets and media dominating traffic · origin bandwidth or load becoming the constraint

**Flow:** `User requests` → `Nearest edge` → `Cache hit served` → `Miss fetches origin` → `Cached for the next user`

**The problem**

A two-megabyte image served from a single region takes hundreds of milliseconds to reach a user on another continent before a single byte of content is transferred, and every user pays that cost independently for identical bytes.

The origin also pays. Serving the same asset a million times consumes bandwidth and connections that could be doing work only the origin can do, and a traffic spike on static content can exhaust capacity needed for dynamic requests.

> **Identical bytes requested by many people should be stored near them**  
> Content that does not vary per user can be copied to locations close to users, where it is served without ever contacting the origin. Latency falls because the distance falls, and origin load falls because most requests terminate at the edge — both from the same change, which is why this is usually the single highest-leverage performance improvement available.

**Mental model**

A global network of caches. A request goes to the nearest one; if it holds the content it answers, otherwise it fetches from the origin and keeps a copy.

1. **Route** — DNS or anycast directs the user to a nearby edge.
2. **Check** — The edge looks for a valid cached copy.
3. **Serve or fetch** — A hit is answered locally; a miss goes to the origin.
4. **Store** — The response is cached according to its headers.
5. **Expire** — Content ages out, is revalidated, or is purged explicitly.

> **Cached content is difficult to recall once distributed**  
> A wrong or sensitive response cached at hundreds of edges is present in hundreds of places, and purging is neither instant nor guaranteed to be complete. This makes cache-control headers a correctness concern rather than a tuning detail: caching something user-specific by accident can serve one user's data to another, and the mistake propagates globally within seconds.

**How it works**

**Cache keys, TTLs and invalidation**

```text
CACHE KEY
  by default: method + host + path (+ query)
  must also vary on anything that changes the
  response:
    Accept-Encoding   (gzip vs brotli)
    device class      (if responses differ)
  -> every extra dimension multiplies the number of
     cached objects and lowers the hit rate

NEVER CACHE BY DEFAULT
  anything varying by user
  anything behind authentication
  -> Cache-Control: private, no-store
  -> a single misconfiguration here leaks data
     between users

TTL STRATEGY
  immutable assets (hashed filenames)
    Cache-Control: public, max-age=31536000, immutable
    -> cache forever; a new version has a new name
  html and frequently changing content
    short max-age + stale-while-revalidate
    -> fresh enough, and always fast
  api responses
    usually private, or very short TTLs

INVALIDATION
  purge         explicit, propagates in seconds to
                minutes, not instant
  versioned url best method: a new name is a new
                object, so nothing needs purging
  short ttl     simplest; content is briefly stale

-> versioned URLs remove the invalidation problem
   entirely and should be the default for assets
```

1. **Use content-hashed filenames for static assets** — Immutable objects can be cached forever and never need purging.
2. **Set cache headers explicitly on every response** — Defaults differ between providers, and the wrong default leaks data.
3. **Keep the cache key as narrow as correctness allows** — Each variation dimension fragments the cache and lowers hit rates.
4. **Use stale-while-revalidate for changing content** — Users get an instant response while the edge refreshes behind them.
5. **Enable origin shielding** — A designated tier absorbs edge misses so the origin sees far fewer.
6. **Monitor hit rate by content type** — An aggregate hit rate hides the specific thing that stopped caching.

**What a CDN actually saves**

```text
LATENCY
  origin in one region, user on another continent
    RTT ~250 ms before any data moves
  nearest edge
    RTT ~10-30 ms
  -> for a multi-request page, this compounds

ORIGIN LOAD
  1,000,000 requests for one asset
  90% hit rate  -> 100,000 reach the origin
  99% hit rate  -> 10,000
  99.9%         -> 1,000
  -> the last percent of hit rate matters more than
     the first ninety

BANDWIDTH
  origin egress is usually the most expensive
  bandwidth you buy
  -> moving it to the edge is often the single
     largest cost saving available

THUNDERING HERD ON A MISS
  a popular object expires
  many edges miss simultaneously
  -> the origin receives a burst
  -> mitigations: request collapsing at the edge,
     origin shielding, staggered TTLs

WHAT IT DOES NOT HELP
  personalised responses
  write requests
  anything requiring the origin's state
```

| Metric | Value | Note |
|---|---|---|
| Latency | 250 ms → 20 ms | **by distance** |
| Origin load | hit rate dependent | 99% → 1% remains |
| Best practice | hashed filenames | no purging needed |
| Risk | caching private data | propagates globally |

> **Caching an authenticated response serves one user's data to everyone**  
> A response that varies by user, cached without a private directive or a user-specific key, is stored at an edge and returned to whoever asks next. The failure is immediate, global and difficult to detect from the inside — the application behaves correctly and the cache is doing exactly what it was told. Explicit cache-control on every authenticated path is what prevents it, and it belongs in the framework rather than in individual handlers.

**Technologies**

| Layer | Options | Note |
|---|---|---|
| Global CDN providers | Managed edge networks | The standard choice; differ in features and pricing |
| Origin shielding | An intermediate cache tier | Collapses edge misses before the origin |
| Edge compute | Functions running at the edge | Personalisation without reaching the origin |
| Image optimisation at the edge | Format and size negotiation | Reduces bytes as well as distance |
| Browser cache | The layer before the CDN | Free, and frequently under-used |
| Object storage with a CDN | Standard static hosting | Cheap origin, cached everywhere |

The browser cache deserves mention because it is the only layer that eliminates the request entirely. Long-lived caching of hashed assets means a returning user makes no request at all, which beats even the fastest edge.

**Trade-offs**

**Serving strategies**

| Approach | Latency | Origin load | Freshness |
|---|---|---|---|
| Origin only | Distance-bound | Full | Always current |
| CDN with short TTL | Low | Reduced | Seconds stale |
| CDN with long TTL | Low | Minimal | Until expiry or purge |
| CDN with versioned URLs | Low | Minimal | Always current |
| Stale-while-revalidate | Low | Minimal | Briefly stale, self-refreshing |

Versioned URLs are the strongest row: because a change produces a new name, content can be cached indefinitely while remaining always current, which is why hashed asset filenames are effectively universal in modern build tooling.

> **Ask before choosing it**  
> What proportion of bytes served are identical for every user? If most traffic is static assets and media, a CDN is the highest-leverage change available; if most is personalised, the benefit is confined to the shared remainder and edge compute becomes the more relevant tool.

**How it fails**

**How CDNs go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Private data served to other users | Authenticated response cached publicly | Explicit private or no-store on those paths |
| Stale content after a deploy | Long TTL with no versioning | Hashed filenames, or purge on release |
| Poor hit rate | Cache key varying on too many dimensions | Normalise the key; strip irrelevant query parameters |
| Origin overwhelmed after expiry | Many edges missing at once | Request collapsing, shielding, staggered TTLs |
| Purge not taking effect | Propagation delay assumed instant | Version URLs instead of relying on purges |
| Everything bypassing the cache | Cookies or headers defeating caching | Strip unnecessary cookies on asset paths |
| Errors cached | Error responses inheriting a long TTL | Short or zero TTL on non-200 responses |

> **Caching an error response distributes an outage that has already ended**  
> If a failing origin returns a 500 with a cacheable header, edges store it and continue serving it long after the origin recovers, so a brief incident becomes a prolonged one that looks unrelated to any current fault. Error responses need explicit short or zero lifetimes, and the mistake is easy to make because the caching configuration was written with successful responses in mind.

**Where it is used**

- **Static asset delivery**, the universal use — scripts, styles, images and fonts.
- **Video and large media**, where bandwidth savings dominate the economics.
- **Software and package distribution**, serving identical artefacts globally.
- **API response caching** for public, non-personalised endpoints.
- **Edge compute deployments**, personalising at the edge rather than at the origin.

**In the interview**

Cover both benefits — latency and origin offload — and treat cache-control as a correctness topic.

- **Quantify both effects**: distance-bound latency falling to tens of milliseconds, and origin load falling with the hit rate.
- **Propose content-hashed filenames** so assets are immutable and invalidation never arises.
- **Treat caching of authenticated responses as a data-leak risk**, not a configuration preference.
- **Raise the thundering herd on expiry** and give shielding or request collapsing as the mitigation.
- **Note that error responses must not inherit long lifetimes**, since a cached failure outlives the incident.

**Practice drill**

A site serves 2 MB of static assets per page view to users worldwide from one region. Design the CDN configuration: cache keys, TTLs, filename strategy and invalidation approach. Estimate the origin load at 90%, 99% and 99.9% hit rates for a million requests. Then identify every response on the site that must not be cached, and describe the mechanism that guarantees it.

**Go deeper**

A CDN serves content from caches close to users, reducing latency by shortening distance and reducing origin load by terminating most requests at the edge.

**It delivers two benefits from one change, which is unusual.** Moving bytes closer cuts the round trip from hundreds of milliseconds to tens, and every request answered at the edge is one the origin never sees. For a site whose traffic is dominated by identical assets, this is typically the largest single performance and cost improvement available, and it requires no application changes beyond correct headers.

**Immutability is what makes caching simple.** Content-hashed filenames mean a changed file is a different object, so assets can be cached indefinitely and still always be current — invalidation never arises because nothing is ever updated in place. This removes the hardest operational aspect of caching entirely, which is why it has become the default in build tooling rather than an optimisation to consider.

**Cache-control on authenticated paths is a correctness concern.** A per-user response cached without a private directive is stored at an edge and served to whoever asks next, which is a data leak that propagates globally in seconds and is invisible from the application's perspective. Because the default behaviour varies between providers and a single misconfigured route is sufficient, the safe pattern is explicit headers applied centrally rather than per handler.

**The last increments of hit rate matter most.** Going from ninety to ninety-nine per cent reduces origin traffic tenfold, and the next nine reduces it tenfold again — so investigating why a specific content type is missing is far more valuable than general tuning. Aggregate hit rate obscures this, since a healthy overall figure can hide one high-volume path bypassing the cache entirely because of a cookie or a query parameter.

**Expiry of popular objects produces a burst at the origin.** When a widely requested object becomes stale, many edges miss simultaneously and forward to the origin together, which is the stampede problem at global scale. Request collapsing at the edge, a shielding tier that absorbs those misses, and staggered lifetimes each address it, and a system relying on a single long TTL without any of them will see periodic origin spikes it cannot explain from its own traffic.

**Its boundary is content that is identical across users.** Personalised responses, writes and anything requiring current origin state are outside what a cache can serve, so the benefit is confined to the shared portion of traffic. Edge compute has narrowed that boundary by allowing per-user assembly near the user, but the underlying principle is unchanged: the CDN helps precisely to the extent that many people are asking for the same bytes.

**Related patterns:** Adaptive Bitrate Streaming · Cache Invalidation · Stale While Revalidate · Object Storage

---

### Adaptive Bitrate Streaming

*Encode video at several qualities, split it into short segments, and let the player switch between them as available bandwidth changes.*

> **When you hear…** Video delivery to varied connections · buffering harming the experience · mobile networks whose capacity changes constantly

**Flow:** `Encode at several bitrates` → `Split into segments` → `Player measures bandwidth` → `Requests matching quality` → `Switches mid-stream`

**The problem**

A single high-quality encode is unwatchable on a slow connection — the player buffers continuously — while a single low-quality encode wastes the capability of fast connections and looks poor on a large screen.

The connection also changes while watching. A viewer who moves between networks or whose signal degrades will experience a stream chosen for conditions that no longer apply, and one fixed choice cannot follow that.

> **Make quality a per-segment decision the player keeps re-making**  
> If the video exists at several bitrates and is divided into short segments that are interchangeable at the boundaries, the player can choose a different quality for each one. Quality then tracks bandwidth continuously — degrading rather than stalling when capacity drops, and improving as soon as it returns — because the decision is revisited every few seconds instead of once at the start.

**Mental model**

Several parallel encodings of the same content, aligned so that segments are interchangeable, described by a manifest the player reads and uses to choose.

1. **Encode** — The source is transcoded to several bitrate and resolution combinations.
2. **Segment** — Each rendition is split into aligned chunks of a few seconds.
3. **Describe** — A manifest lists the renditions and their segments.
4. **Measure** — The player estimates throughput and watches its buffer.
5. **Switch** — It requests the next segment at whatever quality it can sustain.

> **Switching decisions based only on measured throughput oscillate visibly**  
> Bandwidth estimates are noisy, so a player reacting to each measurement flips between qualities every few seconds, which viewers find more objectionable than a consistently lower quality. Buffer occupancy is the more stable signal — a full buffer means there is room to try higher, a draining one means drop now — and good algorithms combine both with hysteresis.

**How it works**

**Ladder, segments and switching**

```text
ENCODING LADDER
  1080p  5.0 Mbps
   720p  2.5 Mbps
   480p  1.0 Mbps
   360p  0.6 Mbps
   240p  0.3 Mbps
  -> roughly a factor of 2 between rungs
  -> closer spacing means more storage and more
     switching for little perceptual gain

SEGMENTS
  2-10 s, typically 4-6 s
  each begins with a keyframe so renditions are
  interchangeable at boundaries
  -> shorter: faster adaptation, more requests,
     more overhead
  -> longer: fewer requests, slower reaction to
     bandwidth changes

STORAGE COST
  5 renditions ≈ 2.3x the top bitrate in total
  plus per-segment overhead
  -> a 1-hour video: several GB across the ladder

SWITCHING LOGIC
  throughput estimate  noisy, reacts fast
  buffer occupancy     stable, reacts late
  combined:
    buffer high AND throughput supports it -> step up
    buffer draining                        -> step
                                              down
    require sustained evidence before stepping up
  -> asymmetric: drop quickly, rise slowly

STARTUP
  begin at a low rendition for a fast first frame
  step up once bandwidth is measured
  -> startup time matters more to abandonment than
     quality does
```

1. **Space the ladder roughly by a factor of two** — Closer rungs add storage and switching without perceptible benefit.
2. **Align segment boundaries across renditions** — Interchangeability at boundaries is what makes switching possible.
3. **Drive switching from buffer occupancy, not throughput alone** — Throughput estimates are too noisy to act on directly.
4. **Step down quickly and up slowly** — A stall is far worse than a few seconds at lower quality.
5. **Start low for a fast first frame** — Startup delay drives abandonment more than quality does.
6. **Serve every segment from a CDN** — Segments are static files, which is what makes this scale.

**Why segmentation makes delivery easy**

```text
EVERY SEGMENT IS A STATIC FILE
  -> cacheable at the edge like any other object
  -> no streaming protocol at the server
  -> no per-viewer server state
  -> ordinary HTTP, so it works everywhere

THIS IS THE REAL INSIGHT
  adaptive streaming turned video from a stateful
  streaming problem into a static file delivery
  problem
  -> which CDNs already solve extremely well

LIVE STREAMING
  same mechanism, manifest updated continuously
  latency = segment duration x buffer depth
    4 s segments, 3 buffered -> ~12 s behind live
  -> low-latency variants use partial segments to
     cut this to 2-5 s

DRM AND ENCRYPTION
  segments encrypted, keys served separately
  -> licence acquisition adds startup latency
  -> another reason to keep the first segment small

QUALITY METRICS THAT MATTER
  startup time
  rebuffering ratio        (the one viewers hate)
  average bitrate delivered
  switch frequency
  -> optimising average bitrate alone produces a
     worse experience than optimising rebuffering
```

| Metric | Value | Note |
|---|---|---|
| Ladder | ~5 renditions | ×2 spacing |
| Segment | 4-6 s | **aligned across renditions** |
| Delivery | static files | CDN does the work |
| Priority | avoid rebuffering | over peak quality |

> **Optimising for the highest sustainable bitrate produces a worse experience than avoiding stalls**  
> Players tuned to maximise delivered quality sit close to the limit of available bandwidth and stall whenever it dips, and viewers consistently rate a rebuffer as far more damaging than a step down in resolution. The correct objective is smooth playback at a reasonable quality, which means keeping a safety margin and preferring to descend early rather than to hold a rendition the connection can barely support.

**Technologies**

| Piece | Options | Note |
|---|---|---|
| Protocols | HLS, DASH | Both segment-and-manifest; differ in details |
| Codecs | H.264, H.265, AV1 | Newer codecs cut bitrate at higher encoding cost |
| Transcoding | Batch or just-in-time | Just-in-time saves storage, adds latency |
| Delivery | CDN | Segments are ordinary cacheable files |
| Players | Client libraries with switching logic | The switching algorithm is where quality of experience is won |
| Per-title encoding | Ladder tuned per video | Animation and live action need very different bitrates |

Per-title encoding is a meaningful refinement: a fixed ladder wastes bitrate on simple content and starves complex content, whereas analysing each title and choosing rungs for it can cut delivered bytes substantially at the same perceived quality.

**Trade-offs**

**Video delivery approaches**

| Approach | Adapts to bandwidth | Storage | Complexity |
|---|---|---|---|
| Single progressive file | No | Lowest | Lowest |
| Several files, user chooses | Only manually | Moderate | Low |
| Adaptive bitrate | Automatically, per segment | Highest | Moderate |
| Just-in-time transcoding | Automatically | Low | High; compute per view |
| Per-title adaptive ladders | Automatically, tuned | High | Highest |

Just-in-time transcoding trades storage for compute, which suits long-tail catalogues where most titles are rarely watched — encoding on demand costs nothing for content nobody requests, while a full ladder for every title does.

> **Ask before choosing it**  
> How varied are the viewers' connections, and how long is the content? A short clip on predictable connections may not justify a ladder at all, while anything watched on mobile networks for more than a minute almost certainly does.

**How it fails**

**How adaptive streaming goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Quality oscillates visibly | Switching on noisy throughput estimates | Buffer-based decisions with hysteresis |
| Frequent rebuffering | Player holding a rendition too aggressively | Step down earlier; keep a margin |
| Slow startup | Starting at a high rendition | Begin low, step up after measuring |
| Switching produces glitches | Segment boundaries not aligned | Align keyframes across renditions |
| Storage costs excessive | Too many rungs, or full ladders for rare titles | Wider spacing; just-in-time for the tail |
| Poor experience on fast connections | Ladder topping out too low | Add a higher rendition |
| Live latency too high | Long segments and deep buffers | Shorter or partial segments |

> **Misaligned segment boundaries make switching visible or impossible**  
> Renditions must be cut at the same points, each starting with a keyframe, or the player cannot splice them cleanly — producing artefacts, audio discontinuities, or an inability to switch at all. The problem originates in the encoding pipeline rather than the player, which is why it is often diagnosed late and why encoding configuration deserves verification against actual switching behaviour.

**Where it is used**

- **Video-on-demand platforms**, where the ladder and player algorithm largely determine perceived quality.
- **Live streaming services**, using the same segmentation with continuously updated manifests.
- **Social video feeds**, where fast startup matters more than peak quality.
- **Education and conferencing platforms**, adapting to widely varying connection quality.
- **Audio streaming**, applying the same principle with a much smaller ladder.

**In the interview**

The point worth making is that segmentation turns streaming into static file delivery.

- **Describe the ladder and segmentation** with concrete numbers — five renditions, four-to-six-second segments.
- **Say segments are static files**, which is what lets a CDN carry the load with no per-viewer server state.
- **Prefer buffer-based switching** and explain why throughput estimates alone cause oscillation.
- **Make the objective explicit**: minimise rebuffering rather than maximise bitrate, because viewers weight stalls heavily.
- **Note the live latency arithmetic** — segment duration times buffer depth — and how partial segments reduce it.

**Practice drill**

Design delivery for a video platform whose viewers range from mobile networks to fibre. Specify the encoding ladder, the segment duration and the storage cost for one hour of content. Describe the player's switching algorithm, including what it uses to decide and how it avoids oscillation. Then compute the live latency your design produces and say how you would reduce it by half.

**Go deeper**

Adaptive bitrate streaming encodes content at several qualities in aligned segments, letting the player choose a rendition for each segment as conditions change.

**Its most consequential property is architectural rather than perceptual.** Splitting video into aligned segments turns a stateful streaming problem into the delivery of ordinary static files, which CDNs already solve extremely well. There is no per-viewer server state, no specialised protocol at the origin, and every segment is cacheable at the edge — which is why global video delivery became tractable on commodity infrastructure.

**Continuous re-decision is what handles changing conditions.** A single quality chosen at the start is wrong as soon as the network changes, whereas a decision revisited every few seconds follows the connection — degrading gracefully when capacity falls and recovering when it returns. The segment duration therefore sets the adaptation speed, and the usual four-to-six-second choice balances responsiveness against request overhead.

**The switching signal matters more than the ladder.** Throughput estimates are noisy enough that acting on them directly produces visible oscillation between qualities, which viewers dislike more than a steady lower resolution. Buffer occupancy is slower but far more stable, and combining the two with hysteresis — plus an asymmetry that drops quickly and rises cautiously — is what separates a player that feels smooth from one that is technically adaptive and unpleasant to watch.

**The objective should be smooth playback, not maximum bitrate.** Viewers weight rebuffering very heavily and resolution changes comparatively lightly, so an algorithm that holds the highest sustainable quality and stalls occasionally scores worse than one that keeps a margin and never stalls. Startup time matters similarly: beginning at a low rendition produces a first frame quickly, and abandonment is driven far more by waiting than by initial quality.

**Alignment across renditions is a pipeline requirement, not a player one.** Segments must be cut at identical points with keyframes at their starts, or splicing produces artefacts or fails outright. Because the symptom appears during playback and the cause is in encoding, this class of problem is typically found late — which makes verifying switching behaviour against actual encoded output part of the delivery pipeline rather than an afterthought.

**Storage and compute are the adjustable costs.** A full ladder multiplies storage by roughly two and a half times the top rendition, which is substantial across a large catalogue, while just-in-time transcoding trades that for compute at view time and suits long tails where most titles are rarely watched. Per-title ladders refine it further, since animation and live action need very different bitrates for the same perceived quality, and a fixed ladder is necessarily wrong for both.

**Related patterns:** CDN · Object Storage · Multipart Upload · Backpressure

---

## Media Storage

### Object Storage

*Store immutable blobs addressed by key in a flat namespace, trading filesystem semantics for effectively unlimited, cheap, durable capacity.*

> **When you hear…** Large files outgrowing a database · media, backups and logs · storage that must scale without capacity planning

**Flow:** `Key and bytes` → `Flat namespace` → `Replicated durably` → `HTTP access` → `Immutable objects`

**The problem**

Uploaded images stored in a database bloat it, slow every backup, and consume the buffer pool that queries need. Stored on a server's local disk they are lost when the instance is replaced and unavailable to every other instance.

A shared filesystem solves the sharing and adds its own limits: capacity must be planned, directories degrade with millions of entries, and throughput is bounded by whatever is mounting it.

> **Give up filesystem semantics and the constraints go with them**  
> Most large files are written once and read many times, never modified in place, and never need directory operations. A store that offers only put, get and delete on whole objects in a flat namespace can distribute them arbitrarily, replicate them for durability, serve them over HTTP, and grow without anyone planning capacity — because none of the guarantees that make filesystems hard to distribute are being provided.

**Mental model**

A distributed key-value store for large values. Keys look like paths but are flat strings; values are opaque bytes plus metadata.

1. **Put** — An object is written whole and becomes immutable.
2. **Replicate** — Copies or erasure-coded fragments are spread across failure domains.
3. **Get** — Objects are retrieved by key, usually over HTTP.
4. **Tier** — Access patterns move objects between cost classes.
5. **Delete** — Objects are removed; there is no in-place modification.

> **There is no rename, no append and no partial update**  
> Changing one byte means rewriting the object, and moving a prefix means copying every object under it. Operations that are trivial on a filesystem are linear in data volume here, which makes naming decisions permanent in practice — a directory structure chosen badly cannot be reorganised cheaply once it holds millions of objects.

**How it works**

**Naming, listing and consistency**

```text
THE NAMESPACE IS FLAT
  "photos/2026/09/14/abc.jpg" is one key
  the slashes are a convention, not a hierarchy
  -> listing by prefix is a scan, not a directory read
  -> listing a prefix with millions of objects is
     slow and paginated

KEY DESIGN
  include everything needed to find the object
  without listing:
    tenant/entity/date/id
  -> the goal is to construct the key from what you
     already know, never to search for it

  avoid sequential prefixes where the provider
  partitions by key range
    20260914-0001, 20260914-0002 ...
    -> all writes land in one partition
  -> prefix with a hash where write rates are high

IMMUTABILITY IS A FEATURE
  content-addressed keys (hash of the bytes)
    -> deduplication for free
    -> caching forever is safe
    -> no invalidation problem

CONSISTENCY
  modern providers give read-after-write for new
  objects
  -> but LIST operations can still lag
  -> never use a list to determine whether a write
     succeeded; use the write's own response
```

1. **Design keys so objects are found by construction, not by listing** — Listing is a scan and becomes unusable at scale.
2. **Avoid sequential key prefixes at high write rates** — Range-partitioned stores concentrate those writes on one partition.
3. **Use content-addressed keys where it fits** — Deduplication, safe permanent caching and no invalidation.
4. **Keep metadata in a database, not in the object store** — Queries, filters and joins need an index the store does not have.
5. **Apply lifecycle policies deliberately** — Tiering and expiry are where most of the cost saving lives.
6. **Serve public objects through a CDN** — Direct reads at scale cost more and are slower than cached ones.

**Cost structure and durability**

```text
WHAT YOU PAY FOR
  storage per GB-month     cheap
  requests                 per thousand; matters at
                           high object counts
  egress                   usually the dominant cost
  -> serving through a CDN cuts egress dramatically

TIERING
  hot      immediate access, highest storage price
  infrequent  cheaper storage, retrieval fee
  archive  very cheap, minutes to hours to restore
  -> lifecycle rules move objects automatically
  -> retrieval fees make aggressive tiering backfire
     for data that is actually read

DURABILITY
  providers quote very high durability figures based
  on replication or erasure coding across failure
  domains
  -> durability is not the same as availability, and
     neither is a backup

WHAT IT DOES NOT PROTECT AGAINST
  deleting the object yourself
  a bug overwriting it
  credentials being compromised
  -> versioning, and separate backups with separate
     access, are what cover these
```

| Metric | Value | Note |
|---|---|---|
| Namespace | flat | **prefix listing is a scan** |
| Mutation | whole object only | no append or rename |
| Dominant cost | egress | use a CDN |
| Durability | not a backup | versioning matters |

> **High durability is not protection against your own mistakes**  
> Replication protects against hardware failure and nothing else. An object deleted by a buggy job, overwritten by a bad deploy, or destroyed by a compromised credential is gone with the same reliability that keeps it safe from disk failure. Versioning, deletion protection and a copy under separate credentials are what address that, and the quoted durability figure says nothing about any of it.

**Technologies**

| Layer | Options | Note |
|---|---|---|
| Cloud object stores | S3 and equivalents | The default; effectively unlimited |
| Self-hosted | MinIO, Ceph | Compatible APIs on your own hardware |
| CDN in front | Any provider | Cuts egress cost and latency |
| Metadata store | A database alongside | The object store cannot answer queries |
| Block storage | Attached volumes | For data needing filesystem semantics |
| File storage | Managed shared filesystems | When applications require POSIX behaviour |

Pairing an object store with a database is the standard arrangement, and the division is worth stating clearly: bytes live in the store, and everything you might want to query — ownership, tags, timestamps, status — lives in the database with the key as a pointer.

**Trade-offs**

**Storage options for large files**

| Option | Scales | Semantics | Cost |
|---|---|---|---|
| Database blobs | Poorly | Transactional | Highest |
| Local disk | Not at all | Full filesystem | Low, and ephemeral |
| Network filesystem | To a point | Full filesystem | Moderate |
| Object storage | Effectively unlimited | Put, get, delete only | Lowest |
| Object storage with a CDN | Effectively unlimited | Same, read-optimised | Lowest, plus cache |

Database blobs are worth a moment of defence: for small files in modest quantities, keeping them transactional with the row that references them removes an entire class of consistency problems, and the cost only becomes unreasonable as volume grows.

> **Ask before choosing it**  
> Do these files ever need to be modified in place, or listed dynamically by attributes? Both are awkward here — the first requires rewriting whole objects, the second requires a database — and discovering either after a hundred million objects exist is expensive.

**How it fails**

**How object storage goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Listing operations time out | Millions of objects under one prefix | Design keys for direct construction |
| Write throughput limited | Sequential key prefixes | Hash or randomise the prefix |
| Egress costs dominate | Serving directly to users | Put a CDN in front |
| Objects deleted irrecoverably | No versioning | Enable versioning and deletion protection |
| Data exposed publicly | Overly broad access policy | Default deny; explicit narrow grants |
| Retrieval costs unexpected | Aggressive archival tiering on active data | Tier by measured access patterns |
| Consistency confusion | Using listings to confirm writes | Trust the write response, not a list |

> **Public access misconfiguration is the most common serious failure in object storage**  
> A bucket or prefix granted broad read access exposes everything under it to anyone who finds the name, and because the store is reachable over ordinary HTTP there is no network boundary providing a second line of defence. The failure is silent — nothing in the application behaves differently — and it is discovered from outside, which is why default-deny policies and automated checking of access configuration are standard practice rather than optional hardening.

**Where it is used**

- **User-uploaded media**, the archetypal use: images, video and documents.
- **Backups and archives**, where durability and cheap cold tiers matter most.
- **Data lakes**, storing partitioned columnar files for batch processing.
- **Static website and asset hosting**, usually behind a CDN.
- **Video segment storage** for adaptive streaming, where every segment is an object.

**In the interview**

Be precise about what is given up, since the constraints are what make the scalability possible.

- **Name the missing semantics** — no append, no rename, no partial update — and note the consequences for key design.
- **Say the namespace is flat** and that prefix listing is a scan, so keys must be constructible rather than searchable.
- **Keep metadata in a database**, with the object key as the pointer, because the store cannot answer queries.
- **Distinguish durability from backup.** Replication does not protect against deletion, bugs or compromised credentials.
- **Raise egress cost and access policy**, which are the two things that most often go wrong in production.

**Practice drill**

Design storage for a product where users upload photos, each with a caption, tags and privacy settings. Specify the key structure, what lives in the database, and how a user's photos are listed. Say what happens when a photo is edited and when it is deleted. Then estimate the monthly cost at ten million photos averaging 2 MB with a million views a day, and identify which line dominates.

**Go deeper**

Object storage holds immutable blobs addressed by key in a flat namespace, offering vast capacity by declining to provide filesystem semantics.

**What it gives up is precisely what makes it scalable.** In-place modification, rename, append and hierarchical directories are the operations that make distributed filesystems difficult, and large files rarely need any of them — they are written once and read many times. Removing those guarantees lets objects be distributed arbitrarily and replicated independently, which is why capacity becomes something nobody plans rather than something that runs out.

**Key design is a permanent decision and should be treated as one.** Because the namespace is flat, finding an object means constructing its key rather than searching for it, and reorganising a prefix means copying every object beneath it. A structure chosen casually at the start becomes expensive to change once millions of objects exist, which makes the naming scheme worth more deliberation than it usually receives.

**The store holds bytes and a database holds everything you might ask about them.** Object stores cannot filter by attribute, sort by date or join, so any query — this user's photos, everything tagged this way, uploads from last week — needs an index elsewhere with the key as a pointer. Systems that attempt to answer such questions by listing prefixes work during development and fail at scale, since listing is a paginated scan.

**Immutability composes well with content addressing.** Naming an object by the hash of its contents gives deduplication automatically, makes indefinite caching safe because the name changes whenever the content does, and removes invalidation entirely. This is the same reasoning that makes hashed asset filenames standard at the CDN layer, applied one level down.

**Egress usually dominates the bill, which surprises people who budget for storage.** Storing bytes is inexpensive; moving them out repeatedly is not, and a popular object served directly to users can cost far more in transfer than in storage. Fronting the store with a CDN addresses both cost and latency, and it is the single most effective economic change available for read-heavy workloads.

**Quoted durability addresses one failure mode and people hear it as addressing all of them.** Replication across failure domains protects against hardware loss and does nothing about a job that deletes the wrong prefix, a deploy that overwrites objects, or a leaked credential. Versioning, deletion protection and a copy held under separate access are what cover those — and because the store is reachable over ordinary HTTP with no network boundary behind it, access policy misconfiguration remains the most common way real data is lost or exposed.

**Related patterns:** Multipart Upload · Presigned URL · CDN · Batch Processing

---

### Multipart Upload

*Split a large upload into independent parts that can be sent in parallel and retried individually, then assemble them server-side.*

> **When you hear…** Files large enough that a single request is fragile · uploads failing near completion · throughput limited by one connection

**Flow:** `Initiate upload` → `Split into parts` → `Upload in parallel` → `Retry failed parts` → `Complete and assemble`

**The problem**

A five-gigabyte upload sent as one request fails after forty minutes because the connection dropped, and every byte must be sent again. On an unreliable network the probability of completing a long single transfer approaches zero as the file grows.

One connection is also slow. A single TCP stream rarely saturates available bandwidth over a long path, so the transfer is limited by one connection's throughput rather than by the network's capacity.

> **Make failure cost one part instead of the whole file**  
> If the upload is divided into independently addressable pieces, a failure retries only the piece that failed, and the pieces can be sent concurrently to use the full available bandwidth. The transfer also becomes resumable across sessions, because completed parts are durable on the server and the client only needs to know which ones remain.

**Mental model**

A three-phase protocol: begin an upload and receive an identifier, send numbered parts independently, then complete by listing the parts and their checksums.

1. **Initiate** — The server returns an upload identifier.
2. **Split** — The client divides the file into numbered parts.
3. **Upload** — Parts are sent concurrently, each acknowledged with a checksum.
4. **Retry** — Failed parts are resent individually.
5. **Complete** — The client submits the part list and the server assembles the object.

> **Incomplete uploads consume storage indefinitely and are invisible**  
> Parts of an upload that is never completed remain stored, billed, and absent from any object listing — so they accumulate silently. Without a lifecycle rule that aborts stale uploads, a system with frequent client failures can accrue substantial cost in data nobody can see and nobody is using.

**How it works**

**Part sizing and parallelism**

```text
PART SIZE
  too small  more requests, more overhead, per-part
             limits reached
  too large  a failure costs more to retry
  typical 5-100 MB; 8-16 MB is a common default

  5 GB file, 10 MB parts -> 500 parts
  5 GB file, 50 MB parts -> 100 parts

PARALLELISM
  4-8 concurrent parts saturates most connections
  more than that contends with itself and with the
  user's other traffic
  -> on mobile, keep it low; battery and radio
     behaviour matter

RETRY
  each part retried independently with backoff
  -> a single failure costs one part, not the file
  -> failure probability per part is small, and the
     whole transfer becomes reliable

RESUMABILITY
  the upload identifier and completed part numbers
  are enough to resume after the app is closed
  -> persist them client-side
  -> query the server for which parts it already has

COMPLETION
  the client sends part numbers with their checksums
  the server verifies and assembles
  -> a mismatch fails the completion rather than
     producing a corrupt object
```

1. **Choose parts in the tens of megabytes** — Small enough to retry cheaply, large enough to avoid request overhead.
2. **Upload a handful of parts concurrently** — Four to eight saturates most links without self-contention.
3. **Persist the upload identifier client-side** — It is what makes resumption across sessions possible.
4. **Verify each part with a checksum** — Corruption should fail completion, not produce a broken object.
5. **Set a lifecycle rule to abort stale uploads** — Otherwise abandoned parts accumulate cost invisibly.
6. **Combine with presigned URLs** — Parts can then go directly from the client to storage.

**Reliability arithmetic**

```text
SINGLE UPLOAD, 5 GB over 40 minutes
  P(connection survives 40 min) on a mobile network
  might be 0.7
  -> 30% of uploads fail and restart from zero
  -> and each retry has the same odds

MULTIPART, 10 MB parts, ~5 s each
  P(a part succeeds) ~0.999
  a failed part costs 5 s, not 40 minutes
  -> effectively every upload completes
  -> total time is lower because parts run in
     parallel

THROUGHPUT
  one TCP stream over a long path is limited by
  window size and loss
  4 parallel streams often deliver 2-4x the
  throughput of one
  -> the speed benefit is frequently larger than
     expected

WHEN NOT TO BOTHER
  files under ~100 MB on reliable networks
  -> the protocol overhead is not worth it
  -> most SDKs switch automatically above a
     threshold
```

| Metric | Value | Note |
|---|---|---|
| Failure cost | one part | **not the file** |
| Throughput | 2-4× faster | parallel streams |
| Part size | 8-16 MB | common default |
| Hazard | abandoned parts | lifecycle rule |

> **Parallelism beyond a handful of parts stops helping and starts hurting**  
> Concurrent streams compete for the same bandwidth, so past roughly eight the aggregate throughput plateaus while contention, memory use and battery consumption keep rising. On mobile clients especially, aggressive parallelism can make transfers slower and noticeably degrade everything else the device is doing — the right number is small and should be verified rather than maximised.

**Technologies**

| Piece | Options | Note |
|---|---|---|
| Object store APIs | Native multipart support | The standard mechanism |
| Client SDKs | Automatic switching above a threshold | Handles splitting and retry |
| Presigned part URLs | Direct client-to-storage upload | Keeps bytes away from your servers |
| Resumable HTTP uploads | Protocols like tus | Similar goals for arbitrary servers |
| Chunked transfer encoding | Streaming a single request | Not resumable; different purpose |
| Lifecycle rules | Abort incomplete uploads | Necessary cost control |

Combining multipart with presigned URLs is the arrangement most large-upload systems use: the application authorises each part and the bytes travel directly from the client to storage, so upload volume never touches application servers.

**Trade-offs**

**Upload strategies**

| Strategy | Failure cost | Throughput | Complexity |
|---|---|---|---|
| Single request | The whole file | One stream | Lowest |
| Single request with resume | Remaining bytes | One stream | Low |
| Multipart | One part | Parallel streams | Moderate |
| Multipart via presigned URLs | One part | Parallel, direct to storage | Moderate |
| Client-side chunking to your API | One chunk | Parallel, through your servers | High; you assemble |

The last row is what teams build when they do not realise the object store already provides this: chunking to an application API means handling assembly, storage of partial data and cleanup, all of which the storage layer does natively.

> **Ask before choosing it**  
> How large are the files and how reliable are the clients' networks? Below about a hundred megabytes on good connections the overhead is not repaid, and most SDKs already switch automatically at a sensible threshold — so the decision is usually about whether to override that default.

**How it fails**

**How multipart uploads go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Storage cost from invisible data | Abandoned uploads never aborted | Lifecycle rule to expire incomplete uploads |
| Uploads still fail entirely | Parts too large | Reduce part size |
| Slow despite parallelism | Too many concurrent parts contending | Reduce concurrency to a handful |
| Corrupt objects assembled | No per-part checksums | Verify each part; fail completion on mismatch |
| Cannot resume after app restart | Upload identifier not persisted | Store it and the completed part numbers |
| Application servers saturated | Bytes routed through your API | Presigned part URLs direct to storage |
| Part count limits exceeded | Very small parts on a very large file | Scale part size with file size |

> **Abandoned multipart uploads are billed storage that appears nowhere**  
> Parts belonging to an upload that was never completed do not appear in object listings, so they are invisible to the usual ways of inspecting what a bucket contains, while continuing to consume storage and accrue charges. In systems where clients frequently abandon uploads this accumulates steadily, and it is typically discovered through a cost investigation rather than through anything in the application — which is why a lifecycle rule aborting stale uploads belongs in the initial configuration.

**Where it is used**

- **Video upload platforms**, where files are large and client networks are unpredictable.
- **Backup and archival tools**, transferring very large artefacts reliably.
- **Mobile applications**, where resumption across sessions and interruptions is essential.
- **Data ingestion pipelines**, moving large datasets into object storage.
- **Desktop sync clients**, uploading large files in the background over long periods.

**In the interview**

Use the reliability arithmetic; it makes the case in one calculation.

- **Contrast failure costs**: one part of a few seconds against a forty-minute transfer starting over.
- **Mention the throughput gain**, since parallel streams often deliver several times a single stream's rate.
- **Give a part size and a concurrency figure**, and say why more parallelism stops helping.
- **Describe resumption** — persisted upload identifier and completed part list — as the client-side requirement.
- **Raise abandoned uploads** and the lifecycle rule, because it is invisible cost that few candidates mention.

**Practice drill**

Design the upload path for 5 GB video files from mobile clients. Choose the part size and concurrency, and justify both. Estimate the transfer time and the probability of completion compared with a single request. Describe exactly what the client persists so an upload survives the app being closed, and specify the storage configuration that prevents abandoned uploads from accumulating.

**Go deeper**

Multipart upload divides a large transfer into independently retryable parts that can be sent concurrently and assembled server-side.

**It changes the unit of failure, which is what makes large uploads viable.** A single long transfer must survive its entire duration, and that probability falls as files grow and networks degrade — so a five-gigabyte upload over a mobile connection may rarely complete at all. Splitting it means a failure costs seconds rather than the whole transfer, and the aggregate reliability becomes high even when individual attempts are not.

**Parallelism provides a throughput gain that is often larger than expected.** A single TCP stream over a long path is limited by window size and packet loss well below the available bandwidth, so several concurrent streams frequently deliver multiples of one stream's rate. The benefit plateaus quickly — beyond a handful of parts the streams contend with each other — which makes a small concurrency figure correct and maximising it counterproductive, particularly on mobile devices where it also affects battery and other traffic.

**Resumability comes almost for free and matters most on the clients that need it.** Because completed parts are durable server-side, an upload can continue after the application is closed or the device changes networks, provided the client persists the upload identifier and knows which parts succeeded. For mobile and desktop sync clients this turns a fragile operation into a background one, which is frequently the difference between a feature working and being abandoned by users.

**Per-part checksums make corruption a failed completion rather than a bad object.** Verifying each part as it arrives, and validating the list at completion, means a transmission error produces an error the client can act on instead of an assembled object with corrupted bytes in the middle. Without it the corruption is discovered later by whoever reads the file, at which point the original is long gone.

**Abandoned uploads are a genuine and invisible cost.** Parts of an upload that never completes do not appear in object listings, so they are absent from the usual ways of examining storage, while continuing to consume and bill. Where clients frequently abandon transfers — which is normal for mobile — this accumulates steadily and is typically discovered through cost analysis rather than through anything in the application, making an abort-incomplete lifecycle rule part of the initial setup rather than a later optimisation.

**It composes naturally with direct-to-storage uploads.** Authorising each part individually and letting the client send bytes straight to the object store keeps upload traffic entirely off application servers, which for a video platform is the difference between a small API tier and one sized for the full ingest bandwidth. The application retains control over who may upload and where, while never handling the data itself.

**Related patterns:** Object Storage · Presigned URL · Retry With Jitter · Idempotency Key

---

### Presigned URL

*Issue a time-limited signed URL that grants one specific operation, so clients read or write storage directly without holding credentials.*

> **When you hear…** Uploads or downloads proxied through application servers · large files consuming API capacity · clients needing storage access without credentials

**Flow:** `Client asks permission` → `Server signs a URL` → `Client uses it directly` → `Storage verifies signature` → `URL expires`

**The problem**

Every upload passes through the application servers so they can authorise it, which means the API tier must be sized for the full bandwidth of user uploads — a video platform's ingest capacity becomes an application scaling problem rather than a storage one.

The alternative of giving clients storage credentials is worse. Credentials in a mobile application are extractable, they usually grant far more than the single operation needed, and revoking them affects every user at once.

> **Sign the permission, not the session**  
> A URL carrying a signature that encodes one operation, one object and an expiry is a capability rather than a credential: it authorises exactly one thing for a short time, cannot be extended by the holder, and needs no account on the storage system. The application keeps full control over who may do what, while the bytes travel directly between the client and the store.

**Mental model**

The application signs a request the client will make later. The storage service verifies the signature and honours precisely what was signed.

1. **Request** — The client asks the application for permission to upload or download.
2. **Authorise** — The application applies its own rules — ownership, quota, content type.
3. **Sign** — It produces a URL encoding the operation, the object, constraints and an expiry.
4. **Use** — The client sends bytes to or from storage directly.
5. **Expire** — The URL stops working; no revocation is needed.

> **A presigned URL is a bearer capability and anyone holding it can use it**  
> The signature does not identify the user, so a URL shared, logged or leaked works for whoever has it until it expires. Short lifetimes are the primary control — minutes rather than hours — and download URLs for sensitive content should be issued per request rather than embedded in pages that might be cached, forwarded or indexed.

**How it works**

**What the signature can constrain**

```text
THE SIGNED REQUEST FIXES
  method        PUT for upload, GET for download
  object key    exactly one object
  expiry        absolute time
  optionally:
    content-type       must match on upload
    content-length     size bounds
    checksum           the client must supply one
  -> anything not constrained is unconstrained

UPLOAD FLOW
  client: "I want to upload a 4 MB jpeg"
  server: checks quota, ownership, plan limits
          generates key: users/42/photos/<uuid>.jpg
          signs a PUT for that key, 15 min expiry,
          content-type image/jpeg, max 10 MB
  client: PUTs the bytes directly to storage
  client: tells the server it finished
  server: verifies the object exists and its size,
          then records it in the database

WHY THE CONFIRMATION STEP MATTERS
  the application never sees the bytes
  -> it does not know the upload happened unless the
     client says so, or storage notifies it
  -> a client that uploads and vanishes leaves an
     orphan object
  -> use storage event notifications where available;
     otherwise reconcile periodically

DOWNLOAD FLOW
  server authorises, signs a GET, short expiry
  -> content-disposition can force a filename
  -> per-request signing keeps the capability tight
```

1. **Keep expiry short** — The URL is a bearer capability; minutes, not hours.
2. **Constrain content type and size on uploads** — An unconstrained signature permits any content of any size.
3. **Generate the object key server-side** — Letting the client choose allows overwriting other objects.
4. **Confirm uploads through storage events where possible** — Relying on the client to report completion loses objects.
5. **Reconcile orphaned objects periodically** — Uploads that are never confirmed accumulate otherwise.
6. **Sign download URLs per request** — Embedding long-lived URLs in pages spreads the capability widely.

**What it saves, and the risks**

```text
WITHOUT PRESIGNED URLS
  10,000 uploads/hour at 5 MB
  = 50 GB/hour through the application tier
  -> servers sized for bandwidth, not for logic
  -> every upload holds a connection for its duration

WITH PRESIGNED URLS
  10,000 signing requests/hour
  each a few hundred bytes and a signature
  -> the API tier is sized for its actual work
  -> bandwidth is storage's problem, where it is
     cheap and elastic

THE RISKS AND THEIR CONTROLS
  leaked URL
    -> short expiry limits the window
  client uploading something unexpected
    -> constrain content-type and size
  client overwriting another object
    -> server chooses the key; never the client
  orphaned uploads
    -> storage event notifications, plus reconciliation
  URL appearing in logs or referrers
    -> short expiry; avoid embedding in HTML where
       possible

WHAT IT IS NOT
  not a substitute for authorisation; the
  application still decides who may do what — it
  just decides once, in advance
```

| Metric | Value | Note |
|---|---|---|
| API bandwidth | 50 GB/h → ~0 | **direct to storage** |
| Capability | one operation | one object |
| Expiry | minutes | the main control |
| Key choice | server-side | never the client |

> **Letting the client choose the object key permits overwriting other people's data**  
> If the upload key comes from the request, a client can request a signature for a path belonging to someone else and overwrite it — the storage service checks the signature, not the intent. The application must construct the key from the authenticated identity and its own naming scheme, and treat any client-supplied path as untrusted input to be discarded rather than used.

**Technologies**

| Piece | Options | Note |
|---|---|---|
| Object store signing | Native presigned URL support | Available across providers |
| Signed policies | Constrain size, type and prefix | Stronger constraints than a plain signed URL |
| CDN signed URLs | Time-limited access to cached content | For protected downloads at the edge |
| Storage event notifications | Server learns of completed uploads | Removes reliance on client confirmation |
| Multipart with signed parts | Large uploads direct to storage | The standard combination |
| Proxying through the API | The alternative | Simple, and sized for full bandwidth |

Event notifications are the piece that makes this robust: rather than trusting a client to report that it finished, the storage service tells the application an object appeared, which closes the gap where uploads complete but are never recorded.

**Trade-offs**

**Client access to storage**

| Approach | API bandwidth | Credential exposure | Control |
|---|---|---|---|
| Proxy through the API | Full | None | Complete, per byte |
| Presigned URLs | Signing requests only | None | Per operation, in advance |
| Client-held credentials | None | High | Coarse, hard to revoke |
| Public bucket | None | n/a | None |
| CDN signed URLs | None | None | Per operation, cached |

Proxying retains one genuine advantage: the application sees the bytes and can scan, transform or validate them inline. Where content inspection is a requirement, presigned uploads must be paired with asynchronous scanning after the fact, which is a different guarantee.

> **Ask before choosing it**  
> Does anything need to inspect the content as it passes? Virus scanning, transcoding triggers and content moderation can all run after the upload, but they then run against an object that already exists — and if the requirement is to reject content before it is stored, proxying is the honest answer.

**How it fails**

**How presigned URLs go wrong**

| Failure | Cause | Fix |
|---|---|---|
| One user overwrites another's object | Client-supplied key | Server constructs every key |
| Unexpected content uploaded | No content-type or size constraint | Constrain both in the signature |
| Objects uploaded but never recorded | Client never confirmed | Storage event notifications; reconciliation |
| URLs shared and reused | Long expiry | Minutes, not hours |
| Sensitive content leaked via logs | URLs recorded in access logs or referrers | Short expiry; per-request signing |
| Storage costs from orphans | Unconfirmed uploads never cleaned | Lifecycle rules and reconciliation |
| Malware stored and served | No scanning after upload | Asynchronous scan before making the object available |

> **Objects that were uploaded but never confirmed become invisible orphans**  
> Because the application never handles the bytes, it learns that an upload succeeded only if the client tells it — and clients close tabs, lose connectivity and crash. The object exists in storage, is billed, and has no database record pointing to it, so it is absent from every view the application provides. Storage event notifications close this properly; without them, periodic reconciliation between storage and the database is the minimum.

**Where it is used**

- **Media upload paths**, letting large files bypass the application tier entirely.
- **Protected downloads**, issuing short-lived links for paid or private content.
- **Mobile applications**, granting storage access without embedding credentials.
- **Third-party integrations**, allowing a partner to deliver files without an account.
- **Multipart upload flows**, signing individual parts for direct transfer.

**In the interview**

Frame it as a capability with constraints, and cover the confirmation gap.

- **Quantify the bandwidth saving**, since sizing the API tier for upload traffic is the problem being solved.
- **List what the signature constrains** — method, key, expiry, content type, size — and note that anything unconstrained is permitted.
- **Insist the server generates the key**, because a client-supplied path allows overwriting other objects.
- **Raise the confirmation gap** and propose storage event notifications rather than trusting the client.
- **Keep expiry short** and explain that the URL is a bearer capability, not an authenticated session.

**Practice drill**

Design a photo upload flow where clients send files directly to storage. Specify what the client requests, what the server checks, and exactly what the signature constrains. Describe how the application learns that an upload completed, and what happens if the client disappears midway. Then extend the design to protected downloads and state the expiry you would choose for each, with reasons.

**Go deeper**

A presigned URL is a time-limited signed capability granting one specific storage operation, letting clients transfer bytes directly while the application retains authorisation.

**It separates the decision from the data path, which is the whole point.** The application applies its rules — ownership, quota, content type, plan limits — once, in advance, and then steps out of the way. Bytes flow between the client and storage, so the API tier is sized for the requests it actually handles rather than for the bandwidth of everything users upload, which for media platforms is a difference of orders of magnitude.

**A signature is a capability, not an identity.** It does not know who is using it, so anyone holding the URL can perform the signed operation until it expires — which makes expiry the primary control and argues for minutes rather than hours. It also argues against embedding download URLs in pages that may be cached, forwarded or logged, since each of those copies is an equally valid capability for as long as it lasts.

**Everything the signature does not constrain is permitted.** A URL signed for an upload with no content type and no size limit allows any content of any size at that key, so constraints are not hardening but part of the authorisation itself. Most importantly the key must be generated server-side from the authenticated identity: accepting a client-supplied path lets a caller request a signature for someone else's object and overwrite it, and the storage service will honour it because the signature is valid.

**Not handling the bytes means not knowing the upload happened.** The application learns of completion only if the client reports it, and clients close tabs, lose connectivity and crash — leaving objects that exist in storage, accrue cost, and have no record pointing to them. Storage event notifications remove the dependency on the client entirely, and where they are unavailable, periodic reconciliation between storage contents and database records is the minimum acceptable substitute.

**Content inspection becomes asynchronous, which is a real change in guarantee.** Virus scanning, moderation and validation can all run after an object appears, but they run against something that has already been stored — so a design requiring content to be rejected before storage must proxy instead. In practice the usual arrangement is to upload to a quarantine location, scan, and only then move the object into a location the application will serve from.

**It composes with multipart uploads to remove the application from large transfers entirely.** Signing individual parts lets a multi-gigabyte upload proceed in parallel, resumably, straight into storage, with the application involved only in authorising and recording. That combination is what allows a modest API tier to support a platform whose ingest volume would otherwise dictate its entire architecture.

**Related patterns:** Object Storage · Multipart Upload · CDN · API Gateway

---

## Storage Internals

### Write Ahead Log

*Append every change to a sequential log and make it durable before applying it, so a crash can be recovered by replaying what was recorded.*

> **When you hear…** Durability required without paying random write cost · crash recovery to a consistent state · the foundation of transactional storage

**Flow:** `Change requested` → `Append to log` → `Flush to disk` → `Apply in memory` → `Replay on recovery`

**The problem**

Committing a transaction means its effects must survive a crash, and those effects are scattered across pages all over the data file. Writing every touched page synchronously turns one logical change into many random writes, which is the slowest thing storage does.

It is also not atomic. A crash partway through writing those pages leaves some updated and some not, with nothing recording which — so the data file is in a state that no correct sequence of operations could have produced.

> **Record the intention sequentially, apply it lazily**  
> If the change is first appended to a log that is written in order, durability costs one sequential write regardless of how many pages the change touches. The data pages can then be updated in memory and flushed whenever convenient, because a crash is recovered by replaying the log — the log is the authority, and the data file is a cache of what the log already says.

**Mental model**

An append-only sequence of change records, flushed before the change is acknowledged, replayed from the last checkpoint after a crash.

1. **Append** — The change is written to the end of the log.
2. **Flush** — The log write is forced to durable storage.
3. **Acknowledge** — Only now is the change reported as committed.
4. **Apply** — Data pages are updated in memory and flushed later.
5. **Recover** — After a crash, the log is replayed from the last checkpoint.

> **The durability guarantee is exactly as good as the flush**  
> A log write that reaches the operating system's cache but not the physical device is lost in a power failure, and the system will have acknowledged a commit that no longer exists. Every layer between the application and the platter — the filesystem, the drive's own write cache, a virtualised disk — can hold data that appears written, which is why correctness depends on an actual durable flush rather than a successful write call.

**How it works**

**Log records, checkpoints and recovery**

```text
RECORD
  LSN 1042 | txn 77 | page 813 | before | after
  -> before and after images allow both redo and undo

COMMIT PROTOCOL
  append records for the change
  append a commit record
  FLUSH the log
  acknowledge to the client
  -> apply to data pages whenever convenient

WHY THIS IS FAST
  one sequential append, one flush
  instead of N random page writes
  sequential throughput is 10-100x random on
  spinning media, and still several times higher on
  flash

CHECKPOINT
  periodically: flush dirty pages, record a
  checkpoint in the log
  -> recovery replays only from that point
  -> without checkpoints, recovery replays the
     entire history

RECOVERY
  find the last checkpoint
  REDO every change after it (committed or not)
  UNDO changes belonging to transactions that never
  committed
  -> the result is exactly the committed state at
     the moment of the crash

GROUP COMMIT
  many transactions flush together
  -> one disk flush serves dozens of commits
  -> adds a little latency, multiplies throughput
```

1. **Flush the log before acknowledging** — A commit acknowledged from a buffer is not durable.
2. **Checkpoint regularly** — Recovery time is proportional to the log since the last one.
3. **Use group commit under load** — Amortising one flush across many transactions is the main throughput lever.
4. **Keep before and after images** — Redo alone cannot roll back uncommitted work.
5. **Retain log segments until the changes are applied** — Truncating too early makes recovery impossible.
6. **Verify that flushes actually reach the device** — Caches at several layers can acknowledge without persisting.

**The trade-offs the log creates**

```text
RECOVERY TIME vs STEADY-STATE COST
  frequent checkpoints
    -> short recovery
    -> more page flushing during normal operation
  rare checkpoints
    -> minimal interference
    -> recovery may take many minutes
  -> choose from how long an outage may last

WRITE AMPLIFICATION
  every change is written twice: once to the log,
  once to the data file
  -> the log write is cheap (sequential)
  -> the data write happens once per page rather
     than once per change, so batching recovers most
     of it

LOG AS A REPLICATION SOURCE
  the same records that recover a crash can be
  shipped to replicas
  -> physical replication is just streaming the log
  -> change data capture reads it too
  -> one mechanism, three uses

IF THE LOG DISK FILLS
  no new writes can be made durable
  -> the database stops accepting writes
  -> this is a very common production outage, and it
     is usually caused by a replication slot or a
     long transaction preventing truncation
```

| Metric | Value | Note |
|---|---|---|
| Durability cost | one sequential flush | **not N random writes** |
| Recovery | replay from checkpoint | bounded by interval |
| Group commit | one flush, many txns | throughput lever |
| Failure | log disk full | writes stop |

> **A full log disk stops the database entirely**  
> Nothing can be made durable once the log cannot grow, so writes fail immediately even though the data disk has space. The usual causes are indirect — a replication slot no longer being consumed, or a very old open transaction preventing old segments from being removed — which means the disk fills for reasons unrelated to write volume and the remedy is rarely simply adding space.

**Technologies**

| System | Name | Note |
|---|---|---|
| Relational databases | WAL, redo log, transaction log | The mechanism underlying durability and recovery |
| LSM-tree stores | Commit log before the memtable | Same role, different storage engine |
| Filesystems | Journals | Metadata consistency by the same principle |
| Distributed logs | Replicated log as the system of record | Consensus protocols build on this idea |
| Replication | Log shipping | Physical replication is log streaming |
| Change data capture | Reading the log | Turns durability machinery into an event source |

The last three rows are the same artefact serving different purposes: a sequential record of every change is exactly what recovery, replication and change capture all need, which is why one mechanism ends up underpinning all three.

**Trade-offs**

**Durability strategies**

| Strategy | Commit cost | Recovery | Data loss on crash |
|---|---|---|---|
| Write pages synchronously | Many random writes | Complex; may be inconsistent | None if complete |
| Write-ahead log with flush | One sequential flush | Replay from checkpoint | None |
| Log with group commit | Amortised flush | Replay from checkpoint | None |
| Log without flush | Buffered write | Replay what survived | Recent commits |
| No log | None | Impossible | Everything unflushed |

The fourth row is a real configuration choice, not a mistake: some systems deliberately relax the flush for workloads where losing the last second of writes is acceptable in exchange for a large throughput gain. It should be a decision someone made, not a default nobody examined.

> **Ask before choosing it**  
> How much recently acknowledged data may be lost in a power failure? Zero requires a genuine flush per commit or a synchronous replica; anything else is a deliberate trade, and the answer determines both the commit protocol and the replication configuration.

**How it fails**

**How write-ahead logging goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Committed data lost in a power failure | Flush not reaching the device | Verify durability through the whole stack |
| Writes stop entirely | Log disk full | Monitor log growth; find what blocks truncation |
| Recovery takes many minutes | Checkpoints too infrequent | Checkpoint more often; accept steady-state cost |
| Throughput limited by flushes | One flush per commit | Group commit |
| Recovery impossible | Log truncated before pages were applied | Retain segments until checkpointed |
| Replica falls behind and blocks truncation | Unconsumed replication slot | Monitor slot lag; bound retention |
| Long transactions block log cleanup | A very old open transaction | Alert on transaction age |

> **A durability guarantee that does not survive power loss is not a durability guarantee**  
> Write caches in the operating system, the filesystem, the drive controller and any virtualisation layer can each report success before data is physically persisted. A system configured for full durability on hardware that lies about flushes will lose acknowledged commits in a power failure, and nothing in normal operation reveals it — the guarantee has to be verified against the actual storage stack rather than assumed from the database's configuration.

**Where it is used**

- **Every relational database**, where the log is the basis of commit, rollback and recovery.
- **LSM-tree stores**, which write to a commit log before the in-memory table.
- **Journaling filesystems**, applying the same principle to metadata consistency.
- **Consensus systems**, replicating a log as the authoritative sequence of operations.
- **Change data capture pipelines**, reading the log to emit events downstream.

**In the interview**

The sequential-versus-random argument is the core insight; make it explicit.

- **State why it is faster**: one sequential flush instead of many random page writes, whatever the change touched.
- **Give the commit order** — append, flush, acknowledge — and say the guarantee lives in the flush.
- **Explain checkpoints as the recovery-time control**, trading steady-state cost against outage length.
- **Mention group commit**, since amortising flushes is the main throughput lever under load.
- **Connect it to replication and change capture**, because the same log serves all three.

**Practice drill**

Explain how a database survives a power failure midway through a transaction. Describe the exact sequence at commit time and state where durability is established. Then describe recovery: what is replayed, what is rolled back, and how the checkpoint interval affects how long it takes. Finally, explain why a replication slot that nobody is reading can stop the database from accepting writes.

**Go deeper**

A write-ahead log records every change sequentially and durably before it is applied, so a crash is recovered by replaying the log.

**It converts random writes into sequential ones, which is the entire performance argument.** A change touching many pages would otherwise require many random writes to be durable; appending a description of that change costs one sequential write and one flush regardless of how scattered its effects are. Sequential throughput exceeds random by a wide margin on every storage medium, so this single change makes durable commits affordable.

**The log becomes the authority and the data file becomes a cache.** Because recovery replays the log, data pages can be updated lazily and flushed in batches whenever convenient, which also means multiple changes to the same page cost one eventual write rather than one each. The apparent write amplification of recording everything twice is largely recovered by that batching.

**Durability is established at the flush and nowhere else.** A commit acknowledged from a buffer is a promise the system cannot keep, and every layer between the application and the physical device — filesystem, drive cache, virtualised storage — can report success before persisting. This makes the guarantee a property of the whole stack rather than of the database configuration, and verifying it requires testing power loss rather than reading settings.

**Checkpoints are the dial between steady-state cost and recovery time.** Replay begins at the last checkpoint, so frequent checkpoints keep recovery short at the cost of more page flushing during normal operation, and infrequent ones do the reverse. The correct setting comes from how long an outage may acceptably last, which is an availability decision rather than a performance one.

**Group commit is what makes high transaction rates possible.** Flushing once per commit bounds throughput by the device's flush rate, whereas batching many commits into one flush amortises that cost across all of them, adding a small amount of latency and multiplying throughput. It is the standard answer when commit rate rather than data volume is the constraint.

**Its most common production failure is the log disk filling for indirect reasons.** Nothing can be made durable once the log cannot grow, so writes stop entirely even with abundant space in the data files — and the cause is usually a replication slot nobody is consuming or a very old open transaction preventing old segments from being released. Because the trigger is unrelated to write volume, monitoring log growth alongside replication lag and transaction age is what turns this from an outage into an alert.

**Related patterns:** MVCC · LSM Tree · Change Data Capture · Two Phase Commit

---

### MVCC

*Keep multiple versions of each row so readers see a consistent snapshot without blocking writers, and writers never block readers.*

> **When you hear…** Long-running reads blocking writes · lock contention between analytics and transactions · snapshot isolation required

**Flow:** `Write creates a version` → `Readers get a snapshot` → `No read locks` → `Old versions retained` → `Vacuum reclaims`

**The problem**

A report scanning a large table holds read locks for its duration, so every write touching those rows waits. Meanwhile a writer holding row locks blocks the report. Readers and writers obstruct each other constantly, and the contention grows with both concurrency and query duration.

Reading without locks is worse: the report sees rows as they change underneath it, producing totals that correspond to no actual state of the database at any point in time.

> **Do not overwrite — create a new version and let readers choose**  
> If an update writes a new version of a row rather than replacing the old one, then a reader can continue seeing the version that was current when it started. Readers need no locks because nothing they are reading changes, and writers need not wait because they are writing somewhere else. The cost is storing versions and eventually reclaiming the ones nobody can see.

**Mental model**

Every row version is stamped with the transaction that created it and the one that removed it. A transaction reads the versions that were visible when its snapshot was taken.

1. **Snapshot** — A transaction records which other transactions had committed when it began.
2. **Write** — An update creates a new version and marks the old one as superseded.
3. **Read** — Each reader sees the version appropriate to its snapshot.
4. **Isolate** — Concurrent transactions see consistent, possibly different, states.
5. **Reclaim** — Versions no transaction can see are eventually removed.

> **Old versions accumulate and must be reclaimed, or the table degrades**  
> Every update leaves a dead version behind, and a table receiving heavy updates can grow far beyond the size of its live data. Scans then read mostly obsolete rows, and performance falls steadily in a way that looks like a query problem. Reclamation is background work that must keep pace with the write rate, and when it cannot, the degradation is gradual and easy to misattribute.

**How it works**

**Visibility and version accumulation**

```text
ROW VERSIONS
  row 42:
    version A  created by txn 100, deleted by 150
    version B  created by txn 150, deleted by null

VISIBILITY RULE  (simplified)
  a version is visible to a transaction if:
    its creating transaction committed before the
    snapshot, AND
    its deleting transaction had not committed
    before the snapshot
  -> readers take no locks at all

WRITE CONFLICTS STILL EXIST
  two transactions updating the same row
  -> the second waits, or fails, depending on the
     isolation level
  -> MVCC removes read-write contention, not
     write-write

VERSION ACCUMULATION
  a table updated 1,000 times per second
  vacuum running every few minutes
  -> hundreds of thousands of dead versions between
     runs
  -> the table occupies several times its live size
  -> scans read mostly dead rows

THE LONG TRANSACTION PROBLEM
  one transaction open for hours holds a snapshot
  -> NO version newer than it can be reclaimed
  -> dead rows accumulate across the whole database
  -> a single forgotten session degrades everything
```

1. **Keep transactions short** — An open transaction blocks reclamation database-wide, not just for its own rows.
2. **Monitor table bloat and reclamation lag** — Degradation is gradual and presents as slow queries.
3. **Alert on the oldest open transaction** — It is the single most common cause of runaway version accumulation.
4. **Understand the isolation level in use** — Snapshot isolation is not serializable and permits specific anomalies.
5. **Expect write-write conflicts to remain** — MVCC eliminates read-write blocking only.
6. **Size storage for versions, not just live data** — Update-heavy tables occupy several times their logical size.

**What MVCC does and does not guarantee**

```text
PROVIDES
  readers never block writers
  writers never block readers
  a consistent snapshot for the whole transaction
  no read locks at all

DOES NOT PROVIDE
  serializability, unless explicitly requested
  freedom from write-write conflicts
  free storage

WRITE SKEW  (the classic snapshot anomaly)
  rule: at least one doctor must remain on call
  two doctors, both on call
  both transactions read: "2 on call, fine"
  both remove themselves
  both commit
  -> zero on call; each decision was valid against
     its own snapshot
  -> snapshot isolation permits this
  -> serializable isolation detects and prevents it

VERSION STORAGE STRATEGIES
  versions in the table   updates leave dead rows in
                          place; vacuum reclaims
  versions in undo space  the table stays compact;
                          long readers can exhaust
                          undo space instead
  -> different systems choose differently, and the
     operational failure modes differ accordingly
```

| Metric | Value | Note |
|---|---|---|
| Readers | never blocked | **no read locks** |
| Writers | never blocked by readers | write-write still conflicts |
| Cost | version storage | plus reclamation |
| Anomaly | write skew | under snapshot isolation |

> **One long-running transaction prevents reclamation across the entire database**  
> Because any version potentially visible to an open snapshot cannot be removed, a single session left open — an idle transaction, a stalled report, a forgotten console — stops cleanup everywhere, not only for the rows it touched. Dead versions then accumulate globally, tables bloat, and performance degrades across unrelated workloads, which makes the age of the oldest open transaction one of the most valuable metrics to alert on.

**Technologies**

| System | Approach | Note |
|---|---|---|
| PostgreSQL | Versions stored in the table | Vacuum reclaims; bloat is the operational concern |
| Oracle and MySQL InnoDB | Versions in undo segments | Table stays compact; undo space can be exhausted |
| Distributed SQL engines | MVCC with timestamps | Snapshots across nodes require clock coordination |
| Key-value stores with versions | Timestamped values | Same principle, simpler visibility rules |
| Two-phase locking | The alternative | Serializable by construction; far more blocking |
| Serializable snapshot isolation | MVCC plus conflict detection | Serializability without most of the locking |

Serializable snapshot isolation is the significant refinement: it keeps the non-blocking reads of MVCC while detecting the dependency patterns that produce anomalies, aborting one transaction when a genuine conflict occurs rather than preventing concurrency in advance.

**Trade-offs**

**Concurrency control approaches**

| Approach | Reader blocking | Storage | Anomalies |
|---|---|---|---|
| Two-phase locking | Readers block writers | Minimal | None at serializable |
| MVCC snapshot isolation | None | Versions retained | Write skew possible |
| Serializable snapshot isolation | None | Versions plus tracking | None; some aborts |
| Read uncommitted | None | Minimal | Many |
| Optimistic concurrency | None | Minimal | Aborts on conflict |

The middle row is what most systems run by default, and the anomalies it permits are subtle enough that many applications contain latent write-skew bugs which appear only under concurrency that the test suite never produces.

> **Ask before choosing it**  
> Does any business rule depend on a condition across rows that a transaction checks and then acts on? That is the shape of write skew, and under snapshot isolation two concurrent transactions can both pass the check and jointly violate the rule — which needs either serializable isolation or an explicit lock.

**How it fails**

**How MVCC goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Tables grow far beyond their data | Reclamation not keeping pace | Tune vacuum; reduce update rate |
| Global performance degradation | A long-open transaction blocking cleanup | Alert on oldest transaction age |
| Queries slow down gradually | Scans reading mostly dead versions | Reclaim; rebuild badly bloated tables |
| Business rules violated under load | Write skew under snapshot isolation | Serializable isolation, or explicit locking |
| Undo space exhausted | Long readers in undo-based systems | Bound query duration; size undo space |
| Unexpected serialization failures | Serializable isolation aborting conflicts | Retry aborted transactions |
| Write conflicts surprising developers | Expecting MVCC to remove all blocking | It removes read-write blocking only |

> **Write skew produces violations that every individual transaction considered legal**  
> Two transactions each read a condition, each conclude their action is permitted, and each commit — leaving a state neither would have allowed had it seen the other. Snapshot isolation permits this by design, so the bug is not in the database and not obviously in the application either. It surfaces only under concurrency, frequently in production, and the remedies are serializable isolation or taking an explicit lock on whatever the rule ranges over.

**Where it is used**

- **PostgreSQL**, where MVCC and vacuum behaviour are central operational concerns.
- **MySQL InnoDB**, using undo segments for versions and consistent reads.
- **Distributed SQL databases**, extending snapshots across nodes with coordinated timestamps.
- **Analytics against transactional databases**, reading consistently without blocking writers.
- **Key-value stores with versioned values**, applying the same visibility principle.

**In the interview**

Say what it removes and what it does not; the write-skew example is what demonstrates real understanding.

- **State the guarantee precisely**: readers never block writers and vice versa, but write-write conflicts remain.
- **Explain visibility by snapshot**, which is why readers need no locks at all.
- **Give the write-skew example** with the on-call doctors, since it shows the limits of snapshot isolation concretely.
- **Raise version accumulation** and the operational cost of reclamation.
- **Name the long-transaction hazard**, where one open session degrades the entire database.

**Practice drill**

Explain how a report scanning ten million rows for two minutes coexists with a thousand updates per second on the same table. Describe what each sees and why neither waits. Then construct a write-skew scenario for a rule requiring at least one administrator per account, show why both transactions commit, and give two different ways to prevent it. Finally, explain what a session left open for six hours does to the database.

**Go deeper**

Multiversion concurrency control keeps several versions of each row so that readers see a consistent snapshot without locking and writers proceed without waiting for readers.

**It removes the largest source of contention in mixed workloads.** Analytical queries and transactional writes obstruct each other severely under lock-based concurrency control, and MVCC eliminates that interaction entirely: readers observe versions that were current when they started, so nothing they read can change, and writers create new versions rather than modifying what readers hold. Write-write conflicts remain, because two transactions genuinely cannot both decide the next value of the same row.

**Versions are the currency, and they must be reclaimed.** Every update leaves an obsolete version, so an update-heavy table accumulates dead rows continuously and can occupy several times the size of its live data. Scans then spend most of their time reading versions nobody can see, which presents as gradually worsening query performance rather than as a storage problem — making bloat and reclamation lag metrics worth watching directly.

**A single long transaction degrades everything.** Reclamation can only remove versions that no open snapshot could see, so one session held open for hours prevents cleanup across the whole database, not merely for the rows it touched. An idle transaction, a stalled report or a forgotten console therefore causes global accumulation, and the age of the oldest open transaction is among the highest-value alerts a system of this kind can have.

**Snapshot isolation is weaker than serializability in a specific and non-obvious way.** Write skew occurs when two transactions each read a condition, each conclude independently that their action is permitted, and together violate the rule neither would have broken alone. Nothing is wrong with either transaction in isolation and nothing is wrong with the database; the anomaly is inherent to reading from a snapshot, which makes it a latent bug in applications whose invariants span rows.

**Where versions are stored changes the operational failure mode.** Keeping them in the table means dead rows accumulate in place and require vacuuming, so the characteristic problem is bloat. Keeping them in a separate undo area keeps tables compact and makes long-running readers a different hazard — they can exhaust undo space and fail, or force other transactions to fail. Neither is better in general, but they fail differently, and the monitoring that matters differs accordingly.

**Serializable snapshot isolation resolves the anomaly without returning to locking.** By tracking read and write dependencies and aborting a transaction when a genuine conflict pattern is detected, it preserves non-blocking reads while providing full serializability, at the cost of occasional aborts that the application must retry. For workloads with invariants spanning rows this is usually a better trade than either accepting write skew or reintroducing explicit locks everywhere.

**Related patterns:** Write Ahead Log · LSM Tree · Consistent Snapshot · Quorum Replication

---

### Bloom Filter

*A compact probabilistic structure that answers definitely-not-present or possibly-present, letting a system skip expensive lookups for keys it does not have.*

> **When you hear…** Expensive lookups for keys that usually do not exist · memory too small for a full index · disk or network reads worth avoiding

**Flow:** `Hash the key` → `Set several bits` → `Query checks bits` → `Any zero means absent` → `All ones means maybe`

**The problem**

Checking whether a key exists means consulting an index that does not fit in memory, so most checks cost a disk read — and in many workloads the majority of those reads discover that the key is absent. The expensive operation was performed to learn nothing.

Keeping a complete set of keys in memory would answer instantly and is exactly what does not fit; that is why the index is on disk in the first place.

> **Accept false positives and the memory requirement collapses**  
> A structure that may occasionally say a key might be present when it is not, but never says absent when it is present, can be built from a handful of bits per key rather than the key itself. Since a definite no eliminates the lookup entirely and a maybe simply proceeds as before, the error is asymmetric in exactly the direction that costs nothing but a wasted check.

**Mental model**

A bit array and several hash functions. Insertion sets the bits a key hashes to; a query reports absent if any of those bits is zero.

1. **Size** — Choose the bit array size and hash count from the expected key count and acceptable error rate.
2. **Insert** — Hash the key several ways and set each resulting bit.
3. **Query** — Hash the same ways and inspect those bits.
4. **Conclude** — Any zero proves absence; all ones means probably present.
5. **Verify** — A probable hit is confirmed by the real lookup.

> **Elements cannot be removed from a standard Bloom filter**  
> Clearing the bits for one key would also clear bits shared with others, producing false negatives — which destroys the one guarantee the structure provides. Deletion requires a counting variant with several bits per position, or rebuilding the filter, and a design that assumes removal works will silently start reporting that present keys are absent.

**How it works**

**Sizing and the error rate**

```text
PARAMETERS
  n  expected number of elements
  m  bits in the array
  k  number of hash functions

OPTIMAL k = (m/n) x ln 2
FALSE POSITIVE RATE ≈ (1 - e^(-kn/m))^k

PRACTICAL SIZING  (bits per element)
  ~4.8 bits  -> 10% false positives
  ~9.6 bits  -> 1%
  ~14.4 bits -> 0.1%
  ~19.2 bits -> 0.01%
  -> each additional ~4.8 bits per element divides
     the error rate by ten

CONCRETE
  1,000,000 keys at 1% error
    9.6 Mbit = 1.2 MB, with k = 7
  compare storing the keys themselves:
    1,000,000 x 20 B = 20 MB
  -> ~17x smaller, and constant regardless of key
     length

THE LONG KEY ADVANTAGE
  a 200-byte URL still costs 9.6 bits
  -> the saving grows with key size

OVERFILLING
  inserting far more than n elements raises the
  error rate towards 100%
  -> the filter degrades into always saying maybe
  -> size for the eventual count, not the current one
```

1. **Size from the eventual element count** — An overfilled filter degrades until it answers maybe to everything.
2. **Choose the error rate from the cost of a false positive** — It is a wasted lookup, so the acceptable rate depends on that lookup's expense.
3. **Use one hash with different seeds** — Computing several independent hashes is unnecessary and slower.
4. **Never rely on removal** — Standard filters cannot delete; use a counting variant or rebuild.
5. **Keep it in memory** — A filter on disk defeats its own purpose.
6. **Monitor the observed false positive rate** — It reveals overfilling before performance degrades noticeably.

**Where the asymmetry pays**

```text
LSM-TREE READS  (the canonical use)
  a read may need to check many sorted files
  each check is a disk read
  a filter per file:
    definitely not here -> skip the file entirely
    maybe               -> read it
  -> most files are skipped
  -> read amplification falls dramatically

CACHE ADMISSION
  do not cache an item until it has been seen twice
  -> a filter records "seen once" cheaply
  -> prevents one-off scans evicting hot data

DISTRIBUTED SET MEMBERSHIP
  send a filter instead of a key set
  -> a peer can check membership locally
  -> anti-entropy and replica comparison use this

WHEN IT DOES NOT HELP
  most queries are for keys that DO exist
  -> the filter says maybe almost every time
  -> you paid memory and saved nothing
  -> the benefit is proportional to the miss rate

THE DECIDING QUESTION
  what fraction of lookups are for absent keys?
  high  -> a filter is extremely effective
  low   -> it is overhead
```

| Metric | Value | Note |
|---|---|---|
| Memory | ~10 bits/key | **at 1% error** |
| Guarantee | no false negatives | absent is certain |
| Benefit | scales with miss rate | useless if all hit |
| Limitation | no deletion | without a variant |

> **The benefit is proportional to how often keys are absent**  
> A filter earns its memory by eliminating lookups, so a workload whose keys nearly always exist receives a maybe on almost every query and skips nothing. The structure is then pure overhead — memory spent and a hash computed per lookup for no saving. Measuring the miss rate before adding one is what separates a large improvement from a small regression.

**Technologies**

| Use | System | Note |
|---|---|---|
| LSM-tree read paths | Cassandra, RocksDB, HBase | One filter per sorted file; the defining use |
| Cache admission | Modern cache implementations | Prevents one-off items evicting hot data |
| Replica comparison | Anti-entropy protocols | Exchange filters rather than key sets |
| Counting Bloom filter | Where deletion is required | Several bits per position; more memory |
| Cuckoo filter | Supports deletion | Comparable size, better locality |
| Exact structures | Hash sets | When memory allows, and certainty is needed |

Cuckoo filters are worth knowing as the modern alternative: they support deletion, achieve similar or better space efficiency at low error rates, and access fewer memory locations per query — which matters when the filter is consulted on every read.

**Trade-offs**

**Membership testing structures**

| Structure | Memory per key | False positives | Deletion |
|---|---|---|---|
| Hash set of keys | Key size plus overhead | None | Yes |
| Bloom filter | ~10 bits at 1% | Yes | No |
| Counting Bloom filter | ~40 bits at 1% | Yes | Yes |
| Cuckoo filter | ~12 bits at 1% | Yes | Yes |
| Sorted key index on disk | Key size | None | Yes |

The counting variant's fourfold memory cost is the price of deletion, which is often enough to change the decision — rebuilding a standard filter periodically is frequently cheaper than paying that overhead continuously.

> **Ask before choosing it**  
> What fraction of lookups are for keys that do not exist, and what does one lookup cost? Those two numbers determine the entire value of the structure, and if most lookups hit, no amount of tuning will make it worthwhile.

**How it fails**

**How Bloom filters go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Error rate approaching 100% | Far more elements than the filter was sized for | Size for the eventual count; rebuild when exceeded |
| Present keys reported absent | Deletion attempted on a standard filter | Counting or cuckoo variant, or rebuild |
| No measurable improvement | Low miss rate in the workload | Measure before adopting; remove if unhelpful |
| Memory larger than expected | Error rate set unnecessarily low | Relax it; the cost is a wasted lookup |
| Slow queries | Too many hash functions | Use the optimal count for the chosen ratio |
| Filter itself becoming a bottleneck | Stored on disk or across the network | It must be in memory to be worthwhile |
| Degradation unnoticed | Observed error rate not tracked | Monitor it and rebuild on a threshold |

> **Attempting deletion turns the one reliable answer into an unreliable one**  
> Clearing bits for a removed key also clears bits that other keys depend on, so those keys begin reporting as definitely absent — and because the entire value of the structure rests on absence being certain, the system will skip lookups for data it actually holds. The result is silent data loss from the caller's perspective, with no error anywhere, which makes this the most damaging way to misuse the structure.

**Where it is used**

- **LSM-tree storage engines**, where a filter per sorted file eliminates most disk reads on lookups.
- **Distributed databases**, avoiding network round trips for keys a node does not hold.
- **Cache admission policies**, keeping one-off items from displacing frequently used data.
- **Anti-entropy protocols**, comparing replica contents by exchanging filters.
- **Web and security systems**, checking membership in large lists without storing them.

**In the interview**

Give the space figure and the asymmetry; both are concrete and immediately convincing.

- **State the guarantee precisely**: no false negatives, so a negative answer is certain and a positive one is a maybe.
- **Quote the space cost** — roughly ten bits per element for one per cent error — and note it is independent of key length.
- **Give the LSM-tree use** as the canonical example, since it shows exactly where the asymmetry pays.
- **Say deletion is not supported** and what happens if it is attempted.
- **Tie the benefit to the miss rate**, because a workload that mostly hits gains nothing.

**Practice drill**

A storage engine checks 20 sorted files per read, each check costing a disk seek, and 80% of reads are for keys that exist in only one file. Size a filter per file for a million keys at 1% error and compute the memory. Estimate the reduction in disk reads. Then explain what happens if a file accumulates ten times the expected keys, and what you would monitor to detect it.

**Go deeper**

A Bloom filter is a compact probabilistic set that never reports a present element as absent, allowing expensive lookups to be skipped with certainty when it says no.

**Its usefulness comes entirely from the asymmetry of its error.** A false positive costs one unnecessary lookup — exactly what would have happened without the filter — while a negative answer is definitive and eliminates the lookup completely. Because the errors only ever cost what the system was already going to pay, the acceptable error rate can be set quite loosely, which is what makes the memory requirement so small.

**Space efficiency comes from storing no keys at all.** Roughly ten bits per element achieves a one per cent error rate regardless of whether the keys are short integers or long URLs, so the saving against storing the keys grows with key size. Each additional five bits or so divides the error rate by ten, which makes the memory-versus-accuracy curve easy to reason about when sizing.

**The benefit is proportional to the miss rate, and this is the question to ask first.** A workload whose lookups almost always find the key receives a maybe nearly every time and skips nothing, so the filter is memory and hashing spent for no return. Measuring how often lookups are for absent keys before adopting one distinguishes a substantial improvement from a small regression, and it is a measurement rather than an intuition.

**Overfilling degrades it gradually into uselessness.** Inserting far more elements than the sizing assumed drives the error rate towards certainty, at which point the filter answers maybe to everything and the system performs exactly as it would without it — while still paying the memory. Because the degradation is smooth and silent, tracking the observed false positive rate is what reveals it before someone investigates a performance regression with no obvious cause.

**Deletion is not merely unsupported; attempting it breaks the guarantee.** Clearing a key's bits also clears bits other keys rely on, so those keys start reporting as definitely absent and the system skips lookups for data it holds. This converts the structure's one reliable answer into an unreliable one and manifests as data that exists but cannot be found, with no error raised — which is why counting variants, cuckoo filters, or periodic rebuilds are the only correct approaches when removal is needed.

**Its defining application is the LSM-tree read path.** A lookup may need to consult many sorted files, each costing a disk read, and most of them do not contain the key. One filter per file turns the majority of those checks into memory operations that skip the file entirely, which is the difference between a read amplification of twenty and one closer to one — and it is the reason the structure appears in essentially every storage engine of that design.

**Related patterns:** LSM Tree · Cache Aside · Anti Entropy · Inverted Index

---

### LSM Tree

*Buffer writes in memory, flush them as sorted immutable files, and merge those files in the background, turning random writes into sequential ones.*

> **When you hear…** Write-heavy workloads · random writes limiting throughput · time-series, logs and event data

**Flow:** `Write to memtable` → `Flush sorted file` → `Files accumulate` → `Compaction merges` → `Reads check several`

**The problem**

A B-tree updates data in place, so every write touches the page holding that key — a random write. At high write rates the device spends its time seeking rather than transferring, and throughput is bounded by random I/O rather than by bandwidth.

Writes also amplify: changing a few bytes rewrites a whole page, and page splits rewrite more. For workloads dominated by inserts and updates this overhead dominates everything else.

> **Never update in place; accumulate and merge**  
> If writes go into a sorted in-memory structure and are flushed as whole immutable files, every disk write is sequential and large. Nothing is ever modified in place, so there are no random writes at all. The cost moves to reads, which must consult several files, and to background merging that keeps the number of files bounded — both of which are more tractable than random write throughput.

**Mental model**

An in-memory sorted buffer, a sequence of immutable sorted files on disk, and a background process merging those files into fewer, larger ones.

1. **Log** — The write is appended to a commit log for durability.
2. **Buffer** — It is inserted into a sorted in-memory table.
3. **Flush** — When the table is full it is written out as an immutable sorted file.
4. **Accumulate** — Files build up, each containing a snapshot of some writes.
5. **Compact** — Background merging combines files, discarding superseded values.

> **Reads may have to consult many files, and compaction competes with live traffic**  
> A key may live in the memtable or in any of the files on disk, so a read potentially examines all of them. Bloom filters and sorted structure make this manageable, and compaction is what keeps the count bounded — but compaction reads and rewrites large amounts of data using the same disks serving queries, which is why LSM systems exhibit latency variability that B-trees do not.

**How it works**

**Write path, read path and compaction**

```text
WRITE
  append to commit log      (durability)
  insert into memtable      (sorted, in memory)
  return
  -> no disk seek, no page read
  -> extremely fast

FLUSH
  memtable full -> write it out as one sorted file
  -> one large sequential write
  -> the file is immutable forever

READ
  check memtable
  then each file, newest first
  stop at the first match
  -> without help this is many disk reads
  -> Bloom filter per file skips most of them
  -> sparse index locates the block within a file

COMPACTION STRATEGIES
  SIZE-TIERED
    merge files of similar size
    + low write amplification
    - more files to read, more space used
    -> good for write-heavy

  LEVELLED
    keep each level's files non-overlapping
    + few files per read, predictable
    - higher write amplification
    -> good for read-heavy

DELETES ARE WRITES
  a delete writes a TOMBSTONE
  -> the key is only truly removed when compaction
     has passed every file containing it
  -> deleting a lot of data temporarily uses MORE
     space
```

1. **Put a Bloom filter on every file** — Without it a read must touch every file on disk.
2. **Choose the compaction strategy from the workload** — Size-tiered favours writes, levelled favours reads.
3. **Throttle compaction** — It uses the same devices as live traffic and will starve it otherwise.
4. **Provision free space for compaction** — Merging requires room for the output before the inputs are released.
5. **Expect tombstones to delay space reclamation** — Deleted data persists until compaction has passed it everywhere.
6. **Monitor the file count and compaction backlog** — A growing backlog degrades reads steadily.

**Amplification and the three-way trade**

```text
THE THREE AMPLIFICATIONS
  write   bytes written to disk per byte of data
  read    files consulted per lookup
  space   disk used per byte of live data
  -> improving any one worsens at least one other

SIZE-TIERED
  write  low   (each byte rewritten a few times)
  read   high  (many files to check)
  space  high  (duplicates until merged)

LEVELLED
  write  high  (a byte may be rewritten ~10x)
  read   low   (one file per level)
  space  low   (little duplication)

B-TREE FOR COMPARISON
  write  high on random writes (page rewrites)
  read   low   (one path down the tree)
  space  moderate (fragmentation)

WHY LSM WINS ON WRITES
  all disk writes are sequential and large
  sequential throughput far exceeds random on every
  medium
  -> an LSM tree can absorb write rates a B-tree
     cannot, on the same hardware

COMPACTION IS THE OPERATIONAL RISK
  it must keep pace with writes
  falling behind -> file count grows -> reads slow
  -> and catching up competes with live traffic,
     which is when latency spikes appear
```

| Metric | Value | Note |
|---|---|---|
| Writes | sequential only | **no random I/O** |
| Reads | several files | Bloom filters help |
| Trade | write/read/space | pick two |
| Risk | compaction backlog | reads degrade |

> **Deleting large amounts of data temporarily increases disk usage**  
> A delete is written as a tombstone, so removing a million rows writes a million records and frees nothing until compaction has merged past every file containing the originals. A system already low on space can therefore fail while attempting to free space, which is counter-intuitive enough that it surprises operators during exactly the incident where they are trying to recover capacity.

**Technologies**

| System | Use | Note |
|---|---|---|
| RocksDB and LevelDB | Embedded storage engines | The reference implementations |
| Cassandra | Distributed store on an LSM engine | Compaction strategy is a per-table decision |
| HBase and Bigtable-style stores | Wide-column storage | Same design lineage |
| Time-series databases | Append-heavy workloads | An excellent fit for the write pattern |
| B-tree engines | The alternative | Better for read-heavy and range-scan workloads |
| Bloom filters | Companion structure | Essential to the read path |

The B-tree comparison is worth keeping concrete: B-trees remain preferable for read-heavy workloads with frequent range scans and in-place updates, while LSM trees dominate where writes are numerous and reads are mostly point lookups.

**Trade-offs**

**LSM against B-tree**

| Property | LSM tree | B-tree |
|---|---|---|
| Write pattern | Sequential only | Random, in place |
| Write throughput | Very high | Bounded by random I/O |
| Point reads | Several files; filters help | One tree traversal |
| Range scans | Merge across files | Sequential along leaves |
| Space | Duplicates until compacted | Fragmentation |
| Latency stability | Compaction causes variance | More predictable |

Latency stability is the row that decides many production choices: LSM engines deliver higher throughput with more variance, and a workload with strict tail-latency requirements may prefer a B-tree even where the write rate would favour an LSM tree.

> **Ask before choosing it**  
> Is this workload write-heavy with point lookups, or read-heavy with range scans? The first is exactly what LSM trees are for; the second is what B-trees do better, and choosing on write throughput alone ignores half the question.

**How it fails**

**How LSM trees go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Reads slowing over time | Compaction falling behind | More compaction throughput; revisit the strategy |
| Latency spikes | Compaction competing with queries | Throttle and schedule compaction |
| Disk full while deleting data | Tombstones consuming space before reclaiming it | Provision headroom; compact deliberately |
| Reads touching every file | No Bloom filters, or overfilled ones | Filters per file, sized correctly |
| Write amplification excessive | Levelled compaction on a write-heavy workload | Size-tiered instead |
| Space usage far above data size | Size-tiered with infrequent compaction | Levelled, or compact more often |
| Deleted data reappearing | Tombstones expiring before full compaction | Ensure repair and compaction outpace tombstone lifetime |

> **Compaction falling behind degrades reads in a way that compounds**  
> When merging cannot keep pace with writes, files accumulate, every read consults more of them, and the extra read I/O competes with the compaction that is already behind. The system slows gradually and then sharply, and catching up requires I/O that the live workload is consuming — which is why compaction backlog is one of the most important metrics in any LSM-based system and why throttling policy deserves attention before it becomes urgent.

**Where it is used**

- **Embedded storage engines**, used inside many databases and applications.
- **Cassandra and wide-column stores**, built directly on this design.
- **Time-series and metrics databases**, whose append-heavy pattern suits it exactly.
- **Event and log storage**, where writes vastly outnumber reads.
- **Key-value caches with persistence**, using the design for durable write throughput.

**In the interview**

Explain the sequential-write insight, then be explicit about the three-way amplification trade.

- **Say why writes are fast**: every disk write is sequential and large, because nothing is ever updated in place.
- **Describe the read path** — memtable then files, newest first — and the role of Bloom filters in making it viable.
- **Lay out write, read and space amplification** and say that improving one worsens another.
- **Contrast the compaction strategies** and tie each to a workload shape.
- **Raise tombstones and compaction backlog**, since both are operational realities that distinguish experience from theory.

**Practice drill**

Design storage for a metrics system ingesting 500,000 writes per second with occasional point lookups. Explain why an LSM tree suits it and what the write path costs. Choose a compaction strategy and justify it against the three amplifications. Then describe what happens operationally when compaction cannot keep up, and what you would monitor to see it coming.

**Go deeper**

An LSM tree buffers writes in memory, flushes them as immutable sorted files, and merges those files in the background, so all disk writes are sequential.

**Eliminating in-place updates is what makes it fast.** A B-tree modifies the page containing a key, which is a random write, and random throughput is far below sequential on every storage medium. Accumulating changes and writing whole sorted files means the device only ever performs large sequential writes, which is why an LSM engine can absorb write rates that a B-tree cannot on identical hardware.

**The cost is moved to reads and to background work.** A key may reside in the memtable or in any file on disk, so a lookup potentially consults all of them — made tractable by a Bloom filter per file that eliminates most checks, and by sorted structure with sparse indexes that locate blocks cheaply. Compaction then keeps the file count bounded, which is what prevents the read path from degrading without limit.

**Write, read and space amplification form a trade in which improving one worsens another.** Size-tiered compaction rewrites data few times but leaves many files and duplicate data, favouring writes; levelled compaction keeps few files per read and little duplication at the cost of rewriting each byte many times. Neither is correct in general, and choosing between them is the main tuning decision an LSM deployment presents.

**Compaction is the operational characteristic that distinguishes these systems.** It consumes the same devices serving live traffic, so it must be throttled to avoid starving queries and must still keep pace with writes — and when it falls behind, files accumulate, reads slow, and the additional read load competes with the compaction that is already late. The degradation compounds, which makes compaction backlog one of the most valuable metrics available and throttling policy something to establish before it is urgent.

**Deletes behave counter-intuitively and surprise operators.** A deletion is a written tombstone rather than a removal, so freeing space requires compaction to merge past every file containing the original values — and in the interim, deleting large volumes increases disk usage. A system already short of space can therefore fail while attempting to reclaim it, precisely when that recovery is most needed.

**It is a workload-shaped choice rather than a generally superior design.** Write-heavy workloads with point lookups — metrics, events, logs, time series — are exactly what it is for, and read-heavy workloads with frequent range scans and in-place updates remain better served by B-trees. Latency stability matters too: LSM engines trade predictable response times for throughput, so a system with strict tail-latency requirements may reasonably choose the lower-throughput structure.

**Related patterns:** Bloom Filter · Write Ahead Log · MVCC · Batch Processing

---

## Search

### Inverted Index

*Map each term to the documents containing it, so text search becomes a lookup and intersection rather than a scan.*

> **When you hear…** Text search over many documents · scans too slow for interactive queries · relevance ranking required

**Flow:** `Analyse text` → `Extract terms` → `Term to document lists` → `Intersect postings` → `Rank results`

**The problem**

Finding documents containing a phrase by scanning every document costs time proportional to the corpus, which for millions of documents is far beyond interactive latency. A database index on the text column does not help, because it orders whole values rather than the words inside them.

Search also needs more than matching. Results must be ordered by relevance, which requires knowing how often terms occur, where, and how distinctive they are — information no ordinary index carries.

> **Index the words, not the documents**  
> Turning the relationship around — from documents containing terms to terms appearing in documents — makes a query a lookup of small sorted lists followed by an intersection. Matching stops depending on corpus size and starts depending on how many documents contain the query terms, and the same structure naturally holds the statistics that ranking needs.

**Mental model**

A dictionary of terms, each pointing to a sorted list of the documents containing it along with positions and frequencies.

1. **Analyse** — Text is tokenised, lowercased, stemmed and filtered.
2. **Index** — Each resulting term gains an entry pointing to this document.
3. **Store** — Postings lists are kept sorted and compressed.
4. **Query** — The same analysis is applied to the query, and postings are intersected.
5. **Rank** — Matches are scored using term statistics and returned in order.

> **The analysis chain must be identical at index and query time**  
> If documents are stemmed and queries are not, a search for running will not match a document indexed as run. The mismatch produces no error and no obviously broken behaviour — simply results that are quietly missing — and it is among the most common causes of search that appears to work while omitting relevant documents.

**How it works**

**Structure, analysis and scoring**

```text
THE INDEX
  "database" -> [3, 17, 42, 108, ...]
  "systems"  -> [8, 17, 42, 99,  ...]
  query "database systems"
    -> intersect the two lists -> [17, 42]
    -> cost is proportional to list length, not
       corpus size

POSTINGS CARRY MORE THAN IDS
  doc id, term frequency, positions
  -> frequency feeds ranking
  -> positions enable phrase queries

ANALYSIS PIPELINE
  "The Running Databases"
    tokenise   -> [The, Running, Databases]
    lowercase  -> [the, running, databases]
    stop words -> [running, databases]
    stem       -> [run, databas]
  -> the SAME pipeline must run on queries

SCORING
  term frequency      more occurrences, more relevant
  inverse document
  frequency           rare terms are more informative
  field length        a match in a short title beats
                      one in a long body
  -> BM25 combines these and is the modern default

COMPRESSION
  postings are sorted, so store gaps not ids
    [3, 17, 42, 108] -> [3, 14, 25, 66]
  -> small numbers compress extremely well
  -> indexes are often smaller than the text they
     index
```

1. **Use one analysis definition for indexing and querying** — Any divergence silently loses matches.
2. **Store positions where phrase search is needed** — They cost space and are the only way to match adjacency.
3. **Choose stemming deliberately per language** — Aggressive stemming raises recall and lowers precision.
4. **Compress postings with gap encoding** — Sorted lists of small differences compress dramatically.
5. **Rebuild rather than update in place** — Segment-based designs make updates cheap by never modifying existing segments.
6. **Keep the source of truth elsewhere** — A search index is derived data and must be rebuildable.

**Updates, segments and the near-real-time gap**

```text
UPDATES ARE AWKWARD
  changing one document could touch the postings
  list of every term in it
  -> so indexes are built in SEGMENTS

SEGMENTS
  new documents go into a new small segment
  segments are immutable once written
  searches query all segments and merge results
  background merging combines small segments into
  large ones
  -> the same shape as an LSM tree

DELETES
  marked in a deletion list, filtered at query time
  -> space reclaimed only when segments merge

NEAR REAL TIME
  a document is searchable once its segment is
  visible
  -> typically ~1 s after indexing, not instantly
  -> forcing visibility per document creates tiny
     segments and severe merge pressure

SIZING
  index size is typically 20-100% of the raw text,
  depending on positions, stored fields and
  compression
  -> storing full documents in the index doubles it,
     and is often unnecessary
```

| Metric | Value | Note |
|---|---|---|
| Query cost | postings length | **not corpus size** |
| Structure | immutable segments | merged in background |
| Visibility | ~1 s | near real time |
| Risk | analysis mismatch | silent missing results |

> **Forcing every document to be searchable immediately destroys indexing throughput**  
> Visibility requires a segment to be published, so demanding per-document immediacy creates enormous numbers of tiny segments, each of which must be searched and later merged. Query latency rises because there are more segments to consult, and merge pressure consumes the resources indexing needs. A one-second delay is the standard compromise, and insisting on less is usually a product requirement that has not been examined.

**Technologies**

| Layer | Options | Note |
|---|---|---|
| Search engines | Elasticsearch, OpenSearch, Solr | Distributed inverted indexes with ranking |
| Libraries | Lucene | The engine underneath most of the above |
| Embedded | SQLite FTS, Postgres full text | Sufficient for modest corpora; no separate system |
| Vector search | Embedding-based retrieval | Complementary; matches meaning rather than terms |
| Database LIKE queries | The naive alternative | Scans; no ranking; unusable at scale |
| Hybrid retrieval | Terms plus vectors | Increasingly the default for quality |

Hybrid retrieval deserves attention: term matching excels at exact and rare terms while vector search captures meaning, and combining their results typically outperforms either alone — which is why modern search systems rarely rely on one mechanism.

**Trade-offs**

**Text search approaches**

| Approach | Query cost | Ranking | Operational cost |
|---|---|---|---|
| Scan with pattern matching | Corpus size | None | None |
| Database full-text index | Postings length | Basic | Low; no new system |
| Dedicated search engine | Postings length | Rich | A cluster to run |
| Vector search | Approximate nearest neighbour | Semantic | Embedding pipeline |
| Hybrid | Both | Best | Highest |

The database full-text row is frequently the right starting point: for corpora in the millions rather than billions, it provides real search with ranking and no additional system to operate, and postponing a dedicated cluster until it is genuinely needed avoids a substantial operational commitment.

> **Ask before choosing it**  
> Is the requirement exact matching or relevance ranking? Filtering by known attributes is a database question, and treating it as search brings a whole system to solve something an index already handles — while genuine free-text relevance is exactly what this structure exists for.

**How it fails**

**How inverted indexes go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Relevant documents never returned | Analysis differing between index and query | One shared analysis definition |
| Phrase queries not working | Positions not stored | Enable position indexing |
| Indexing throughput collapses | Forcing immediate visibility | Accept near-real-time refresh |
| Queries slowing over time | Too many small segments | Allow merging to keep pace |
| Index far larger than expected | Storing full documents and all fields | Store only what is retrieved |
| Poor relevance | Default scoring unsuited to the corpus | Tune field weights; consider hybrid retrieval |
| Index and database disagree | Index treated as the source of truth | Keep it derived and rebuildable |

> **Treating the search index as a system of record makes data loss permanent**  
> Search indexes are derived structures with lossy analysis — stemmed terms, dropped stop words, normalised text — so the original content cannot be reconstructed from them. A system that writes only to the index has no authoritative copy, and any corruption, mapping change or accidental deletion is unrecoverable. The index must always be rebuildable from a source that holds the real data.

**Where it is used**

- **Site and product search**, the most common application.
- **Log and observability platforms**, searching enormous volumes of text.
- **Document management and knowledge bases**, where relevance ranking is the point.
- **Code search**, with analysis chains adapted to identifiers and symbols.
- **Retrieval for language model applications**, usually combined with vector search.

**In the interview**

The inversion and the cost model are the core; the analysis pipeline is where practical experience shows.

- **State the inversion** and why it changes the cost from corpus size to postings length.
- **Describe the analysis pipeline** and insist it is identical at index and query time.
- **Explain segments** — immutable, merged in the background — and the near-real-time visibility that follows.
- **Cover ranking briefly**: term frequency, rarity and field length, combined by a modern scoring function.
- **Say the index is derived**, never the system of record, and must be rebuildable.

**Practice drill**

Design search over ten million product listings with titles, descriptions and attributes. Specify the analysis pipeline, which fields carry positions, and how relevance is scored across fields. State how quickly a newly listed product becomes searchable and why. Then describe what happens when the analysis configuration changes, and how you would deploy that change without losing search.

**Go deeper**

An inverted index maps terms to the documents containing them, turning search into a lookup and intersection of sorted lists.

**The inversion changes the cost model fundamentally.** Scanning costs time proportional to the corpus, while intersecting postings costs time proportional to how many documents contain the query terms — which for a specific query is a tiny fraction of the whole. This is what makes interactive search over very large collections possible at all, and it is why no amount of optimising a scan approaches the same result.

**Analysis is where correctness actually lives.** Tokenisation, lowercasing, stop-word removal and stemming transform text into terms, and the same transformation must apply to queries or matches are lost. A mismatch produces no error and no visibly broken behaviour — only results that quietly omit relevant documents — which makes a single shared analysis definition an architectural requirement rather than a configuration convenience.

**Segment-based construction solves the update problem the same way LSM trees do.** Modifying one document could touch the postings of every term it contains, so indexes are built as immutable segments that are searched together and merged in the background. The consequences follow directly: deletes are filtered rather than removed, space is reclaimed only on merge, and a document becomes searchable when its segment is published rather than when it is indexed.

**Near-real-time visibility is a deliberate compromise worth defending.** Publishing a segment per document creates vast numbers of tiny segments, each searched on every query and each requiring later merging, which degrades both query latency and indexing throughput. The customary one-second delay is what keeps segments large enough to be efficient, and a requirement for instant visibility usually turns out, on examination, not to be a real requirement.

**Ranking is inseparable from the structure.** The same postings that answer whether a term appears also record how often and where, which is exactly what relevance scoring needs — frequency within the document, rarity across the corpus, and the length of the field matched. This is why search engines exist as distinct systems rather than as an index type: matching is the easy half, and ordering is what users actually judge.

**It is derived data and must be treated as such.** Analysis is lossy, so the original text cannot be recovered from the index, and any mapping change, corruption or mistaken deletion is unrecoverable without a separate authoritative copy. Systems that write only to the index have no source of truth, and the ability to rebuild from scratch is also what makes analysis changes deployable — since altering the pipeline means reindexing everything it ever processed.

**Related patterns:** Spatial Index · Scatter Gather · Materialized View · LSM Tree

---

### Spatial Index

*Index locations so that proximity queries examine only nearby candidates rather than scanning everything.*

> **When you hear…** Find everything within a radius · nearest neighbour queries · maps, delivery and location-based matching

**Flow:** `Points in space` → `Partition the plane` → `Query defines a region` → `Fetch nearby cells` → `Filter by true distance`

**The problem**

Finding every driver within two kilometres by computing the distance to every driver costs a full scan, and the arithmetic itself is expensive. At a hundred thousand drivers and thousands of queries a second, this is not viable.

Ordinary indexes do not help either. An index on latitude narrows one dimension and leaves a band stretching around the world; combining two single-column indexes still produces a rectangle far larger than the region actually wanted.

> **Reduce two dimensions to one while preserving locality**  
> If space is divided into cells and each cell is given an identifier such that nearby cells have nearby identifiers, then a proximity query becomes a small number of range lookups in an ordinary sorted index. The two-dimensional problem is mapped onto the one-dimensional structures databases are already good at, and the query examines only the cells the region touches.

**Mental model**

A partition of space into cells, each with an identifier that preserves locality, so that nearby points have nearby keys.

1. **Partition** — Space is divided into cells, uniformly or adaptively.
2. **Encode** — Each point is assigned the identifier of its cell.
3. **Query** — The search region is converted into the set of cells it covers.
4. **Fetch** — Candidates are retrieved from those cells.
5. **Refine** — True distances are computed to remove candidates outside the region.

> **The index returns candidates, not answers**  
> Cells are rectangles or squares and query regions are usually circles, so retrieved candidates include points outside the true radius. The exact distance filter is mandatory, and omitting it returns results that are visibly wrong at the edges — a driver four kilometres away appearing in a two-kilometre search.

**How it works**

**Geohash, grids and trees**

```text
GEOHASH  (interleave and encode)
  interleave the bits of latitude and longitude
  encode into a string
    "u4pruyd"  -> a cell a few metres across
    "u4pruy"   -> a larger cell containing it
  -> a shared prefix means spatial proximity
  -> a prefix range query finds everything in a cell

THE EDGE PROBLEM
  two points either side of a cell boundary can be
  metres apart with completely different geohashes
  -> ALWAYS query the target cell plus its 8
     neighbours
  -> forgetting this is the classic geohash bug

QUADTREE  (adaptive)
  split a cell into four when it holds too many
  points
  -> dense cities get fine cells, oceans get coarse
     ones
  -> uniform candidate counts regardless of density

R-TREE  (bounding boxes)
  hierarchical rectangles around groups of shapes
  -> handles polygons and lines, not only points
  -> what most spatial databases use

S2 AND H3
  sphere-aware cell systems
  -> no distortion near the poles
  -> hexagons (H3) have uniform neighbour distances,
     which suits movement and coverage problems
```

1. **Always include neighbouring cells in a query** — Points just across a boundary are near in space and far in key order.
2. **Choose the cell size from the typical query radius** — Cells much smaller than the radius mean many lookups; much larger means many candidates.
3. **Always filter by exact distance afterwards** — Cells approximate the region; they do not define it.
4. **Prefer adaptive structures where density varies** — A uniform grid returns thousands of candidates downtown and none in the countryside.
5. **Use a sphere-aware system for global data** — Flat grids distort badly at high latitudes.
6. **Consider update cost for moving objects** — Frequently moving points change cells and rewrite index entries.

**Density, updates and what to index**

```text
DENSITY VARIES ENORMOUSLY
  1 km cell in a city centre   -> 5,000 drivers
  1 km cell in farmland        -> 0
  -> a uniform grid is wrong everywhere at once
  -> adaptive structures keep candidate counts even

MOVING OBJECTS
  100,000 drivers, position every 4 s
  = 25,000 updates/s
  -> most updates stay within the same cell
  -> only rewrite the index entry when the cell
     changes
  -> this optimisation is frequently the difference
     between viable and not

NEAREST NEIGHBOUR
  expanding ring search:
    query the containing cell
    if fewer than k results, widen to the
    surrounding ring
    repeat
  -> bounded work in dense areas, correct in sparse
     ones

WHAT NOT TO DO
  computing distance to every row       full scan
  latitude BETWEEN ... AND longitude    a rectangle
  BETWEEN ...                           spanning far
                                        more than
                                        intended, and
                                        only usable
                                        with a
                                        composite
                                        index
```

| Metric | Value | Note |
|---|---|---|
| Query | cells, not scans | **candidates only** |
| Boundary | query 9 cells | not 1 |
| Density | adaptive structures | even candidate counts |
| Moving points | update on cell change | not every ping |

> **Querying only the containing cell misses everything just across a boundary**  
> Two points a metre apart on opposite sides of a cell edge fall in different cells with unrelated identifiers, so a query restricted to the containing cell omits genuinely near results. The symptom is subtle — nearby items occasionally missing, depending on exactly where the query point sits — and the fix is to always include the surrounding cells, which is cheap and non-negotiable.

**Technologies**

| Option | Structure | Note |
|---|---|---|
| PostGIS | R-tree indexes | Full geometry support in a relational database |
| Geohash in any key-value store | Prefix ranges | Simple; requires neighbour queries |
| S2 | Spherical cells | Sphere-aware; used for large-scale geospatial systems |
| H3 | Hexagonal cells | Uniform neighbour distances; good for movement and coverage |
| Redis geospatial commands | Sorted sets over geohashes | Convenient for real-time proximity |
| Search engines with geo queries | Combined with text filters | Useful when search and location are both needed |

Hexagonal systems have a property that matters for movement problems: every neighbour is the same distance away, whereas a square cell's diagonal neighbours are further than its edge neighbours — which distorts anything that reasons about spreading, coverage or travel between cells.

**Trade-offs**

**Spatial indexing approaches**

| Approach | Handles density | Shapes | Update cost |
|---|---|---|---|
| Full scan with distance | n/a | Any | None |
| Uniform grid or geohash | Poorly | Points | Low |
| Quadtree | Well | Points | Moderate |
| R-tree | Well | Points and polygons | Higher |
| S2 or H3 cells | Well, with level choice | Points and regions | Low |

Geohash remains popular despite its weaknesses because it requires nothing beyond an ordinary sorted index — a string prefix query in any database provides spatial search, which for many applications is enough and needs no specialised system.

> **Ask before choosing it**  
> Are the objects moving, and how unevenly are they distributed? Static, evenly spread points work with almost any approach; rapidly moving points in wildly varying density are what distinguish the structures, and they are also exactly the case most location-based products have.

**How it fails**

**How spatial indexes go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Nearby results missing | Only the containing cell queried | Include neighbouring cells |
| Results outside the radius | No exact distance filter | Refine candidates by true distance |
| Slow queries in dense areas | Uniform cells returning thousands of candidates | Adaptive structure or finer levels |
| Empty results in sparse areas | Cells too small for the density | Expanding ring search |
| Index write load excessive | Reindexing on every position update | Update only when the cell changes |
| Distortion at high latitudes | Flat grid over a sphere | Sphere-aware cell system |
| Distance calculations wrong | Euclidean arithmetic on latitude and longitude | Proper geodesic distance, or a projection |

> **Treating latitude and longitude as flat coordinates gives wrong distances**  
> A degree of longitude is about 111 kilometres at the equator and near zero at the poles, so Euclidean distance over raw coordinates is increasingly wrong as latitude rises — a two-kilometre search in northern Europe can return points several kilometres away. Correct results require geodesic distance or a suitable projection, and the error is easy to miss when testing near the equator or over small areas.

**Where it is used**

- **Ride-hailing and delivery platforms**, matching moving vehicles to requests in real time.
- **Mapping and local search**, finding places within a viewport or radius.
- **Geofencing**, detecting entry and exit from defined regions.
- **Location-based social features**, surfacing nearby users or content.
- **Logistics and coverage planning**, where hexagonal cells simplify adjacency reasoning.

**In the interview**

The locality-preserving reduction is the idea; the boundary problem is the detail that shows practical experience.

- **Explain the reduction**: two dimensions mapped to one key such that nearby points have nearby keys.
- **Raise the boundary problem** and say that a query must include the surrounding cells.
- **Insist on the exact distance filter**, because cells are approximations of the region.
- **Address density**, since uniform cells fail simultaneously in cities and in open country.
- **Discuss update cost for moving objects**, and the optimisation of reindexing only on cell change.

**Practice drill**

Design driver location search for a ride-hailing service: 100,000 drivers updating every 4 seconds, with queries for all drivers within 2 kilometres. Choose the index structure and cell size, and justify both. Describe the query, including how you handle cell boundaries and exact distance. Then compute the index write load and explain the optimisation that reduces it, and say how your design behaves in a dense city centre compared with a rural area.

**Go deeper**

A spatial index organises locations so that proximity queries examine only the region of interest rather than the whole dataset.

**It works by reducing dimensions while preserving locality.** Assigning each point a cell identifier where nearby cells have nearby identifiers turns a two-dimensional problem into range queries over an ordinary sorted index — which is precisely what existing storage engines do well. Composing two single-column indexes cannot achieve this, because each narrows one dimension independently and the intersection is a rectangle far larger than the region actually wanted.

**Cell boundaries create the characteristic bug of the whole family.** Two points a metre apart on opposite sides of an edge receive unrelated identifiers, so querying only the containing cell silently omits genuinely nearby results. Including the surrounding cells is inexpensive and mandatory, and the reason the mistake persists is that the failure is intermittent and depends on where the query point happens to fall.

**The index produces candidates and the application produces answers.** Cells are rectangles or hexagons while query regions are typically circles, so retrieved points include some outside the true radius. The exact distance filter is part of the query rather than an optimisation, and computing that distance correctly matters too — Euclidean arithmetic on raw coordinates is wrong by a factor that grows with latitude.

**Density variation is what separates the structures.** A uniform grid sized for a city centre returns nothing useful in open country, and one sized for open country returns thousands of candidates downtown; adaptive structures subdivide where points are dense so that candidate counts stay roughly even regardless of location. Since most location-based products serve exactly this kind of wildly uneven distribution, the choice usually matters.

**Moving objects change the economics entirely.** A fleet reporting position every few seconds generates enormous index write volume, and the optimisation that makes it viable is recognising that most updates do not change the cell — so the index entry need not be rewritten. This single observation frequently determines whether a real-time location system is affordable, and it is invisible if the problem is considered only as a query problem.

**Cell system choice encodes assumptions worth being explicit about.** Flat grids distort on a sphere and become badly wrong at high latitudes; spherical systems avoid that; hexagonal systems additionally give every neighbour an equal distance, which matters for anything reasoning about spread, coverage or movement between cells. Selecting one because it is familiar rather than because its geometry suits the problem is a decision that becomes expensive to revisit once identifiers are stored everywhere.

**Related patterns:** Inverted Index · Consistent Hashing · Sharding · Scatter Gather

---

## Scheduling

### Priority Queue

*Order pending work by importance rather than arrival, so limited capacity is spent on what matters most first.*

> **When you hear…** Mixed workloads with different urgency · bulk work delaying interactive work · capacity insufficient for everything at once

**Flow:** `Work arrives with priority` → `Ordered by importance` → `Workers take the highest` → `Low priority waits` → `Ageing prevents starvation`

**The problem**

A queue processes a password reset email behind twenty thousand marketing messages because they arrived first. The user waits ten minutes for something that should take seconds, while the system works on items nobody is waiting for.

Separating everything into its own queue helps until there are eight queues, each with its own workers, each idle while another is overwhelmed — capacity is partitioned rather than shared, and the partitioning is wrong as soon as the traffic mix changes.

> **Order by what the work is worth, not by when it arrived**  
> First-in-first-out is only correct when all items are equally urgent, which is rarely true. Ordering by priority means that when capacity is scarce — exactly when the ordering matters — it is spent on the most valuable work, and when capacity is ample the ordering costs nothing because everything is processed promptly anyway.

**Mental model**

A single pool of pending work ordered by a priority value, with workers always taking the highest-priority item available.

1. **Classify** — Each item is assigned a priority when it is enqueued.
2. **Order** — The queue maintains ordering by priority, then by arrival.
3. **Dequeue** — Workers always take the most important available item.
4. **Age** — Long-waiting items gain priority so they are not starved.
5. **Observe** — Wait time is tracked per priority, not in aggregate.

> **Low-priority work can wait forever if higher-priority work never stops arriving**  
> A queue that always serves the most important item will never serve the least important one when the arrival rate of important work exceeds capacity. The low-priority backlog grows silently and indefinitely, and because the aggregate metrics look healthy — most items are processed quickly — the starvation is invisible until someone notices work from three weeks ago.

**How it works**

**Priorities, ageing and fairness**

```text
KEEP THE LEVELS FEW
  1  interactive: password resets, verification
  2  user-visible: order confirmations
  3  background: reports, exports
  4  bulk: marketing, analytics
  -> three to five levels
  -> more than that and nobody can say what the
     difference between 6 and 7 means

AGEING  (the anti-starvation mechanism)
  effective = base_priority - (waiting_minutes / 10)
  -> a level-4 item waiting 30 minutes competes with
     level 1
  -> guarantees eventual service without abandoning
     the ordering

WEIGHTED FAIR ALTERNATIVE
  reserve capacity per class instead of ordering
    70% interactive, 20% background, 10% bulk
  -> every class always progresses
  -> high priority may wait behind reserved bulk
     capacity
  -> better when starvation is unacceptable and
     strict ordering is not required

IMPLEMENTATION NOTES
  a heap in memory is trivial and not durable
  a database queue: ORDER BY priority, created_at
                    FOR UPDATE SKIP LOCKED
  brokers vary: some support priorities natively,
  many do not and need separate queues per level
```

1. **Use a small number of meaningful levels** — Fine-grained priorities cannot be assigned consistently by anyone.
2. **Implement ageing from the start** — Starvation is certain under sustained load and invisible in aggregate metrics.
3. **Monitor wait time per priority level** — An overall percentile hides a starved class entirely.
4. **Consider weighted capacity instead of strict ordering** — Reserving shares guarantees progress for every class.
5. **Do not let callers set their own priority** — Everything becomes urgent; priority must be assigned by policy.
6. **Keep the ordering cheap** — A priority computation that queries other systems becomes the bottleneck.

**Where priorities go wrong in practice**

```text
PRIORITY INFLATION
  teams choose the priority for their own work
  -> everything becomes level 1 within a quarter
  -> the queue is FIFO again, with extra machinery
  -> priorities must be assigned by policy, centrally

FALSE PRECISION
  priority 1 through 100
  -> nobody can justify 43 over 47
  -> and the distinction does not change behaviour
  -> use levels people can name

THE REAL QUESTION IS CAPACITY
  if high-priority work alone exceeds capacity,
  prioritisation only decides who is disappointed
  -> it reorders, it does not create throughput
  -> a permanently deep queue is a capacity problem
     wearing a scheduling costume

MEASURING CORRECTLY
  overall p99 wait: 2 s   -> looks excellent
  level 4 p99 wait: 6 days -> nobody is looking
  -> always segment by priority
```

| Metric | Value | Note |
|---|---|---|
| Ordering | by value | **not arrival** |
| Required | ageing | or starvation |
| Levels | 3-5 | nameable |
| Metric | wait per level | not aggregate |

> **Letting each team set its own priority converts the queue back into first-in-first-out**  
> When the caller chooses, everything becomes urgent — not through bad faith, but because every team's work genuinely matters to them. Within a couple of quarters every item is at the top level and the ordering carries no information, while the system retains all the complexity of supporting it. Priority has to be assigned by a policy that compares work across teams, which is an organisational decision rather than a technical one.

**Technologies**

| Option | Mechanism | Note |
|---|---|---|
| Database-backed queue | Order by priority with skip locked | Simple, durable, flexible ordering |
| Brokers with native priorities | Priority field on messages | Support and semantics vary considerably |
| Separate queues per level | Workers prefer higher queues | Works everywhere; capacity is partitioned |
| Weighted fair queueing | Reserved shares per class | Guarantees progress for every class |
| In-memory heaps | Within a single process | Fast, not durable |
| Work queue without priorities | First in, first out | Correct when urgency is uniform |

Separate queues per level with workers that prefer the higher ones is the most portable implementation and the most commonly deployed, because many brokers either lack priority support or implement it with surprising semantics under load.

**Trade-offs**

**Scheduling policies**

| Policy | Urgent work | Starvation risk | Complexity |
|---|---|---|---|
| First in, first out | Waits its turn | None | Lowest |
| Strict priority | Served first | High | Low |
| Priority with ageing | Served first | Bounded | Moderate |
| Weighted fair queueing | Served within its share | None | Moderate |
| Separate queues and pools | Isolated | None | Partitioned capacity |

Weighted fair queueing is underused relative to strict priority: guaranteeing every class a share of capacity removes starvation by construction, and for most systems the difference between urgent work being served first and being served within its reserved share is not worth the risk of a class that never progresses.

> **Ask before choosing it**  
> Is the queue permanently deep, or only deep during bursts? Prioritisation reorders a backlog and does not shorten it, so a queue that never drains is a capacity problem — and scheduling will merely determine which work is permanently delayed.

**How it fails**

**How priority queues go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Low-priority work never processed | Strict priority under sustained load | Ageing, or reserved capacity per class |
| Everything at the top level | Callers choosing their own priority | Central policy assigns priorities |
| Starvation unnoticed for weeks | Only aggregate metrics tracked | Wait time per priority level |
| Priorities meaningless | Too many levels | Reduce to a few nameable classes |
| Ordering becomes a bottleneck | Expensive priority computation | Compute it at enqueue time and store it |
| Capacity partitioned badly | One queue and pool per level | Shared pool with ordering |
| Urgent work still delayed | Long-running low-priority items holding workers | Bound task duration; preempt where possible |

> **Starvation hides behind healthy aggregate metrics**  
> When ninety-five per cent of items are high priority and served in seconds, the overall percentiles look excellent while the remaining class waits for days. Nothing alerts, nothing errors, and the problem surfaces when someone notices an export request from a fortnight ago still pending. Segmenting wait-time metrics by priority is what makes this visible, and it must be in place before the load that causes it arrives.

**Where it is used**

- **Notification and email delivery**, separating transactional messages from bulk sends.
- **Job processing platforms**, running interactive tasks ahead of scheduled reports.
- **Support and incident systems**, where severity determines processing order.
- **Media processing pipelines**, prioritising user-initiated work over re-encoding.
- **Operating system and database schedulers**, the original application of the idea.

**In the interview**

Raise starvation before being asked; it is the failure that defines the pattern.

- **Give a small set of named levels** rather than a numeric range nobody can apply consistently.
- **Introduce ageing immediately** as the mechanism that bounds how long low-priority work waits.
- **Insist on per-level metrics**, because aggregate percentiles conceal a starved class completely.
- **Warn about priority inflation** and say assignment must be by central policy, not by the caller.
- **Distinguish reordering from capacity**: a permanently deep queue needs throughput, not scheduling.

**Practice drill**

A queue handles password resets, order confirmations, scheduled reports and marketing emails with capacity for 60% of peak volume. Define the priority levels and the ageing function. Compute how long a marketing email waits during a peak hour with and without ageing. Then describe the metrics you would publish, and explain how you would recognise starvation if ageing were not implemented.

**Go deeper**

A priority queue orders pending work by importance so that scarce capacity is spent on the most valuable items first.

**It only matters when capacity is insufficient, which is exactly when it matters most.** With ample throughput everything is processed promptly and ordering is irrelevant; under load the ordering determines whether a user waits seconds or minutes for something urgent. This means the value of the mechanism is concentrated entirely in the periods that define the product's perceived reliability.

**Strict priority guarantees starvation whenever important work saturates capacity.** If high-priority arrivals alone exceed throughput, the lowest class is never reached, and its backlog grows without bound. Ageing — raising effective priority with waiting time — bounds the delay while preserving the ordering, and weighted capacity reservation avoids the problem entirely by guaranteeing each class a share. One of the two is necessary; strict priority without either is a design that works until it is needed.

**The failure is invisible in aggregate measurements.** When most items are high priority and served quickly, overall percentiles look excellent while one class waits for days. Because nothing errors and nothing alerts, starvation is typically discovered by a person noticing stale work rather than by monitoring — which makes segmenting wait times by priority level a prerequisite for operating the pattern rather than an enhancement.

**Priority assignment is an organisational problem in technical clothing.** When each team sets the priority of its own work, everything reaches the top level within a few quarters, not through bad faith but because every team's work genuinely is important to them. Meaningful ordering requires a policy that compares work across teams, and a system that cannot make that comparison will end up with a first-in-first-out queue carrying the overhead of a priority one.

**Few levels beat many.** A range of a hundred priorities invites distinctions nobody can justify or apply consistently, and the fine gradations do not change behaviour. Three to five classes that people can name — interactive, user-visible, background, bulk — are assignable without debate and produce the same scheduling outcomes, which is the practical test of whether a level is worth having.

**Scheduling reorders a backlog; it does not shorten one.** A queue that is permanently deep has a capacity shortfall, and prioritisation only decides which work is permanently delayed. Recognising that distinction prevents a great deal of wasted tuning: if the queue drains between bursts, ordering is the right tool, and if it never drains, the answer is more throughput or less work regardless of how the remaining capacity is allocated.

**Related patterns:** Work Queue · Delay Queue · Load Shedding · Backpressure

---

### Delay Queue

*Schedule work to become available at a future time, so retries, reminders and deferred actions need no polling loop.*

> **When you hear…** Retries with backoff · reminders and timeouts · anything that must happen later rather than now

**Flow:** `Enqueue with a delay` → `Invisible until due` → `Becomes available` → `Worker processes it` → `Reschedule if needed`

**The problem**

A retry after thirty seconds, a reminder in three days and a cart abandonment check in an hour all require something to happen at a future moment. Implementing this by scanning a table every minute for due rows means a query that scales with pending work and a granularity no finer than the scan interval.

The scan is also wasteful and fragile. Most executions find nothing, the query competes with production traffic as the table grows, and a missed or delayed scan silently postpones everything scheduled within it.

> **Make the queue itself aware of time**  
> If an item can be enqueued with a time before which it is invisible, then scheduling is a property of the message rather than a separate scanning mechanism. Nothing polls, nothing scans, and the work becomes available exactly when it is due — with the queue handling ordering, durability and delivery as it does for immediate work.

**Mental model**

A queue where each item carries a visibility time. Items exist but cannot be received until that moment arrives.

1. **Enqueue** — The item is submitted with a delay or an absolute due time.
2. **Hide** — It is durable but invisible to consumers until due.
3. **Surface** — At the due time it becomes available like any other item.
4. **Process** — A worker receives and handles it normally.
5. **Reschedule** — Work needing another attempt is re-enqueued with a new delay.

> **Delivery is at-least-once and approximately on time, not exactly either**  
> Delayed items can be delivered late under load and can be delivered more than once, so anything scheduled must be idempotent and must tolerate arriving somewhat after its due time. Designs treating a delay queue as a precise timer — for financial cut-offs or hard deadlines — are relying on a guarantee it does not provide.

**How it works**

**Implementations and their limits**

```text
NATIVE QUEUE DELAY
  enqueue with delay_seconds
  + simple, durable, no extra infrastructure
  - many brokers cap the delay (commonly 15 minutes)
  -> fine for retries, not for three-day reminders

SORTED SET BY DUE TIME
  score = due timestamp
  a poller fetches everything with score <= now
  + arbitrary delays
  + efficient: one range query regardless of size
  - one component must poll, and must be reliable

DATABASE TABLE
  WHERE due_at <= now() ORDER BY due_at
  FOR UPDATE SKIP LOCKED
  + transactional with business data
  + arbitrary delays, easy inspection
  - polling load grows with table size unless indexed
    and pruned

DEAD LETTER AND REDRIVE
  delayed retries commonly implement backoff:
    attempt fails -> re-enqueue with 2^n seconds
    after N attempts -> dead letter queue

LONG DELAYS
  for delays beyond the broker's cap, chain them:
    enqueue with 15 min, re-enqueue on arrival
  -> works, but multiplies message volume
  -> a sorted set or table is usually cleaner
```

1. **Match the mechanism to the delay length** — Broker delays suit seconds to minutes; long delays need a scheduled store.
2. **Make every delayed task idempotent** — Delivery is at-least-once, and duplicates are normal.
3. **Store the due time, not the delay** — An absolute time survives restarts and re-enqueues unambiguously.
4. **Include cancellation in the design** — Scheduled work is frequently made obsolete before it runs.
5. **Expect late delivery under load** — The due time is the earliest it can run, not a guarantee.
6. **Prune completed and cancelled items** — A scheduling table that only grows eventually degrades the scan.

**Cancellation and the check-on-execution rule**

```text
THE PROBLEM
  schedule "cart abandonment email in 1 hour"
  the user completes the purchase 10 minutes later
  -> the email must not be sent

OPTION 1  DELETE THE SCHEDULED ITEM
  requires knowing its identifier and the store
  supporting removal
  -> queues generally do NOT support cancelling an
     enqueued message
  -> works with a database table or sorted set

OPTION 2  CHECK CONDITIONS ON EXECUTION
  the task re-reads current state when it runs:
    "is this cart still abandoned?"
    if not, do nothing
  -> works with ANY mechanism
  -> the task must be written to expect obsolescence
  -> this is the robust default

-> even where cancellation is possible, the check on
   execution should remain: a cancellation can be
   missed, and the state may have changed in ways
   nobody anticipated

TIMING EXPECTATIONS
  due at 10:00 might run at 10:00:03 or 10:04
  -> acceptable for reminders and retries
  -> unacceptable for anything with a hard deadline
```

| Metric | Value | Note |
|---|---|---|
| Scheduling | in the queue | **no scanning** |
| Delivery | at-least-once | idempotent tasks |
| Timing | earliest, not exact | late under load |
| Cancellation | check on execution | the robust default |

> **Most queues cannot cancel a message once it has been enqueued**  
> Scheduled work is frequently invalidated before it runs — the user acts, the order is cancelled, the condition resolves — and a broker that accepted a delayed message usually offers no way to withdraw it. The reliable pattern is for the task to verify its own preconditions when it executes and do nothing if they no longer hold, which works regardless of the mechanism and also covers cases nobody thought to cancel.

**Technologies**

| Option | Delay range | Note |
|---|---|---|
| Queue native delay | Seconds to minutes | Simplest; commonly capped around 15 minutes |
| Redis sorted set by due time | Arbitrary | Efficient range query; needs a reliable poller |
| Database table with a due column | Arbitrary | Transactional; easy to inspect and cancel |
| Scheduler services | Arbitrary | Managed; per-schedule cost and limits |
| Durable workflow engines | Arbitrary | Delays are a first-class step |
| Cron | Fixed times | For recurring work, not per-item scheduling |

The distinction between cron and a delay queue is worth stating: cron runs the same job on a schedule, while a delay queue schedules an individual item — and using cron plus a scan to emulate per-item scheduling is exactly the polling approach the pattern replaces.

**Trade-offs**

**Ways to do something later**

| Approach | Granularity | Scales with pending items | Cancellation |
|---|---|---|---|
| Polling scan | Scan interval | Poorly | Easy |
| Queue native delay | Seconds | Well | Usually impossible |
| Sorted set by due time | Seconds | Well | Easy |
| Scheduler service | Varies | Well | Usually possible |
| Workflow engine timer | Seconds | Well | Built in |
| In-process timer | Milliseconds | Poorly | Easy, and lost on restart |

In-process timers are the shortcut that fails quietly: a scheduled callback held in memory disappears on deploy or crash, so the work simply never happens and nothing records that it was lost.

> **Ask before choosing it**  
> How long is the delay, and what happens if the work runs twice or arrives late? Short delays with idempotent tasks suit a queue's native mechanism; long delays and anything needing cancellation point to a store you can query and modify.

**How it fails**

**How delay queues go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Scheduled work lost on deploy | In-process timers | Durable scheduling store |
| Obsolete actions taken | No condition check at execution | Verify preconditions when the task runs |
| Long delays unsupported | Broker delay cap | Sorted set, table, or chained re-enqueues |
| Scheduling scan slows over time | Table never pruned | Delete completed items; index the due column |
| Duplicate actions | At-least-once delivery | Idempotent tasks |
| Everything due at once | Round-number scheduling | Jitter the due times |
| Retry storms | Fixed backoff across many items | Exponential backoff with jitter |

> **Scheduling large numbers of items for the same instant produces a self-inflicted spike**  
> Delays computed as round numbers — an hour from now, midnight tomorrow — cluster due times, so thousands of tasks become available simultaneously and overwhelm the workers. The pattern is easy to create accidentally, since every natural way of expressing a delay produces alignment, and the remedy is to add randomness to due times whenever many items are scheduled together.

**Where it is used**

- **Retry with backoff**, the most common use, re-enqueuing failures with growing delays.
- **Reminders and notifications**, scheduled days or weeks ahead.
- **Abandonment and follow-up flows**, acting if a condition persists after an interval.
- **Timeouts on long-running operations**, scheduling a check that fires if nothing completes.
- **Durable workflows**, where waiting for a period is an ordinary step.

**In the interview**

Contrast it with polling, then cover cancellation — which is where most designs are incomplete.

- **Explain why polling is the wrong shape**: cost grows with pending work and granularity is bounded by the interval.
- **Match the mechanism to the delay length**, since broker delays are typically capped at minutes.
- **Say delivery is at-least-once and approximate**, so tasks must be idempotent and tolerate lateness.
- **Raise cancellation** and propose checking preconditions at execution as the mechanism-independent answer.
- **Add jitter to due times**, because natural scheduling expressions cluster and produce spikes.

**Practice drill**

Design scheduling for three cases: retry a failed payment in 30 seconds, send a reminder in 3 days, and check cart abandonment in 1 hour. Choose a mechanism for each and justify it. Describe how the abandonment check avoids emailing someone who has already purchased. Then explain what happens if fifty thousand reminders are all scheduled for 09:00 tomorrow, and how you would prevent it.

**Go deeper**

A delay queue makes work available at a future time, so deferred actions need no polling loop or scanning job.

**It moves scheduling into the queue, which changes the cost model.** Scanning a table for due items costs work proportional to pending volume and offers granularity no finer than the scan interval, while a queue that understands visibility times surfaces each item exactly when it is due with no periodic query at all. The scheduling becomes a property of the message rather than a separate mechanism that must itself be reliable.

**The mechanism has to match the delay length.** Broker-native delays are typically capped at minutes, which covers retries and short timeouts but not reminders days away — those need a store that can hold arbitrary due times and be queried by range. Chaining short delays to emulate long ones works and multiplies message volume, which is usually a sign that a sorted set or table is the better structure.

**Timing is a lower bound, not a guarantee.** Items can surface late under load and can be delivered more than once, so tasks must be idempotent and must tolerate running after their intended moment. Treating the mechanism as a precise timer — for a financial cut-off or a hard deadline — relies on a property it does not have, and such requirements need a design that verifies the time itself rather than trusting delivery.

**Cancellation is where most designs are incomplete.** Scheduled work is frequently invalidated before it runs, and queues generally offer no way to withdraw an enqueued message. The reliable approach is for the task to check its own preconditions when it executes and do nothing if they no longer hold, which works with every mechanism and additionally covers the cases nobody anticipated cancelling — making it the right default even where removal is possible.

**Aligned due times produce self-inflicted spikes.** Every natural way of expressing a delay — an hour from now, nine o'clock tomorrow — clusters items at the same instant, so a large batch becomes available simultaneously and saturates the workers. Adding randomness to due times whenever many items are scheduled together spreads the load, and it is the same reasoning that makes jitter essential in retry policies.

**In-process timers are the shortcut that loses work silently.** A callback scheduled in memory disappears on deploy, crash or scale-in, and nothing records that it was lost — the reminder simply never arrives and no error is generated anywhere. Any scheduled action that matters needs a durable store behind it, which is the minimum the pattern provides and the reason it exists as infrastructure rather than as a language feature.

**Related patterns:** Work Queue · Priority Queue · Retry With Jitter · Durable Workflow

---

### Durable Workflow

*Persist each step's completion so a multi-step process survives crashes, restarts and long waits without losing its place.*

> **When you hear…** Multi-step business processes · operations spanning minutes to months · orchestration that must survive deploys

**Flow:** `Define steps` → `Execute one` → `Persist the result` → `Crash safe` → `Resume where it stopped`

**The problem**

An onboarding process creates an account, sends a verification email, waits up to three days for confirmation, provisions resources and schedules a follow-up. Holding this in a function means the process dies with the process — and three days will certainly contain a deploy.

Rebuilding it from queue messages and database flags works and scatters the logic: the sequence exists nowhere in readable form, and answering where a particular customer's onboarding has reached requires querying several systems and inferring.

> **Make progress durable so the code can be written as a sequence**  
> If the result of every completed step is persisted, execution can be resumed by replaying that history and skipping what already happened. The process can then be expressed as ordinary sequential code — including waits of arbitrary length — because no crash, deploy or restart loses its position, and the code itself is the readable definition of the flow.

**Mental model**

A durably recorded history of completed steps. On resumption the history is replayed to reconstruct state, and execution continues from the first step with no recorded result.

1. **Define** — The workflow is written as a sequence of steps with waits and branches.
2. **Execute** — Each step runs, and its result is persisted before moving on.
3. **Wait** — Timers and external signals suspend the workflow without consuming resources.
4. **Recover** — After any interruption the history is replayed and execution resumes.
5. **Complete** — The final state and the full history remain available for inspection.

> **Workflow code must be deterministic, and this constrains how it is written**  
> Because resumption replays the recorded history, the code must take the same path given the same results — so current time, random values and direct external calls cannot appear in workflow logic. They must be performed inside steps whose results are recorded. Code that ignores this appears to work and fails only on recovery, which is both rare and exactly when correctness matters most.

**How it works**

**Steps, determinism and versioning**

```text
WHAT THE ENGINE RECORDS
  step 1 createAccount   -> {id: 4471}
  step 2 sendEmail       -> {sent: true}
  step 3 awaitConfirm    -> pending
  -> on resume, steps 1 and 2 are not re-executed;
     their recorded results are returned

DETERMINISM RULES
  NOT in workflow code:
    current time, random numbers, uuid generation
    direct network or database calls
    iteration over an unordered collection
  INSTEAD:
    perform them inside a step, so the result is
    recorded and replayed identically

-> the workflow decides; steps act

WAITS ARE FREE
  sleep for 3 days
  -> the workflow is suspended, not running
  -> no thread, no memory, no polling
  -> this is what makes long processes practical

VERSIONING IS THE HARD PART
  a workflow started last week is running the old
  code path
  deploy new code with different steps
  -> replay may not match the recorded history
  -> engines provide versioning primitives; they
     must be used deliberately
  -> in practice: avoid changing the step sequence
     of long-running workflows; branch on a version
     marker instead
```

1. **Keep all non-determinism inside steps** — Replay must follow the same path; time and randomness in workflow code break that.
2. **Make every step idempotent** — A step may be retried after a crash between execution and recording.
3. **Use the engine's waiting primitives** — A suspended workflow consumes nothing; a polling loop consumes everything.
4. **Plan versioning before long workflows are deployed** — In-flight instances outlive several releases.
5. **Keep workflow logic free of business rules that change often** — Frequent changes to long-running definitions are where versioning pain concentrates.
6. **Expose workflow state to operations** — Being able to answer where an instance is stuck is a principal benefit.

**What it replaces, and what it costs**

```text
WITHOUT AN ENGINE
  state machine in a database table
  queue messages between steps
  a scheduler for waits
  retry logic per step
  reconciliation for stuck instances
  -> all of this is written, tested and operated by
     you
  -> and the sequence is not visible anywhere

WITH AN ENGINE
  the sequence is the code
  progress, retries, waits and history are provided
  -> plus a service to operate, or a hosted one to
     pay for

COSTS
  an engine to run or a service to buy
  determinism constraints on how code is written
  versioning complexity for long workflows
  a new operational surface to understand

WHEN IT IS WORTH IT
  several multi-step processes, not one
  processes lasting longer than a deploy cycle
  failures that currently require manual repair
  a real need to answer "where is this instance?"

WHEN IT IS NOT
  a single three-step flow
  -> a saga with an outbox is lighter and sufficient
```

| Metric | Value | Note |
|---|---|---|
| Progress | durable per step | **survives anything** |
| Waits | free | no resources held |
| Constraint | determinism | in workflow code |
| Hard part | versioning | long-running instances |

> **Changing a workflow definition while instances of it are running is the principal operational hazard**  
> Instances started before a deploy replay their recorded history against the new code, and if the step sequence has changed the replay may not match — producing a failure that affects only in-flight instances and only on recovery. Long-running workflows routinely outlive several releases, so versioning is not an edge case but the normal condition, and engines provide explicit primitives precisely because naive changes break running work.

**Technologies**

| Option | Model | Note |
|---|---|---|
| Workflow engines | Code-as-workflow with replay | Full featured; an engine to operate or buy |
| Cloud step functions | State machines as configuration | Managed; less expressive than code |
| Saga orchestration by hand | Your own state machine and queues | Fine for a small number of short flows |
| Job scheduler with state | Table-driven progress | Simple; you build resumption yourself |
| Event sourcing | State derived from recorded events | Overlapping idea, broader commitment |
| Queue chaining | Each step enqueues the next | Works; the flow is invisible and hard to query |

Queue chaining is what most teams build before adopting an engine, and its defining weakness is not reliability but legibility: the process works, and nobody can see it, so answering an operational question means reconstructing the sequence from message handlers spread across services.

**Trade-offs**

**Orchestration approaches**

| Approach | Survives crashes | Long waits | Flow visible |
|---|---|---|---|
| In-process function | No | No | Yes, in code |
| Queue chaining | Yes | Yes | No |
| State machine table | Yes | Yes | Partly |
| Managed step functions | Yes | Yes | Yes, as configuration |
| Durable workflow engine | Yes | Yes | Yes, as code |

The last column is the one that distinguishes durable workflows from the alternatives that also survive crashes: several approaches keep a process running reliably, and few of them let a person read the process or ask where a particular instance currently is.

> **Ask before choosing it**  
> How many multi-step processes are there, and how long do they run? One short flow does not justify an engine, while several processes spanning days — with failures currently repaired by hand — is exactly the situation the pattern exists for.

**How it fails**

**How durable workflows go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Replay fails after a deploy | Step sequence changed with instances in flight | Version workflows explicitly |
| Non-deterministic behaviour on recovery | Time or randomness in workflow code | Move it into steps |
| Steps executed twice | Crash between execution and recording | Idempotent steps |
| Resources consumed while waiting | Polling instead of the engine's timers | Use durable sleep and signals |
| Instances stuck indefinitely | No timeout on external signals | Bound every wait; define what happens on expiry |
| History growing without limit | Very long-lived workflows with many steps | Continue as a new instance; archive history |
| Engine becomes a single point of failure | All processes depending on it | Run it highly available; understand its failure modes |

> **Non-determinism in workflow code fails only during recovery, which is the worst possible time**  
> Reading the clock or generating a value directly in workflow logic produces correct behaviour on the first execution and a different path on replay, so the defect is invisible in testing and normal operation. It appears when a workflow resumes after a crash — precisely when the process was relying on durability — and manifests as an instance that cannot continue rather than as an obvious error in the code that caused it.

**Where it is used**

- **Order fulfilment and provisioning**, coordinating steps across several services with compensation.
- **Onboarding and approval flows**, waiting days for human action without holding resources.
- **Payment and settlement processes**, requiring durable progress and full auditability.
- **Infrastructure automation**, where a long provisioning sequence must survive operator restarts.
- **Data pipeline orchestration**, with dependencies, retries and long-running steps.

**In the interview**

Lead with durable progress enabling sequential code, then name determinism and versioning as the costs.

- **Say each step's result is persisted**, which is what allows the process to be written as ordinary sequential code.
- **Note that waits are free** — a suspended workflow holds no resources — which is what makes multi-day processes practical.
- **State the determinism constraint** and give the specific things that cannot appear in workflow code.
- **Raise versioning unprompted** as the main operational difficulty, since long instances outlive deploys.
- **Be honest about when it is unnecessary**: one short flow is better served by a saga with an outbox.

**Practice drill**

Design onboarding that creates an account, sends verification, waits up to 3 days for confirmation, provisions resources and schedules a follow-up in 7 days. Write it as a sequence and mark which parts must be steps rather than workflow logic. Say what happens if the service restarts during the 3-day wait and if it restarts between provisioning and recording that result. Then describe how you would add a new step without breaking instances already running.

**Go deeper**

A durable workflow persists the result of every completed step so a multi-step process can resume after any interruption.

**Durable progress is what allows the process to be written as ordinary code.** Because each step's outcome is recorded, execution can be reconstructed by replaying that history and continuing from the first unfinished step — so a sequence spanning days can be expressed as a readable series of statements rather than scattered across message handlers and database flags. The legibility is not a side benefit; for most teams it is the principal one.

**Waiting costs nothing, which changes what is practical.** A suspended workflow holds no thread, no memory and no connection, so waiting three days for a human to act is as cheap as waiting three seconds. Processes that would otherwise be decomposed into scheduled jobs and reconciliation passes, purely to avoid holding resources, can be written as they are actually understood.

**Replay imposes determinism, and this constrains how the code is written.** Reading the clock, generating random values or calling an external system directly inside workflow logic produces a different path on replay than on the original execution. Such code behaves correctly during normal operation and fails only on recovery — when durability was the entire point — which makes the rule about keeping non-determinism inside recorded steps a correctness requirement rather than a style preference.

**Versioning is the dominant operational difficulty.** Instances started before a release replay their recorded history against the new definition, and a changed step sequence can make that replay inconsistent. Since long-running workflows routinely outlive several deploys, this is the normal condition rather than an edge case, and it argues for keeping frequently changing business rules out of workflow definitions and inside steps where they can be altered freely.

**Steps must be idempotent because the crash window is real.** A process can fail between performing a step's effect and recording its result, so recovery re-executes it. Without idempotence that means the effect happens twice, which for anything touching money or external systems is exactly what the durability was meant to prevent — making step design the same problem as any at-least-once message consumer.

**It is worth its cost when there are several long processes, not one.** An engine to operate, determinism constraints and versioning complexity are a real commitment, and a single three-step flow is better served by a saga with an outbox. The signal that it is warranted is a collection of processes spanning longer than a deploy cycle, with failures currently repaired by hand and no reliable way to answer where a particular instance has reached — which is precisely the situation the pattern was built to remove.

**Related patterns:** Saga · Delay Queue · Work Queue · Transactional Outbox

---
