# Curriculum · API Design

[← System Design index](../README.md)

> 8 lessons in **API Design**. Part of the Curriculum — Build the fundamentals.

## Contents

- **API Design** (8): [Resource-oriented HTTP APIs](#resource-oriented-http-apis) · [RPC and gRPC](#rpc-and-grpc) · [GraphQL execution](#graphql-execution) · [Cursor pagination](#cursor-pagination) · [Idempotency keys](#idempotency-keys) · [Rate limiting algorithms](#rate-limiting-algorithms) · [API versioning and compatibility](#api-versioning-and-compatibility) · [Authentication and authorization boundaries](#authentication-and-authorization-boundaries)

## API Design

### Resource-oriented HTTP APIs

*Model nouns with stable identities and use HTTP's verbs, status codes and caching semantics as designed — so infrastructure and clients behave correctly without special knowledge.*

**Flow:** `Resource identity` → `HTTP method` → `Validation` → `State transition` → `Representation`

> **The 30-second version**  
> Give things stable URLs and use HTTP's methods and status codes as specified, so caches, proxies and client libraries behave correctly without knowing your domain.

**The problem**

An API designed as a set of remote procedure calls over HTTP — `POST /getUser`, `POST /updateOrderStatus`, `POST /doThing` — works, but every piece of infrastructure between client and server is now blind. Caches cannot cache, proxies cannot retry, and load balancers cannot route, because everything is an opaque POST.

Resource orientation is not aesthetic preference. HTTP's methods carry machine-readable semantics: `GET` is safe and cacheable, `PUT` and `DELETE` are idempotent, `POST` is neither. Using them correctly means browsers, CDNs, proxies, service meshes and client libraries all make correct decisions without knowing anything about your domain.

> **What the semantics actually buy**  
> A correctly-shaped `GET` can be cached by a CDN, retried automatically by any proxy, prefetched by a browser, and served stale during an outage. A correctly-shaped `PUT` can be retried after a timeout without risk. None of this requires the intermediary to understand your business — it only requires you to have used the methods as specified.

**Mental model**

Identify the nouns in your domain, give each a stable URL, and express operations as standard transformations of those resources. The URL identifies the thing; the method says what you are doing to it.

1. **Resource** — A thing with identity and a representation: an order, a user, a subscription. Collections are resources too.
2. **Identifier** — A stable URL. It should not change when the resource changes, and it should not encode implementation details like table names.
3. **Method** — `GET` read (safe, cacheable), `POST` create or non-idempotent action, `PUT` full replace (idempotent), `PATCH` partial update, `DELETE` remove (idempotent).
4. **Representation** — The document returned or sent. Content negotiation lets the same resource have several.
5. **Status code** — A machine-readable outcome class: 2xx success, 4xx the client must change something, 5xx the server failed and a retry may help.

> **Not everything is a resource, and forcing it produces worse APIs**  
> Some operations are genuinely verbs: `POST /orders/123/cancel`, `POST /transfers`, `POST /searches`. Contorting them into resource updates — `PATCH /orders/123 {status:"cancelled"}` — hides a state machine behind a field assignment, loses the ability to take parameters, and makes authorisation harder. Model the action as a resource when it has its own lifecycle, and as a sub-resource verb when it does not.

**How it works**

**Method semantics and why they matter**

```text
METHOD   SAFE  IDEMPOTENT  CACHEABLE   MEANING
GET      yes   yes         yes         read; no side effects
HEAD     yes   yes         yes         metadata only
PUT      no    yes         no          replace entirely
DELETE   no    yes         no          remove
PATCH    no    no*         no          partial update
POST     no    no          rarely      anything else

* PATCH can be made idempotent with a precondition
  (If-Match) or a merge format that is naturally idempotent

WHAT "SAFE" ENABLES
  a proxy may retry a GET freely
  a browser may prefetch it
  a CDN may cache and serve it stale

WHAT "IDEMPOTENT" ENABLES
  a client may retry after a timeout without risk
  -> this is why POST needs an idempotency key and
     PUT does not
```

1. **Use status codes as a contract, not decoration** — A 4xx tells the client its request was wrong and retrying unchanged will not help. A 5xx tells it the server failed and a retry may succeed. Returning 200 with an error body destroys this distinction and every generic retry layer along the path.
2. **Return the created resource's location** — `201 Created` with a `Location` header lets the client follow up without guessing the URL, and makes the creation self-describing.
3. **Use conditional requests for concurrency** — `ETag` plus `If-Match` gives optimistic concurrency at the protocol level, with `412 Precondition Failed` as the conflict signal — no custom version field needed.
4. **Design errors as a consistent structured type** — A stable machine-readable error shape — a type, a code, a human message, and field-level details — lets clients handle failures programmatically instead of matching on strings.
5. **Keep URLs stable and free of implementation detail** — A URL is a long-lived identifier. Encoding internal service names, table names or versions of internal structure guarantees breakage later.
6. **Do not tunnel everything through POST** — Each method you misuse removes a capability from every intermediary. The cost is invisible until you want caching or automatic retry and discover you cannot have it.

**Status codes that carry real information**

```text
2xx
  200 OK               read or update succeeded
  201 Created          + Location header
  202 Accepted         async: work queued, not done
  204 No Content       success, nothing to return

4xx  the CLIENT must change something
  400 Bad Request      malformed
  401 Unauthorized     no or invalid credentials
  403 Forbidden        authenticated but not permitted
  404 Not Found        (also: hiding existence from the
                        unauthorised - deliberate choice)
  409 Conflict         state conflict, e.g. duplicate
  412 Precondition     If-Match failed -> optimistic concurrency
  422 Unprocessable    syntactically fine, semantically invalid
  429 Too Many         + Retry-After

5xx  the SERVER failed; retry MAY help
  500 Internal         unexpected
  502/504              upstream failed or timed out
  503 Unavailable      + Retry-After, overload or maintenance

THE DIVIDING LINE THAT MATTERS MOST
  4xx: do not retry unchanged.  5xx: retry with backoff.
  Generic client libraries depend on this.
```

> **`202 Accepted` is the honest answer for async work**  
> When a request starts work that will finish later, returning `200 OK` lies about completion and returning an error lies about failure. `202 Accepted` with a `Location` pointing at a status resource tells the client exactly what happened and where to look — and it makes the asynchronous nature part of the contract rather than a surprise.

**Worked example**

Designing an orders API, contrasting RPC-over-HTTP with a resource-oriented shape.

**The same functionality, two designs**

```text
RPC-STYLE
  POST /createOrder
  POST /getOrder            {id}
  POST /updateOrderStatus   {id, status}
  POST /cancelOrder         {id}
  POST /listOrdersForUser   {userId}

  every call is POST -> nothing cacheable, nothing safely
  retryable, no conditional requests, no meaningful codes

RESOURCE-ORIENTED
  POST   /orders                 -> 201 + Location
  GET    /orders/{id}            -> 200, ETag, cacheable
  GET    /orders?customer={id}   -> 200, paginated
  PATCH  /orders/{id}            -> 200, If-Match required
  POST   /orders/{id}/cancel     -> 200 or 409
  DELETE /orders/{id}            -> 204, idempotent

WHY cancel IS A POST SUB-RESOURCE
  cancellation is a state TRANSITION with rules, not a
  field assignment. PATCH {status:"cancelled"} would:
    - hide the transition rules behind a field
    - make it impossible to pass a cancellation reason
    - make authorisation harder (who may set which field?)
  POST /orders/{id}/cancel is explicit, auditable and
  can take a body.

WHAT THE RESOURCE SHAPE GAINS
  GETs cached at the edge; conditional requests prevent
  lost updates; a proxy can retry idempotent calls;
  429 and 503 with Retry-After work without custom code
```

| Metric | Value | Note |
|---|---|---|
| RPC style | 1 method | no HTTP semantics |
| Resource style | full semantics | **caching + retry** |
| Concurrency | If-Match | no custom version field |
| Actions | sub-resource POST | explicit transitions |

> **Model state transitions explicitly, not as field assignments**  
> The most common design mistake in resource-oriented APIs is expressing every change as a `PATCH` on a status field. That collapses a state machine with rules, permissions and side effects into an assignment, and it gives the client no place to supply the context a transition needs. Transitions that have their own rules deserve their own endpoints.

**When to use it**

- **Public and partner APIs**, where clients you do not control benefit most from standard semantics.
- **Anything fronted by CDNs, proxies or a service mesh**, since those intermediaries act on HTTP semantics.
- **CRUD-shaped domains**, where resources map naturally onto entities.
- **Browser-facing APIs**, where caching, conditional requests and prefetching are free wins.
- **Long-lived contracts**, since HTTP semantics are stable in a way custom conventions are not.

**When to avoid it**

- **Do not force verbs into resources.** Some operations are actions and deserve explicit endpoints.
- **Do not return 200 with an error body**, which breaks every generic retry and monitoring layer.
- **Do not tunnel reads through POST** unless the query genuinely cannot fit in a URL — and then say so explicitly.
- **Do not expose internal structure in URLs**, which turns refactoring into a breaking change.
- **Do not use resource orientation for high-frequency internal RPC**, where gRPC's efficiency and typing are usually better.

**Advantages**

- **Intermediaries work correctly for free** — caching, retries, routing, rate limiting, all driven by standard semantics.
- **Conditional requests give optimistic concurrency** at the protocol level with no custom mechanism.
- **Status codes are a machine-readable contract**, so generic client libraries handle errors sensibly.
- **Discoverable and conventional**, so new consumers need less documentation.
- **Stable over time**, since HTTP semantics change far more slowly than any internal convention.

**Disadvantages**

- **Not every operation fits a resource shape**, and forcing it produces contorted designs.
- **Multiple round trips** for composite views, unless you add aggregate endpoints or a different protocol.
- **Over- and under-fetching** — the representation is fixed while client needs vary.
- **Verbose compared to binary protocols**, which matters at very high internal call volume.
- **Weak typing** compared with schema-first protocols, unless you add a specification and generate from it.

**Trade-offs**

**API styles**

|  | Resource-oriented HTTP | RPC (gRPC) | GraphQL |
|---|---|---|---|
| Intermediary support | Excellent | Limited | Poor (single POST endpoint) |
| Caching | Native | Manual | Manual, per query |
| Typing | Via spec (OpenAPI) | Native, generated | Native schema |
| Efficiency | Moderate | High (binary, HTTP/2) | Reduces round trips |
| Fit for composite views | Multiple calls | Multiple calls | Excellent |
| Best for | Public, browser, partner | Internal, high-volume | Diverse client needs |

> **Framing the choice**  
> “Public API goes resource-oriented over HTTP so CDNs and client libraries do the right thing without custom logic, with `ETag` and `If-Match` for concurrency. Internal service-to-service calls use gRPC for typing and efficiency — and the boundary between them is where we translate, not a place where either style leaks.”

**How it fails**

**API design failures**

| Failure | Cause | Fix |
|---|---|---|
| Clients retry and duplicate | Non-idempotent POST retried after a timeout | Idempotency keys on POST; use PUT where semantics allow |
| Nothing is cacheable | All operations tunnelled through POST | Use GET for reads with correct cache headers |
| Generic retry layers misbehave | Errors returned as 200 with a body | Use accurate status codes |
| Lost updates | Concurrent PATCH with no precondition | Require `If-Match` with an `ETag` |
| Breaking change on refactor | URLs encoding internal structure | Stable identifiers independent of implementation |
| Clients cannot distinguish error types | Unstructured error messages | Consistent machine-readable error type and code |
| Overload cascades | 429 and 503 without `Retry-After` | Include `Retry-After`; clients honour it |

**Limits**

> **Practical guidance**
>
> - **URL length** is bounded in practice (~2,000 characters is a safe limit), which is why very complex queries sometimes need a POST — stated explicitly.
> - **`ETag` plus `If-Match`** gives optimistic concurrency with `412` as the conflict signal.
> - **4xx versus 5xx** is the single most important distinction for automated clients.
> - **`Retry-After`** on 429 and 503 is what turns client retry behaviour from guesswork into cooperation.
> - **Cache headers** (`Cache-Control`, `ETag`, `Last-Modified`) are what make an edge cache useful at all.

**Alternatives**

| Style | Best for | Trade |
|---|---|---|
| Resource-oriented HTTP | Public, browser, partner APIs | Round trips; fixed representations |
| gRPC | Internal high-volume, typed | Poor intermediary and browser support |
| GraphQL | Diverse clients, composite views | Caching and rate limiting are hard |
| RPC over HTTP POST | Quick internal endpoints | Loses all HTTP semantics |
| Async messaging | Fire-and-forget work | No synchronous result |
| Server-sent events / WebSocket | Push and streaming | Different infrastructure |

**In real systems**

- **Stripe's API** is a widely copied reference for resource orientation with explicit action sub-resources, idempotency keys and structured errors.
- **GitHub's REST API** demonstrates conditional requests at scale — `ETag`-based caching is how clients stay within rate limits.
- **Google's API design guide** codifies resource orientation with standard methods plus explicitly named custom methods for genuine actions.
- **CDNs** rely entirely on correct `GET` semantics and cache headers; an API that POSTs its reads gets no edge caching at all.
- **OpenAPI specifications** restore the typing that gRPC has natively, enabling generated clients and contract testing.

**Common mistakes**

- **Everything as POST**, discarding caching, safe retry and conditional requests.
- **200 OK with an error payload**, breaking generic retry and alerting.
- **Expressing state transitions as `PATCH` on a status field**, hiding rules and losing parameters.
- **No `If-Match` on updates**, permitting lost updates.
- **URLs encoding internal implementation**, making refactoring a breaking change.
- **429 or 503 with no `Retry-After`**, leaving clients to guess and amplify.
- **Unstructured error bodies**, forcing clients to parse prose.

**The staff-level view**

HTTP semantics are a free coordination mechanism with every intermediary and every client library — and abandoning them costs capabilities that are hard to add back later.

- **Insist on accurate status codes in review.** The 4xx/5xx distinction drives retry behaviour in libraries, meshes and monitoring, and returning 200 for errors quietly breaks all of it.
- **Standardise a structured error type across the organisation**, so clients can handle failures programmatically rather than matching message strings.
- **Require idempotency keys on any POST clients may retry.** Ambiguous timeouts are routine and duplicate side effects are expensive.
- **Treat URLs as long-lived identifiers**, independent of internal structure, because they outlive the implementation they were derived from.
- **Choose per boundary, not per organisation.** Resource-oriented HTTP at the edge and gRPC internally is a coherent architecture; mandating one everywhere is not.

**Go deeper**

HTTP methods carry machine-readable semantics: `GET` is safe and cacheable, `PUT` and `DELETE` are idempotent, `POST` is neither. Using them correctly means CDNs cache, proxies retry, browsers prefetch and generic client libraries handle failures — none of which requires the intermediary to understand your domain. Tunnelling everything through POST discards all of it, and the loss is invisible until you need caching or safe retry.

Status codes are the other half of the contract, and the 4xx/5xx boundary is the critical one: 4xx means retrying unchanged will fail, 5xx means a retry may succeed. Returning 200 with an error body breaks every retry layer, mesh policy and alert that depends on it. Conditional requests with `ETag` and `If-Match` give optimistic concurrency at the protocol level, with `412` as the conflict signal and no custom version field.

The common structural mistake is expressing every change as a `PATCH` on a status field, which collapses a state machine with rules, permissions and side effects into an assignment. Transitions with their own rules deserve their own endpoints — `POST /orders/{id}/cancel` — while `PATCH` stays for genuine attribute edits. And choose per boundary: resource-oriented HTTP facing clients, gRPC internally where typing and efficiency matter more than intermediary support.

Resource orientation is usually taught as a style question and is better understood as an interoperability decision: HTTP's semantics are a contract with every intermediary and client library between you and the caller.

**What the methods buy.** `GET` being safe and cacheable means a CDN can cache it, a proxy can retry it, a browser can prefetch it, and an edge can serve it stale during an origin outage. `PUT` and `DELETE` being idempotent means a client may retry after an ambiguous timeout without risk, which is why `POST` — neither safe nor idempotent — requires an idempotency key to be retried safely. None of this requires any intermediary to know anything about the domain; it requires only that the methods were used as specified. An API that POSTs its reads has silently opted out of all of it.

**Status codes as machine-readable outcome.** The 4xx/5xx boundary is the single most consequential detail: 4xx says the client must change something and an identical retry will fail identically, while 5xx says the server failed and a retry may succeed. Service meshes, retry libraries, circuit breakers and alerting all key off this. Returning 200 with an error payload is therefore not a stylistic choice but a functional regression that disables infrastructure the organisation already paid for. Finer distinctions matter too — `409` for state conflicts, `412` for failed preconditions, `429` and `503` with `Retry-After` so clients cooperate rather than amplify.

**Concurrency at the protocol level.** `ETag` returned on reads and `If-Match` required on writes gives optimistic concurrency with `412 Precondition Failed` as the conflict signal, understood by generic clients without any bespoke version field. It is the same compare-and-set pattern that appears in databases and key-value stores, expressed in a form the transport already knows about.

**Where resource orientation breaks down.** Not every operation is a noun. Cancelling an order, transferring funds, or running a search are actions with rules, parameters and distinct authorisation. Expressing them as `PATCH` on a status field collapses a state machine into an assignment, removes the ability to pass context such as a cancellation reason, and makes field-level authorisation awkward. The right shape is an explicit sub-resource action — `POST /orders/{id}/cancel` — keeping `PATCH` for genuine attribute edits. Forcing everything into resources produces worse APIs than acknowledging the exceptions.

**Choosing per boundary.** The benefits of HTTP semantics accrue where intermediaries and heterogeneous clients exist: public APIs, browsers, partners. Between two internal services in one cluster, edge caching and browser prefetch are irrelevant while typed generated clients, binary encoding and streaming matter a great deal, which is where gRPC wins. A coherent architecture uses resource-oriented HTTP outward and gRPC inward with translation at the boundary — and mandating one style organisation-wide trades a genuine cost for a consistency nobody benefits from.

**What is worth standardising.** A structured error type used by every service, because without it each client writes bespoke string-matching and error handling never improves. Correct status codes, enforced in review, because one service returning 200 for failures corrupts fleet-wide retry behaviour. And a shared idempotency-key implementation for POSTs that clients may retry, because ambiguous timeouts are routine and duplicate side effects are the most expensive class of API defect. All three are cheap at design time and painful to retrofit once clients depend on the existing shape.

**Prove it — interview questions**

1. **[Basic] Why does it matter which HTTP method you use?**

   <details><summary>Model answer</summary>

   Because the methods carry machine-readable semantics that every intermediary acts on. `GET` is safe and cacheable, so proxies can retry it, browsers can prefetch it, and CDNs can cache it. `PUT` and `DELETE` are idempotent, so a client can retry after a timeout without risk. `POST` is neither, which is why it needs an idempotency key to be retried safely. Tunnelling everything through POST means none of that infrastructure can help you, and the loss is invisible until you want caching or automatic retry.

   </details>

2. **[Basic] What is the most important distinction in status codes?**

   <details><summary>Model answer</summary>

   4xx versus 5xx. A 4xx means the client must change something — retrying the identical request will fail identically. A 5xx means the server failed and a retry may well succeed. Generic client libraries, service meshes and monitoring all depend on that distinction, which is why returning 200 with an error body is so damaging: it makes every failure invisible to infrastructure that would otherwise handle it correctly.

   </details>

3. **[Senior] How do you handle concurrency in a REST API?**

   <details><summary>Model answer</summary>

   With conditional requests. The `GET` returns an `ETag` representing the current version; the client sends it back as `If-Match` on the update, and the server applies the change only if the resource is still at that version, returning `412 Precondition Failed` otherwise. That gives optimistic concurrency at the protocol level with no custom version field, and it is understood by generic clients. The alternative — a version field in the body — works but is bespoke, so every client must know about it.

   </details>

4. **[Senior] When is a sub-resource action better than a PATCH?**

   <details><summary>Model answer</summary>

   When the change is a state transition with rules rather than a field assignment. Cancelling an order is not the same as setting `status` to `cancelled`: there are conditions on whether it is permitted, side effects like refunds and stock release, possibly a reason to record, and different authorisation from editing an address. Expressing it as `POST /orders/{id}/cancel` makes the transition explicit, lets it take a body, gives it its own authorisation rule, and keeps the field-level `PATCH` for genuine attribute edits. Collapsing transitions into field assignments hides a state machine and is the most common structural mistake in these APIs.

   </details>

5. **[Staff] When would you choose gRPC over a resource-oriented HTTP API?**

   <details><summary>Model answer</summary>

   For internal service-to-service communication at high volume, where the benefits reverse. gRPC gives generated typed clients, binary encoding, HTTP/2 multiplexing and streaming, all of which matter when two services exchange millions of calls. The HTTP semantics that make resource orientation valuable — edge caching, browser prefetch, generic proxy retry — are largely irrelevant between two services in the same cluster. Conversely, at a public or browser-facing boundary gRPC's poor intermediary and browser support outweighs its efficiency. So I would draw the line at the trust or network boundary: resource-oriented HTTP facing outward, gRPC inward, with translation at the edge rather than either style leaking across.

   </details>

6. **[Principal] What API conventions would you standardise across an organisation, and why those?**

   <details><summary>Model answer</summary>

   Three, chosen because each one is expensive to retrofit and because inconsistency between services compounds into real client cost. First, a structured error type used by every service — a stable machine-readable code, a category, a human message and field-level details — because without it every client writes bespoke string-matching for every service and error handling never improves. Second, correct status code usage enforced in review, particularly the 4xx/5xx boundary, since meshes, retry libraries and alerting all key off it and a single service returning 200 for errors quietly corrupts fleet-wide retry behaviour. Third, idempotency keys on any POST a client may retry, with a shared implementation, because ambiguous timeouts are routine and duplicate side effects are the most expensive class of API bug. I would deliberately not standardise the protocol itself — resource-oriented HTTP outward and gRPC inward is a coherent architecture, and mandating one everywhere trades a real efficiency or usability cost for a consistency that nobody actually benefits from.

   </details>

---

### RPC and gRPC

*Call a remote procedure as if it were local — with a schema, generated clients and binary encoding — while never forgetting that the network makes it unlike a local call.*

**Flow:** `Client stub` → `Encoded request` → `RPC transport` → `Service handler` → `Typed response`

> **The 30-second version**  
> Schema-defined remote calls with generated typed clients, binary encoding, HTTP/2 multiplexing and propagated deadlines — excellent internally, awkward at the browser boundary.

**The problem**

Two internal services exchange millions of calls per day. Encoding each as JSON over HTTP/1.1 costs parsing time, verbose payloads, a connection per concurrent request, and a contract that exists only in documentation — so a field rename breaks the caller at runtime rather than at compile time.

RPC frameworks address all four: a schema defines the contract, code generation produces typed clients and servers in every language, binary encoding is compact and fast to parse, and HTTP/2 multiplexes many concurrent calls over one connection.

> **The fallacy the abstraction invites**  
> Making a remote call look like a local one hides that it can be slow, can fail partially, can time out ambiguously, and consumes a connection and a thread. The original distributed computing fallacies — the network is reliable, latency is zero, bandwidth is infinite — are exactly the assumptions a transparent RPC abstraction encourages. Every remote call needs a deadline, a retry policy and an idempotency story that a local call does not.

**Mental model**

A schema is the contract; generated stubs are the implementation of that contract on both sides. What travels between them is a compact binary encoding over a multiplexed connection.

1. **Service definition** — Methods with typed request and response messages, written in a schema language and checked into version control.
2. **Code generation** — Typed client stubs and server interfaces in every language, so the contract is enforced at compile time.
3. **Binary encoding** — Field numbers rather than names, varint integers, no whitespace — typically several times smaller and faster to parse than JSON.
4. **HTTP/2 transport** — Many concurrent calls on one connection, with per-call cancellation and streaming.
5. **Deadlines** — A per-call time budget propagated through the chain, so downstream work stops when the caller has already given up.

> **Deadline propagation is the feature that matters most**  
> A deadline set by the original caller travels with the request through every hop. Each service knows how much time remains and refuses to start work it cannot finish. That turns a chain of independent timeouts — which multiply — into a single budget, and it means a slow dependency wastes no capacity on requests whose callers have already timed out.

**How it works**

**Schema, streaming modes, and what each is for**

```text
SERVICE DEFINITION
  service Orders {
    rpc GetOrder    (GetOrderRequest) returns (Order);
    rpc ListOrders  (ListRequest)     returns (stream Order);
    rpc UploadItems (stream Item)     returns (UploadResult);
    rpc Sync        (stream Event)    returns (stream Event);
  }

FOUR CALL SHAPES
  unary            request -> response      (most calls)
  server streaming request -> stream        (large result sets,
                                             live updates)
  client streaming stream  -> response      (bulk upload)
  bidirectional    stream  <-> stream       (chat, sync,
                                             long-lived sessions)

WHY STREAMING MATTERS
  a large list as unary must be fully materialised in
  memory on both sides; as a stream it flows incrementally
  with flow control, bounded memory, and the client can
  stop early.
```

1. **Set a deadline on every call and propagate it** — A call without a deadline can block a thread indefinitely. Propagating the remaining budget downstream means a chain has one total bound rather than a product of timeouts.
2. **Evolve schemas additively using field numbers** — Protocol Buffers identify fields by number, so renaming is harmless and removal reserves the number. Adding optional fields is free; changing a field's type or meaning is not.
3. **Use per-request load balancing, not per-connection** — gRPC multiplexes over a long-lived HTTP/2 connection, so an L4 balancer pins all of a client's calls to one backend. This is a correctness issue for load distribution, not a tuning detail.
4. **Add a maximum connection age** — Long-lived connections never discover new backends. Periodic `GOAWAY` with jitter forces reconnection and rebalancing, without which autoscaling adds capacity that receives no traffic.
5. **Map errors to status codes deliberately** — gRPC's status codes carry retryability semantics much as HTTP's do. `UNAVAILABLE` is retryable; `INVALID_ARGUMENT` is not. Getting this wrong breaks generic retry middleware.
6. **Do not expose it directly to browsers** — Browsers cannot speak gRPC's framing natively. gRPC-Web plus a proxy works, but a resource-oriented HTTP boundary is usually the better choice for public and browser traffic.

**Deadlines versus independent timeouts**

```text
INDEPENDENT TIMEOUTS (the common mistake)
  gateway timeout 5s
    -> service A timeout 5s
      -> service B timeout 5s
        -> database timeout 5s
  worst case: 5s at each hop, and the gateway gave up at 5s
  while B and the database keep working for the full 5s
  -> capacity spent on results nobody will receive

PROPAGATED DEADLINE
  gateway sets deadline = now + 2s
    A receives: 1.9s remaining -> sets its own calls within it
      B receives: 1.4s remaining
        database query bounded by 1.3s
  any hop with insufficient time remaining fails FAST
  rather than starting work it cannot finish

EFFECT UNDER LOAD
  during a slowdown, work is abandoned early at every level
  instead of accumulating -> the system sheds naturally
```

> **gRPC's efficiency is real but rarely the binding constraint**  
> Binary encoding and multiplexing are genuine improvements, but most services are not bottlenecked on serialisation. The durable benefits are the schema as an enforced contract, generated clients that fail at compile time, deadline propagation, and streaming. Choosing gRPC purely for speed usually under-values what actually pays off.

**Worked example**

Migrating an internal service boundary from JSON over HTTP to gRPC, and what it actually changes.

**What improves, and what needs attention**

```text
BEFORE  JSON over HTTP/1.1, contract in a wiki page
  field renamed in the producer -> consumers break at runtime,
    discovered in production
  one connection per concurrent request -> connection churn
  no deadlines -> a slow dependency exhausts caller threads
  payload ~4 KB per call

AFTER  gRPC
  schema in version control, generated clients
    -> a breaking change fails the CONSUMER's build
  HTTP/2 multiplexing -> one connection, many concurrent calls
  deadline propagated from the gateway
  payload ~800 B

WHAT NEEDED ATTENTION DURING MIGRATION
  1  load balancing
       an L4 balancer pinned each client's long-lived
       connection to one pod -> severe skew
       fix: per-request L7 balancing or client-side
            subchannel balancing
  2  connection age
       existing connections never discovered new pods
       fix: server-side max connection age with jitter
  3  client-side queueing
       MAX_CONCURRENT_STREAMS exceeded -> calls queued
       invisibly in the client, looking like server latency
       fix: raise the limit and export outstanding-call count
  4  error mapping
       everything returned UNKNOWN -> retry middleware
       could not distinguish retryable from permanent
       fix: deliberate status code mapping
```

| Metric | Value | Note |
|---|---|---|
| Payload | 4 KB → 800 B | 5× smaller |
| Contract | compile-time | **was runtime** |
| Deadlines | propagated | chain-wide budget |
| Balancing | must be L7 | or severe skew |

> **The operational surprises come from HTTP/2, not from gRPC**  
> Load skew, stale connections and invisible client-side queueing are all consequences of long-lived multiplexed connections, not of the RPC framework. Teams migrating to gRPC meet them together and attribute them to gRPC — but the same issues appear with any HTTP/2 client, and the fixes are per-request balancing, bounded connection age, and exporting client-side concurrency.

**When to use it**

- **Internal service-to-service communication**, especially at high call volume.
- **Polyglot environments**, where generated clients remove hand-written integration code in every language.
- **Streaming workloads** — large result sets, bulk upload, bidirectional sync — where unary calls would be awkward.
- **Where a strong contract matters**, since schema plus code generation catches breakage at build time.
- **Latency-sensitive internal paths**, where deadline propagation and multiplexing genuinely help.

**When to avoid it**

- **Do not expose it directly to browsers**; use a resource-oriented HTTP boundary or gRPC-Web with a proxy.
- **Do not use it for public APIs** without strong justification — tooling, debuggability and client familiarity all favour HTTP there.
- **Do not use L4 load balancing in front of it**, which pins connections and skews load badly.
- **Do not omit deadlines**, which lets one slow dependency exhaust caller resources.
- **Do not treat remote calls as local ones** because the syntax looks the same — partial failure and latency are still real.

**Advantages**

- **Schema-enforced contracts** with compile-time breakage instead of runtime surprises.
- **Generated clients in every language**, removing hand-written serialisation and HTTP plumbing.
- **Compact binary encoding** and fast parsing, typically several times smaller than JSON.
- **HTTP/2 multiplexing** with per-call cancellation, so one connection serves many concurrent calls.
- **First-class deadlines and streaming**, which are awkward to retrofit onto plain HTTP.

**Disadvantages**

- **Poor browser support** without a proxy layer.
- **Harder to debug** than text protocols — you cannot read a capture or issue a call with `curl` without tooling.
- **Requires L7 or client-side load balancing**, since connection-level balancing skews badly.
- **Intermediaries understand less**, so caching and generic HTTP tooling do not apply.
- **Schema and codegen add build-time machinery** that a simple JSON endpoint does not need.
- **The local-call illusion** encourages designs that ignore latency and partial failure.

**Trade-offs**

**gRPC versus JSON over HTTP**

|  | gRPC | JSON/HTTP |
|---|---|---|
| Contract | Schema, compile-time | Documentation or OpenAPI |
| Payload size | Compact binary | Verbose text |
| Streaming | Native, four modes | Awkward (SSE, chunked) |
| Deadlines | Built in and propagated | Manual |
| Browser | Needs a proxy | Native |
| Debuggability | Needs tooling | `curl` and a browser |
| Intermediaries | Limited understanding | Full HTTP semantics |

> **Framing the boundary**  
> “gRPC inside the cluster for the contract enforcement, deadline propagation and streaming — with L7 per-request balancing and a bounded connection age, because HTTP/2's long-lived connections otherwise skew load and hide new pods. Resource-oriented HTTP at the public edge, where browser support and intermediary semantics matter more than payload size.”

**How it fails**

**gRPC operational failures**

| Symptom | Cause | Fix |
|---|---|---|
| Severe backend load skew | L4 balancing pinning long-lived connections | L7 per-request or client-side subchannel balancing |
| New pods receive no traffic | Connections never re-established | Server-side max connection age with jitter |
| High client latency, healthy server metrics | Calls queued behind `MAX_CONCURRENT_STREAMS` | Raise the limit; export outstanding-call count |
| Retry middleware misbehaves | All errors mapped to `UNKNOWN` | Deliberate status mapping with retryability in mind |
| Threads exhausted by one slow dependency | No deadlines | Set and propagate deadlines on every call |
| Breaking change ships silently | Schema changed incompatibly | Field numbers; additive-only evolution; compatibility checks in CI |
| Cannot debug a production issue | Binary protocol, no tooling | Reflection enabled; `grpcurl`; structured logging of requests |

**Limits**

> **Operating parameters**
>
> - **Payload size**: typically 3–10× smaller than equivalent JSON, with proportionally faster parsing.
> - **`MAX_CONCURRENT_STREAMS`**: commonly 100–250; excess calls queue in the client where server metrics cannot see them.
> - **Max connection age**: 30–60 minutes with jitter, so connections rebalance onto new backends.
> - **Deadlines** should be set on every call and propagated; a chain without them has an unbounded worst case.
> - **Field numbers are permanent**: reserve removed numbers so they cannot be reused with a different meaning.

**Alternatives**

| Option | Best for | Trade |
|---|---|---|
| gRPC | Internal, typed, high-volume, streaming | Browser support; debuggability |
| JSON over HTTP | Public, browser, partner | Verbose; weak contract |
| gRPC-Web | Browser clients wanting gRPC contracts | Proxy required; limited streaming |
| GraphQL | Diverse client data needs | Caching and rate limiting are hard |
| Message queue | Asynchronous work | No synchronous result |
| Thrift / Avro RPC | Similar goals, different ecosystems | Smaller communities |

**In real systems**

- **Kubernetes and its ecosystem** use gRPC extensively for control-plane communication, relying on streaming for watch semantics.
- **Envoy's xDS protocol** is a bidirectional gRPC stream, which is how configuration is pushed to proxies in sub-second time.
- **Google's internal RPC infrastructure** is where gRPC's design originated, including deadline propagation as a first-class concept.
- **Service meshes** provide the per-request load balancing that gRPC requires, which is one reason meshes and gRPC are so often adopted together.
- **gRPC-Web with an Envoy proxy** is the standard pattern where browser clients need to reach gRPC services directly.

**Common mistakes**

- **No deadlines**, letting a slow dependency exhaust caller resources.
- **L4 load balancing** in front of gRPC, pinning connections and skewing load.
- **No maximum connection age**, so new backends never receive traffic.
- **Mapping every error to `UNKNOWN`**, defeating retry middleware.
- **Treating remote calls as local** because the syntax is identical.
- **Changing field types or meanings** rather than adding new fields.
- **Exposing gRPC directly to browsers** without a proxy.

**The staff-level view**

The valuable parts of gRPC are the contract and the deadlines; the operational surprises are HTTP/2's, and both should be addressed at the platform level.

- **Make deadline propagation a platform default.** A call chain without a shared budget has an unbounded worst case, and individual timeouts multiply rather than compose.
- **Mandate L7 or client-side balancing for any gRPC path.** Connection-level balancing produces severe skew that no health check corrects.
- **Set a jittered maximum connection age fleet-wide**, or autoscaling silently adds capacity that receives no traffic.
- **Enforce schema compatibility in CI**, since the value of a generated contract disappears if incompatible changes can ship.
- **Keep the boundary deliberate.** gRPC internally and HTTP at the edge is coherent; pushing gRPC to browsers or public partners usually costs more than it saves.

**Go deeper**

gRPC defines services in a schema, generates typed clients and servers in every language, encodes messages compactly in binary, and runs over HTTP/2 so many concurrent calls share one connection with per-call cancellation. It also offers four streaming shapes and first-class deadlines. The contract enforcement and the deadlines usually matter more than the efficiency, even though efficiency is what gets advertised.

Deadline propagation is the standout feature: a budget set by the original caller travels through every hop, so each service knows how much time remains and refuses work it cannot finish. That replaces a chain of independent timeouts — whose worst case multiplies and which leave downstream services working on abandoned requests — with a single bound, and it makes the system shed doomed work naturally under load.

The operational surprises come from HTTP/2 rather than gRPC. Long-lived multiplexed connections mean L4 load balancing pins a client's entire workload to one backend, so per-request L7 or client-side balancing is mandatory; connections never discover new pods without a jittered maximum connection age; and calls beyond the negotiated stream limit queue invisibly in the client. Use it internally, translate to resource-oriented HTTP at the browser and public boundary.

gRPC is best understood as three separable things bundled together: a schema-driven contract with code generation, an efficient binary encoding, and an HTTP/2-based transport with deadlines and streaming. They are valuable in roughly that order, though efficiency is what gets quoted.

**The contract is the durable benefit.** A service definition in version control, compiled into typed clients and server interfaces in every language, means an incompatible change fails the consumer's build rather than surfacing as a runtime error in production. Field numbers rather than names as identity make additive evolution genuinely free — renaming is harmless, adding optional fields costs nothing, and removed numbers can be reserved so they are never recycled with different meaning. What remains unsafe is changing a field's type, or changing its semantics while keeping the type, which passes every mechanical check and corrupts data silently.

**Deadlines are the operational benefit.** A budget set by the originating caller propagates through every hop, so each service knows the remaining time and declines to start work it cannot complete. This is qualitatively different from per-hop timeouts, whose worst case multiplies down the chain and which leave downstream services burning capacity on requests their callers abandoned long ago. Under load, propagated deadlines cause work to be shed early at every level rather than accumulating — a form of automatic backpressure that is awkward to retrofit onto plain HTTP.

**Streaming covers shapes unary calls handle badly.** Server streaming avoids materialising a large result set in memory on both sides and lets the client stop early. Client streaming handles bulk upload with flow control. Bidirectional streaming supports long-lived sessions such as configuration push or synchronisation — which is exactly how service mesh control planes deliver updates in sub-second time.

**The operational costs belong to HTTP/2, not to RPC.** Because a long-lived connection carries all of a client's calls, an L4 balancer makes a single routing decision and pins that workload to one backend, producing skew that no health signal corrects — so per-request L7 or client-side subchannel balancing is a correctness requirement rather than an optimisation. Those same connections never discover backends added later, so a jittered maximum connection age is needed or autoscaling adds capacity that receives nothing. And calls exceeding the negotiated concurrent-stream limit queue inside the client, invisible to server metrics, producing the confusing pattern of high client latency alongside healthy server dashboards.

**The abstraction's danger.** Making a remote call look like a local one invites designs that assume the network is reliable, latency is negligible and failures are total rather than partial. A local call cannot time out ambiguously, cannot succeed while its response is lost, and does not consume a connection. Every remote call needs a deadline, a retry policy keyed to error retryability, and an idempotency story — none of which the syntax reminds you about, which is why status code mapping and deadline defaults belong in shared middleware rather than in each service.

**Draw the boundary deliberately.** Inside a cluster, contract enforcement, deadlines and streaming justify the operational work, and the HTTP semantics given up — edge caching, browser prefetch, generic proxy behaviour — are irrelevant between services you control. At a public or browser boundary that reverses entirely: unfamiliar tooling, proxy requirements and reduced intermediary support outweigh payload size. Translating at the edge is a coherent architecture; mandating one protocol everywhere trades a real cost for a uniformity nobody benefits from.

**Prove it — interview questions**

1. **[Basic] What does gRPC give you over JSON on HTTP?**

   <details><summary>Model answer</summary>

   A schema that generates typed clients and servers, so contract breakage is caught at compile time rather than in production; compact binary encoding that is several times smaller and faster to parse; HTTP/2 multiplexing so many concurrent calls share one connection with per-call cancellation; native streaming in four shapes; and first-class deadlines that propagate through a call chain. The contract and the deadlines usually matter more than the efficiency, though efficiency is what gets quoted.

   </details>

2. **[Basic] Why does a deadline matter more than a timeout?**

   <details><summary>Model answer</summary>

   Because a deadline is a shared budget rather than a per-hop limit. With independent timeouts, each service in a chain waits its own full duration, so the worst case multiplies and downstream services keep working on requests whose callers gave up long ago. A propagated deadline means every hop knows how much time remains and refuses to start work it cannot finish — so under load the system abandons doomed work early at every level rather than accumulating it.

   </details>

3. **[Senior] Why does gRPC break L4 load balancing?**

   <details><summary>Model answer</summary>

   Because it multiplexes many calls over a single long-lived HTTP/2 connection, and an L4 balancer makes one routing decision per connection. Every call from that client therefore lands on the same backend for the connection's lifetime, so a heavy client saturates one pod while others idle, and the balancer cannot move traffic even if that pod degrades. The fix is per-request balancing — an L7 proxy, a sidecar, or client-side subchannel balancing — plus a bounded connection age so connections are periodically re-established and redistributed.

   </details>

4. **[Senior] How do you evolve a gRPC schema safely?**

   <details><summary>Model answer</summary>

   Additively, using field numbers rather than names as the identity. Adding a new optional field is free, since old clients ignore unknown numbers and new clients see a default for old messages. Renaming a field is harmless because the number is what matters. What is unsafe is changing a field's type, reusing a retired number for a different meaning, or changing semantics while keeping the type — the last of which passes every compatibility check and corrupts data silently. Removed numbers should be explicitly reserved so they cannot be recycled, and compatibility should be enforced in CI so an incompatible change fails the build.

   </details>

5. **[Staff] You are migrating internal services to gRPC. What operational work does it require?**

   <details><summary>Model answer</summary>

   Mostly work created by HTTP/2 rather than by gRPC itself. Load balancing must become per-request — L7 or client-side — because connection-level balancing pins a client's entire workload to one backend and skews severely. A jittered maximum connection age is needed, or long-lived connections never discover pods added by autoscaling, so new capacity sits idle while the scaling system appears broken. Client-side outstanding-call count must be exported, because calls beyond the negotiated concurrent-stream limit queue inside the client where no server metric sees them, producing high client latency with healthy server dashboards. And error codes need deliberate mapping, since defaulting everything to `UNKNOWN` prevents retry middleware from distinguishing retryable from permanent failures. None of these are exotic, but teams meet them all at once during migration and attribute them to gRPC.

   </details>

6. **[Principal] Where would you draw the boundary between gRPC and HTTP in an architecture?**

   <details><summary>Model answer</summary>

   At the point where I stop controlling both ends and where intermediaries start mattering. Inside the cluster, gRPC's advantages — enforced contracts with compile-time breakage, generated clients across languages, deadline propagation, streaming — are worth real operational work, and the HTTP semantics it gives up, like edge caching and browser prefetch, are irrelevant between two of my own services. At the public or browser boundary that reverses: clients I do not control benefit enormously from standard HTTP semantics, caching, and the ability to debug with ordinary tools, while gRPC needs a proxy and unfamiliar tooling. So I would translate at the edge rather than let either style leak across, and I would resist the pull toward uniformity in both directions — mandating gRPC everywhere pushes a poor fit onto public clients, and mandating HTTP everywhere gives up contract enforcement and deadline propagation where they matter most.

   </details>

---

### GraphQL execution

*Clients declare exactly the data they need and the server resolves a tree of fields — solving over-fetching while making caching, cost control and authorisation much harder.*

**Flow:** `Query document` → `Schema validation` → `Resolver plan` → `Batched sources` → `Result graph`

> **The 30-second version**  
> Clients request an exact data shape and the server resolves a field tree — removing over-fetching, while relocating N+1 to the server and removing HTTP caching, cost predictability and endpoint-level authorisation.

**The problem**

A mobile screen needs a user, their last five orders, each order's items, and each item's product name and image. Over a resource-oriented API that is one request for the user, one for the orders, and then a request per order for items — the classic N+1 round-trip problem — or a bespoke aggregate endpoint that exists only for this screen and must change whenever the screen does.

GraphQL lets the client describe the shape it wants in one query. The server validates it against a schema and resolves each field, so a single request returns exactly that shape. Over-fetching, under-fetching and screen-specific endpoints all disappear — and are replaced by a different set of problems around caching, cost and authorisation.

> **The trade is specificity for control**  
> A REST endpoint is a fixed contract the server controls: it knows the cost, the cache key and the authorisation surface in advance. A GraphQL query is composed by the client at runtime, so the server knows none of those until it arrives. Every difficulty in operating GraphQL follows from that inversion.

**Mental model**

The schema is a graph of types and fields. A query is a traversal request over that graph. The server walks the requested tree, calling a resolver function for each field, and assembles the result in the same shape as the query.

1. **Schema** — Types, fields and their relationships, plus the root query, mutation and subscription entry points. A strongly typed contract.
2. **Query document** — The client's requested shape. Validated against the schema before any execution begins.
3. **Resolver** — A function producing the value for one field, given its parent. Resolvers compose into a tree matching the query.
4. **Execution** — Breadth-first per level, so all fields at one depth resolve before the next — which is what makes batching possible.
5. **DataLoader / batching** — A per-request layer that collects the keys requested at one level and fetches them in a single call, eliminating N+1.

> **The N+1 problem moves rather than disappears**  
> A query for fifty orders, each with its customer, calls the customer resolver fifty times. Without batching, that is fifty database queries where a REST endpoint would have written one join. GraphQL does not remove N+1; it relocates it from the client's round trips to the server's data access, where it is invisible unless you look. Batching per request is not optional infrastructure — it is a prerequisite.

**How it works**

**Execution order and why batching works**

```text
QUERY
  { orders(last: 50) { id customer { name } } }

WITHOUT BATCHING
  orders resolver      -> 1 query returning 50 orders
  customer resolver    -> called 50 times, once per order
                          -> 50 separate queries
  total: 51 queries    <- the N+1 problem, server-side

WITH DATALOADER
  execution is breadth-first: all 50 customer resolvers
  are invoked before the next level begins
  DataLoader collects the 50 customer ids in a request-scoped
  buffer, then issues ONE query:
    SELECT * FROM customers WHERE id IN (...)
  total: 2 queries

WHY IT REQUIRES THE EXECUTION MODEL
  batching only works because all fields at a level resolve
  together. A depth-first executor could not collect the keys.
  DataLoader must also be REQUEST-SCOPED, or its cache leaks
  data between users.
```

1. **Make batching a framework default, not a per-resolver choice** — Every field that loads by id needs it, and a single missed resolver reintroduces N+1 invisibly.
2. **Bound query cost before executing** — Compute a static cost from depth, breadth and field weights, and reject queries above a budget. Without this, a single client can request a query that traverses the graph exponentially.
3. **Authorise per field, not per endpoint** — The query shape is arbitrary, so authorisation cannot live at the entry point. Each resolver must enforce access to the data it returns, which is more work but also more precise.
4. **Use persisted queries in production** — Clients register queries ahead of time and send an identifier. That restores a known, finite set of shapes — so cost, caching and authorisation become analysable again.
5. **Do not expect HTTP caching to work** — Everything is a POST to one endpoint, so CDNs and proxies cannot help. Caching must happen at the resolver or data-source layer, or via persisted queries served as GETs.
6. **Return partial results with errors** — GraphQL can return data for the fields that resolved and errors for those that did not, which is a better fit for composite views than failing the whole request.

**Query cost: why depth limits alone are insufficient**

```text
MALICIOUS OR ACCIDENTAL
  { user { friends { friends { friends { name } } } } }
  depth 4, but if each user has 200 friends:
    200 x 200 x 200 = 8,000,000 nodes
  a depth limit of 5 permits this.

COST ANALYSIS
  assign each field a cost, multiply by list sizes:
    friends(first: N) costs N x cost(selection set)
  reject if total exceeds a budget
  requires clients to paginate: friends(first: 20)

COMPLEMENTARY CONTROLS
  depth limit          - blunt, cheap, catches recursion
  breadth/complexity   - the real control
  per-client budget    - amortised cost over time
  timeouts             - the backstop
  persisted queries    - the strongest: only known shapes run

WITHOUT COST CONTROL, ONE QUERY CAN BE A DENIAL OF SERVICE.
```

> **Schema-wide exposure is an authorisation surface**  
> A single schema reachable by any client means every field is potentially requestable by any caller in any combination. Authorisation that assumed an endpoint boundary — “only the admin endpoint returns emails” — no longer holds, because a client can reach that field through any path in the graph. Field-level authorisation is not a refinement; it is the only correct model.

**Worked example**

A mobile app screen: what GraphQL improves and what it costs to operate.

**One query, and the infrastructure it needs**

```text
QUERY (one round trip)
  {
    me {
      name
      orders(last: 5) {
        id placedAt total
        items { quantity product { name imageUrl } }
      }
    }
  }

WHAT IT REPLACED
  REST: 1 + 1 + 5 + (5 x items) requests, or a bespoke
  /mobile/home endpoint that changes with every design tweak

WHAT THE SERVER MUST NOW DO
  batching:      product lookups across all items -> 1 query
                 (without it: up to 25 queries)
  cost control:  orders(last: 5) bounded; without a limit
                 a client could request last: 10000
  authorisation: "me" is scoped to the caller, but the
                 product and order resolvers must each
                 verify access independently
  caching:       no HTTP caching (single POST endpoint)
                 -> cache at the data-source layer, keyed
                    by entity id, not by query

PRODUCTION HARDENING
  persisted queries: the app registers this query at build
  time and sends a hash. The server then:
    - knows every possible query shape in advance
    - can pre-compute cost and cache policy
    - can serve it as a GET, restoring HTTP caching
    - rejects any unregistered query
```

| Metric | Value | Note |
|---|---|---|
| Round trips | 1 | was 1 + 1 + 5 + N |
| Batching | required | **or N+1 server-side** |
| Cost control | required | or DoS by query |
| Persisted queries | restores caching | and bounds shapes |

> **Persisted queries reverse most of the operational cost**  
> Registering queries ahead of time gives back what dynamic queries take away: a finite known set of shapes, so cost is pre-computable, caching becomes possible as GETs with stable URLs, and unregistered queries can be rejected outright. The flexibility was valuable during development; in production it is mostly a liability, and persisted queries let you keep the developer experience while removing the runtime exposure.

**When to use it**

- **Diverse clients with different data needs** — mobile, web and partner apps served by one schema.
- **Composite views** requiring data from several sources in one round trip.
- **Rapidly changing front-ends**, where new screens should not require new backend endpoints.
- **Aggregating multiple backends** behind one graph, where the alternative is a bespoke backend-for-frontend per client.
- **With persisted queries**, which make it operable in production without giving up the development benefits.

**When to avoid it**

- **Do not use it for simple CRUD with one client**, where REST is simpler and caching works for free.
- **Do not expose a dynamic-query endpoint publicly** without cost analysis — one query can be a denial of service.
- **Do not rely on HTTP caching**; it does not apply to a single POST endpoint.
- **Do not authorise at the endpoint**, since any field is reachable through any path.
- **Do not skip batching**, which reintroduces N+1 on the server where it is harder to see.

**Advantages**

- **Clients fetch exactly what they need**, eliminating over- and under-fetching.
- **One round trip for composite views**, which matters most on high-latency mobile networks.
- **Strongly typed schema** with introspection, enabling excellent tooling and generated clients.
- **Front-end changes need no backend change**, decoupling release cycles.
- **Partial results with field-level errors**, which suits composite views better than all-or-nothing failure.

**Disadvantages**

- **HTTP caching does not apply**, so caching must be rebuilt at the resolver or data layer.
- **Query cost is unbounded by default**, making denial of service trivially easy.
- **N+1 is relocated to the server** and requires batching infrastructure everywhere.
- **Authorisation must be per field**, which is more work and easy to get wrong.
- **Observability is harder** — every request is a POST to one endpoint, so per-operation metrics need explicit instrumentation.
- **Rate limiting by request count is meaningless**, since queries vary enormously in cost.

**Trade-offs**

**GraphQL versus REST**

|  | GraphQL | REST |
|---|---|---|
| Over/under-fetching | Solved | Endemic, or bespoke endpoints |
| Round trips for composite views | One | Several |
| HTTP caching | Not applicable | Native |
| Cost predictability | Unbounded without analysis | Known per endpoint |
| Authorisation | Per field | Per endpoint |
| Rate limiting | Must be cost-based | Request count works |
| Observability | Needs operation-level instrumentation | Per-endpoint by default |

> **A balanced framing**  
> “GraphQL for the mobile and web clients, because one query per screen matters on high-latency networks and the front-end teams stop waiting on backend endpoints. In production we run persisted queries only — which bounds the shapes, makes cost pre-computable, and lets us serve them as cacheable GETs — so we keep the development benefit without the runtime exposure.”

**How it fails**

**GraphQL failures**

| Failure | Cause | Fix |
|---|---|---|
| Database overwhelmed by one query | N+1 in resolvers without batching | Request-scoped DataLoader on every id-based field |
| A single query takes the service down | No cost analysis; exponential traversal | Complexity scoring; mandatory pagination; persisted queries |
| Data leaked between users | DataLoader cache not request-scoped | Instantiate loaders per request, never globally |
| Unauthorised field returned | Authorisation at the endpoint rather than per field | Field-level checks in resolvers |
| No useful latency metrics | All traffic is POST to one path | Instrument per operation name; require named operations |
| Rate limiting ineffective | Counting requests, not cost | Budget by computed complexity per client |
| Cannot cache anything | Single POST endpoint | Persisted queries as GETs; cache at the data layer by entity id |

**Limits**

> **Operating guidance**
>
> - **Complexity budget** per query, computed from field weights and list sizes, with mandatory pagination arguments.
> - **Depth limit** as a cheap backstop against recursion, but breadth is where the real cost is.
> - **DataLoader must be request-scoped**; a global instance is a cross-tenant data leak.
> - **Persisted queries** turn an unbounded shape space into a finite, analysable one.
> - **Instrument per operation name**, since a single endpoint makes default HTTP metrics useless.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| GraphQL | Diverse clients, composite views | Caching, cost, authorisation complexity |
| REST with aggregate endpoints | Known screens, stable clients | Endpoint proliferation |
| Backend-for-frontend | A few well-known clients | One BFF per client to maintain |
| gRPC | Internal typed calls | No client-shaped queries |
| REST + sparse fieldsets | Mild over-fetching problems | Partial solution; non-standard |
| Persisted GraphQL only | Production GraphQL | Loses ad-hoc query flexibility |

A backend-for-frontend is the honest alternative when there are only two or three clients: a thin aggregation service per client gives one round trip and full server-side control of cost, caching and authorisation, without the schema-wide exposure. GraphQL wins as the number and diversity of clients grows.

**In real systems**

- **Facebook built GraphQL** for mobile clients on slow networks, where round trips dominated perceived latency — which remains its strongest use case.
- **GitHub's GraphQL API** applies an explicit point-based rate limit computed from query complexity, demonstrating that request counting cannot work.
- **Apollo's persisted queries** are the standard production hardening, restoring cacheability and bounding the query space.
- **DataLoader** was created at Facebook precisely because per-field resolution otherwise produces N+1 against every data source.
- **Federation approaches (Apollo Federation)** compose a single graph from multiple services, which is how large organisations avoid one monolithic schema owner.

**Common mistakes**

- **No batching**, producing server-side N+1 that looks like mysterious database load.
- **Global DataLoader instances**, leaking data between requests and tenants.
- **No query cost analysis**, allowing one query to exhaust the service.
- **Endpoint-level authorisation**, leaving fields reachable through unexpected paths.
- **Rate limiting by request count**, which is meaningless when costs vary by orders of magnitude.
- **Expecting CDN caching to work** on a single POST endpoint.
- **No named operations**, making per-operation observability impossible.

**The staff-level view**

GraphQL's costs are all operational and all deferred: they appear after adoption, in caching, cost control and authorisation, long after the developer-experience benefits were demonstrated.

- **Require persisted queries in production from the start.** They restore caching, bound the shape space, make cost pre-computable, and are far easier to adopt early than to retrofit.
- **Treat query cost analysis as a launch requirement**, not a hardening task. Without it, a single client can trivially take the service down, accidentally or otherwise.
- **Make batching a framework default.** One resolver without it reintroduces N+1 invisibly, and the symptom appears as unexplained database load.
- **Insist on field-level authorisation.** The schema is one reachable surface, so endpoint-level assumptions silently stop holding.
- **Ask honestly whether a backend-for-frontend would do.** With two or three clients it usually would, with far less operational surface.

**Go deeper**

A GraphQL schema is a typed graph; a query is a requested traversal of it. The server validates the query, then resolves each field, assembling a result matching the requested shape. That eliminates over- and under-fetching and collapses a composite view into one round trip — which matters most on high-latency mobile networks and lets front-end teams add screens without waiting on new backend endpoints.

The costs are structural. N+1 moves from the client's round trips to the server's data access: fifty orders each resolving a customer means fifty queries unless a request-scoped batching layer collects the keys at each level and issues one. Query cost is unbounded by default, and depth limits are insufficient because breadth dominates — so complexity scoring with mandatory pagination is a launch requirement, not hardening. HTTP caching does not apply at all, since everything is a POST to one endpoint. And authorisation must be per field, because any field is reachable through any path in the graph.

Persisted queries reverse most of this: clients register queries at build time and send a hash, so the shape space becomes finite and known, cost is pre-computable, queries can be served as cacheable GETs, and anything unregistered is rejected. Where there are only two or three clients, a backend-for-frontend achieves the same composite views with far less operational surface.

GraphQL inverts who decides the response shape, and every one of its benefits and costs follows from that inversion.

**What it solves.** A REST endpoint returns a representation the server chose, so clients either receive unneeded fields or make several requests to assemble a view — and the usual remedy, screen-specific aggregate endpoints, couples backend work to front-end design changes. GraphQL lets the client express the exact tree it needs, resolved in one round trip. On mobile networks where round trips dominate perceived latency, and in organisations where many independent client teams would otherwise queue for backend endpoints, this is a substantial win.

**Where N+1 goes.** Execution is breadth-first: all fields at a level resolve before the next begins. That is what makes batching possible, and also what makes it necessary — fifty orders each resolving a customer invoke the customer resolver fifty times. A request-scoped loader collects the keys at each level and issues a single query, turning fifty-one queries into two. Two details matter operationally: batching must be a framework default, because one unbatched resolver reintroduces the problem as unexplained database load, and loaders must be request-scoped, because a global instance caches across users and becomes a cross-tenant data leak.

**Cost is unbounded by construction.** The server cannot know what a query will cost until it arrives. Depth limits are a cheap backstop against recursion but miss the real hazard, which is breadth: a depth-four traversal over a graph where each node has two hundred neighbours requests millions of nodes. The working control is complexity scoring — per-field weights multiplied by requested list sizes, with pagination arguments mandatory — plus per-client budgets amortised over time and timeouts as the final backstop. It also means rate limiting by request count is meaningless, since query costs differ by orders of magnitude.

**Caching and authorisation both lose their usual home.** Everything being a POST to a single endpoint means CDNs, proxies and browser caches cannot participate, and there is no stable cache key because the shape is client-determined; caching must be rebuilt inside the server keyed by entity identity. Similarly, a single reachable schema means any field can be requested through any path, so authorisation cannot live at an entry point — assumptions like “only the admin endpoint exposes emails” silently stop holding. Field-level enforcement in resolvers is the only correct model, and it is more work than an endpoint check.

**Persisted queries restore most of what was lost.** Registering queries at build time and sending a hash makes the shape space finite and known in advance: cost can be pre-computed, cache policy assigned, the request served as a GET with a stable URL so HTTP caching works again, and anything unregistered rejected outright — which removes the arbitrary-query attack surface entirely. The dynamic flexibility that made development pleasant is mostly a production liability, and persisted queries keep the former while eliminating the latter. Adopting them from the start is far easier than retrofitting.

**Know when not to.** With one or two well-known clients, a backend-for-frontend delivers the same single-round-trip composite views with complete server-side control of cost, caching and authorisation, and no schema-wide surface — a materially smaller commitment for the same user-facing result. GraphQL earns its operational cost as client diversity and team independence grow, and in that setting a federated graph distributes schema ownership so no single team becomes the bottleneck for every field.

**Prove it — interview questions**

1. **[Basic] What problem does GraphQL solve?**

   <details><summary>Model answer</summary>

   Over-fetching and under-fetching. A REST endpoint returns a fixed representation, so a client either receives fields it does not need or must make several requests to assemble a view. GraphQL lets the client specify the exact shape it wants in one query, which removes both problems and eliminates the need for screen-specific aggregate endpoints that change whenever the front-end does. That matters most on high-latency mobile networks where round trips dominate perceived performance.

   </details>

2. **[Basic] What is the N+1 problem in GraphQL?**

   <details><summary>Model answer</summary>

   A query returning fifty orders, each with a customer, invokes the customer resolver fifty times — producing fifty separate data-source queries where a REST endpoint would have used one join. GraphQL does not remove N+1; it moves it from the client's round trips to the server's data access, where it is far less visible. Batching with a request-scoped loader collects all the keys requested at one level and issues a single query, which is why it is a prerequisite rather than an optimisation.

   </details>

3. **[Senior] Why is caching harder with GraphQL?**

   <details><summary>Model answer</summary>

   Because everything is a POST to one endpoint, so CDNs, proxies and browser caches cannot participate — they see one opaque URL with a body they will not inspect. The response shape is also determined by the client, so there is no stable cache key. Caching therefore has to be rebuilt inside the server, at the resolver or data-source layer keyed by entity identity rather than by query. Persisted queries partially restore the HTTP layer: because the query is identified by a hash, it can be sent as a GET with a stable URL and cached normally.

   </details>

4. **[Senior] How do you prevent a single query from overwhelming the service?**

   <details><summary>Model answer</summary>

   By computing a cost before executing. A depth limit alone is insufficient because breadth dominates — a depth-four query over a graph where each node has two hundred neighbours can request millions of nodes. So each field gets a weight, list fields multiply by their requested size, pagination arguments are mandatory, and queries exceeding a budget are rejected. Per-client budgets amortised over time handle the case of many individually acceptable queries. And rate limiting must be cost-based rather than request-based, because query costs vary by orders of magnitude — counting requests is meaningless.

   </details>

5. **[Staff] How would you harden GraphQL for production?**

   <details><summary>Model answer</summary>

   Persisted queries first, because they reverse most of the operational cost at once: clients register their queries at build time and send a hash, so the server knows every possible shape in advance, can pre-compute cost and cache policy, can serve them as cacheable GETs, and can reject anything unregistered — which eliminates the arbitrary-query attack surface entirely. Then request-scoped batching on every field that loads by id, since one missed resolver reintroduces N+1 as unexplained database load. Then field-level authorisation, because a single schema means any field is reachable through any path and endpoint-level assumptions silently stop holding. And finally per-operation instrumentation with required operation names, since a single POST endpoint makes default HTTP metrics useless for telling which query is slow.

   </details>

6. **[Principal] When would you advise against GraphQL?**

   <details><summary>Model answer</summary>

   When the client diversity that justifies it does not exist. With one or two well-known clients, a backend-for-frontend gives the same one-round-trip composite views with full server-side control of cost, caching and authorisation, and without a schema-wide reachable surface — that is a materially smaller operational commitment for the same user-facing benefit. I would also be cautious where the team cannot invest in the supporting infrastructure, because GraphQL's costs are all deferred and all operational: the developer-experience benefit is visible in week one, while the caching, cost-control and authorisation work surfaces months later under load, and a partially hardened GraphQL endpoint is a genuine denial-of-service risk rather than merely a slow one. Where it does earn its place is a large organisation with many independent client teams whose release cycles should not be coupled to backend endpoint work — and there the right form is a federated graph with persisted queries, so ownership is distributed and the runtime surface stays bounded.

   </details>

---

### Cursor pagination

*Page by remembering where you stopped rather than counting how many to skip — which stays correct under concurrent writes and stays fast at any depth.*

**Flow:** `Sort order` → `First page` → `Last tuple` → `Opaque cursor` → `Next range`

> **The 30-second version**  
> Page from a remembered position rather than a count: constant cost at any depth, and correct even while items are inserted and deleted around you.

**The problem**

Offset pagination is the obvious design: `LIMIT 20 OFFSET 40` for page three. It breaks in two ways, both of which appear only at scale or under concurrency, which is why it survives review and fails in production.

First, it is slow at depth: the database must scan and discard every skipped row, so `OFFSET 100000` reads a hundred thousand rows to return twenty. Second, it is incorrect under concurrent writes: if a row is inserted before your current position between page requests, every subsequent page shifts and you see an item twice — or, on deletion, miss one entirely.

> **The reframing**  
> Offset asks “skip N rows.” Cursor asks “give me rows after this specific point in the sort order.” The second question has a stable answer regardless of what was inserted or deleted elsewhere, and it can be answered by an index seek rather than a scan — which is why it fixes correctness and performance with the same change.

**Mental model**

A cursor encodes a position in a total order. The next page is everything strictly after that position. Because the position is a value rather than a count, insertions and deletions elsewhere in the list do not move it.

1. **Sort key** — The column or columns defining the order. Must be stable and, combined with a tiebreaker, unique.
2. **Tiebreaker** — A unique column — usually the primary key — appended to the sort key so no two rows share a position.
3. **Cursor** — An opaque encoding of the last row's sort key values. Opaque so clients cannot construct or depend on its internals.
4. **Seek predicate** — `WHERE (sort_key, id) > (last_sort_key, last_id)` — a range scan the index can satisfy directly.
5. **Page size** — Bounded, with a maximum enforced server-side, since the client controls it.

> **A non-unique sort key silently breaks pagination**  
> Sorting by `created_at` alone, where several rows share a timestamp, means `WHERE created_at > last` can skip rows sharing that timestamp, while `>=` returns duplicates. The fix is a compound cursor including a unique tiebreaker, compared as a tuple. This bug is invisible until two rows land in the same millisecond, which at scale is constant.

**How it works**

**Offset versus seek, in SQL and in cost**

```text
OFFSET (broken at depth and under writes)
  SELECT * FROM posts
  ORDER BY created_at DESC
  LIMIT 20 OFFSET 100000;
  -> the engine must produce and discard 100,000 rows
  -> cost grows linearly with page number

SEEK / KEYSET (correct and constant-cost)
  SELECT * FROM posts
  WHERE (created_at, id) < ('2024-06-01T10:00:00Z', 918273)
  ORDER BY created_at DESC, id DESC
  LIMIT 20;
  -> index seek straight to the position, read 20 rows
  -> cost is the same for page 1 and page 100,000

REQUIRED INDEX
  CREATE INDEX ON posts (created_at DESC, id DESC);
  the index order must match the ORDER BY exactly,
  including direction, or the engine sorts instead.

TUPLE COMPARISON MATTERS
  (created_at, id) < (t, i)  is NOT the same as
  created_at < t AND id < i
  the first is correct; the second loses rows.
```

1. **Always include a unique tiebreaker in the sort key** — Without it, ties at the boundary either duplicate or skip rows, and the bug appears only when two rows share a value.
2. **Make the cursor opaque** — Base64-encode it, or sign it. Clients that parse a cursor will depend on its structure, which prevents you from ever changing the sort key or adding a field.
3. **Match the index to the sort exactly** — Including direction. A mismatched index means the engine sorts the whole result set, which reintroduces the cost you were eliminating.
4. **Return the cursor, not the page number** — The response should carry `next` and, if supported, `previous` cursors. Clients never compute positions themselves.
5. **Enforce a maximum page size server-side** — The client supplies a limit, so it must be bounded, or one request can ask for the entire table.
6. **Do not offer arbitrary page jumps** — Cursors are inherently sequential. If the product genuinely needs “jump to page 500”, that is a different feature requiring a different mechanism — and usually it does not.

**Why offset is wrong under concurrent writes**

```text
list sorted newest first, 20 per page

t0  client fetches page 1 (OFFSET 0)   -> items 1..20
t1  a new item is inserted at the top
t2  client fetches page 2 (OFFSET 20)
    the list has shifted by one
    -> item 20 is now at position 21
    -> the client SEES ITEM 20 TWICE

deletion has the mirror problem:
t1  an item above the current position is deleted
t2  OFFSET 20 now starts one item later
    -> one item is NEVER SHOWN

WITH A CURSOR
  page 2 asks for "everything after item 20"
  insertions above and deletions above are irrelevant
  -> no duplicates, no skips, regardless of concurrency

THIS IS WHY INFINITE-SCROLL FEEDS USE CURSORS:
duplicate and missing items are extremely visible there.
```

> **Cursors can encode more than a position**  
> Because the cursor is opaque, it can carry the sort field, the direction, and a snapshot identifier — which lets the server validate that a client is not mixing a cursor from one sort order into a request using another, and lets it change the internal encoding later without breaking clients. Signing it additionally prevents clients from fabricating positions.

**Worked example**

Paginating an activity feed, and handling the cases that make cursors awkward.

**Design and the hard cases**

```text
ENDPOINT
  GET /feed?limit=20
  GET /feed?limit=20&after=eyJ0IjoiMjAyNC0wNi0wMVQxMDowMDowMFoiLCJpZCI6OTE4MjczfQ

RESPONSE
  { "items": [...],
    "page": { "next": "eyJ0...", "hasMore": true } }

INDEX
  (user_id, created_at DESC, id DESC)
  -> the user_id prefix scopes the feed; the rest matches
     the ORDER BY exactly

HARD CASE 1  new items while scrolling
  cursor pagination naturally EXCLUDES items inserted
  above the current position - the user scrolling down
  does not see them, which is correct
  a separate "N new posts" indicator handles them, using
  a cursor pointing at the TOP of the first page

HARD CASE 2  bidirectional paging
  provide both "after" and "before" cursors
  the response carries next and previous
  the index must support both directions - same index,
  scanned backwards

HARD CASE 3  the client wants a total count
  cursors give no total. COUNT(*) over a large filtered
  set is expensive.
  options: omit it (most feeds do), approximate it, or
  cap it ("500+ results")

HARD CASE 4  sort changes mid-session
  a cursor is valid only for the sort it was issued under
  -> encode the sort in the cursor and reject mismatches
```

| Metric | Value | Note |
|---|---|---|
| Page cost | constant | any depth |
| Duplicates | impossible | **under concurrency** |
| Total count | unavailable | usually fine |
| Page jumps | not supported | rarely needed |

> **The lost features are usually not needed**  
> Cursors give up total counts and arbitrary page jumps. In practice, feeds, timelines, logs and activity streams need neither — users scroll, they do not navigate to page 47. Where a total genuinely matters, it is usually acceptable to approximate or cap it, and where page jumps genuinely matter the dataset is typically small enough that offset pagination is fine. Requirements should be checked rather than assumed.

**When to use it**

- **Infinite scroll and feeds**, where duplicates and skips are highly visible.
- **Large datasets**, where offset cost at depth is prohibitive.
- **Frequently changing data**, where concurrent inserts and deletes make offsets incorrect.
- **APIs iterated by scripts**, which walk the full set and must not miss or repeat records.
- **Log and event streams**, where the natural access pattern is sequential from a position.

**When to avoid it**

- **Do not use it when arbitrary page jumps are a genuine requirement** — that needs offsets or a different design.
- **Do not use it without a unique tiebreaker** in the sort key.
- **Do not expose cursor internals**, which freezes your sort implementation.
- **Do not mix cursors across sort orders** without validating that the cursor matches the requested sort.
- **Do not promise total counts**, which cursors cannot provide cheaply.

**Advantages**

- **Constant cost per page** regardless of depth, because it is an index seek rather than a scan.
- **Correct under concurrent writes** — no duplicates and no skipped rows.
- **Natural fit for streaming and iteration**, where the client resumes from where it stopped.
- **Resumable**: a client can store a cursor and continue days later from exactly that position.
- **Opaque encoding** lets the server change the internal representation without breaking clients.

**Disadvantages**

- **No arbitrary page jumps**, since positions are values rather than counts.
- **No total count**, or only at significant cost.
- **Requires a matching index**, including direction, or the performance benefit disappears.
- **Bidirectional paging needs extra care**, with both cursors and a reversible index scan.
- **Cursors are sort-specific**, so changing the sort invalidates outstanding cursors.

**Trade-offs**

**Pagination approaches**

|  | Offset | Cursor / keyset | Page token (opaque state) |
|---|---|---|---|
| Cost at depth | Linear in offset | Constant | Constant |
| Correct under writes | No | Yes | Yes |
| Arbitrary page jumps | Yes | No | No |
| Total count | Cheap-ish | Not available | Not available |
| Implementation | Trivial | Moderate | Moderate; server-side state |
| Best for | Small, stable admin tables | Feeds, logs, large datasets | Complex queries, search |

Page tokens are the generalisation: an opaque token that may encode a keyset position, a search engine's scroll context, or a snapshot identifier. They give the same client-facing contract while letting the server choose the mechanism per query shape — which is why large APIs standardise on tokens rather than exposing keysets directly.

**How it fails**

**Pagination failures**

| Symptom | Cause | Fix |
|---|---|---|
| Duplicate items in a feed | Offset pagination with concurrent inserts | Cursor pagination |
| Items never shown | Offset pagination with concurrent deletes | Cursor pagination |
| Deep pages time out | `OFFSET` scanning and discarding rows | Keyset seek with a matching index |
| Rows skipped or repeated at boundaries | Non-unique sort key without a tiebreaker | Compound cursor with the primary key, compared as a tuple |
| Pagination slow despite cursors | Index order does not match `ORDER BY` | Index on exactly the sort columns and directions |
| Client breaks after a backend change | Cursor internals exposed and depended upon | Opaque, ideally signed, cursors |
| Wrong results after changing sort | Cursor reused across sort orders | Encode the sort in the cursor; reject mismatches |

**Limits**

> **Practical guidance**
>
> - **Offset cost** grows linearly: `OFFSET 100000` reads 100,000 rows before returning any.
> - **Cursor cost is constant** — an index seek plus the page size.
> - **The index must match the sort exactly**, including column order and direction.
> - **Tuple comparison** `(a, b) > (x, y)` is required; comparing columns independently loses rows.
> - **Maximum page size** must be enforced server-side, typically 100–1,000 depending on payload size.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Offset | Small, stable, admin-facing lists | Wrong under writes; slow at depth |
| Keyset cursor | Feeds, logs, large ordered sets | No jumps or totals |
| Opaque page token | Search, complex queries | Server may hold state |
| Time-bucketed windows | Time-series and logs | Uneven page sizes |
| Snapshot pagination | Consistent view across a long iteration | Snapshot cost and expiry |
| Client-side full fetch | Genuinely small datasets | Does not scale |

Snapshot pagination is worth knowing for exports and reports: the server takes a consistent point-in-time view and pages within it, so a long iteration sees a coherent dataset rather than a moving target — at the cost of holding the snapshot, which in an MVCC database means preventing cleanup of versions for its duration.

**In real systems**

- **Twitter, Facebook and most feed APIs** use cursors precisely because duplicates and gaps in an infinite scroll are immediately visible to users.
- **Stripe's list endpoints** use `starting_after` and `ending_before` object ids, which is keyset pagination with the id as the cursor.
- **Elasticsearch's `search_after`** is keyset pagination, introduced because deep `from`/`size` paging is prohibitively expensive in a distributed search engine.
- **Google's API design guide** mandates opaque page tokens rather than exposing offsets or keysets, letting the server change mechanism freely.
- **Database replication and CDC** are conceptually cursor-based: the consumer stores a position and resumes from it, which is why offsets and log sequence numbers exist.

**Common mistakes**

- **Offset pagination on a feed**, producing visible duplicates and gaps.
- **Sort key without a unique tiebreaker**, skipping or repeating rows at page boundaries.
- **Comparing columns independently** instead of as a tuple, which loses rows.
- **Index direction not matching `ORDER BY`**, causing a full sort.
- **Exposing cursor internals**, freezing the implementation.
- **No server-side maximum page size**, letting a client request everything.
- **Reusing a cursor across a changed sort order.**

**The staff-level view**

Pagination is a small design decision that becomes very expensive to change once clients depend on it, and offset pagination's failures are invisible in testing.

- **Make cursor pagination the default in API guidelines**, since offset's correctness bug appears only under concurrency and its performance bug only at depth — neither of which reviews or tests catch.
- **Require opaque tokens rather than exposed keysets**, so the server can change sort implementation, switch mechanisms, or add snapshotting without breaking clients.
- **Check the index matches the sort exactly** in review, including direction; a mismatch silently reintroduces a full sort.
- **Challenge total-count and page-jump requirements.** They are frequently assumed rather than needed, and they are what force offset pagination.
- **Insist on a unique tiebreaker.** The boundary bug from a non-unique sort key is subtle, intermittent and hard to reproduce.

**Go deeper**

Offset pagination fails twice. It is slow at depth, because the database must generate and discard every skipped row, so `OFFSET 100000` reads a hundred thousand rows to return twenty. And it is incorrect under concurrency: an insertion above your position shifts everything down, so an item appears on two consecutive pages, while a deletion causes one to be skipped entirely.

A cursor encodes a position in the sort order instead of a count. The next page is everything strictly after that position, expressed as a tuple comparison — `(created_at, id) < (t, i)` — which an index matching the sort order satisfies with a seek rather than a scan. Cost becomes constant at any depth, and insertions or deletions elsewhere in the list are irrelevant, so duplicates and gaps become impossible.

Two details matter. The sort key needs a unique tiebreaker, usually the primary key, or rows sharing a boundary value are skipped or repeated — a bug that appears only when two rows share a timestamp. And the cursor should be opaque and ideally signed, so clients cannot depend on its structure and the server remains free to change the sort, the mechanism, or the encoding. What you give up — arbitrary page jumps and cheap totals — is usually assumed rather than required.

Pagination looks like a minor interface decision and is in fact a correctness decision, because offset pagination is wrong in a way that testing does not surface and that clients experience as visible bugs.

**Two independent failures of offset.** Performance: the engine must materialise and discard every skipped row, so cost grows linearly with page number and deep pages eventually time out. Correctness: an offset is a count into a list that is changing underneath the client. An insertion before the current position shifts everything, so the last item of one page reappears at the start of the next; a deletion causes an item to be skipped and never shown. In an infinite-scroll feed both are immediately visible to users, which is why feed APIs universally use cursors.

**The seek formulation.** A cursor records the sort key values of the last row returned, and the next page asks for everything strictly after that position: `WHERE (created_at, id) < (t, i) ORDER BY created_at DESC, id DESC LIMIT 20`. With an index on exactly those columns in exactly those directions, this is a seek followed by a sequential read, so page one and page one hundred thousand cost identically. Two implementation details are easy to get wrong and both lose rows: the comparison must be a genuine tuple comparison rather than independent column comparisons, and the index direction must match the sort or the engine performs a full sort and the benefit evaporates.

**Uniqueness at the boundary.** If the sort key is not unique — several rows sharing a timestamp — then a strict comparison skips rows at the boundary while a non-strict one duplicates them. Appending a unique tiebreaker, normally the primary key, makes every position distinct. This bug is intermittent, dependent on data collisions, and therefore rarely reproduced in development while being constant at production scale.

**Opacity is a contract decision.** Anything visible in a cursor will eventually be parsed and depended upon by a client, which freezes the sort key, the mechanism and the encoding permanently. Base64-encoding it, and ideally signing it, keeps the implementation server-side and allows later migration to a search engine's scroll context, a snapshot identifier, or a different sort — none of which is possible once clients construct their own. Encoding the sort order inside the cursor additionally lets the server reject a cursor being reused under a different sort, which would otherwise return silently wrong results.

**What you give up, and whether it matters.** Cursors cannot support arbitrary page jumps, because positions are values rather than counts, and they cannot cheaply produce a total count. Both are more often assumed than required: users of feeds and logs scroll rather than navigating to page forty-seven, and totals over large filtered sets are expensive regardless of pagination style, with most products satisfied by omission, approximation, or a cap. These requirements deserve explicit challenge, because accepting them uncritically is what forces offset pagination and defers its failures into production.

**The organisational form.** Standardising on opaque page tokens rather than exposed keysets gives a uniform client contract while leaving each service free to choose its mechanism per query shape and to change it later without a breaking change. Paired with a server-enforced maximum page size, a required unique tiebreaker, and a review check that indexes match sort orders including direction, that makes correct pagination the default rather than something each team rediscovers after an incident.

**Prove it — interview questions**

1. **[Basic] Why is offset pagination incorrect under concurrent writes?**

   <details><summary>Model answer</summary>

   Because an offset is a count, and the count shifts when rows are inserted or deleted before your position. If a new item is added at the top between fetching page one and page two, everything moves down by one, so the last item of page one appears again at the start of page two. A deletion produces the mirror problem: one item is skipped entirely and never shown. A cursor asks for everything after a specific value instead, which is unaffected by changes elsewhere in the list.

   </details>

2. **[Basic] Why is offset slow at depth?**

   <details><summary>Model answer</summary>

   Because the database has to produce and discard every skipped row. `OFFSET 100000` means generating a hundred thousand rows in sort order and throwing them away before returning twenty, so cost grows linearly with page number. A keyset cursor instead seeks directly into the index at the recorded position and reads forward, so page one and page one hundred thousand cost the same.

   </details>

3. **[Senior] Why does a cursor need a unique tiebreaker?**

   <details><summary>Model answer</summary>

   Because a non-unique sort key makes the boundary ambiguous. Sorting by `created_at` alone, if several rows share a timestamp, means a strict comparison skips the rows sharing the boundary value while a non-strict one returns duplicates. Appending the primary key makes each position unique, and the comparison must be a tuple comparison — `(created_at, id) > (t, i)` — rather than comparing the columns independently, which is a different and incorrect predicate that loses rows. The bug only appears when two rows share a value, which at scale is constant but in testing is rare.

   </details>

4. **[Senior] Why should cursors be opaque?**

   <details><summary>Model answer</summary>

   Because anything a client can read, it will eventually depend on. If the cursor is a visible timestamp and id, clients start constructing them, parsing them, or storing them with assumptions about their structure — and then you cannot change the sort key, add a field, switch to a search engine's scroll context, or introduce snapshotting without breaking them. Encoding it opaquely, and ideally signing it, keeps the mechanism a server-side implementation detail and lets it evolve. It also prevents clients fabricating positions they were never given.

   </details>

5. **[Staff] What do you lose with cursor pagination, and does it matter?**

   <details><summary>Model answer</summary>

   Two things: arbitrary page jumps and cheap total counts. Both are usually assumed rather than required. Users of feeds, timelines and logs scroll; they do not navigate to page forty-seven, and where they do the dataset is usually small enough for offsets to be fine. Totals are more often requested, but a count over a large filtered set is expensive regardless of pagination style, and most products are satisfied by omitting it, approximating it, or capping it at something like “500+”. I would push back on both requirements explicitly, because accepting them uncritically is what forces offset pagination — and then the correctness bug under concurrency and the performance cliff at depth arrive later as production incidents rather than as design decisions.

   </details>

6. **[Principal] What pagination standard would you set for an organisation's APIs?**

   <details><summary>Model answer</summary>

   Opaque page tokens as the universal client-facing contract, with keyset pagination as the default implementation behind them. The token matters as much as the mechanism: it means individual services can use a keyset, a search engine's scroll context, or a snapshot identifier depending on the query shape, and can change that choice later without a breaking change — whereas exposing offsets or raw keysets freezes the implementation on day one. Alongside that I would mandate a server-enforced maximum page size, a unique tiebreaker in every sort, and a review check that the index matches the sort order including direction, since that last one silently reintroduces a full sort and is easy to miss. And I would explicitly discourage total counts in list responses, because they are the requirement that most often forces teams back to offsets, and they are almost always negotiable once someone asks what the number is actually used for.

   </details>

---

### Idempotency keys

*A client-supplied key makes a retried request return the original result instead of performing the operation twice — which is what allows clients to retry at all.*

**Flow:** `Operation key` → `Deduplication check` → `Atomic mutation` → `Stored result` → `Retry response`

> **The 30-second version**  
> The client names its operation and reuses that name on retries; the server stores the result against the name and replays it. That is what makes retrying after a timeout safe.

**The problem**

A client sends a payment request and the connection drops before the response arrives. The payment may have succeeded; it may not. The client cannot tell. If it retries and the first attempt succeeded, the customer is charged twice. If it does not retry and the first attempt failed, the order is never paid.

This ambiguity is not an edge case. Timeouts are routine during ordinary network degradation, and they are most frequent precisely when systems are stressed — meaning duplicate-charge risk is highest exactly when retries are most needed.

> **The key changes who answers the question**  
> Without a key, only the server knows whether the operation happened, and the client cannot ask because it has no way to refer to its previous attempt. A client-supplied key gives that attempt a name. The retry then means “did you already do the thing called X?” — which the server can answer definitively.

**Mental model**

The client names the operation before performing it. The server records that name alongside the result, atomically. A second request with the same name is not a new operation; it is a request for the previous one's outcome.

1. **Key generation** — The client creates a unique key per logical operation — typically a UUID — and reuses it across retries of that operation.
2. **First request** — The server records the key as in-progress, performs the work, stores the response with the key, and returns it.
3. **Retry** — The server finds the key, and returns the stored response verbatim. Indistinguishable from the first call.
4. **Concurrent retry** — A request arriving while the first is still running must not execute; it should return a conflict or wait, not proceed.
5. **Retention** — Keys are kept longer than the maximum plausible retry window, then expired.

> **Storing a flag is not enough — store the response**  
> A record that merely says “this happened” lets the server avoid repeating the work, but the best it can then return is a conflict, which tells the client something went wrong when it did not. The client must then implement special handling to interpret that. Storing and replaying the original response makes the retry genuinely transparent, which is the entire point of offering idempotency.

**How it works**

**The full flow, including the concurrent case**

```text
REQUEST  POST /payments   Idempotency-Key: abc-123

SERVER
  BEGIN
    row = SELECT * FROM idempotency WHERE key = 'abc-123'
          FOR UPDATE
    if row exists:
        if row.state = 'done':   COMMIT; return row.response
        if row.state = 'running': COMMIT; return 409 in-progress
    INSERT (key='abc-123', state='running', started_at=now())
  COMMIT

  ... perform the payment (external call) ...

  BEGIN
    UPDATE idempotency
       SET state='done', response=<body>, status=201
     WHERE key='abc-123'
  COMMIT

  return the response

CRASH RECOVERY
  a row stuck in 'running' beyond a timeout must be
  resolvable: either query the downstream for the
  operation's outcome, or mark it failed and allow retry.
  Leaving it 'running' forever blocks the client permanently.
```

1. **Require the key on every non-idempotent write** — Clients cannot retry safely without one, so omitting it forces them either to risk duplicates or to give up on failures — both bad outcomes.
2. **Bind the key to the request content** — Store a hash of the request body with the key. If the same key arrives with a different body, that is a client bug and should return an error rather than silently replaying an unrelated response.
3. **Scope keys per client or per account** — A globally shared key namespace lets one client collide with another's key, accidentally or deliberately.
4. **Make the record and the effect atomic where possible** — If the effect is a local database write, write both in one transaction. If it is an external call, pass the key through so the remote system deduplicates too.
5. **Handle the in-progress case explicitly** — A retry arriving while the original is still running must not execute concurrently. Returning a conflict with a retry hint is the usual answer.
6. **Recover stuck in-progress records** — A process that dies mid-operation leaves a record that will otherwise block that key forever. A timeout plus an outcome check is required.

**Why the key must come from the client**

```text
SERVER-GENERATED KEY (does not work)
  server assigns an id when it receives the request
  client retries -> a NEW request arrives -> a NEW id
  -> the server cannot tell it is the same logical operation
  -> two payments

DERIVED KEY (works sometimes)
  key = hash(account, amount, timestamp, description)
  + no client change needed
  - two GENUINELY distinct identical payments collide
    (buying the same coffee twice in one minute)
  - timestamp granularity determines the failure mode

CLIENT-SUPPLIED KEY (correct)
  client generates a UUID when it FORMS the intent,
  reuses it for every retry of that intent
  -> distinguishes "retry of X" from "a new, similar X"
  -> the only scheme that gets both cases right

THE CLIENT KNOWS SOMETHING THE SERVER CANNOT INFER:
whether this is a new intention or a repeat of an old one.
```

> **The gap between the effect and the record**  
> If the effect is performed and the process dies before the response is stored, the key remains in-progress while the operation has already happened. The retry must not re-execute. This is why the in-progress state needs a resolution path — query the downstream system for the outcome, or use the same key on the external call so it deduplicates — rather than simply timing out and allowing a retry.

**Worked example**

Idempotency across a chain of services, showing where each key catches what.

**Key propagation through a payment flow**

```text
MOBILE APP
  generates key on "Pay" tap:  Idempotency-Key: 7f3a...
  reuses it for every retry of that tap
  -> catches: user double-tap, network retry, app restart

API GATEWAY
  passes the key through unchanged
  -> catches: proxy-level retries of 502s

PAYMENT SERVICE
  checks its own idempotency table on 7f3a...
  -> returns the stored response on retry
  passes 7f3a... (or a derived key) to the PSP
  -> catches: its own crash between effect and record,
              because the PSP also deduplicates

PSP (external)
  deduplicates on the key it received
  -> the authoritative deduplication for the money movement

LEDGER CONSUMER (async)
  dedupes on the payment event's id
  -> catches: broker redelivery

WHAT BREAKS IF THE KEY IS NOT PROPAGATED
  the payment service crashes after calling the PSP but
  before storing the response
  -> retry re-calls the PSP with a NEW key
  -> double charge, despite the payment service having
     a perfectly correct idempotency table
```

| Metric | Value | Note |
|---|---|---|
| Client key | catches retries | the important one |
| Propagated | to the PSP | **closes the crash gap** |
| Stored | the response | transparent replay |
| Retention | 24 h | > max retry window |

> **Propagation is what makes it correct, not the table**  
> A service with a perfect idempotency table still double-charges if it crashes between calling the payment provider and recording the result — unless it passed the key downstream so the provider deduplicates too. Idempotency is a chain property: it holds only if the key travels the whole way to wherever the irreversible effect actually happens.

**When to use it**

- **Any non-idempotent write a client may retry**, which in practice means every POST that has consequences.
- **Payments, orders, transfers and provisioning**, where a duplicate is expensive and visible.
- **Public and partner APIs**, where you cannot control client retry behaviour.
- **Behind unreliable networks** — mobile clients especially, where connection loss is routine.
- **Anywhere a proxy, gateway or SDK may retry automatically** without the application knowing.

**When to avoid it**

- **Do not omit it because “clients should not retry”** — they will, and so will proxies and SDKs.
- **Do not generate the key server-side**, which cannot distinguish a retry from a new request.
- **Do not derive the key from request content alone**, which conflates genuinely repeated operations with retries.
- **Do not return a conflict on retry** when you could return the original response.
- **Do not leave in-progress records without a resolution path**, which blocks that key permanently.

**Advantages**

- **Clients can retry safely**, which is the behaviour you want during degradation.
- **Transparent replay** means no client-side special handling for the retry case.
- **Catches duplicates from every layer** — user double-taps, SDK retries, proxy retries — when the key is generated early.
- **Simple to implement** relative to the class of bug it prevents.
- **Composes across services** when propagated, closing crash windows that local tables cannot.

**Disadvantages**

- **Storage and retention** for keys and stored responses, which must be managed.
- **Atomicity with external effects is impossible**, so correctness depends on downstream cooperation.
- **In-progress handling is subtle**, and getting it wrong either blocks clients or permits duplicates.
- **Clients must participate**, generating and reusing keys correctly.
- **Key scoping and content binding** add cases that are easy to overlook.

**Trade-offs**

**Key strategies**

| Source | Catches retries | Distinguishes new requests | Client change |
|---|---|---|---|
| Server-generated id | No | n/a | None |
| Content hash | Yes | No — identical requests collide | None |
| Content hash + time bucket | Within the bucket | Partially | None |
| Client-supplied UUID | Yes | Yes | Required |
| Natural business key | Yes | Yes, if genuinely unique | Depends |

A natural business key — an order id the client already generated, or a unique reference from the caller's system — is often the best of both worlds: it requires no new concept for the client, it is genuinely unique per intention, and it is meaningful in logs and support conversations.

**How it fails**

**Idempotency failures**

| Failure | Cause | Fix |
|---|---|---|
| Double charge despite an idempotency table | Key not propagated to the external system | Pass the key downstream; make the remote deduplicate |
| Retry returns 409 and the client gives up | Storing a flag rather than the response | Store and replay the original response |
| Client blocked forever | In-progress record never resolved after a crash | Timeout plus downstream outcome check |
| Unrelated response returned | Key reused for a different request | Bind the key to a content hash; reject mismatches |
| Cross-client key collision | Global key namespace | Scope keys per client or account |
| Genuine second purchase rejected | Content-derived key | Client-supplied key per intention |
| Table grows without bound | No retention policy | Expire beyond the maximum retry window |

**Limits**

> **Practical parameters**
>
> - **Retention**: 24 hours is a common default — longer than any plausible retry window, short enough to bound storage.
> - **Key format**: a client-generated UUID per logical intention, reused across retries of that intention.
> - **Scope**: per client or per account, never global.
> - **In-progress timeout**: minutes, after which the outcome must be determined rather than assumed.
> - **Content binding**: store a hash of the request so key reuse with different content is detected and rejected.

**Alternatives**

| Approach | Handles | Limitation |
|---|---|---|
| Client idempotency key | Retries from every layer | Requires client participation |
| Natural business key | Same, with domain meaning | Only where one exists |
| Conditional write (version) | Duplicate updates to one entity | Single entity only; not for creation |
| Naturally idempotent design | Everything, with no mechanism | Only some operations qualify |
| Reconciliation after the fact | Detection, not prevention | Damage already done |
| Accept duplicates | Where genuinely harmless | Requires proof of harmlessness |

Reformulating to be naturally idempotent is always worth attempting first: setting a value rather than incrementing, writing a specific version rather than appending, or creating a resource at a client-chosen URL with `PUT` rather than `POST`. Where it applies, it needs no key, no storage and no retention.

**In real systems**

- **Stripe's `Idempotency-Key` header** is the reference implementation, storing the full response and replaying it for 24 hours, with content binding to catch key reuse.
- **AWS API client tokens** serve the same purpose for resource creation, ensuring a retried create returns the existing resource rather than making a second one.
- **HTTP `PUT` semantics** are naturally idempotent, which is why creating at a client-chosen URL avoids the problem entirely where the API shape allows it.
- **Payment networks** require idempotency at every hop, because the irreversible effect is at the far end and a local table upstream cannot protect it.
- **Message consumers** apply the same pattern with the event id as the key, which is why at-least-once delivery is workable.

**Common mistakes**

- **Generating the key server-side**, which cannot recognise a retry.
- **Deriving the key from content**, conflating genuine repeats with retries.
- **Storing a flag instead of the response**, so retries get a conflict.
- **Not propagating the key** to the system where the effect actually occurs.
- **No in-progress resolution**, permanently blocking a key after a crash.
- **Global key scope**, allowing cross-client collisions.
- **No content binding**, so a reused key returns an unrelated response.

**The staff-level view**

Idempotency keys are a small mechanism that prevents the most expensive class of API bug, and the implementation details that matter are the ones teams most often skip.

- **Require a key on every consequential POST**, provided by a shared library rather than implemented per service, because the subtleties — content binding, scoping, in-progress handling — are identical everywhere and identically easy to get wrong.
- **Insist the key is propagated to wherever the irreversible effect occurs.** A local table does not protect against a crash between the external call and the record.
- **Store the response, not a flag**, so retries are transparent and clients need no special handling.
- **Design the in-progress resolution path explicitly.** Without it, a crashed request blocks that key forever, which is a support burden that surfaces months later.
- **Prefer naturally idempotent designs where the API shape allows it**, since they remove the mechanism, the storage and the retention question entirely.

**Go deeper**

A timeout is ambiguous: the operation may have succeeded and the response been lost. Without a way to refer to the previous attempt, a client must either risk a duplicate by retrying or risk losing the operation by not retrying. A client-supplied idempotency key names the logical operation, so the retry asks “did you already do X?” and the server answers definitively.

The key must come from the client, because only the client knows whether this is a new intention or a repeat. A server-generated id is new on every request; a content-derived key cannot distinguish a retry from a genuinely repeated identical purchase. And the server must store the full response, not a flag — replaying it verbatim makes the retry transparent, whereas returning a conflict forces client-side special handling and misreports a success as a problem.

Two details decide correctness in practice. A retry arriving while the original is still running must not execute concurrently, and the resulting in-progress record needs a resolution path or it blocks that key forever. And the key must be propagated to wherever the irreversible effect actually happens: a service with a perfect local table still double-charges if it crashes between calling the payment provider and recording the result, unless the provider was given the same key.

Idempotency keys exist because timeouts are ambiguous, and that ambiguity is most common precisely when systems are degraded and retries are most needed.

**Why the key must be client-supplied.** The server cannot infer whether an incoming request is a retry or a new intention. A server-generated identifier is fresh each time, so it recognises nothing. A key derived from request content recognises retries but conflates them with genuinely repeated identical operations — two coffees bought in the same minute become one. Only a key created at the moment the client forms the intent, and reused across every retry of that intent, distinguishes both cases. The client holds information the server structurally cannot deduce.

**Store the response, not a flag.** A record asserting only that the operation occurred lets the server avoid repeating work but leaves it with nothing to return except a conflict — which misreports a success and forces the client to implement special handling to interpret it. Replaying the original response verbatim makes a retry indistinguishable from the first call, which is the property that lets clients retry freely without bespoke logic. Binding a hash of the request to the key completes it, so a client reusing a key for a different request is rejected rather than receiving an unrelated response.

**The in-progress state is where implementations get subtle.** A retry arriving while the first attempt is still executing must not run concurrently, so the key is claimed under a lock and a concurrent retry receives a conflict with a retry hint. But a process that dies mid-operation leaves that claim outstanding, and without a resolution path the key is blocked permanently — a failure that surfaces as a support ticket months later. The resolution requires determining the actual outcome, typically by querying the downstream system with the same key, rather than either waiting forever or optimistically re-executing.

**Idempotency is a chain property.** The most damaging failure mode occurs with a locally perfect implementation: the service calls an external payment provider, the charge succeeds, and the process dies before the response is stored. The local table shows in-progress while money has moved. A retry that calls the provider with a new key charges twice. The only fix is propagating the same key to the point where the irreversible effect occurs, so the provider deduplicates — which is precisely why payment networks require idempotency at every hop rather than trusting upstream callers to be correct.

**Scope, retention and reuse.** Keys must be scoped per client or account, since a global namespace permits collisions between tenants, accidental or malicious. Retention must exceed the longest plausible retry window — twenty-four hours is a common default — and then expire, or the table grows unbounded. And the key should ideally be a natural business identifier the client already has, since that removes a concept from the client's mental model and makes the key meaningful in logs and support conversations.

**The best version is the one you do not need.** Reformulating operations to be naturally idempotent removes the mechanism entirely: creating a resource at a client-chosen URL with `PUT` rather than `POST`, setting a value rather than incrementing, writing at a specific version rather than appending. Where the API shape permits it, that is strictly better than any key scheme — no storage, no retention, no in-progress state, and no chance of a subtly wrong implementation.

**Prove it — interview questions**

1. **[Basic] What problem do idempotency keys solve?**

   <details><summary>Model answer</summary>

   Ambiguous outcomes. When a request times out, the client cannot tell whether it succeeded, and retrying risks performing the operation twice. A client-supplied key names the logical operation, so a retry becomes a question the server can answer — “have you already done the thing called X?” — and the server returns the original result instead of executing again.

   </details>

2. **[Basic] Why must the key come from the client?**

   <details><summary>Model answer</summary>

   Because only the client knows whether this is a new intention or a repeat of a previous one. A server-generated identifier is new on every request, so it cannot recognise a retry. A key derived from the request content recognises retries but also collides with genuinely repeated operations — buying the same coffee twice in a minute becomes indistinguishable from retrying one purchase. Only a key created when the client forms the intent, and reused across retries of that intent, distinguishes both cases correctly.

   </details>

3. **[Senior] Why store the response rather than just recording that the operation happened?**

   <details><summary>Model answer</summary>

   Because the goal is that a retry is indistinguishable from the first call. If the record is just a flag, the server can avoid repeating the work but has nothing to return except a conflict — which tells the client something went wrong when it did not, and forces client-side special handling to interpret it. Replaying the original response verbatim means the client's ordinary retry logic works with no special case, which is the entire reason for offering idempotency.

   </details>

4. **[Senior] How do you handle a retry arriving while the original is still in flight?**

   <details><summary>Model answer</summary>

   It must not execute concurrently, because that would produce exactly the duplicate the mechanism exists to prevent. The usual answer is to record the key as in-progress under a lock, so a concurrent retry finds it and returns a conflict with a hint to retry shortly. The harder part is recovery: if the process dies mid-operation, that record stays in-progress and would block the key forever. So there needs to be a timeout after which the outcome is determined — typically by querying the downstream system with the same key — rather than either blocking indefinitely or optimistically allowing a re-execution.

   </details>

5. **[Staff] Where can idempotency still fail even with a correct implementation?**

   <details><summary>Model answer</summary>

   In the gap between performing an external effect and recording that it happened. If the payment service calls the provider, the charge succeeds, and the process dies before writing the response, the local table shows in-progress while the money has moved. A retry that re-calls the provider with a fresh key charges twice, despite the local implementation being flawless. The fix is propagation: pass the same idempotency key to the provider so it deduplicates at the point where the irreversible effect occurs. Idempotency is a chain property — it holds only if the key travels all the way to where the effect happens, which is why payment networks require it at every hop rather than trusting callers.

   </details>

6. **[Principal] How would you make idempotency reliable across an organisation?**

   <details><summary>Model answer</summary>

   By shipping it as shared infrastructure rather than as guidance, because the details that matter — content binding so a reused key cannot return an unrelated response, scoping per client so keys cannot collide across tenants, storing the full response for transparent replay, and resolving stuck in-progress records — are identical in every service and identically easy to omit. A shared middleware plus a standard header means a service gets correct behaviour by default. Alongside that, I would make key propagation a reviewed requirement for any call chain ending in an irreversible external effect, since that is the failure a local implementation cannot prevent and the one that costs real money. And I would encourage naturally idempotent API shapes where possible — creating at a client-chosen URL with `PUT`, setting values rather than incrementing — because those remove the mechanism, the storage and the retention policy entirely, and the best version of this problem is the one that does not exist.

   </details>

---

### Rate limiting algorithms

*Decide who may proceed and who must wait, using an algorithm whose burst behaviour, memory cost and fairness match what you are actually protecting.*

**Flow:** `Identity` → `Cost estimate` → `Bucket state` → `Allow or reject` → `Retry guidance`

> **The 30-second version**  
> Bound what each client may consume, using an algorithm whose burst and memory behaviour fit the use — and remember it enforces fairness, not overload protection.

**The problem**

A service has finite capacity. Without a limit, one client — a misbehaving retry loop, a runaway script, a scraper, a legitimate customer who just launched a campaign — can consume all of it, and every other client experiences an outage caused by someone else's behaviour.

Rate limiting is the mechanism that makes capacity a per-client resource rather than a shared free-for-all. The design questions are which algorithm, keyed on what identity, applied at which layer, and what happens to requests that exceed it.

> **Rate limiting is fairness first, protection second**  
> Its primary job is to stop one participant's behaviour from degrading everyone else's experience. Protecting absolute capacity is a related but distinct goal, better served by load shedding and concurrency limits — which react to the system's actual state rather than to a fixed per-client number. Conflating them leads to limits set too low for fairness and too high for protection.

**Mental model**

Every algorithm answers the same question — has this identity consumed more than its allowance in the relevant window — and they differ in how they define the window, how much burst they permit, and how much state they need.

1. **Fixed window** — Count requests per calendar interval. Simple and cheap; permits double the limit across a boundary.
2. **Sliding window log** — Store a timestamp per request and count those within the window. Exact, but memory grows with the rate.
3. **Sliding window counter** — Weight the previous window's count by how far into the current one you are. Approximate, cheap, and usually the right default.
4. **Token bucket** — Tokens refill at a constant rate up to a capacity; each request consumes one. Permits a burst up to the capacity, then enforces the rate.
5. **Leaky bucket** — Requests enter a queue drained at a fixed rate. Smooths output completely; adds latency rather than rejecting.

> **Token bucket is usually the right default**  
> It has two parameters that map directly onto product requirements: the sustained rate a client may use, and how large a burst they may make after being idle. That matches real client behaviour — mostly quiet with occasional bursts — far better than a flat window, and it costs only two numbers of state per identity.

**How it works**

**The algorithms, compared on the same traffic**

```text
LIMIT: 100 requests per minute

FIXED WINDOW
  counter resets at each minute boundary
  client sends 100 at 10:00:59 and 100 at 10:01:01
  -> 200 requests in 2 seconds, both windows satisfied
  -> the boundary burst problem

SLIDING WINDOW LOG
  store every request timestamp; count those in the last 60s
  exact, no boundary problem
  -> memory = one timestamp per request per client
  -> at high rates this is the dominant cost

SLIDING WINDOW COUNTER
  count(current) + count(previous) x (fraction of window remaining)
  at 10:01:15, 25% into the window:
    effective = current + previous x 0.75
  -> approximate, bounded memory, no boundary burst
  -> the pragmatic default for request counting

TOKEN BUCKET
  capacity 100, refill 100/60s = 1.67 tokens/s
  idle client accumulates up to 100 tokens -> can burst 100
  then sustained at 1.67/s
  -> state: token count + last refill timestamp (2 numbers)
  -> burst and rate are independently configurable
```

1. **Choose the identity deliberately** — API key, user id, account, IP, or a combination. IP is weak — shared NATs punish innocents and attackers rotate addresses — so authenticate first and key on the account wherever possible.
2. **Limit by cost, not by request count** — A query scanning a million rows and a health check are not equivalent. Weighted limits, or a token cost per operation, prevent an expensive endpoint from being rate-limited identically to a trivial one.
3. **Always return `Retry-After`** — Without it, clients guess, and their guesses synchronise into another spike. With it, retry behaviour becomes cooperative.
4. **Apply limits at multiple layers** — A global edge limit for abuse, a per-account limit for fairness, and a per-endpoint limit for expensive operations. Each protects a different thing.
5. **Decide between rejection and queueing** — Token bucket rejects immediately; leaky bucket queues and smooths. Rejection is right for interactive requests; queueing suits background work where latency is flexible.
6. **Make distributed counting approximate on purpose** — Exact global counting requires a round trip per request. Local counters with periodic reconciliation, or a shared store with atomic operations, trade precision for latency — and slight over-admission is almost always acceptable.

**Distributed rate limiting: the consistency trade**

```text
PROBLEM  10 gateway instances, one global limit of 1000/min

OPTION A  central store, atomic increment per request
  correct to within one request
  cost: a network round trip on EVERY request
  risk: the store becomes a hard dependency and a bottleneck

OPTION B  local limits of 100/min each
  no coordination at all
  problem: uneven load means some instances reject while
           others have headroom -> effective limit < 1000

OPTION C  local counters + periodic sync (usually best)
  each instance tracks locally, syncs every few seconds
  slight over-admission during the sync interval
  no per-request coordination
  degrades to local-only if the sync fails

OPTION D  central store, batched reservations
  an instance reserves 20 tokens at a time
  1 round trip per 20 requests instead of per request
  good balance of accuracy and cost

ACCEPT APPROXIMATION. A limit is a policy, not an invariant;
admitting 1,050 instead of 1,000 harms nobody.
```

> **Rate limiting is not overload protection**  
> A per-client limit set for fairness does not know whether the system is currently healthy. During a dependency slowdown, every client can stay within its limit while the service collapses, because the limit was calibrated for normal conditions. Overload needs concurrency limits and load shedding that react to observed latency and queue depth — mechanisms that measure the system rather than the client.

**Worked example**

Designing limits for a public API with free and paid tiers.

**Layered limits, each protecting something different**

```text
LAYER 1  EDGE, per IP  -  abuse protection
  1,000 req/min, sliding window counter
  purpose: absorb scrapers and unauthenticated floods
           before they reach anything expensive
  note: IP is weak (NAT, rotation) - this is a blunt filter,
        not the real limit

LAYER 2  PER API KEY  -  fairness and tiering
  free:  100 req/min,  burst 20
  paid:  5,000 req/min, burst 500
  token bucket: burst matches real client behaviour
  purpose: one customer cannot degrade another

LAYER 3  PER ENDPOINT COST  -  protecting expensive work
  each endpoint has a token cost:
    GET /items          1
    POST /reports       50
    GET /search         10
  the key's bucket is charged the endpoint's cost
  purpose: 100 report generations != 100 item reads

LAYER 4  CONCURRENCY LIMIT  -  overload protection
  max 200 in-flight requests per instance
  sheds when the system is actually struggling, regardless
  of whether clients are within their quotas
  purpose: the thing rate limits CANNOT do

RESPONSES
  429 + Retry-After + X-RateLimit-Remaining / -Reset
  -> clients back off cooperatively instead of guessing
```

| Metric | Value | Note |
|---|---|---|
| Edge | per IP | abuse |
| Per key | token bucket | **fairness** |
| Per cost | weighted | expensive endpoints |
| Concurrency | in-flight cap | real overload |

> **Cost-weighted limits are the underused layer**  
> Counting requests treats a trivial read and an expensive report as equivalent, which means the limit must be set low enough for the worst case and is therefore needlessly restrictive for the common case. Charging each endpoint a token cost lets a client make thousands of cheap calls or a handful of expensive ones — which is both fairer and a more accurate proxy for the capacity they are actually consuming.

**When to use it**

- **Public and partner APIs**, where client behaviour is outside your control.
- **Multi-tenant systems**, to prevent one tenant degrading others.
- **Protecting expensive operations** — report generation, search, exports — with cost-weighted limits.
- **Abuse and scraping mitigation** at the edge, as a blunt first filter.
- **Alongside concurrency limits and load shedding**, which handle the overload case rate limits cannot.

**When to avoid it**

- **Do not use rate limits as overload protection**; they do not know the system's state.
- **Do not key solely on IP address**, which punishes shared networks and is trivially rotated.
- **Do not return 429 without `Retry-After`**, leaving clients to guess and synchronise.
- **Do not count all requests equally** when their costs differ by orders of magnitude.
- **Do not require exact global counting**, which puts a network round trip on every request.

**Advantages**

- **Fairness between clients**, so one participant cannot consume shared capacity.
- **Predictable cost and capacity planning**, since per-client consumption is bounded.
- **Token bucket maps onto product tiers naturally**, with rate and burst as separate dials.
- **Cheap** — a few numbers of state per identity for most algorithms.
- **Cooperative when implemented well**, with `Retry-After` and remaining-quota headers guiding client behaviour.

**Disadvantages**

- **Does not protect against overload**, because it is blind to system health.
- **Distributed enforcement requires approximation** or per-request coordination.
- **Legitimate bursts get rejected** if burst allowances are set too tightly.
- **Identity is hard**: IP is weak, API keys can be shared, and accounts can be created freely.
- **Cost weighting requires knowing endpoint costs**, which drift as implementations change.

**Trade-offs**

**Algorithm comparison**

| Algorithm | Burst | Memory | Accuracy | Best for |
|---|---|---|---|---|
| Fixed window | 2× at boundaries | One counter | Poor at boundaries | Simple internal limits |
| Sliding window log | None | One entry per request | Exact | Low-rate, high-value limits |
| Sliding window counter | Slight | Two counters | Good approximation | General request counting |
| Token bucket | Configurable | Two numbers | Exact for its model | Public APIs, tiered plans |
| Leaky bucket | None — smoothed | Queue | Exact output rate | Background work; traffic shaping |

> **Framing the design**  
> “Token bucket per API key, because rate and burst are separate dials that map onto our tiers, with endpoint costs weighting the charge so an expensive report is not equivalent to a cheap read. Distributed enforcement uses local counters with a few seconds of sync — slight over-admission is fine, a round trip per request is not. And separately, a concurrency limit for actual overload, because rate limits cannot see system health.”

**How it fails**

**Rate limiting failures**

| Failure | Cause | Fix |
|---|---|---|
| Double the limit at window boundaries | Fixed window counting | Sliding window counter or token bucket |
| Clients retry in synchronised waves | 429 with no `Retry-After`, or no jitter | Return `Retry-After`; clients jitter |
| Service collapses while everyone is within limits | Rate limits used as overload protection | Add concurrency limits and load shedding |
| Legitimate customers blocked | Shared NAT behind an IP-based limit | Authenticate and key on account |
| Expensive endpoint abused within limits | All requests counted equally | Cost-weighted token charges |
| Rate limiter becomes the bottleneck | Central store consulted per request | Local counters with periodic sync, or batched reservations |
| Limit ineffective across instances | Independent local limits with uneven load | Shared approximate state, or reservations |

> **The rate limiter as a single point of failure**  
> A limiter that consults a central store on every request has made that store a hard dependency of every request. If it is slow, every request is slow; if it is down, you must choose between failing everything and admitting everything. The design must state which — and failing open is usually correct for fairness limits, while failing closed may be right for abuse protection.

**Limits**

> **Design parameters**
>
> - **Token bucket**: capacity sets the burst, refill rate sets the sustained limit — configure them independently.
> - **Sliding window counter** approximation error is small and bounded; exact counting is rarely worth its cost.
> - **Distributed sync interval** of a few seconds gives slight over-admission and removes per-request coordination.
> - **`Retry-After`** should reflect actual reset time, and clients should add jitter on top.
> - **Concurrency limits** belong alongside rate limits, sized from Little's law rather than from client quotas.

**Alternatives**

| Mechanism | Protects against | Blind to |
|---|---|---|
| Rate limiting | One client consuming shared capacity | System health |
| Concurrency limiting | Actual overload | Per-client fairness |
| Load shedding | Overload, by dropping work | Which client is at fault |
| Queueing (leaky bucket) | Bursts, by smoothing | Adds latency |
| Quotas (per day/month) | Long-term consumption | Short bursts |
| Pricing | Economic over-consumption | Abuse and free tiers |

Quotas and rate limits answer different questions: a rate limit says how fast, a quota says how much in total. Most commercial APIs need both — a burst-tolerant rate limit for moment-to-moment fairness, and a monthly quota for commercial terms.

**In real systems**

- **GitHub's API** uses a points-based limit reflecting query cost, which is the only workable approach for GraphQL where request count is meaningless.
- **Stripe** returns `429` with clear rate-limit headers and documents the expected client backoff behaviour explicitly.
- **Envoy and service meshes** implement both local token buckets and a global rate-limit service, letting operators choose the coordination trade.
- **Cloudflare and other edge providers** apply blunt IP-based limits as a first filter, explicitly framed as abuse mitigation rather than fairness.
- **Netflix's concurrency-limits library** exists precisely because rate limits cannot detect overload — it adapts admission from observed latency instead.

**Common mistakes**

- **Fixed windows**, permitting double the limit across a boundary.
- **IP-based limits only**, punishing shared networks and trivially evaded.
- **No `Retry-After`**, causing synchronised retry waves.
- **Counting all requests equally**, forcing limits down to the most expensive endpoint.
- **Expecting rate limits to prevent overload**, which they cannot detect.
- **Per-request central coordination**, making the limiter a bottleneck and a hard dependency.
- **Undefined behaviour when the limiter itself fails.**

**The staff-level view**

The most common mistake with rate limiting is asking it to do a job it structurally cannot: protect the system from overload.

- **Separate fairness from protection explicitly.** Rate limits enforce per-client fairness; concurrency limits and shedding handle overload. A system with only the first will collapse with everyone inside their quotas.
- **Weight by cost, not request count**, so limits can be generous for cheap calls without exposing expensive ones.
- **Accept approximation in distributed enforcement.** A limit is a policy, not an invariant, and per-request coordination makes the limiter a bottleneck and a dependency.
- **Decide and document the fail-open or fail-closed behaviour** of the limiter itself, because it will fail and the choice differs between fairness limits and abuse protection.
- **Return `Retry-After` and remaining-quota headers as a standard.** Cooperative clients are cheaper than adversarial ones, and most clients want to cooperate if told how.

**Go deeper**

Rate limiting makes capacity a per-client resource so one participant cannot degrade everyone else. The algorithms differ in burst behaviour and state cost: fixed windows are cheap but permit double the limit across a boundary; sliding window logs are exact but store a timestamp per request; sliding window counters approximate well with two counters; and token buckets, the usual default, give independently configurable sustained rate and burst size for just two numbers of state.

Three design choices matter as much as the algorithm. Key on an authenticated account rather than IP, since shared NATs punish innocents and attackers rotate addresses. Charge by cost rather than request count, so an expensive report is not equivalent to a cheap read — otherwise the limit must be set low enough for the worst endpoint. And always return `Retry-After` with remaining-quota headers, or clients guess and their guesses synchronise into another spike.

The structural mistake is expecting rate limits to protect against overload. They encode assumptions about normal conditions and cannot see system health, so during a dependency slowdown every client stays within quota while the service collapses. Overload needs concurrency limits and load shedding driven by observed latency and in-flight work. Distributed enforcement should be deliberately approximate — local counters with periodic sync — because a per-request round trip makes the limiter a bottleneck and a hard dependency.

Rate limiting is primarily a fairness mechanism and only incidentally a protection mechanism, and conflating the two produces limits that are simultaneously too restrictive for users and too permissive for the system.

**The algorithms and what distinguishes them.** Fixed windows count per calendar interval — one counter, trivially cheap, and vulnerable at boundaries where a client can spend a full allowance at the end of one window and again at the start of the next. Sliding window logs store a timestamp per request and are exact, but memory scales with request rate, which makes them suitable only for low-volume, high-value limits. Sliding window counters weight the previous window's count by position in the current one, giving a good approximation for two counters. Token buckets refill at a rate up to a capacity, which separates sustained throughput from burst size — two dials that map directly onto commercial tiers, for two numbers of state. Leaky buckets queue and drain at a fixed rate, smoothing output entirely at the cost of latency, which suits background work rather than interactive requests.

**Identity, cost and cooperation.** Keying on IP is a blunt filter: shared NATs mean punishing innocent users, and rotation makes evasion trivial, so it belongs at the edge as abuse mitigation rather than as the real limit. Authenticated account or API key is the meaningful identity. Equally important is charging by cost rather than by request, since an endpoint scanning a million rows and a health check are not equivalent — counting them identically forces the limit down to the worst case and needlessly restricts everything else. And responses must carry `Retry-After` and remaining-quota headers, because without guidance clients guess, and independent guesses synchronise into the next spike.

**Distribution requires deliberate approximation.** Exact global counting means a round trip to a shared store on every request, which puts that store in the critical path of everything and makes it both a bottleneck and a hard dependency. Static division of the limit among instances avoids coordination but under-admits when load is uneven. The workable approaches are local counters with a few seconds of synchronisation, or batched reservations claiming a block of tokens at a time. Both over-admit slightly during the interval, which is fine — a rate limit is a policy rather than an invariant, and admitting a few percent over harms nobody while a slow limiter harms everyone.

**Why it cannot protect capacity.** A per-client limit encodes assumptions about normal conditions and has no visibility into current system state. When a downstream dependency slows, latency rises and queues build while every client remains comfortably within quota — the limiter sees nothing wrong because nothing about client behaviour changed. Protection requires mechanisms that measure the system: concurrency limits sized from in-flight work, and load shedding triggered by observed latency or queue depth. Organisations that rely on rate limits for protection typically respond to incidents by lowering the limits, which restricts legitimate users without protecting anything — the worst of both outcomes.

**The limiter's own failure mode.** Whatever the design, the limiter will sometimes be unavailable, and the behaviour must be chosen in advance rather than defaulted. Failing open is usually correct for fairness limits, since admitting everyone briefly is better than rejecting everyone. Failing closed may be right for abuse protection, where the limit is the only thing standing between an attacker and expensive work. Leaving it undecided means the choice is made by an implementation detail and discovered during an incident.

**Prove it — interview questions**

1. **[Basic] What is the boundary problem with fixed-window rate limiting?**

   <details><summary>Model answer</summary>

   The counter resets at each window boundary, so a client can consume its full allowance at the very end of one window and again at the start of the next — effectively double the intended rate in a very short period. A limit of a hundred per minute permits two hundred requests across two seconds spanning the boundary. Sliding window counters and token buckets both avoid this by not having a hard reset point.

   </details>

2. **[Basic] Why is token bucket a good default?**

   <details><summary>Model answer</summary>

   Because its two parameters map directly onto product requirements: the refill rate is the sustained throughput a client may use, and the bucket capacity is how large a burst they may make after being idle. That matches how clients actually behave — mostly quiet with occasional bursts — far better than a flat window, and it costs only a token count and a timestamp per identity. It also makes tiering natural, since a paid plan is simply a larger bucket refilling faster.

   </details>

3. **[Senior] Why can't rate limits protect against overload?**

   <details><summary>Model answer</summary>

   Because they are calibrated against client behaviour rather than system state. If every client stays within its quota but a downstream dependency slows down, the system can collapse while the limiter sees nothing wrong — the limits were set for normal conditions and have no visibility into current latency or queue depth. Overload protection needs mechanisms that measure the system: concurrency limits sized from in-flight work, and load shedding triggered by observed latency. Rate limits and concurrency limits solve different problems and a system needs both.

   </details>

4. **[Senior] How do you enforce a global limit across many instances?**

   <details><summary>Model answer</summary>

   By accepting approximation. Consulting a central store on every request is accurate but puts a network round trip in every request path and makes the store a hard dependency and a bottleneck. Dividing the limit statically among instances avoids coordination but under-admits when load is uneven. The practical answer is local counters synchronised every few seconds, or batched reservations where an instance claims a block of tokens at a time — both trade slight over-admission during the sync interval for no per-request coordination. A rate limit is a policy rather than an invariant, so admitting a few percent over is harmless; making the limiter a bottleneck is not.

   </details>

5. **[Staff] Design rate limiting for a public API with tiers.**

   <details><summary>Model answer</summary>

   Layered, because each layer protects something different. At the edge, a blunt per-IP limit as abuse mitigation, understood to be weak since NATs and rotation defeat it — it exists to stop unauthenticated floods reaching anything expensive. Then a per-API-key token bucket for fairness, with rate and burst set per tier, which is where the commercial policy lives. Then cost weighting, charging each endpoint a token cost so an expensive report is not equivalent to a cheap read — without this the limit has to be set low enough for the worst endpoint and is needlessly restrictive for everything else. And separately a concurrency limit per instance for actual overload, because none of the above can detect that the system is struggling. Responses return `429` with `Retry-After` and remaining-quota headers so clients back off cooperatively rather than guessing and synchronising.

   </details>

6. **[Principal] What is the most common structural mistake organisations make with rate limiting?**

   <details><summary>Model answer</summary>

   Treating it as the answer to capacity protection, which leaves them with no defence when the system degrades for reasons unrelated to client behaviour. A dependency slows, latency climbs, queues build, and every client is comfortably within quota while the service falls over — because the limits encode assumptions about normal conditions and have no feedback from the system's actual state. The fix is architectural rather than a tuning change: rate limits for fairness, and adaptive concurrency limiting plus load shedding for protection, with the second sized from in-flight work rather than from client quotas. The second-most-common mistake is closely related and follows from the first: setting rate limits low in the hope they will protect capacity, which makes them restrictive for legitimate users while still failing to protect anything — the worst of both outcomes. I would put both mechanisms in the platform's default request path so teams inherit them, because each one alone produces a predictable and different class of incident.

   </details>

---

### API versioning and compatibility

*You cannot deploy your clients, so every change must be compatible — and versioning is the expensive escape hatch for the changes that cannot be.*

**Flow:** `Existing contract` → `Additive change` → `Compatibility tests` → `Client migration` → `Retirement`

> **The 30-second version**  
> You deploy the server and cannot deploy your clients, so almost every change must be additive. Expand-contract handles the rest; versions are the expensive exception, not the routine.

**The problem**

An API is a contract with software you do not control and cannot redeploy. A mobile app version from eighteen months ago is still installed; a partner integration was written by a team that has since dissolved; a script somewhere depends on a field nobody remembers adding. Any change that breaks them is an outage you caused for someone else.

The instinct is to version everything and move on — `/v2`, then `/v3`. But every version you publish is a version you must operate, test and eventually retire, and retirement requires persuading clients to migrate, which is a coordination problem rather than an engineering one.

> **Versioning is a failure to be compatible**  
> A new major version is not a feature; it is the cost of a change that could not be made compatibly. The goal is to make almost every change additively, so versions are rare events rather than a release cadence. Organisations that version routinely end up operating many versions simultaneously and retiring none.

**Mental model**

Think in terms of what each side may assume. Clients assume fields they read will be present with the same meaning; servers assume fields they require will be sent. A change is compatible if it does not violate an assumption either side was entitled to make.

1. **Backward compatible** — Existing clients keep working against the new server. This is the one that matters, because you deploy first.
2. **Additive change** — New optional fields, new endpoints, new enum values with a documented default. Free if clients were written tolerantly.
3. **Breaking change** — Removing or renaming a field, changing a type, making an optional field required, changing semantics. Requires a version or a migration.
4. **Tolerant reader** — A client that ignores unknown fields and does not fail on additions. This is what makes additive evolution possible at all.
5. **Deprecation** — A stated intention to remove, with a date, a migration path, and measurement of who still depends on it.

> **The most dangerous change is the one that still parses**  
> Renaming a field produces an obvious error. Changing `amount` from pounds to pence, redefining `status: pending` to mean something new, or altering a default keeps every client parsing successfully while producing wrong results. No schema check detects it and no test catches it, because the shape is unchanged. Semantic changes are breaking changes and require a new field.

**How it works**

**Compatible and incompatible changes**

```text
SAFE (backward compatible)
  add a new optional response field
  add a new optional request parameter with a default
  add a new endpoint
  add a new enum value       <- ONLY if clients have a
                                 documented unknown branch
  relax a validation rule    (accept more than before)
  make a required request field optional

BREAKING
  remove or rename a field
  change a field's type
  make an optional request field required
  tighten validation         (reject what was accepted)
  change a default value
  change units or semantics  <- parses fine; corrupts silently
  change error codes clients branch on
  change pagination or ordering behaviour

THE ASYMMETRY
  RESPONSE fields: adding is safe, removing breaks clients
  REQUEST fields:  adding optional is safe, requiring breaks
  -> the direction of safety is opposite on each side
```

1. **Deploy the server first, always** — Since you cannot deploy clients, every change must work with old clients before new ones exist. That constraint alone rules out most breaking changes.
2. **Write clients as tolerant readers** — Ignore unknown fields, handle unknown enum values with a default branch, do not fail on additions. A generation of clients that reject unknown fields makes all future evolution breaking.
3. **Remove in stages, never in place** — Deprecate with a date, measure usage, notify the remaining consumers, then remove. Each step is independently deployable and reversible.
4. **Measure who actually uses a field** — The blocker on retirement is almost never the mechanism; it is not knowing who depends on what. Per-field usage telemetry converts a guess into a decision.
5. **Prefer expanding the contract to versioning it** — A new optional field alongside the old one, both populated during a transition, lets clients migrate independently with no version at all.
6. **When you must version, version the resource, not the whole API** — A breaking change to one endpoint does not justify forcing every client to migrate every integration.

**Versioning strategies and their real costs**

```text
URL PATH        /v1/orders, /v2/orders
  + explicit, cacheable, obvious in logs and dashboards
  - the version leaks into every URL; "v2" often means
    "v2 of everything" even for unchanged resources

HEADER          Accept: application/vnd.api.v2+json
  + URLs stay stable; per-resource versioning is natural
  - invisible in logs and caches unless you add it to the
    cache key; easy for clients to omit by accident

QUERY PARAM     /orders?version=2
  + simple
  - pollutes caching; easy to forget

DATE-BASED      Stripe-Version: 2024-06-01
  + every client pins a date; the server holds a chain of
    transformations from each old version to current
  + new clients get current behaviour by default
  - you maintain every transformation forever

NO VERSION, ADDITIVE ONLY
  + no versions to operate or retire
  - requires discipline and tolerant clients
  -> this is the target state; versions are the exception
```

> **The date-pinned model is powerful and expensive**  
> Pinning each client to the API as it behaved on a date, with the server applying a chain of transformations from that date to current, gives clients complete stability and lets the team change the underlying API freely. The cost is that every transformation must be maintained and tested forever, so the internal complexity grows monotonically. It suits organisations with many long-lived integrations and the engineering capacity to carry that weight.

**Worked example**

Changing a money field from a single amount to a currency-aware pair, without breaking anyone.

**Expand, migrate, contract — over four releases**

```text
CURRENT
  { "id": 1, "totalPence": 4999 }
  every client assumes GBP

NAIVE (breaking, and worse: silent)
  { "id": 1, "totalPence": 4999, "currency": "EUR" }
  -> old clients still read totalPence as pounds
  -> revenue reporting silently wrong for non-GBP orders
  -> parses perfectly; no error anywhere

STAGE 1  EXPAND
  { "id": 1,
    "totalPence": 4999,              <- unchanged, still GBP only
    "amount": { "value": 4999, "currency": "GBP" } }
  old clients unaffected; new clients use "amount"
  non-GBP orders are NOT yet exposed to old clients
    -> they are filtered, or return an error for old clients,
       rather than being misrepresented

STAGE 2  MIGRATE
  deprecate totalPence with a date in the docs and a
  response header
  measure per-client usage of the field
  contact the remaining consumers directly

STAGE 3  VERIFY
  usage telemetry shows zero reads of totalPence for 30 days
  (this is the step teams skip, and why fields never get removed)

STAGE 4  CONTRACT
  remove totalPence

NO VERSION WAS NEEDED. The change was made compatibly
by expanding first and contracting later.
```

| Metric | Value | Note |
|---|---|---|
| Releases | 4 | each reversible |
| Version bumps | 0 | **expand-contract** |
| Blocker | usage telemetry | not mechanism |
| Silent risk | avoided | no reused field |

> **The hard part of removal is knowing who is affected**  
> Teams carry deprecated fields for years not because removal is technically difficult but because nobody can prove the field is unused, and the cost of being wrong is breaking a customer. Per-field, per-client usage telemetry turns that from an unanswerable question into a report — and it is the single highest-leverage investment in API evolution, far more than the choice of versioning scheme.

**When to use it**

- **Additive evolution** for the overwhelming majority of changes — new fields, new endpoints, relaxed validation.
- **Expand-contract migration** for changes to existing fields, avoiding a version entirely.
- **Explicit versioning** only when a change genuinely cannot be made compatibly.
- **Date-based pinning** where there are many long-lived external integrations and the capacity to maintain transformations.
- **Per-resource versioning** rather than whole-API, so one breaking change does not force every client to migrate everything.

**When to avoid it**

- **Do not version routinely.** Every version is one you must operate, test and retire.
- **Do not change a field's meaning or units** while keeping its name — it parses and corrupts silently.
- **Do not add enum values** unless clients are documented to handle unknown ones.
- **Do not deprecate without measuring usage**, or the field will never be removed.
- **Do not assume clients are tolerant readers** unless you wrote them or documented the requirement.

**Advantages**

- **Additive evolution costs nothing** and requires no client action, so APIs can grow indefinitely.
- **Expand-contract removes the need for versions** in most cases, including changes to existing fields.
- **Versioning provides a genuine escape hatch** for the rare change that cannot be made compatibly.
- **Date pinning gives clients total stability** while letting the team evolve freely.
- **Usage telemetry turns retirement into a decision** rather than a permanent unknown.

**Disadvantages**

- **Fields accumulate**, because removal requires a multi-release process and proof of non-use.
- **Every published version is permanent operational load** until it is retired, and retirement requires client cooperation.
- **Semantic changes evade all mechanical checks**, so discipline is the only defence.
- **Date pinning grows internal complexity monotonically**, since transformations are never deleted.
- **Client tolerance cannot be assumed** for third parties, which constrains what counts as additive.

**Trade-offs**

**Versioning strategies**

| Strategy | Client burden | Server burden | Visibility |
|---|---|---|---|
| Additive only | None | Discipline; accumulating fields | n/a |
| URL path version | Migrate whole API | Operate N versions | Excellent — visible in logs |
| Header version | Set a header | Operate N versions | Poor unless instrumented |
| Per-resource version | Migrate one endpoint | Operate N versions per resource | Good |
| Date pinning | Pin once, migrate when ready | Maintain all transformations forever | Good |

> **A grounded framing**  
> “I'd make this change with expand-contract rather than a version: add the new field alongside the old, deprecate the old with a date, measure per-client usage until it reaches zero, then remove it. That is four reversible releases and no version to operate — and the blocker is usage telemetry, not the mechanism, which is why I'd want that in place first.”

**How it fails**

**Versioning and compatibility failures**

| Failure | Cause | Fix |
|---|---|---|
| Old clients break after a deploy | Field removed, renamed, or type changed | Expand-contract; never modify in place |
| Silent data corruption downstream | Units or semantics changed on an existing field | New field; treat semantic change as breaking |
| Client crashes on a new enum value | Exhaustive branching with no default | Document unknown handling; treat enums as open |
| Deprecated field can never be removed | No usage measurement | Per-field, per-client telemetry |
| Six versions in production | Versioning used as the default response to change | Additive evolution; per-resource versioning |
| Client breaks on an added field | Strict schema validation rejecting unknowns | Document tolerant-reader expectations; generate clients accordingly |
| Tightened validation rejects existing traffic | Stricter rule applied without a transition | Warn first, measure, then enforce |

**Limits**

> **Practical guidance**
>
> - **Deploy order**: the server always goes first, so backward compatibility is the constraint that matters.
> - **Removal takes three to four releases**: expand, deprecate with a date, verify zero usage, contract.
> - **Verification window**: 30 days of zero usage is a common bar before removal.
> - **Enum additions** are only safe if clients are documented to handle unknown values.
> - **Every live version has a cost** in testing, operating and supporting — count them and set a maximum.

**Alternatives**

| Approach | When | Trade |
|---|---|---|
| Additive only | Almost always | Fields accumulate |
| Expand-contract | Changing an existing field | Multi-release process |
| New endpoint | Substantially different behaviour | Two endpoints to maintain |
| Major version | Genuinely incompatible redesign | Operate and retire N versions |
| Date pinning | Many long-lived integrations | Transformations maintained forever |
| Feature flag per client | Gradual behaviour change | Combinatorial testing surface |

Introducing a new endpoint is often a better answer than a version: `POST /v2/orders` forces every client to migrate everything, while `POST /orders/bulk` or a differently named resource lets old and new coexist naturally with no version concept at all.

**In real systems**

- **Stripe's date-based versioning** pins each account to the API as of a date, with the server transforming responses — widely admired and genuinely expensive to maintain.
- **Google's API design guide** mandates backward compatibility and treats major versions as rare, with explicit rules on what constitutes a breaking change.
- **Protocol Buffers' field numbers** make additive evolution safe by construction, which is why gRPC APIs evolve more smoothly than ad-hoc JSON ones.
- **Kubernetes API groups** version per resource rather than globally, with a documented promotion path from alpha to beta to stable.
- **Mobile app clients** are the hardest constraint in practice, since old versions remain installed for years and cannot be forced to update.

**Common mistakes**

- **Versioning as the default response to change**, producing many live versions and no retirements.
- **Changing units or meanings** on an existing field, which parses and corrupts silently.
- **Removing a field in place**, breaking clients you cannot redeploy.
- **Deprecating with no usage measurement**, so removal never happens.
- **Adding enum values** where clients branch exhaustively.
- **Tightening validation without a warning period**, rejecting traffic that previously worked.
- **Assuming clients ignore unknown fields** when nothing required them to.

**The staff-level view**

The discipline that matters is treating a new version as an admission of failure rather than a routine step, and investing in the telemetry that makes removal possible.

- **Establish expand-contract as the default change pattern**, so versions become rare. Most changes teams believe require a version do not.
- **Invest in per-field, per-client usage telemetry.** The blocker on retiring anything is never mechanism, it is not knowing who depends on it — and this single capability unlocks years of accumulated deprecations.
- **Define breaking changes explicitly, including semantic ones.** Units, meanings and enum sets are part of the contract even though no tool checks them.
- **Cap the number of live versions** and make retiring one a precondition for publishing another, or version count only grows.
- **Publish tolerant-reader expectations** and generate clients that honour them, since a generation of strict clients makes all future evolution breaking.

**Go deeper**

An API is a contract with software you cannot redeploy — old mobile versions, partner integrations, forgotten scripts. Since the server always deploys first, backward compatibility is the constraint that matters: existing clients must keep working. Adding optional fields, new endpoints and relaxed validation is free if clients ignore unknown fields; removing, renaming, retyping or requiring anything is not.

Most changes teams believe need a version can be made with expand-contract: add the new shape alongside the old, run both, migrate clients, verify nobody reads the old one, then remove it. Each stage is independently deployable and reversible, and no version is published — which matters because every version is one you must operate, test and eventually retire, and retirement needs cooperation from clients you do not control.

Two things deserve particular attention. Semantic changes — altering units, redefining a status value, changing a default — pass every mechanical check while corrupting data silently, so they must be treated as breaking and handled with a new field. And the real blocker on removing deprecated fields is never mechanism but visibility: teams carry them for years because nobody can prove they are unused. Per-field, per-client usage telemetry is the highest-leverage investment in API evolution.

API evolution is constrained by a single asymmetry: you deploy your server and you cannot deploy your clients. Every rule in this area follows from that.

**What is safe, and the direction of safety.** On responses, adding fields is safe and removing them breaks clients. On requests, adding optional parameters is safe and making them required breaks clients. The directions are opposite, which is a frequent source of confusion. Relaxing validation is safe; tightening it rejects traffic that previously worked. Adding an enum value is safe only if clients are documented to handle unknown values — otherwise a legitimate new status crashes an exhaustive switch, which makes enums one of the most commonly underestimated breaking changes.

**Expand-contract removes most versions.** Rather than modifying a field, add the new representation alongside it, populate both, deprecate the old one with a date, verify nobody reads it, and only then remove it. Every stage is independently deployable and reversible, and existing clients are unaffected throughout. This handles the majority of changes teams assume require a version — renaming, restructuring, changing representation — which is why a major version should signal that a change genuinely could not be made compatibly rather than being a routine release step.

**Semantic change is the silent category.** Renaming a field errors visibly. Changing `amount` from pounds to pence, redefining what `pending` means, or altering a default value leaves every client parsing successfully and producing wrong results, typically discovered much later in a report that does not reconcile. No schema validator, contract test or compatibility checker can detect it, because nothing structural changed. The only defence is an explicit rule that units, meanings, valid value sets and error codes clients branch on are part of the contract, and changing any of them requires a new field rather than a redefinition.

**The real blocker on removal is visibility.** Teams carry deprecated fields for years, not because deletion is hard but because they cannot prove nobody depends on them, and the cost of being wrong is breaking a customer. So the safe choice is always to wait, indefinitely. Per-field, per-client usage telemetry converts this into a report: thirty days of zero reads makes removal a decision rather than a gamble. This capability unlocks every deprecation an organisation has already accumulated and is worth more than any choice of versioning scheme.

**When you must version, version narrowly.** A path version implies the whole API changed and forces every client to migrate every integration, even for resources that did not change. Per-resource versioning limits the blast radius. Date pinning — each client pinned to the API as of a date, with the server transforming between versions — gives clients complete stability and the team full freedom, at the cost of maintaining every transformation forever, which makes internal complexity grow monotonically. It suits organisations with many long-lived external integrations and the capacity to carry that weight, and is a poor fit otherwise.

**Organisational forcing functions matter.** Without one, version count only increases: publishing is easy and retiring requires other people's cooperation. Capping live versions and making the retirement of one a precondition for publishing another creates the pressure that discipline alone does not. Similarly, tolerant-reader expectations must be published and honoured by generated clients, because a generation of strict clients that reject unknown fields makes every future change breaking — turning an organisation's own client library into the constraint on its API evolution.

**Prove it — interview questions**

1. **[Basic] What makes a change to an API backward compatible?**

   <details><summary>Model answer</summary>

   That existing clients keep working against the new server. Since you deploy the server and cannot deploy your clients, that is the direction that matters. Adding an optional response field or a new endpoint is safe if clients ignore unknown fields; adding an optional request parameter with a default is safe. Removing or renaming a field, changing its type, or requiring a previously optional parameter all break clients that were entitled to assume otherwise.

   </details>

2. **[Basic] Why is versioning something to avoid rather than adopt?**

   <details><summary>Model answer</summary>

   Because every version you publish is one you must implement, test, operate, document and eventually retire — and retirement requires persuading clients you do not control to migrate, which is a coordination problem rather than an engineering one. Organisations that version routinely accumulate many live versions and retire none. A new major version should be the cost of a change that genuinely could not be made compatibly, not a normal step in a release process.

   </details>

3. **[Senior] What is expand-contract and why does it usually avoid a version?**

   <details><summary>Model answer</summary>

   You add the new shape alongside the old rather than replacing it, run both for a period, migrate clients, verify nobody uses the old one, and then remove it. Because the old field is never modified or removed while anyone depends on it, existing clients are unaffected at every stage, and each stage is an independently deployable and reversible release. Most changes teams believe require a version — renaming a field, restructuring a value, changing a representation — can be made this way, which is why versions should be rare.

   </details>

4. **[Senior] Why are semantic changes more dangerous than structural ones?**

   <details><summary>Model answer</summary>

   Because they still parse. Renaming a field produces an obvious error that is caught immediately. Changing `amount` from pounds to pence, or redefining what a status value means, or altering a default keeps every client working and producing wrong results — silently, and often discovered weeks later in a reconciliation that does not balance. No schema validation, contract test or compatibility checker detects it, because the shape is unchanged. The only defence is a stated rule that units, meanings and valid value sets are part of the contract, and changing them requires a new field.

   </details>

5. **[Staff] What actually prevents teams from removing deprecated fields?**

   <details><summary>Model answer</summary>

   Not the mechanism — it is not knowing who depends on them. A team can deprecate a field, document it, and announce a date, but removing it means risking breaking a customer whose integration nobody has visibility into, and the downside of being wrong is severe enough that the safe choice is always to wait. So fields accumulate for years. Per-field, per-client usage telemetry converts that from an unanswerable question into a report: if no client has read this field in thirty days, removal is a decision rather than a gamble. I would treat that telemetry as the highest-leverage investment in API evolution, well above the choice of versioning scheme, because it unlocks every deprecation the organisation has already accumulated.

   </details>

6. **[Principal] How would you set API evolution policy across an organisation?**

   <details><summary>Model answer</summary>

   Three rules, chosen because each prevents a failure that is expensive and hard to reverse. First, expand-contract is the default and a major version requires justification — specifically, an explanation of why the change cannot be made additively — because the alternative is a version proliferation that nobody ever unwinds. Second, breaking changes are defined explicitly to include semantic ones: units, meanings, enum sets and error codes clients branch on, since those evade every automated check and are the ones that corrupt data rather than merely erroring. Third, per-field usage telemetry is a platform capability rather than a per-team project, because it is what makes deprecation actionable and no individual team will build it for themselves. Alongside those I would cap the number of live versions per API and make retiring one a precondition for publishing another — without a forcing function, version count only ever increases, and the operational and testing burden compounds silently until someone notices the team is maintaining six behaviours for one product.

   </details>

---

### Authentication and authorization boundaries

*Establish who the caller is once, at the edge, then decide what they may do close to the data — because only the resource owner knows its access rules.*

**Flow:** `Credential` → `Verified identity` → `Resource lookup` → `Policy decision` → `Authorized action`

> **The 30-second version**  
> Verify who the caller is once at the edge; decide what they may do at each resource, because only the owning service knows the ownership rules.

**The problem**

Authentication asks who you are; authorisation asks what you may do. Conflating them produces systems where the gateway checks a token and every downstream service assumes the request is fully permitted — so any service that can be reached internally can act as anyone.

Placing both at the edge fails for a structural reason: the gateway does not know that document 4711 belongs to a team the caller left last month, or that this account is suspended, or that the field they requested is restricted. Those facts live with the data, and only the service owning the data can evaluate them.

> **Authenticate once, authorise everywhere**  
> Identity is expensive to establish — verifying signatures, checking sessions, contacting an identity provider — and it does not change during a request, so establish it once at the edge and propagate it. Authorisation depends on the specific resource being touched, so it must be evaluated at each point where a resource is accessed. The two have opposite placement rules and are constantly conflated.

**Mental model**

The edge turns a credential into a verified identity. That identity travels inward as a trustworthy assertion. Each service then combines the identity with the specific resource and its own policy to decide whether the action is permitted.

1. **Credential** — A password, token, certificate or key. Verified once; never propagated inward as-is.
2. **Verified identity** — Who the caller is, plus context — account, roles, scopes, tenant, device. Propagated as a signed assertion.
3. **Resource context** — What is being accessed: its owner, its tenant, its classification. Known only to the owning service.
4. **Policy decision** — Identity plus resource plus action, evaluated against rules. The answer is permit or deny.
5. **Audit** — The record of who did what to which resource, which requires all three pieces to be present at the decision point.

> **Edge-only authorisation is a lateral-movement vulnerability**  
> If services trust any internal call because “the gateway already checked”, then compromising one low-value service grants the ability to call every other service as any user. The blast radius of any single vulnerability becomes the entire system. Internal calls need authenticated identity — service identity and propagated user identity — and authorisation decisions of their own.

**How it works**

**Where each decision belongs**

```text
EDGE (once)
  verify the credential
    - validate token signature and expiry
    - check session, or contact the identity provider
  establish identity: user id, account, roles, scopes
  COARSE authorisation only:
    - is this token allowed to call this API at all?
    - does it have the required scope for this endpoint class?
  propagate identity inward as a SIGNED assertion

SERVICE (every resource access)
  receive propagated identity (verified, not just trusted)
  load the resource
  FINE authorisation:
    - does this user own / belong to the tenant of / have a
      role granting access to THIS resource?
    - are they permitted THIS action on it?
    - is any field restricted for this caller?
  emit an audit record

WHY THE SPLIT
  the edge cannot know resource ownership without loading
  every resource, which is the service's job anyway
  the service cannot cheaply verify credentials on every
  call, and should not need the raw credential at all
```

1. **Never propagate the raw credential inward** — A downstream service holding a user's bearer token can impersonate them anywhere, forever, to any system that accepts it. Propagate a short-lived signed assertion scoped to this request instead.
2. **Authenticate service-to-service calls too** — mTLS or signed service tokens, so a compromised service cannot simply call others. Network position must not be an authorisation grant.
3. **Make the authorisation decision at the data, not before it** — The check must happen after the resource is loaded, because ownership and classification are properties of the resource. Checking beforehand means checking against assumptions.
4. **Fail closed** — An unreachable policy service, a missing identity, or an unrecognised resource type must deny. Defaulting to permit on error turns a policy outage into a total exposure.
5. **Separate the policy decision from the enforcement point** — A policy engine evaluating rules, with enforcement inline in each service, means rules are consistent and auditable rather than reimplemented per service.
6. **Include the resource in the audit record** — “User X performed action Y” is not auditable. “User X performed action Y on resource Z at time T, permitted by rule R” is.

**Authorisation models, and where each fits**

```text
RBAC  role-based
  user -> roles -> permissions
  simple, familiar, coarse
  breaks down when permissions depend on the RESOURCE:
  "editor" of WHAT? -> role explosion (editor_team_17)

ABAC  attribute-based
  decide from attributes of user, resource, action, context
  rule: allow if user.dept == resource.dept AND time < 18:00
  flexible; hard to answer "who can access X?" (no reverse index)

ReBAC  relationship-based
  permission derives from a graph of relationships
  "user is a member of a team that owns the folder
   containing this document"
  matches how people actually think about access
  needs graph traversal; reachability must be fast

PRACTICAL SHAPE
  RBAC for coarse capability (can this user use the admin API)
  ReBAC or ABAC for resource-level decisions
  both evaluated at the service, not the gateway
```

> **Authorisation checks are hard to test and easy to omit**  
> Missing an authorisation check produces no error, no test failure and no log entry — the request simply succeeds when it should not. The defences are structural: make the data-access layer require an identity and a permission, so an unchecked query cannot compile or cannot run, rather than relying on every developer to remember every check on every path.

**Worked example**

A document platform: where each check happens and why.

**Request path with decisions at each boundary**

```text
REQUEST  GET /documents/4711     Authorization: Bearer <jwt>

EDGE GATEWAY
  verify JWT signature and expiry          -> AUTHENTICATION
  check the token has scope "documents:read"
                                           -> COARSE AUTHZ
  reject if the account is suspended
  mint a short-lived internal assertion:
    { sub: user_88, account: acct_12, scopes: [...],
      req: <request id>, exp: now+30s }
  forward inward over mTLS

DOCUMENT SERVICE
  verify the internal assertion signature  -> do NOT just trust
  load document 4711
  evaluate: does user_88 have read access to THIS document?
    - direct grant?
    - member of a team with access?
    - inherited from the parent folder?        -> FINE AUTHZ
  if the document is in another account -> DENY (404, not 403,
    to avoid leaking existence)
  filter restricted fields for this caller
  emit audit: user_88 read doc 4711 via team_5 membership

STORAGE LAYER
  the document service is authenticated as a service
  storage does not re-evaluate user permissions - it trusts
  the document service as the owner of that policy
  -> the boundary of policy ownership is explicit

WHAT WOULD GO WRONG WITHOUT THE SERVICE-LEVEL CHECK
  any internal caller reaching the document service could
  read any document, because the gateway cannot know
  ownership without loading the document itself.
```

| Metric | Value | Note |
|---|---|---|
| Edge | identity + scope | once |
| Service | resource access | **every call** |
| Assertion | 30 s, signed | not the raw token |
| Not found | 404 not 403 | avoid leaking existence |

> **Return 404 rather than 403 for resources the caller may not see**  
> A 403 confirms the resource exists, which leaks information — an attacker can enumerate valid identifiers by distinguishing 403 from 404. Where existence itself is sensitive, denying with a not-found response is the correct choice. Where it is not sensitive, 403 gives better developer experience. This should be a deliberate decision per resource type rather than an accident of implementation.

**When to use it**

- **Every system with more than one user**, which is effectively all of them.
- **Multi-tenant systems**, where tenant isolation is the most consequential authorisation boundary.
- **Microservice architectures**, where internal calls must not be implicitly trusted.
- **Regulated domains**, where the audit record must show who accessed which resource under which rule.
- **Anywhere resources have owners**, since ownership can only be evaluated where the resource lives.

**When to avoid it**

- **Do not authorise only at the edge**, which cannot know resource ownership and leaves internal calls unchecked.
- **Do not propagate raw user credentials inward**, which lets any downstream service impersonate the user anywhere.
- **Do not treat network position as authorisation** — being inside the perimeter is not a permission.
- **Do not fail open** when the policy service is unavailable.
- **Do not scatter authorisation logic across services** without a shared policy definition, or rules will diverge silently.

**Advantages**

- **Expensive credential verification happens once**, while cheap policy decisions happen where the context exists.
- **Lateral movement is contained**, because each service enforces its own rules on every call.
- **Auditability is real**, since the decision point has identity, resource and rule together.
- **Policy can evolve independently** of services when a shared engine evaluates it.
- **Field-level and row-level decisions become possible**, which an edge check cannot express.

**Disadvantages**

- **Every service must implement enforcement**, which is more work and more places to omit a check.
- **Distributed policy is harder to reason about** — answering “who can access X?” requires querying the owning service.
- **A policy service becomes a hot dependency**, needing caching with a bounded staleness that conflicts with prompt revocation.
- **Identity propagation adds plumbing** and a signing infrastructure to operate.
- **Missing checks fail silently**, succeeding when they should deny.

**Trade-offs**

**Where to enforce**

| Placement | Knows the resource | Cost per call | Risk |
|---|---|---|---|
| Edge only | No | Once | Internal calls unchecked; no resource-level rules |
| Service only | Yes | Per service | Expensive credential verification repeated |
| Edge authn + service authz | Yes | Once + cheap checks | Requires identity propagation |
| Data layer enforcement (RLS) | Yes | Per query | Policy in the database; harder to reason about |
| Sidecar policy enforcement | Yes, via context | Per call | Consistent; another component |

Row-level security in the database is the strongest structural defence, because an unchecked query is impossible rather than merely discouraged — the policy is enforced by the storage engine for every writer including migrations and admin tools. Its cost is that policy lives in the database and is harder to version and reason about than application code.

**How it fails**

**Authorisation failures**

| Failure | Cause | Fix |
|---|---|---|
| Any internal service can act as any user | Raw credential propagated inward | Short-lived scoped assertions; mTLS service identity |
| Cross-tenant data exposure | Tenant check omitted on one query path | Enforce tenant scoping in the data-access layer, not per query |
| Enumeration of valid ids | 403 distinguishes existing from non-existing | Return 404 where existence is sensitive |
| Policy outage grants access | Failing open on policy service errors | Fail closed; cache decisions with a bounded TTL |
| Revoked access persists | Long-lived tokens or over-cached decisions | Short token lifetimes; revocation list; bounded decision cache |
| Rules diverge between services | Authorisation reimplemented per service | Shared policy definition; centralised evaluation |
| Audit log is not actionable | Records the action but not the resource or rule | Log identity, resource, action and the rule that permitted it |

> **The missing check is silent**  
> An omitted authorisation check causes no error, no failed test and no log line — the request simply succeeds. This is why authorisation is one of the few areas where structural enforcement matters more than discipline: a data-access layer that requires an identity and a permission argument makes the unchecked query impossible to write, which no amount of code review reliably achieves.

**Limits**

> **Operating parameters**
>
> - **Internal assertion lifetime**: seconds to a minute — long enough for one request, short enough to be useless if leaked.
> - **Decision cache TTL** bounds how long revoked access persists; seconds, not minutes, for anything security-relevant.
> - **Access tokens**: 5–15 minutes is the usual compromise between lookup cost and revocation lag, with a revocable refresh token behind them.
> - **Fail closed** on every error path — policy unavailable, identity missing, resource type unknown.
> - **Audit records** need identity, resource, action, time and the permitting rule to be genuinely useful.

**Alternatives**

| Model | Best for | Weakness |
|---|---|---|
| RBAC | Coarse capabilities | Role explosion when permissions are resource-specific |
| ABAC | Contextual rules (time, location, classification) | Hard to answer who can access a given resource |
| ReBAC | Ownership and hierarchy (folders, teams) | Requires fast graph reachability |
| ACLs per resource | Small, explicit permission sets | Does not scale to inherited structures |
| Row-level security | Structural enforcement | Policy in the database |
| Capability tokens | Delegated, time-bounded access | Revocation is hard |

**In real systems**

- **Google's Zanzibar** models authorisation as a relationship graph with reachability checks, and is the reference design for ReBAC at scale.
- **OAuth2 and OIDC** separate the concerns explicitly: authentication produces an identity token, authorisation produces scoped access tokens.
- **Service meshes with mTLS** give each workload a cryptographic identity, so internal calls are authenticated rather than trusted by network position.
- **PostgreSQL row-level security** enforces tenant and ownership rules in the storage engine, making an unchecked query structurally impossible.
- **The recurring class of IDOR vulnerabilities** — reading another user's record by changing an id — is precisely the failure of authorising at the edge rather than at the resource.

**Common mistakes**

- **Authorising only at the gateway**, leaving resource-level rules unenforced and internal calls trusted.
- **Forwarding the user's raw token** to downstream services.
- **Treating internal network position as authorisation.**
- **Failing open** when the policy service is unreachable.
- **Returning 403 for resources whose existence is sensitive**, enabling enumeration.
- **Reimplementing rules per service**, letting them diverge.
- **Audit records without the resource or the rule**, making them unusable for investigation.

**The staff-level view**

Authorisation is the area where structural enforcement beats discipline most decisively, because the failure mode is silent success rather than a visible error.

- **Make the data-access layer require identity and permission**, so an unchecked query cannot be written. Tenant scoping in particular should be structurally impossible to omit.
- **Propagate short-lived signed assertions, never raw credentials.** A downstream service holding a user's token can impersonate them everywhere, which converts one vulnerability into total compromise.
- **Authenticate service-to-service calls.** Network position is not a permission, and treating it as one means the blast radius of any single compromise is the whole system.
- **Centralise policy definition, distribute enforcement.** Rules reimplemented per service diverge silently, and nobody discovers it until an audit or an incident.
- **Fail closed everywhere, and bound decision caches.** A policy outage must reduce access, not expand it, and a cached permit must expire quickly enough that revocation means something.

**Go deeper**

Authentication verifies a credential and is expensive, so it happens once at the edge. Authorisation decides whether an identity may perform an action on a specific resource, which depends on facts — ownership, tenancy, classification — that only the service holding the resource can load. The two therefore have opposite placement rules, and conflating them produces systems where every internal caller is implicitly trusted.

The identity established at the edge should travel inward as a short-lived signed assertion, never as the user's original credential: a downstream service holding a bearer token can impersonate that user anywhere it is accepted, which turns one compromised service into total exposure. Service-to-service calls need their own authentication too, because network position is not a permission — otherwise the blast radius of any single vulnerability is the entire system.

The defining hazard is that a missing authorisation check fails silently: no error, no test failure, no log entry, just a request that succeeds when it should not. So enforcement should be structural rather than remembered — a data-access layer requiring an identity and permission, or row-level security in the database, makes the unchecked query impossible rather than merely discouraged. And everything should fail closed, with decision caches short enough that revocation is meaningful.

Authentication and authorisation are constantly discussed together and have opposite design constraints, which is the root of most failures in this area.

**Opposite placement rules.** Verifying a credential is expensive — signature checks, session lookups, identity provider round trips — and the answer does not change during a request, so it belongs at the edge, performed once. Deciding whether an identity may act on a resource depends on that resource's owner, tenant, classification and inherited permissions, none of which the edge can know without loading the resource, which is the owning service's job anyway. So authentication centralises and authorisation distributes, and any design that centralises both is either doing resource lookups at the gateway or not doing resource-level authorisation at all.

**Propagation must not carry the credential.** A downstream service that receives the user's bearer token can act as that user against anything accepting it, for the token's full lifetime. That converts a vulnerability in the least important service into complete compromise. The correct form is a short-lived signed assertion — valid for seconds, scoped to this request, carrying identity and context — which downstream services verify cryptographically and which is worthless if intercepted. Paired with mutual TLS giving each workload its own identity, this replaces network-position trust with cryptographic identity and contains lateral movement.

**The failure is silent.** An omitted authorisation check produces no exception, no failing test and no log line; the request succeeds. Neither testing nor review reliably catches this, because absence is invisible. That makes authorisation one of the few areas where structural enforcement decisively beats discipline: a data-access layer that takes identity and required permission as mandatory arguments makes an unchecked query impossible to write, and row-level security in the database enforces tenancy for every writer including migration scripts and admin tools. Cross-tenant leakage in particular — the highest-consequence failure in multi-tenant systems — almost always results from one forgotten predicate on one query path, which is exactly what structural enforcement prevents.

**Models and their fit.** Role-based access works for coarse capabilities and collapses when permissions are resource-specific, producing role explosion like `editor_team_17`. Attribute-based rules handle context — time, location, classification — but cannot easily answer “who can access this resource”, since there is no reverse index. Relationship-based access derives permission from a graph — a user belongs to a team that owns the folder containing this document — which matches how people actually reason about access and is why it underpins large-scale authorisation systems, at the cost of needing fast reachability. Practically, coarse capability checks at the edge combine with relationship or attribute decisions at the service.

**The permanent tension is caching versus revocation.** Evaluating policy per request is expensive, so decisions get cached; but a cached permit means revoked access persists for the cache's lifetime. Teams resolve this toward latency by default because staleness is invisible until it matters, which is precisely when it matters most. The decision cache TTL should therefore be treated as a security parameter with a named owner rather than a performance tuning knob, because it determines whether revoking someone's access actually does anything. Similarly, every error path — policy service unreachable, identity missing, resource type unrecognised — must deny, since failing open converts a policy outage into total exposure.

**Centralise definition, distribute enforcement.** Rules reimplemented per service diverge silently and nobody discovers it until an audit or an incident. A shared policy definition evaluated consistently, with enforcement inline where the resource lives, gives both correctness and context. And audit records need identity, resource, action, time and the rule that permitted it — “user X performed action Y” is not auditable, because it cannot answer the question an investigation actually asks.

**Prove it — interview questions**

1. **[Basic] What is the difference between authentication and authorisation?**

   <details><summary>Model answer</summary>

   Authentication establishes who the caller is by verifying a credential. Authorisation decides whether that identity may perform a specific action on a specific resource. They have opposite placement rules: authentication is expensive and does not change during a request, so it happens once at the edge, while authorisation depends on the resource being accessed and must therefore be evaluated wherever that resource is loaded.

   </details>

2. **[Basic] Why can't the gateway do all the authorisation?**

   <details><summary>Model answer</summary>

   Because it does not know the resource. Deciding whether a user may read document 4711 requires knowing who owns that document, which team it belongs to, whether it is inherited from a folder, and whether any fields are restricted — all of which are properties of the data that only the owning service can load. The gateway can check coarse things like whether the token has a scope permitting this class of endpoint, but resource-level decisions have to happen after the resource is fetched.

   </details>

3. **[Senior] Why should you not propagate the user's credential to downstream services?**

   <details><summary>Model answer</summary>

   Because a service holding a bearer token can use it anywhere that token is accepted, for as long as it is valid, impersonating the user completely. That means compromising one low-value service grants the ability to act as any user against every system. The correct pattern is for the edge to verify the credential once and mint a short-lived signed assertion scoped to this request — valid for seconds, carrying the identity and relevant context — which downstream services verify but which is useless if it leaks. Combined with mutual TLS for service identity, that contains lateral movement instead of amplifying it.

   </details>

4. **[Senior] Why is a missing authorisation check particularly dangerous?**

   <details><summary>Model answer</summary>

   Because it fails silently. There is no exception, no failed test and no log entry — the request simply succeeds when it should have been denied, and the only way to notice is a security review or an incident. That is why this is one of the few areas where structural enforcement genuinely beats discipline: a data-access layer that requires an identity and a permission as arguments makes the unchecked query impossible to write, and tenant scoping enforced in the storage layer or by row-level security cannot be omitted by any code path, including migrations and admin tools.

   </details>

5. **[Staff] How would you structure authorisation across a microservice system?**

   <details><summary>Model answer</summary>

   Authentication once at the edge, producing a verified identity that is propagated inward as a short-lived signed assertion rather than as the original credential, over mutually authenticated connections so service identity is cryptographic rather than positional. Each service then performs its own authorisation after loading the resource, because ownership and classification are properties of the data. Policy definition should be centralised — a shared rule set or policy engine — while enforcement stays inline at each service, so rules cannot diverge but decisions happen where the context exists. Everything fails closed: an unreachable policy service, a missing identity or an unknown resource type denies. And I would push tenant scoping into the data-access layer or the database itself, since cross-tenant leakage is the highest-consequence failure and the one most likely to result from a single forgotten predicate.

   </details>

6. **[Principal] What makes authorisation hard to get right at organisational scale?**

   <details><summary>Model answer</summary>

   Three things compound. First, it is distributed by necessity — only the owning service knows its resources — which means the rules live in many places and drift apart unless policy definition is deliberately centralised. Second, the failure mode is silent success, so neither testing nor code review reliably catches an omission, which is why the answer has to be structural: making the unsafe thing impossible to express rather than asking people to remember. Third, there is a permanent tension between caching decisions for latency and revoking access promptly, and teams resolve it toward latency by default because the cost of stale permissions is invisible until it matters. My priorities would be to put tenant and ownership enforcement into the data-access layer so it cannot be bypassed, to make short-lived scoped assertions the only form of propagated identity, and to treat the decision cache TTL as a security parameter with an owner rather than a performance tuning knob — because that number is what determines whether revoking someone's access actually does anything.

   </details>

---
