# System Design Fundamentals Handbook
## A Staff Engineer's Guide to First-Principles Architecture

---

> *This handbook is your prerequisite. Before you design WhatsApp, you must understand TCP. Before you design a Payment System, you must understand ACID. Before you design Netflix, you must understand caching. Read this first.*

---

## About This Handbook

This handbook exists because most system design resources teach you *what* without teaching you *why*.

They show you boxes and arrows. They list technologies. They tell you "use Kafka here" and "add Redis there." But they never tell you:

- Why does Kafka exist at all?
- What breaks if you don't add Redis?
- What tradeoff are you accepting when you choose eventual consistency?
- What does that choice cost you at 3 AM when the system breaks?

This handbook teaches the *why*.

You will learn the same way a Staff Engineer at Google, Meta, or Stripe reasons: from first principles, through tradeoffs, toward real production consequences.

---

## Who This Is For

- **Senior Engineers** preparing for Staff Engineer interviews
- **Lead Engineers** wanting to deepen distributed systems fundamentals
- **Staff Engineer Candidates** building a vocabulary for architectural decisions
- **Software Architects** seeking a first-principles grounding

---

## How to Read This Handbook

Every chapter answers these seven questions:

1. **Why does this exist?** — What pain existed before this concept?
2. **What problem does it solve?** — The core job to be done
3. **How does it work internally?** — Mechanics, not just surface
4. **What are the tradeoffs?** — What you gain and lose
5. **What production failures occur?** — Real consequences of misunderstanding
6. **How does it appear in interviews?** — What interviewers probe for
7. **How is it used in real systems?** — Grounded production examples

---

## The Mental Model Underlying This Handbook: SCC

Every distributed system problem reduces to three fundamental forces:

```
STATE           — Where does data live? Who owns it?
COORDINATION    — How do components agree? Who waits for whom?
CONCENTRATION   — Where does load accumulate? Where are the hotspots?
```

Every technology in this handbook either:
- Solves a STATE problem
- Solves a COORDINATION problem  
- Solves a CONCENTRATION problem

...and in doing so, usually creates a new one.

When you internalize this, you stop memorizing architectures and start *reasoning* toward them.

---

## What This Handbook Is Not

- Not a tutorial for any specific tool
- Not a list of technologies to memorize
- Not a shortcut for shallow interview prep
- Not a substitute for building systems yourself

---

## Companion Case Studies

After completing this handbook, you will be ready to design:

| Domain | System |
|--------|--------|
| Web Infrastructure | URL Shortener, API Gateway, Rate Limiter |
| Messaging | WhatsApp, Notification Service |
| Transportation | Uber, Lyft |
| Payments | Payment System, Ledger, Fraud Detection |
| Identity | IAM Platform, SSO, OAuth Server |
| Streaming | Netflix, YouTube |
| Storage | Dropbox, Google Drive |
| Social | News Feed, Twitter Timeline |
| Search | Search Engine, Typeahead |
| Data | Distributed Cache, Event Streaming Platform |

---

## Table of Contents

### Part 1: Networking Foundations
- Chapter 1: DNS — The Internet's Phone Book
- Chapter 2: TCP vs UDP — Reliability vs Speed
- Chapter 3: HTTP and HTTPS — The Language of the Web
- Chapter 4: HTTP/2 and HTTP/3 — The Evolution of Speed
- Chapter 5: TLS and SSL — The Envelope That Can't Be Opened
- Chapter 6: REST — Resources, Not Procedures
- Chapter 7: gRPC — When REST Is Too Slow
- Chapter 8: WebSockets — Keeping the Line Open

### Part 2: Databases
- Chapter 9: Relational Databases — The Bedrock
- Chapter 10: Normalization and Denormalization — Organizing Truth
- Chapter 11: Transactions and ACID — Making Promises
- Chapter 12: Isolation Levels — How Much Do You Trust Your Neighbors?
- Chapter 13: Indexes — The Art of Finding Fast
- Chapter 14: Query Optimization — Teaching the Database to Think
- Chapter 15: Partitioning — Cutting the Table
- Chapter 16: Replication — Keeping Copies
- Chapter 17: Sharding — Splitting the Database Itself
- Chapter 18: PostgreSQL Fundamentals
- Chapter 19: MySQL Fundamentals
- Chapter 20: NoSQL Databases — When Relational Isn't Enough

### Part 3: Caching
- Chapter 21: Why Caching Exists
- Chapter 22: Cache-Aside
- Chapter 23: Write-Through
- Chapter 24: Write-Back
- Chapter 25: Read-Through
- Chapter 26: Cache Invalidation — The Hard Problem
- Chapter 27: TTL — Time as a Consistency Tool
- Chapter 28: Hot Keys — When One Key Breaks Everything
- Chapter 29: Cache Stampede — The Self-Inflicted DDoS

### Part 4: Messaging Systems
- Chapter 30: Queues — Work Buffering
- Chapter 31: Pub/Sub — Decoupled Broadcasting
- Chapter 32: Event-Driven Architecture
- Chapter 33: Producers and Consumers
- Chapter 34: Consumer Groups — Parallel Processing
- Chapter 35: Ordering — The Hardest Guarantee
- Chapter 36: Retries — When to Try Again
- Chapter 37: Dead Letter Queues — The Poison Pill Handler
- Chapter 38: Kafka Fundamentals
- Chapter 39: RabbitMQ Fundamentals

### Part 5: Distributed Systems
- Chapter 40: Scalability — Vertical and Horizontal
- Chapter 41: Load Balancing — Distributing the Burden
- Chapter 42: Consistency Models
- Chapter 43: CAP Theorem — The Impossible Triangle
- Chapter 44: PACELC — CAP's More Complete Sibling
- Chapter 45: Replication Models
- Chapter 46: Partitioning and Sharding (Advanced)
- Chapter 47: Consensus — How Distributed Systems Agree

### Part 6: Reliability
- Chapter 48: Availability — SLA, SLO, Error Budgets
- Chapter 49: Fault Tolerance Patterns
- Chapter 50: Disaster Recovery
- Chapter 51: Idempotency — Safe to Retry

### Part 7: Architecture Patterns
- Chapter 52: Monolith
- Chapter 53: Microservices
- Chapter 54: Event-Driven Architecture (Advanced)
- Chapter 55: CQRS — Splitting Reads and Writes
- Chapter 56: Saga Pattern — Distributed Transactions

### Part 8: Performance Engineering
- Chapter 57: Latency — The Speed of Slowness
- Chapter 58: Throughput — How Much Can You Handle?
- Chapter 59: Capacity Planning
- Chapter 60: Backpressure — Saying No Gracefully

### Part 9: Security
- Chapter 61: Authentication — Proving Who You Are
- Chapter 62: Authorization — Proving What You Can Do
- Chapter 63: Secrets and Encryption

### Part 10: Observability
- Chapter 64: Logging
- Chapter 65: Metrics — The Four Golden Signals
- Chapter 66: Distributed Tracing

### Part 11: Staff Engineer Topics
- Chapter 67: Data Modeling
- Chapter 68: Tradeoff Analysis
- Chapter 69: Multi-Region Design
- Chapter 70: Failure Mode Analysis
- Chapter 71: Operational Excellence
- Chapter 72: Cost Optimization
- Chapter 73: Incident Response
- Chapter 74: Architecture Reviews

### Final Section
- System Design Interview Framework

---

*Let's begin.*
# Part 1: Networking Foundations

> Before you can design distributed systems, you must understand how machines talk to each other. Every architecture sits on top of these protocols. Every failure eventually traces back to them.

---

# Chapter 1: DNS — The Internet's Phone Book

## Interview Question

> *"A user types `bank.com` in their browser. Walk me through everything that happens before the first byte of HTML arrives."*

---

## Why Should You Care?

DNS (Domain Name System) is invisible until it breaks. When it breaks, every service in your architecture stops working simultaneously. No database connection. No API call. No microservice communication. Just timeouts.

At scale, DNS becomes a performance bottleneck, a security attack surface, and a multi-region failover mechanism — all at once.

**Interviewers care because:** engineers who don't understand DNS make poor decisions about caching, TTL, and failover. They also can't debug the class of "everything is broken, nothing changed" incidents that DNS causes.

---

## Plain English Explanation

Every computer on the internet has a number: an IP address like `104.21.30.91`. But humans don't remember numbers well. We remember names like `bank.com`.

DNS is the system that translates names to numbers.

When you type `bank.com`, your computer asks: *"Hey, what's the IP address for `bank.com`?"* DNS answers that question. Then your computer connects to that IP address.

Simple. Except the "asking" part is a multi-step journey through several servers around the world.

---

## Internal Mechanics

### The Journey of a DNS Query

```
User types bank.com
       |
       v
1. Browser Cache — "Do I know this already?"
       |
       v (miss)
2. OS Cache — "Does the operating system know?"
       |
       v (miss)
3. Recursive Resolver (your ISP or 8.8.8.8)
       |
       v
4. Root Server — "I don't know bank.com, but I know who handles .com"
       |
       v
5. TLD Server (.com) — "I don't know bank.com, but the authoritative
                        server is ns1.bankdns.com"
       |
       v
6. Authoritative Server (ns1.bankdns.com)
       |
       v
   "bank.com = 104.21.30.91"
       |
       v
   Recursive resolver caches the result, returns it
       |
       v
   Browser connects to 104.21.30.91
```

### Key Components

**Recursive Resolver**
The first server your query reaches. Often run by your ISP, Google (8.8.8.8), or Cloudflare (1.1.1.1). Does the work of asking the chain of servers on your behalf. Caches results to avoid repeating the journey.

**Root Servers**
Thirteen sets of servers (labeled A through M) that know where TLD servers live. They don't know IP addresses for websites — they just know *who to ask next*. Replicated globally via anycast.

**TLD Servers (Top-Level Domain)**
Know which authoritative servers handle each domain within their TLD. `.com` TLD servers know that `bank.com` is handled by Verisign's nameservers.

**Authoritative Server**
The final answer. This server actually has the DNS records for your domain. This is what your company controls. This is where you point `bank.com` to your load balancer.

### DNS Record Types

| Record | Purpose | Example |
|--------|---------|---------|
| `A` | Domain → IPv4 | `bank.com → 104.21.30.91` |
| `AAAA` | Domain → IPv6 | `bank.com → 2606:4700::...` |
| `CNAME` | Domain → Another domain | `www.bank.com → bank.com` |
| `MX` | Mail server | `bank.com → mail.bank.com` |
| `TXT` | Arbitrary text | SPF, DKIM, domain verification |
| `NS` | Authoritative nameserver | `bank.com → ns1.bankdns.com` |
| `SOA` | Start of authority | Zone metadata |

### TTL — Time to Live

Every DNS record has a TTL: a number in seconds telling resolvers how long to cache the answer.

```
TTL = 300  → Cache for 5 minutes
TTL = 3600 → Cache for 1 hour
TTL = 86400 → Cache for 1 day
```

**Low TTL:** Changes propagate fast. More DNS queries. More load on your authoritative server.  
**High TTL:** Changes propagate slowly. Fewer DNS queries. Cached everywhere.

This TTL decision is critical for failover.

---

## Production Example

### IAM Platform: DNS-Based Failover

Your identity platform serves `login.company.com`. You run primary servers in `us-east-1` and failover servers in `us-west-2`.

**Strategy:** Use low TTL (60 seconds) on the A record pointing to `us-east-1`. When `us-east-1` degrades, your health check automation updates the DNS record to point to `us-west-2`.

**Problem:** Users who cached the old record for 60 seconds will keep hitting `us-east-1` for up to 60 more seconds. You can't instantly cut over all traffic.

**Production lesson:** Always pre-reduce TTL *before* a planned failover. Reduce `86400 → 300` 24+ hours in advance. Otherwise your "instant failover" takes 24 hours to complete.

```
Normal operation:
  login.company.com A 54.x.x.x TTL=86400  (us-east-1)

Pre-failover (24h before):
  login.company.com A 54.x.x.x TTL=300    (reduce TTL)

Failover:
  login.company.com A 52.x.x.x TTL=300    (us-west-2)

Recovery:
  login.company.com A 54.x.x.x TTL=300    (us-east-1 restored)
  ...then raise TTL back after confirming stability
```

### Payments: GeoDNS

A payment processor serves users globally. London users should hit EU servers for latency and data residency. US users should hit US servers.

**GeoDNS** returns different IP addresses based on the resolver's geographic location:

```
payments.company.com
  → From EU: 185.x.x.x  (Dublin)
  → From US: 54.x.x.x   (Virginia)
  → From APAC: 52.x.x.x (Singapore)
```

This is DNS doing load balancing at the geographic level — before the request even leaves the user's country.

---

## ASCII Architecture Diagram

```
User Browser (London)
        |
        | DNS query: payments.company.com
        v
  Recursive Resolver (ISP, London)
        |
        | Cache miss → asks Root Server
        v
  Root Server
        |
        | ".com is handled by Verisign TLD"
        v
  .com TLD Server
        |
        | "payments.company.com → ns1.company.com"
        v
  Authoritative NS (ns1.company.com) — GeoDNS enabled
        |
        | "London resolver → return EU IP: 185.x.x.x"
        v
  Recursive Resolver caches: payments.company.com = 185.x.x.x (TTL 300)
        |
        v
  Browser connects to 185.x.x.x
        |
        v
  EU Load Balancer → EU Application Servers
```

---

## Common Failure Modes

### 1. Stale DNS After Failover
**Scenario:** Server fails. DNS record updated. But clients cached the old record and keep connecting to the failed server for TTL duration.  
**Prevention:** Pre-reduce TTL. Accept eventual consistency.

### 2. DNS Amplification Attack (DDoS)
**Scenario:** Attacker sends small queries with spoofed source IP (victim's IP). DNS servers send large responses to victim.  
**Prevention:** Rate limiting, Response Rate Limiting (RRL), source IP validation.

### 3. DNS Hijacking
**Scenario:** Attacker poisons a resolver's cache with false records. Users sent to malicious IPs.  
**Prevention:** DNSSEC (DNS Security Extensions) — cryptographic signatures on records.

### 4. Resolver Outage
**Scenario:** Your service hardcodes DNS resolver IP. Resolver goes down. All outgoing connections fail.  
**Prevention:** Use multiple resolvers. Don't rely on a single resolver.

### 5. TTL Too High Before a Launch
**Scenario:** You're migrating from old servers to new servers. TTL is 86400. Users hit old servers for 24 hours after migration.  
**Prevention:** Standard DNS migration playbook: reduce TTL days before, migrate, then restore TTL.

### 6. CNAME Chains
**Scenario:** `api.company.com` → `api.prod.company.com` → `api.alb.us-east-1.amazonaws.com` → IP. Each CNAME requires an extra lookup. Deep chains add latency and fragility.  
**Prevention:** Flatten CNAME chains. Use `ALIAS` records at zone apex where supported.

---

## Tradeoffs

| Decision | Benefit | Cost |
|----------|---------|------|
| Low TTL | Fast propagation, fast failover | More queries, more DNS server load |
| High TTL | Fewer queries, better performance | Slow failover, stale caches |
| GeoDNS | Lower latency globally | More complex routing, resolver location ≠ user location |
| DNSSEC | Integrity protection | Larger responses, key management complexity |
| Single resolver | Simple | Single point of failure |

---

## Interview Discussion

**Interviewers look for:**
- Understanding that DNS is cached at multiple levels
- Knowing TTL affects failover timing
- Understanding that DNS is eventual — not instant
- Ability to describe the full recursive resolution path

**Common beginner mistake:** Saying "just update the DNS record" for failover without mentioning TTL propagation delay. This is a red flag that shows shallow understanding.

**Follow-up questions you should expect:**
- *"How would you design zero-downtime DNS migration?"*
- *"If users are hitting the wrong server after DNS update, what's happening?"*
- *"How does Cloudflare use anycast for DNS?"*

---

## Staff-Level Thinking

A Staff Engineer doesn't just understand DNS resolution. They think about:

**Operational reliability:** What happens to your DNS if your DNS provider goes down? (Use multiple providers. Have a fallback strategy.)

**Security surface:** DNS is unencrypted by default. DNS-over-HTTPS (DoH) and DNS-over-TLS (DoT) exist to encrypt queries. Without them, your ISP can see every domain you query.

**Health-aware routing:** Modern DNS providers (Route53, Cloudflare) integrate with health checks. Combine DNS TTL with health-check intervals — your effective failover time is `max(TTL, health_check_interval)`.

**Internal vs external DNS:** Most large companies run split-horizon DNS. Internal services resolve to private IPs inside the network. External users resolve to public IPs. Same domain, different answers.

```
From inside corporate network:
  api.company.com → 10.0.1.50 (private IP, direct)

From internet:
  api.company.com → 54.x.x.x (public IP, via load balancer)
```

---

# Chapter 2: TCP vs UDP — Reliability vs Speed

## Interview Question

> *"Explain why DNS uses UDP but your database uses TCP. What would break if you switched them?"*

---

## Why Should You Care?

Every network connection you make uses either TCP or UDP. The choice affects latency, reliability, and what your application must do to compensate. Understanding why helps you reason about protocol choices in system design and why certain failure modes exist.

---

## Plain English Explanation

Imagine sending documents between two offices.

**TCP** is like certified mail:
- You send a package.
- You get a confirmation receipt.
- If no receipt arrives, you resend.
- Packages arrive in order.
- Reliable, but slower.

**UDP** is like dropping flyers:
- You throw packets into the void.
- No confirmation.
- No resend.
- No ordering.
- Fast, but unreliable.

The choice depends on whether reliability or speed matters more.

---

## Internal Mechanics

### TCP — Transmission Control Protocol

TCP provides **reliable, ordered, error-checked** delivery of data.

**Three-Way Handshake:**
```
Client                    Server
  |                          |
  |-------- SYN ----------->|   "I want to connect"
  |                          |
  |<------- SYN-ACK --------|   "OK, I heard you, you hear me?"
  |                          |
  |-------- ACK ----------->|   "Yes, connection established"
  |                          |
  |===== DATA TRANSFER ======|
```

**Why this matters:** The handshake adds ~1 RTT (Round Trip Time) before any data flows. In high-latency networks (cross-continent), this is ~150ms of overhead just to start talking.

**What TCP guarantees:**
1. **Reliability** — Lost packets are retransmitted
2. **Ordering** — Segments arrive in the correct sequence
3. **Flow Control** — Sender won't overwhelm receiver (sliding window)
4. **Congestion Control** — Sender won't overwhelm the network

**TCP Connection State:**
```
Connections use ports. A TCP connection is a 4-tuple:
(source IP, source port, destination IP, destination port)

Each connection has state maintained in the kernel.
10,000 concurrent TCP connections = 10,000 kernel state entries.
This is why C10K problem was hard.
```

**Four-Way Teardown:**
```
Client                    Server
  |                          |
  |-------- FIN ----------->|   "I'm done sending"
  |<------- ACK ------------|   "Got it"
  |<------- FIN ------------|   "I'm done too"
  |-------- ACK ----------->|   "Got it, goodbye"
  |                          |
  Both sides in TIME_WAIT for 2*MSL (~60 seconds)
```

**TIME_WAIT problem:** Each closed connection stays in `TIME_WAIT` state for ~60 seconds. On a high-traffic server, you can exhaust port numbers. This is why connection pooling exists.

---

### UDP — User Datagram Protocol

UDP provides **fast, connectionless, best-effort** delivery.

```
Client                    Server
  |                          |
  |-------- DATA ---------->|   (no handshake)
  |-------- DATA ---------->|   (no acknowledgment)
  |-------- DATA ---------->|   (no ordering)
  |                          |
  (Server may receive 0, 1, 2, or 3 of these. No notification either way.)
```

UDP has almost no overhead. The header is just 8 bytes (vs TCP's 20+ bytes).

**What UDP does NOT guarantee:**
- Delivery (packets can be lost silently)
- Ordering (packet 3 may arrive before packet 1)
- No duplicates
- No congestion control (you can hammer the network)

**Why UDP is still useful:**
- Speed: No handshake, no ACK, no retransmit wait
- Broadcast/Multicast support
- Fine when application handles reliability itself, or doesn't need it

---

### When to Use Each

| Scenario | Protocol | Reason |
|----------|---------|--------|
| HTTP/Database connections | TCP | Data must be complete and correct |
| DNS queries | UDP | Small, fast. Client retries if no answer |
| Video streaming | UDP (or QUIC) | Occasional dropped frame is OK; stalling isn't |
| Online gaming | UDP | Old position data is useless; latest is all that matters |
| VoIP/Video calls | UDP | Slight packet loss < pausing to retransmit |
| File transfers | TCP | Corruption is unacceptable |

---

## Production Example

### Payments: Why TCP Matters

A payment authorization request must arrive at the bank completely and in the right order. If a TCP packet is lost, TCP retransmits it. The request arrives complete.

If we used UDP for payments:
- A packet containing "$" might arrive
- A packet containing "1000.00" might be lost
- The bank sees a partial message
- What does it do?

The application would have to implement its own reliability mechanism — essentially reinventing TCP poorly. TCP handles this for you.

### IAM Platform: LDAP over TCP

LDAP (the protocol used to look up users in Active Directory and similar directories) runs over TCP. An LDAP query returns a user's entire attribute set — groups, roles, email, phone. This data must arrive complete and in order.

If a packet is dropped and the user record arrives truncated, your IAM system might grant no access (if roles are missing) or fail to authenticate — causing an outage. TCP's retransmission prevents this.

### DNS: Why UDP is Correct

DNS queries are small (under 512 bytes traditionally). The question is: *"What is the IP for bank.com?"* The answer is one record.

If the UDP packet is lost, the resolver retries after a timeout. Total overhead: one small packet, one small reply. No handshake, no state, low latency.

For large DNS responses (many records, DNSSEC signatures), DNS falls back to TCP automatically. This is the "UDP first, TCP fallback" pattern.

---

## ASCII Architecture Diagram

```
TCP Connection Lifecycle:

Client              Network             Server
  |                    |                   |
  |--SYN-------------->|------------------>|  Handshake: 1 RTT
  |<--SYN-ACK----------|<------------------|
  |--ACK + HTTP Req---->|------------------>|
  |                    |                   |  Server processes
  |<--HTTP Response----|<------------------|
  |--ACK-------------->|                   |
  |                    |                   |
  |--FIN-------------->|------------------>|  Teardown: 1.5 RTT
  |<--FIN-ACK----------|<------------------|
  |--ACK-------------->|                   |


UDP Transaction (DNS):

Client              Network             Server
  |                    |                   |
  |--Query (1 packet)->|------------------>|  No handshake
  |<--Response (1 pkt)-|<------------------|  Immediate reply
  |                    |                   |
  Total: ~0.5 RTT
```

---

## Common Failure Modes

### 1. TCP Connection Exhaustion (TIME_WAIT)
**Scenario:** Short-lived HTTP connections close rapidly. Server accumulates thousands of sockets in TIME_WAIT. Port range exhausted. New connections fail.  
**Fix:** Enable `SO_REUSEPORT`, use connection pooling, tune `net.ipv4.tcp_tw_reuse`.

### 2. Head-of-Line Blocking in TCP
**Scenario:** In HTTP/1.1, one slow request blocks all subsequent requests on the same TCP connection (even if later requests are ready).  
**Fix:** HTTP/2 multiplexing; HTTP/3 (QUIC); connection pools with multiple connections.

### 3. TCP Retransmit Storms
**Scenario:** Network congestion triggers retransmits. Retransmits add more load. Congestion worsens. Spiral downward.  
**Fix:** TCP's congestion control algorithms (Cubic, BBR) are designed to handle this. But misconfigured buffers can break them.

### 4. UDP Packet Loss Under Load
**Scenario:** Application uses UDP without reliability layer. Network drops 0.1% of packets. For small traffic, unnoticed. At scale, thousands of dropped messages per minute.  
**Fix:** If using UDP for non-trivial data, implement application-level ACK and retry. Or use QUIC (which adds reliability on top of UDP).

### 5. Slow Retransmission Timeout (RTO)
**Scenario:** TCP packet lost. RTO (Retransmission Timeout) starts at ~1 second, doubles with each failure. Three lost packets → 7 seconds of waiting. High-latency connection appears "hung."  
**Fix:** Tune TCP parameters. Use TCP Fast Open where available. Consider application-level timeouts below TCP's.

---

## Tradeoffs

| | TCP | UDP |
|--|-----|-----|
| **Reliability** | Guaranteed delivery | Best-effort |
| **Ordering** | In-order delivery | No ordering |
| **Latency** | +1 RTT for handshake | Near-zero overhead |
| **State** | Per-connection kernel state | Stateless |
| **Congestion** | Built-in congestion control | None — can saturate network |
| **Use case** | Database, HTTP, file transfer | DNS, video, gaming, QUIC |

---

## Interview Discussion

**What interviewers look for:**
- Understanding that TCP's reliability comes with latency cost
- Knowing that UDP is appropriate when the application handles retries or doesn't need reliability
- Understanding TIME_WAIT and connection pool implications

**Staff-Level Depth:**
- Know that QUIC (HTTP/3) runs on UDP but reimplements reliability at the application layer with better performance than TCP because it avoids head-of-line blocking at the transport layer
- Know that TCP's slow-start algorithm means the first few packets of a new connection are artificially slow — important for latency-sensitive services

---

## Staff-Level Thinking

TCP was designed in 1974. The internet looked very different. Today, the tradeoffs of TCP's retransmission and head-of-line blocking are increasingly unacceptable for low-latency applications.

QUIC (the protocol underlying HTTP/3) is Google's answer: build reliability on top of UDP, but smarter. Each stream within a connection is independent — one stream's packet loss doesn't block other streams. This solves TCP's head-of-line blocking at the connection level.

A Staff Engineer recognizes that protocol choice is a performance and reliability decision, not just a convention. When you're designing a system where hundreds of milliseconds matter (payments, real-time bidding, IAM token validation), understanding the network stack beneath you is not optional.

---

# Chapter 3: HTTP and HTTPS — The Language of the Web

## Interview Question

> *"Walk me through a complete HTTP request lifecycle from the browser to your application server and back. Include everything."*

---

## Why Should You Care?

HTTP is the universal language of web services. Your REST APIs speak it. Your microservices call each other over it. Your CDN optimizes it. Your load balancer routes it. Misunderstanding HTTP leads to security holes, performance problems, and baffling debugging sessions.

---

## Plain English Explanation

HTTP (HyperText Transfer Protocol) is the agreed-upon language that browsers and servers use to communicate. It defines:

- **How to ask for things:** GET, POST, PUT, DELETE
- **How to describe requests:** Headers (metadata about what you're sending)
- **How to describe responses:** Status codes (200 OK, 404 Not Found)
- **How to carry data:** Request and response bodies

HTTP is stateless by design. Each request is independent. The server remembers nothing about previous requests. This is intentional — it makes servers horizontally scalable. State lives in cookies, tokens, or sessions stored outside the server.

---

## Internal Mechanics

### The Complete HTTP Request Lifecycle

```
1. User types https://bank.com/account/123
2. DNS resolution → IP address (Chapter 1)
3. TCP connection to IP:443 (Chapter 2)
4. TLS handshake (Chapter 5)
5. HTTP request sent over encrypted TCP
6. Server processes request
7. HTTP response returned
8. Browser renders
```

### HTTP Request Structure

```
GET /account/123 HTTP/1.1          ← Request line: method + path + version
Host: bank.com                      ← Required header: target host
Authorization: Bearer eyJ...       ← Auth token
Accept: application/json            ← Expected response format
Content-Type: application/json      ← Body format (for POST/PUT)
User-Agent: Mozilla/5.0...          ← Client identification
Cookie: session=abc123              ← Session state
Cache-Control: no-cache             ← Caching directive

{                                   ← Body (empty for GET, present for POST/PUT)
  "filter": "active"
}
```

### HTTP Response Structure

```
HTTP/1.1 200 OK                     ← Status line: version + code + message
Content-Type: application/json      ← Body format
Content-Length: 412                 ← Body size in bytes
Cache-Control: max-age=300          ← Cache this for 5 minutes
Set-Cookie: session=new123; HttpOnly; Secure
X-Request-Id: abc-def-123          ← Trace ID for debugging

{                                   ← Body
  "id": "123",
  "balance": 5000.00,
  ...
}
```

### HTTP Methods

| Method | Idempotent? | Safe? | Use |
|--------|------------|-------|-----|
| `GET` | Yes | Yes | Read resource |
| `POST` | No | No | Create resource |
| `PUT` | Yes | No | Replace resource |
| `PATCH` | No | No | Partial update |
| `DELETE` | Yes | No | Remove resource |
| `HEAD` | Yes | Yes | GET but no body |
| `OPTIONS` | Yes | Yes | CORS preflight |

**Idempotent:** Repeating the request has the same effect as doing it once.  
**Safe:** Request doesn't change server state.

**Why idempotency matters in payments:** A `POST /payment` request should NOT be idempotent by default — but you need to make it so through idempotency keys. A network retry of a non-idempotent payment = duplicate charge.

### HTTP Status Codes

```
1xx — Informational (100 Continue, 101 Switching Protocols)
2xx — Success
    200 OK
    201 Created
    204 No Content (success, no body)
3xx — Redirects
    301 Moved Permanently (update your bookmark)
    302 Found (temporary redirect)
    304 Not Modified (use your cache)
4xx — Client Errors
    400 Bad Request (you sent garbage)
    401 Unauthorized (not authenticated)
    403 Forbidden (authenticated but not allowed)
    404 Not Found
    409 Conflict (optimistic lock failure)
    429 Too Many Requests (rate limited)
5xx — Server Errors
    500 Internal Server Error
    502 Bad Gateway (upstream service failed)
    503 Service Unavailable (overloaded or down)
    504 Gateway Timeout (upstream too slow)
```

**Critical distinction: 401 vs 403**
- `401 Unauthorized` = Not logged in. Fix: log in.
- `403 Forbidden` = Logged in, but not permitted. Fix: request access.

This distinction matters in IAM systems. Getting them backwards confuses clients and reveals information about what resources exist.

### Headers That Matter

**Security-Critical Headers:**
```
Strict-Transport-Security: max-age=31536000; includeSubDomains
  → "Only talk to me over HTTPS, ever"

Content-Security-Policy: default-src 'self'
  → "Only load resources from my own domain"

X-Frame-Options: DENY
  → "Don't let anyone embed me in an iframe"

X-Content-Type-Options: nosniff
  → "Don't guess at my content type"

Referrer-Policy: strict-origin-when-cross-origin
  → "Don't leak my URL to other sites"
```

**Caching Headers:**
```
Cache-Control: max-age=3600, public
  → CDN and browser can cache for 1 hour

Cache-Control: no-store
  → Don't cache at all (sensitive data)

ETag: "abc123"
  → Version identifier for conditional requests

Last-Modified: Thu, 01 Jan 2026 00:00:00 GMT
  → When was this resource last changed?
```

**CORS Headers (Cross-Origin Resource Sharing):**
```
Access-Control-Allow-Origin: https://app.company.com
  → "Only allow this origin to use my API"

Access-Control-Allow-Methods: GET, POST
  → "Only these methods allowed cross-origin"
```

### HTTPS = HTTP + TLS

HTTPS is not a different protocol. It's HTTP running over a TLS-encrypted TCP connection. The `S` stands for Secure.

```
HTTP:  Browser --[plaintext]--> Server
HTTPS: Browser --[TLS tunnel]--> Server
                    |
             Encrypted and authenticated
```

Without HTTPS:
- Your ISP can read your requests
- Attackers on the same WiFi can read your requests
- Attackers can modify responses (inject scripts)
- No proof the server is who it claims to be

With HTTPS:
- Traffic is encrypted end-to-end
- Server identity is verified via certificate
- Modifications detected via TLS MAC

---

## Production Example

### IAM Platform: OAuth 2.0 Token Exchange

When a user logs in via OAuth 2.0:

```
1. Browser → Authorization Server
   GET /authorize?
     client_id=app123&
     redirect_uri=https://app.company.com/callback&
     response_type=code&
     scope=openid+profile&
     state=random-csrf-token

2. Authorization Server → Browser (after user authenticates)
   HTTP/1.1 302 Found
   Location: https://app.company.com/callback?
     code=AUTHORIZATION_CODE&
     state=random-csrf-token

3. App Server → Authorization Server (server-to-server)
   POST /token
   Content-Type: application/x-www-form-urlencoded

   grant_type=authorization_code&
   code=AUTHORIZATION_CODE&
   redirect_uri=https://app.company.com/callback&
   client_id=app123&
   client_secret=SECRET

4. Authorization Server → App Server
   HTTP/1.1 200 OK
   {
     "access_token": "eyJ...",
     "token_type": "Bearer",
     "expires_in": 3600,
     "refresh_token": "..."
   }
```

Every step is HTTP. The security model depends entirely on:
- HTTPS (encryption and server authentication)
- Correct status codes (302 redirect, 200 success, 401 unauthorized)
- Proper headers (Cache-Control: no-store on token responses)
- State parameter (CSRF protection)

### Payments: Idempotency Keys

```
POST /v1/payments
Idempotency-Key: unique-request-id-12345
Authorization: Bearer token
Content-Type: application/json

{
  "amount": 1000,
  "currency": "USD",
  "to_account": "acc_xyz"
}
```

If the network times out and the client retries with the same `Idempotency-Key`, the server returns the same response as the first request — even if the payment already completed. This prevents double-charging.

---

## Common Failure Modes

### 1. HTTP Keep-Alive Connection Exhaustion
**Scenario:** HTTP/1.1 keeps TCP connections alive between requests. Under high load, connection pool exhausted. New requests hang waiting for a connection.  
**Fix:** Tune connection pool size. Set appropriate keep-alive timeouts.

### 2. Ignoring Response Codes
**Scenario:** Client code checks `if response.ok` but doesn't handle `429 Too Many Requests`. Hammers API, gets banned.  
**Fix:** Implement proper response code handling. Implement exponential backoff for 429 and 503.

### 3. Missing Security Headers
**Scenario:** Application serves sensitive financial data but omits `Cache-Control: no-store`. CDN caches the response. User A's account balance appears in User B's response.  
**Fix:** Explicitly set `Cache-Control: private, no-store` on all authenticated endpoints.

### 4. HTTP to HTTPS Downgrade (SSL Stripping)
**Scenario:** User types `bank.com` (HTTP). Browser follows redirect to HTTPS. Attacker intercepts the initial HTTP request, serves fake HTTP page.  
**Fix:** HSTS (Strict-Transport-Security header). Once seen, browser goes direct to HTTPS.

### 5. Verbose Error Responses
**Scenario:** `500 Internal Server Error` response includes stack trace. Attacker learns your framework, version, file paths.  
**Fix:** Log details internally. Return generic error messages externally.

---

## Tradeoffs

| Design Decision | Benefit | Cost |
|-----------------|---------|------|
| Stateless HTTP | Horizontal scalability | State must live somewhere (tokens, sessions) |
| HTTPS everywhere | Security | TLS handshake overhead (small but real) |
| Long-lived connections | Fewer handshakes | Connection state on server, limits per server |
| Large response bodies | Fewer round trips | More bandwidth, longer time-to-first-byte |
| Aggressive caching | Speed, less backend load | Stale data risk |

---

## Interview Discussion

**Interviewers look for:**
- Clean understanding of request/response structure
- Correct use of HTTP methods and status codes
- Understanding of statelessness and its implications
- Security header knowledge

**Red flags:**
- Using POST for all operations (ignoring idempotency)
- Confusing 401 and 403
- Not knowing what CORS is or why it exists
- Not understanding that HTTP/1.1 is text-based (and why this matters for HTTP/2)

---

## Staff-Level Thinking

Staff Engineers think about HTTP at the infrastructure level:

**Observability:** Every HTTP response code should have a counter. Alert on 5xx rate, 4xx rate, latency percentiles. The four golden signals start with HTTP.

**API versioning:** How does HTTP URL structure (`/v1/payments` vs `/v2/payments`) affect client migration? How do you run both versions simultaneously?

**Protocol evolution:** HTTP/1.1 → HTTP/2 → HTTP/3 (QUIC). Each shift changes performance characteristics. Your load balancers, CDNs, and clients must be updated together.

**Cost:** TLS offloading, request parsing, header processing — all have CPU cost at scale. Large companies run dedicated TLS termination clusters.

---

# Chapter 4: HTTP/2 and HTTP/3 — The Evolution of Speed

## Interview Question

> *"Your API has high latency on mobile. An engineer suggests upgrading to HTTP/2. What does that buy you? What doesn't it fix?"*

---

## Why Should You Care?

HTTP/1.1 was designed in 1999 for a different web. Modern applications load dozens of resources per page — CSS, JavaScript, images, fonts, API calls. HTTP/1.1 handles this poorly. HTTP/2 and HTTP/3 solve the performance problems of HTTP/1.1 at the protocol level.

---

## Plain English Explanation

**HTTP/1.1 problem:** Imagine ordering at a restaurant where the waiter takes one order, goes to the kitchen, waits for it, brings it back, *then* takes the next order. Slow.

**HTTP/2 fix:** Same waiter takes all orders at once, delivers as items come out of the kitchen. Much faster.

**HTTP/3 (QUIC) fix:** Same idea, but now if the kitchen is slow with one dish, other dishes still come out. HTTP/2 still has a problem where one slow request blocks others on the same connection.

---

## Internal Mechanics

### HTTP/1.1 Limitations

**1. Sequential Requests (No Pipelining in Practice)**
```
Client                    Server
  |--Request 1 ---------->|
  |<--------- Response 1--|   Must wait
  |--Request 2 ---------->|
  |<--------- Response 2--|   Must wait
  |--Request 3 ---------->|
  |<--------- Response 3--|
```
Time = sum of all request/response times.

**2. Head-of-Line Blocking**
Even with pipelining enabled (rarely used), response 1 must complete before response 2 is sent. One slow response blocks everything behind it.

**3. Redundant Headers**
Every request re-sends all headers. `User-Agent`, `Authorization`, `Accept-Encoding` — same bytes, sent thousands of times per session. Wasteful.

**4. Text-Based Protocol**
HTTP/1.1 is human-readable text. Efficient for debugging; inefficient for parsing.

---

### HTTP/2 Solutions

**Core Innovation: Multiplexing over a single TCP connection**

```
HTTP/1.1:
  Connection 1: [Request 1 ----] [Response 1 ----]
  Connection 2: [Request 2 ----] [Response 2 ----]
  Connection 3: [Request 3 ----] [Response 3 ----]
  (Multiple connections, one request per connection)

HTTP/2:
  Single Connection:
    Stream 1: [Req1 Frame][Req1 Frame][Res1 Frame][Res1 Frame]
    Stream 2: [Req2 Frame][Res2 Frame]
    Stream 3: [Req3 Frame][Res3 Frame][Res3 Frame]
    (Interleaved frames, multiple streams, one connection)
```

**Key features:**

**1. Binary Framing**
HTTP/2 converts messages into binary frames. Smaller, faster to parse, less error-prone than text.

**2. Multiplexing**
Multiple requests and responses interleaved on a single connection. No waiting.

**3. Header Compression (HPACK)**
Headers compressed using shared lookup table. `Authorization: Bearer eyJ...` sent once, referenced by ID thereafter. Significant savings for API-heavy apps.

**4. Server Push**
Server can proactively send resources the client will need:
```
Client: GET /index.html
Server: Here's index.html (also sending style.css and app.js you'll need next)
```
Used by CDNs for performance optimization. Often disabled because it's easy to over-push.

**5. Stream Prioritization**
Client can tell server: "Render-blocking CSS is more important than tracking pixel."

---

### HTTP/2 Remaining Problem: TCP Head-of-Line Blocking

HTTP/2 multiplexes over a *single* TCP connection. This solves HTTP-level head-of-line blocking. But TCP has its own head-of-line blocking.

```
TCP Stream carrying HTTP/2:
  [Stream1 Pkt1][Stream2 Pkt1][Stream3 Pkt1][Stream1 Pkt2][LOST][Stream3 Pkt2]...

TCP must retransmit the lost packet before delivering anything after it.
Even though Stream1 and Stream3 don't need the lost packet (it belongs to Stream2),
all streams are blocked waiting for TCP retransmission.
```

This is especially painful on mobile networks with frequent packet loss.

---

### HTTP/3 and QUIC

HTTP/3 replaces TCP with **QUIC** — a protocol built on top of UDP that re-implements reliability without head-of-line blocking.

**QUIC innovations:**

**1. Per-stream reliability**
Each stream has its own retransmission mechanism. A lost packet in stream 2 only blocks stream 2. Streams 1 and 3 continue unaffected.

```
QUIC Connection:
  Stream 1: [Pkt1][Pkt2][Pkt3] ← unaffected by Stream 2 loss
  Stream 2: [Pkt1][LOST][Pkt3] ← retransmitting, only blocks Stream 2
  Stream 3: [Pkt1][Pkt2]       ← unaffected
```

**2. 0-RTT and 1-RTT Connection Establishment**
HTTP/1.1 + TLS: 3-4 RTT before first byte (TCP handshake + TLS handshake)
HTTP/2 + TLS: 2-3 RTT
HTTP/3 (QUIC): 1 RTT first connection, **0 RTT for returning connections**

```
QUIC first connection:
  [QUIC Initial + TLS ClientHello]  →
  ←  [QUIC ServerHello + TLS Certs + First Data]
  [Client Finish + First Request]   →
  ←  [Response]

Total: 1 RTT
```

**3. Connection Migration**
QUIC connections use a Connection ID (not IP:port). If your phone switches from WiFi to 4G (IP changes), the QUIC connection survives. With TCP, the connection would be torn down.

This is huge for mobile IAM sessions — a user shouldn't have to re-authenticate because their IP changed.

---

## Production Example

### IAM Platform: Parallel Token Validation

An API gateway validates JWTs before routing requests. With HTTP/1.1, each token introspection call is sequential. With HTTP/2 to the IAM service, the gateway multiplexes 20 concurrent token validation requests over one connection.

```
HTTP/1.1 (10 concurrent requests, each 5ms):
  Serial: 50ms minimum

HTTP/2 (10 concurrent requests, each 5ms, multiplexed):
  Parallel: ~5ms + overhead
```

### Payments: 0-RTT for Repeat Customers

A payment SDK on mobile devices uses HTTP/3. For returning customers, the 0-RTT feature sends the payment request on the very first packet — before the cryptographic handshake completes.

**Risk:** 0-RTT is vulnerable to replay attacks. An attacker could capture the 0-RTT packet and replay it to duplicate the payment request.  
**Mitigation:** Payment APIs must implement idempotency keys. 0-RTT should only be used for safe operations, or idempotent ones.

---

## ASCII Architecture Diagram

```
HTTP/1.1 vs HTTP/2 vs HTTP/3 Latency Profile

Connection Phase:
  HTTP/1.1+TLS: [DNS][TCP][TCP][TLS][TLS][TLS][REQ][RESP]  = 4-5 RTT
  HTTP/2+TLS:   [DNS][TCP][TLS combined][REQ][RESP]          = 2-3 RTT
  HTTP/3+QUIC:  [DNS][QUIC+TLS 1-RTT][REQ+DATA][RESP]       = 1 RTT
  HTTP/3+QUIC:  [DNS][0-RTT DATA in first packet]            = 0 RTT*

Multiplexing:
  HTTP/1.1:
    Conn1: ████████████  (req 1)
    Conn2: ████████████  (req 2, separate TCP connection)
    Conn3: ████████████  (req 3, separate TCP connection)

  HTTP/2:
    Single Conn:
    Stream 1: ██░░██░░██  (frames interleaved)
    Stream 2: ░██░░██░░█
    Stream 3: ░░██░░██░░

  HTTP/3:
    Single QUIC Conn:
    Stream 1: ██░░██░░██  (independent reliability)
    Stream 2: ░██[LOSS]█  (only stream 2 blocked on retransmit)
    Stream 3: ░░██░░██░░  (unaffected by stream 2 loss)
```

---

## Common Failure Modes

### 1. HTTP/2 + Load Balancer Misconfiguration
**Scenario:** Clients connect via HTTP/2 to load balancer. Load balancer connects to backend servers via HTTP/1.1. You lose multiplexing benefit for backend calls.  
**Fix:** Enable HTTP/2 on backend connections too. Or use gRPC (which uses HTTP/2 internally).

### 2. Server Push Over-Eagerness
**Scenario:** Server pushes 20 resources preemptively. Client already has 18 cached. Wasted bandwidth. Browser must cancel pushes.  
**Fix:** Use server push sparingly. Prefer `Link: rel=preload` header hints instead.

### 3. HTTP/3 Blocked by Firewall
**Scenario:** QUIC runs on UDP port 443. Some corporate firewalls block UDP on 443 (non-standard). Clients can't upgrade to HTTP/3.  
**Fix:** HTTP/3 falls back to HTTP/2 automatically via `Alt-Svc` header. Not a hard failure, just a missed optimization.

### 4. 0-RTT Replay Attack
**Scenario:** Attacker captures 0-RTT QUIC packet for a POST request. Replays it. Request processed twice.  
**Fix:** Servers must reject 0-RTT for non-idempotent requests. Use idempotency keys.

---

## Tradeoffs

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 (QUIC) |
|---------|---------|--------|---------------|
| Connections needed | Many (6 per domain) | One | One |
| Head-of-line blocking | HTTP + TCP | TCP only | None |
| Setup latency | 3-4 RTT | 2 RTT | 1 RTT (0 RTT repeat) |
| Mobile robustness | Poor | Moderate | Excellent (connection migration) |
| Infrastructure support | Universal | Widespread | Growing |
| Debugging | Easy (text) | Hard (binary) | Hard (binary + UDP) |
| Firewall compatibility | Universal | Universal | Issues with UDP blocking |

---

## Staff-Level Thinking

HTTP/2 and HTTP/3 are not magic. They solve specific problems:

- Many small requests (API calls, asset loading) → HTTP/2 multiplexing helps greatly
- Mobile with packet loss → HTTP/3 removes TCP head-of-line blocking
- Returning connections → HTTP/3 0-RTT reduces latency

They don't help:
- A single large file download (one stream, no multiplexing benefit)
- Slow application logic (protocol doesn't fix your database)
- High-latency networks (1 RTT is still 1 RTT to Australia)

A Staff Engineer asks: "Where is my latency actually coming from?" before recommending a protocol upgrade.
# Chapter 5: TLS and SSL — The Envelope That Can't Be Opened

## Interview Question

> *"Explain TLS to a non-technical executive, then explain to me what happens during the TLS handshake. Then tell me what happens when a certificate expires."*

---

## Why Should You Care?

Every HTTPS connection uses TLS. Every OAuth token exchange is protected by TLS. Every mTLS service mesh connection uses TLS. When certificates expire, your services go dark. When TLS is misconfigured, your data is readable. TLS is the foundation of secure communication at every layer.

---

## Plain English Explanation

Imagine you need to pass a secret message through a crowd of strangers.

You have two problems:
1. **Encryption:** Can strangers read the message?
2. **Authentication:** Is the person I'm sending to actually who they claim to be?

TLS solves both:
- **Encryption:** The message is sealed in an envelope that only the recipient can open.
- **Authentication:** The recipient shows a government-verified ID (certificate) proving they're who they say.

Without TLS: a coffee shop WiFi owner can read your bank password.  
With TLS: they see random garbage.

---

## Internal Mechanics

### TLS Handshake (TLS 1.3)

```
Client                              Server
  |                                    |
  |--- ClientHello ------------------>|
  |    (TLS version, cipher suites,   |
  |     random nonce, SNI hostname)   |
  |                                    |
  |<--- ServerHello ------------------|
  |     (chosen cipher suite,         |
  |      random nonce,                |
  |      server certificate,          |
  |      server's public key,         |
  |      encrypted extensions)        |
  |                                    |
  |  [Client validates certificate]   |
  |  [Derives session keys from       |
  |   key exchange]                   |
  |                                    |
  |--- {Finished} ------------------->|  ← Encrypted with session key
  |                                    |
  |<--- {Finished} -------------------|  ← Encrypted with session key
  |                                    |
  |=== Encrypted Application Data ====|
```

**TLS 1.3 achieves this in 1 RTT.** TLS 1.2 needed 2 RTT.

### Certificates and PKI

**The problem TLS solves:** How do you trust that `bank.com` is *actually* bank.com and not an attacker?

Answer: **Certificate Authorities (CAs)** — trusted third parties that verify domain ownership and sign certificates.

```
Certificate Authority (e.g., DigiCert, Let's Encrypt)
  ↓ signs
Server Certificate for bank.com
  ↓ presented to
Browser
  ↓ verifies against
Root CA store (pre-installed in OS and browsers)
```

**Certificate contents:**
```
Subject: bank.com
Issuer: DigiCert Global Root CA
Valid From: 2026-01-01
Valid Until: 2027-01-01
Public Key: [RSA 2048-bit or ECDSA P-256]
Signature: [CA's signature over the above]
SANs: bank.com, www.bank.com, api.bank.com
```

**Wildcard Certificates:** `*.bank.com` covers all subdomains but not sub-subdomains. Simple, but revocation affects everything.

**SAN Certificates (Subject Alternative Names):** List explicit hostnames. More specific, easier to reason about.

### Symmetric vs Asymmetric Encryption

**Asymmetric (during handshake):**
- Public key: anyone can use it to encrypt
- Private key: only server has it; used to decrypt
- Slow (RSA 2048 operations are CPU-intensive)

**Symmetric (after handshake):**
- Both parties have the same session key
- Used for actual data encryption (AES-GCM)
- Fast

TLS uses asymmetric cryptography only to securely exchange a symmetric session key. Then all data is encrypted symmetrically.

### Mutual TLS (mTLS)

Standard TLS: only the server presents a certificate. Client authenticates via username/password or token inside the encrypted channel.

mTLS: **both** sides present certificates.

```
Client (presents client cert)       Server (presents server cert)
  |--- ClientHello ----------------->|
  |<--- ServerHello + cert ----------|
  |     [validate server cert]       |
  |--- client cert + Finished ------>|
  |           [validate client cert] |
  |<--- Finished ---------------------|
  |====== Encrypted Channel =========|
```

**Where mTLS appears:**
- Service mesh (Istio, Linkerd) — every service authenticates to every other service
- API clients with client certificates (PCI-DSS requirement for some financial APIs)
- IoT device authentication
- Zero-trust network architectures

In an IAM platform context: mTLS is the network-level authentication layer. Even if a request has a valid JWT, mTLS ensures the calling service is a known service — not a rogue process that somehow obtained a token.

---

## Production Example

### IAM Platform: Certificate Lifecycle

Your authorization service uses mutual TLS. Every microservice has a certificate issued by an internal CA.

**Problem:** Certificates expire. If you don't rotate them before expiration, *every service that trusts the certificate stops trusting it* the moment midnight passes on the expiry date.

```
Normal flow:
  service-a.internal:443 cert expires 2026-12-01
  Rotation job runs 30 days before: 2026-11-01
  New cert issued, services updated, old cert retired.

Failure flow:
  Nobody notices cert expiring.
  2026-12-01 00:00:00 UTC
  Payment service → IAM service: TLS handshake fails
  IAM service cert: EXPIRED
  All payment authorization: DEAD
  On-call engineer wakes up at 3 AM.
```

**Real world:** The LinkedIn, Azure, Stripe, and many other major outages were caused by expired certificates. Automate certificate rotation. Alert 60, 30, 14, 7 days before expiry.

### Payments: TLS and PCI-DSS

PCI-DSS (Payment Card Industry Data Security Standard) requires:
- TLS 1.2 minimum; TLS 1.3 preferred
- No SSL, no TLS 1.0, no TLS 1.1 (these are deprecated and vulnerable)
- Strong cipher suites only (no RC4, no 3DES, no export-grade)
- Certificate validity managed and monitored

Failing a PCI audit over a cipher suite misconfiguration is embarrassing and costly.

---

## ASCII Architecture Diagram

```
Certificate Chain of Trust:

Root CA (in OS trust store, never online)
  └─ Intermediate CA (signs leaf certificates)
       └─ bank.com certificate
              ↓ presented to browser
       Browser verifies chain:
         bank.com cert → signed by Intermediate CA → signed by Root CA → trusted ✓

Certificate Revocation:
  bank.com private key compromised
       ↓
  CA revokes bank.com cert
       ↓
  OCSP (Online Certificate Status Protocol) or CRL updated
       ↓
  Browsers check OCSP before trusting cert
  (OCSP Stapling: server includes OCSP response in TLS handshake — faster)
```

---

## Common Failure Modes

### 1. Expired Certificate (Most Common)
**Scenario:** Certificate expires. All connections refused. Site appears down.  
**Fix:** Automated rotation (certbot/ACME protocol for Let's Encrypt, AWS Certificate Manager auto-renewal). Alert chains.

### 2. Certificate Not Trusted (Self-Signed)
**Scenario:** Developer uses self-signed cert in production. Browsers reject it. App shows "NET::ERR_CERT_AUTHORITY_INVALID".  
**Fix:** Use certificates from a recognized CA, even for internal services. Use your internal PKI for service mesh.

### 3. Hostname Mismatch
**Scenario:** Certificate issued for `api.bank.com`. Request goes to `bank.com`. TLS rejects ("hostname mismatch").  
**Fix:** Use SAN certificates with all hostnames listed. Use wildcard certs carefully.

### 4. Weak Cipher Suite
**Scenario:** Server accepts TLS 1.0 or RC4 ciphers. Attacker uses POODLE or BEAST attack to decrypt sessions.  
**Fix:** Disable deprecated protocols and ciphers. Regularly audit with `nmap --script ssl-enum-ciphers` or similar.

### 5. Private Key Compromise
**Scenario:** Server's private key leaked (in logs, in public repo, in breach). All past TLS sessions using that key are now decryptable.  
**Fix:** Revoke immediately. Use Perfect Forward Secrecy (PFS) cipher suites — session keys are generated fresh per session and not derived from the server's private key.

---

## Tradeoffs

| Consideration | Decision | Tradeoff |
|--------------|---------|---------|
| Certificate validity period | Longer = less rotation overhead | Longer window if key compromised |
| Wildcard vs SAN | Wildcard = simpler | Wildcard revocation affects all subdomains |
| OCSP vs CRL | OCSP = real-time | OCSP = privacy leak (CA knows who connects to what) |
| Self-signed vs CA-signed | Self-signed = free, instant | Self-signed = no chain of trust |
| mTLS | Strong service auth | Certificate lifecycle complexity multiplied by service count |

---

## Staff-Level Thinking

Certificate management at scale is an operational discipline, not a one-time setup. A Staff Engineer designs:
- Automated issuance via ACME/Let's Encrypt or internal CFSSL/Vault PKI
- Automated rotation before expiry with zero-downtime rotation
- Monitoring dashboards showing days until expiry for every certificate
- Runbooks for emergency revocation

**SPIFFE/SPIRE** is the emerging standard for workload identity in microservices — every pod gets a cryptographic identity certificate that rotates automatically. This is the direction service meshes are moving.

---

# Chapter 6: REST — Resources, Not Procedures

## Interview Question

> *"Design the REST API for a payment service that supports creating payments, checking status, and refunding. Include idempotency, versioning, and error handling."*

---

## Why Should You Care?

REST is the dominant style for web APIs. Getting REST right means clean, evolvable APIs. Getting it wrong means breaking clients on every change, confusing consumers, and creating security holes. Every system design case study involves API design decisions.

---

## Plain English Explanation

Before REST, many APIs used RPC (Remote Procedure Call) style: you called functions over the network.

```
# RPC style — verbs as operations
POST /createPayment
POST /getPayment
POST /cancelPayment
POST /refundPayment
```

REST inverts this: instead of *verbs on things*, you define *things* (resources) and use standard *verbs* (HTTP methods) to operate on them.

```
# REST style — nouns as resources, HTTP verbs as operations
POST   /payments        → create payment
GET    /payments/{id}   → get payment
DELETE /payments/{id}   → cancel payment
POST   /payments/{id}/refunds → create refund
```

REST stands for Representational State Transfer. Resources have representations (JSON, XML). You transfer those representations between client and server.

---

## Internal Mechanics

### REST Constraints

**1. Stateless**
Each request must contain all information needed to process it. Server holds no session state.

```
# Stateful (bad):
Session on server: {"user": "alice", "cart": [item1]}
POST /checkout → server looks up session to get cart

# Stateless (REST):
POST /checkout
Body: {"user_id": "alice", "cart": [item1], "auth_token": "..."}
→ All context in the request
```

**2. Uniform Interface**
Standard HTTP methods with consistent semantics. `GET` always reads. `DELETE` always removes. No guessing.

**3. Resource-Oriented**
Everything is a resource with a URI. Resources are nouns.

```
GOOD resources:
  /users
  /users/{id}
  /users/{id}/addresses
  /payments
  /payments/{id}/refunds

BAD resources (RPC leaking in):
  /getUser
  /createPayment
  /processRefund
```

**4. Representation**
Resources can be represented in multiple formats. Accept header negotiates format:
```
Accept: application/json        → returns JSON
Accept: application/xml         → returns XML
Accept: application/hal+json    → returns HAL (hypermedia)
```

**5. HATEOAS** (Hypermedia As The Engine Of Application State)
Responses include links to related actions. Rarely fully implemented in practice.
```json
{
  "id": "pay_123",
  "status": "pending",
  "_links": {
    "self": {"href": "/payments/pay_123"},
    "cancel": {"href": "/payments/pay_123", "method": "DELETE"},
    "refund": {"href": "/payments/pay_123/refunds", "method": "POST"}
  }
}
```

---

### REST API Design Patterns

**URL Conventions:**
```
Collections (plural):    /payments
Individual resource:     /payments/{payment_id}
Nested resource:         /payments/{payment_id}/line-items
Nested collection:       /accounts/{account_id}/transactions
```

**Query parameters for filtering, sorting, pagination:**
```
GET /payments?status=pending&from=2026-01-01&to=2026-12-31
GET /payments?sort=created_at&order=desc
GET /payments?page=2&page_size=20
GET /payments?cursor=eyJpZCI6MTIzfQ==  ← cursor-based pagination
```

**API Versioning Strategies:**

| Strategy | Example | Pros | Cons |
|---------|--------|------|------|
| URL versioning | `/v1/payments` | Explicit, easy routing | URL pollution |
| Header versioning | `Accept: application/vnd.api.v1+json` | Clean URLs | Harder to test in browser |
| Query param | `/payments?version=1` | Easy to override | Messy |

URL versioning is most common in practice: `/v1/`, `/v2/`. Simple, debuggable.

---

## Production Example

### Payments REST API

```
# Create a payment
POST /v1/payments
Authorization: Bearer {token}
Idempotency-Key: idem-{client-generated-uuid}
Content-Type: application/json

{
  "amount": 1000,
  "currency": "USD",
  "source": {"card_token": "tok_abc123"},
  "description": "Order #987654",
  "metadata": {"order_id": "ord_987654"}
}

Response: 201 Created
{
  "id": "pay_xyz789",
  "status": "processing",
  "amount": 1000,
  "currency": "USD",
  "created_at": "2026-05-31T10:00:00Z",
  "_links": {
    "self": {"href": "/v1/payments/pay_xyz789"},
    "refund": {"href": "/v1/payments/pay_xyz789/refunds"}
  }
}

# Get payment status
GET /v1/payments/pay_xyz789
Authorization: Bearer {token}

Response: 200 OK
{
  "id": "pay_xyz789",
  "status": "succeeded",   ← polling until terminal state
  ...
}

# Create refund
POST /v1/payments/pay_xyz789/refunds
Idempotency-Key: idem-{another-uuid}
{
  "amount": 500,
  "reason": "customer_request"
}

Response: 201 Created
{
  "id": "ref_abc123",
  "payment_id": "pay_xyz789",
  "amount": 500,
  "status": "pending"
}
```

### IAM REST API

```
# Create a user
POST /v1/users
{
  "email": "alice@company.com",
  "name": "Alice",
  "roles": ["viewer"]
}
→ 201 Created, Location: /v1/users/usr_123

# Assign role
PUT /v1/users/usr_123/roles
{
  "roles": ["viewer", "editor"]
}
→ 200 OK

# Get user permissions (ABAC/RBAC resolved)
GET /v1/users/usr_123/permissions?resource_type=document&resource_id=doc_456
→ 200 OK
{
  "can_read": true,
  "can_write": false,
  "can_delete": false
}
```

---

## Common Failure Modes

### 1. Non-Idempotent POST Without Idempotency Keys
**Scenario:** Network timeout on payment POST. Client retries. Payment charged twice.  
**Fix:** Idempotency keys. Server deduplicates requests by key.

### 2. Wrong HTTP Method Leading to Cache Poisoning
**Scenario:** GET request modifies state (e.g., `GET /approve-payment?id=123`). CDN or proxy caches the response. Other users "see" the same response (pre-approved state).  
**Fix:** GET must never modify state. Use POST/PUT/PATCH for mutations.

### 3. Breaking API Change Without Versioning
**Scenario:** Rename field `amount` to `total_amount` in v1 API. All existing clients break.  
**Fix:** Never remove or rename fields in existing version. Introduce new version. Support old version for documented deprecation period.

### 4. Returning 200 for Errors
**Scenario:** API returns `{"error": "not found"}` with HTTP 200. Clients must parse body to detect failure. Logs show all 200s even on errors.  
**Fix:** Use correct HTTP status codes. 404 for not found. 400 for bad input. 500 for server error.

---

## Staff-Level Thinking

REST is not a perfect protocol for every use case:

- **Chatty APIs:** REST requires many round trips for complex operations. A user profile page might need 5 API calls. GraphQL (or BFF pattern) reduces this.
- **Real-time:** REST is request-response. For real-time, add WebSockets or SSE on top.
- **Binary data:** REST with JSON is verbose. For high-throughput internal services, gRPC is more efficient.

A Staff Engineer designs REST as the external API contract (for its simplicity and tooling) and uses gRPC or message queues internally where performance matters.

---

# Chapter 7: gRPC — When REST Is Too Slow

## Interview Question

> *"Why does Google use gRPC internally instead of REST? When would you choose gRPC over REST?"*

---

## Why Should You Care?

REST over HTTP/1.1 with JSON is the standard for external APIs. But inside a large microservices architecture, it's often too slow and too verbose. gRPC is the standard for high-performance internal service communication. Understanding it is essential for system design at scale.

---

## Plain English Explanation

REST sends requests as human-readable text (JSON). The server reads it, processes it, sends back text.

gRPC sends requests as compressed binary (Protocol Buffers). The server reads binary, processes it, sends back binary.

Binary is smaller and faster to parse than text. For services making millions of calls per second to each other, this difference compounds significantly.

gRPC also uses HTTP/2 natively, getting multiplexing and streaming for free.

---

## Internal Mechanics

### Protocol Buffers (Protobuf)

You define your API in a `.proto` file:

```protobuf
syntax = "proto3";

service PaymentService {
  rpc CreatePayment(CreatePaymentRequest) returns (PaymentResponse);
  rpc GetPayment(GetPaymentRequest) returns (PaymentResponse);
  rpc StreamPayments(StreamPaymentsRequest) returns (stream PaymentEvent);
}

message CreatePaymentRequest {
  int64  amount    = 1;
  string currency  = 2;
  string source_id = 3;
}

message PaymentResponse {
  string id     = 1;
  string status = 2;
  int64  amount = 3;
}
```

**Code generation:** `protoc` compiles `.proto` into client and server stubs in Go, Java, Python, etc.

```java
// Generated client code (Java)
PaymentServiceGrpc.PaymentServiceBlockingStub stub =
    PaymentServiceGrpc.newBlockingStub(channel);

CreatePaymentRequest request = CreatePaymentRequest.newBuilder()
    .setAmount(1000)
    .setCurrency("USD")
    .setSourceId("src_abc123")
    .build();

PaymentResponse response = stub.createPayment(request);
```

No manual JSON parsing. No URL construction. Type-safe, auto-generated.

### Serialization Comparison

```
JSON:
{
  "amount": 1000,
  "currency": "USD",
  "source_id": "src_abc123"
}
→ ~55 bytes, requires JSON parser

Protobuf binary:
08 E8 07 12 03 55 53 44 1A 0A 73 72 63 5F 61 62 63 31 32 33
→ ~20 bytes, decoded by schema-aware parser
→ ~3x smaller, ~5-10x faster to parse
```

### gRPC Communication Patterns

```
1. Unary (like REST):
   Client → single request → Server → single response

2. Server Streaming:
   Client → single request → Server → stream of responses
   (e.g., "stream me all payment events from today")

3. Client Streaming:
   Client → stream of requests → Server → single response
   (e.g., "I'll send you 1000 records, process them, tell me the result")

4. Bidirectional Streaming:
   Client ←→ stream of messages ←→ Server
   (e.g., real-time order book updates)
```

### gRPC vs REST Comparison

| Aspect | REST/JSON | gRPC/Protobuf |
|--------|---------|--------------|
| Payload size | Larger (text) | Smaller (binary, ~3-5x) |
| Parse speed | Slower | Faster (schema-driven) |
| Human readable | Yes | No (need tools) |
| Browser support | Universal | Limited (needs grpc-web) |
| Streaming | Workarounds (SSE, polling) | Native (4 modes) |
| Schema | Optional (OpenAPI) | Required (proto) |
| Code generation | Optional | Central to workflow |
| Load balancing | L7 HTTP, easy | Requires L7 gRPC-aware LB |
| Error model | HTTP status codes | Rich error codes |

---

## Production Example

### IAM: Token Introspection at Scale

An API gateway validates tokens for every request. With REST+JSON, each introspection request:
- Creates JSON: `{"token": "eyJ..."}`
- Sends HTTP/1.1 POST
- Server parses JSON
- Creates JSON response: `{"active": true, "user_id": "123", "scopes": [...]}`
- Client parses JSON

At 100,000 requests/second, this JSON serialization/deserialization becomes measurable CPU overhead.

With gRPC:
- Binary protobuf encoding
- HTTP/2 multiplexing (multiple introspections over one connection)
- Schema-driven parsing (no dynamic type inference)
- ~3x smaller payloads

Result: same infrastructure handles more load.

### Payments: Streaming Transaction Events

```protobuf
rpc SubscribeTransactions(SubscribeRequest) returns (stream TransactionEvent) {}
```

A fraud detection service streams real-time transaction events from the payment processor:

```
Fraud Service → PaymentService: SubscribeRequest{account_id: "acc_123"}
PaymentService → Fraud Service: stream of TransactionEvent (as they occur)
```

vs REST alternative: Polling every 100ms (CPU waste, latency). WebSocket (works but not type-safe, harder code generation).

---

## Common Failure Modes

### 1. gRPC Load Balancing Pitfall
**Scenario:** gRPC uses HTTP/2 long-lived connections. Standard L4 load balancer distributes connections (not requests). All requests on a connection go to the same backend.  
**Fix:** Use L7 gRPC-aware load balancing (Envoy, Nginx, cloud LBs). Or implement client-side load balancing.

### 2. Protobuf Schema Breaking Change
**Scenario:** Team removes field `amount` from protobuf, adds `total_amount`. Old clients still send `amount`. New server ignores it. Silent data loss.  
**Fix:** Never remove fields. Deprecate with `[deprecated=true]`. Add new fields with new field numbers. Keep old servers running during rollout.

### 3. Missing Deadlines
**Scenario:** gRPC call hangs. No timeout set. Thread blocks forever. Thread pool exhausted.  
**Fix:** Always set deadlines. `withDeadline(Instant.now().plusSeconds(5))`.

### 4. gRPC Not Browser-Compatible
**Scenario:** You want to call your gRPC service from a browser. HTTP/2 trailers (required for gRPC) not supported in browser XHR.  
**Fix:** Use `grpc-web` (proxy translates between browser HTTP/1.1 and gRPC HTTP/2). Or expose REST API for browser clients.

---

## Staff-Level Thinking

gRPC is the standard for internal microservice communication at Google, Netflix, Uber, and most large tech companies. It's not a replacement for REST — it's the *internal* communication layer.

**Design pattern:** 
- External API: REST (universally accessible, human-debuggable)
- Internal API: gRPC (performance, type safety, code generation)
- Boundary: API Gateway translates REST → gRPC internally

```
Mobile Client
    ↓ REST/JSON
API Gateway
    ↓ gRPC/Protobuf
Internal Services (IAM, Payments, Orders...)
```

---

# Chapter 8: WebSockets — Keeping the Line Open

## Interview Question

> *"Design a real-time notification system for a payment app. Users should see transaction alerts within 100ms. How do you push notifications to 10 million concurrent users?"*

---

## Why Should You Care?

HTTP is request-response. The client always initiates. The server can only respond.

But some features require the server to *push* to the client: notifications, live prices, chat messages, real-time order status. WebSockets solve this. Understanding WebSockets is essential for any real-time feature design.

---

## Plain English Explanation

HTTP is like a letter exchange. You write a letter (request), wait for a reply (response), then start over.

WebSocket is like a phone call. Once connected, either party can speak at any time. The connection stays open. The server can push data without waiting for the client to ask.

Without WebSockets, clients simulate real-time via **polling**: ask the server "anything new?" every second. Wasteful — 999 out of 1000 responses might be "no."

---

## Internal Mechanics

### WebSocket Handshake

WebSocket upgrades an existing HTTP connection:

```
Client → Server:
GET /notifications HTTP/1.1
Host: app.bank.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

Server → Client:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

[Connection is now a persistent bidirectional WebSocket]
```

After the 101 handshake, the HTTP protocol is abandoned. The raw TCP connection is kept open and used for WebSocket frames.

### WebSocket Frames

```
WebSocket messages are transmitted as frames:

0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - -+
```

Opcodes: `0x1` = text frame, `0x2` = binary frame, `0x8` = close, `0x9` = ping, `0xA` = pong.

**Ping/Pong:** Server sends ping, client must respond with pong. Detects dead connections (unlike HTTP, which doesn't know the connection is broken until the next request).

### Connection Architecture at Scale

Maintaining 10 million concurrent WebSocket connections is a distributed systems problem.

```
10M users → You cannot put all connections on one server

Architecture needed:

                    Users (10M mobile apps)
                         |
                    WebSocket CDN / Proxy Layer
                    (Cloudflare, AWS API Gateway)
                         |
              ┌──────────┴──────────┐
              │     WebSocket        │
              │   Connection Layer   │
              │  (stateful servers)  │
              │   Server A: 50K conns│
              │   Server B: 50K conns│
              │   Server C: 50K conns│
              │   ...N servers...    │
              └──────────┬──────────┘
                         |
              ┌──────────┴──────────┐
              │    Message Bus       │
              │  (Kafka/Redis PubSub)│
              └──────────┬──────────┘
                         |
              ┌──────────┴──────────┐
              │   Notification       │
              │   Service            │
              │   (stateless,        │
              │    scales easily)    │
              └─────────────────────┘
```

**Key problem:** When a payment event occurs, which WebSocket server holds the connection for that user?

**Solutions:**
1. **Message bus fan-out:** Publish event to Kafka/Redis. All WebSocket servers subscribe. Each server checks if it holds a connection for that user. Broadcasts to it.
2. **Routing table:** Store `{user_id → websocket_server_id}` in Redis. Route events directly to the correct server.
3. **User-partitioned servers:** Consistent hashing ensures user always connects to the same server. Events routed by same hash.

---

## Production Example

### Payments: Real-Time Transaction Alerts

```
Payment completes in processing service
    ↓
Publish to Kafka: PaymentCompleted{user_id: "u123", amount: 500, ...}
    ↓
Notification Service consumes event
    ↓
Looks up WebSocket connection: Redis → "user u123 is on WS Server B"
    ↓
Pushes to WS Server B via internal message
    ↓
WS Server B sends to user's open connection:
{
  "type": "PAYMENT_COMPLETE",
  "payment_id": "pay_xyz",
  "amount": 500,
  "timestamp": "2026-05-31T10:05:00Z"
}
    ↓
User sees push notification on app: "₹500 received!"
```

### IAM: Session Invalidation

When an admin revokes a user's access:
```
Admin: POST /v1/users/{id}/sessions/revoke-all
    ↓
Session revocation published to message bus
    ↓
WebSocket connection for that user receives:
{
  "type": "SESSION_INVALIDATED",
  "reason": "admin_revocation"
}
    ↓
App clears local tokens, redirects to login
```

This is more immediate than waiting for the JWT to expire.

---

## Polling vs WebSockets vs SSE

| Approach | How | Server Push? | Overhead | Use Case |
|---------|-----|-------------|---------|---------|
| Short Polling | Client polls every N seconds | No | High (many requests) | Simple, low-frequency |
| Long Polling | Client asks, server holds until data | Effectively yes | Medium | Pre-WebSocket real-time |
| SSE (Server-Sent Events) | HTTP/1.1 stream, server pushes | Yes (one-way) | Low | Notifications, events |
| WebSockets | Persistent bidirectional connection | Yes (both ways) | Low per-message | Chat, collaboration, games |

**SSE is underused:** For one-directional server-push (notifications, live prices), SSE is simpler than WebSockets. HTTP/2 SSE is especially efficient. No upgrade handshake. Reconnects automatically. Works through standard HTTP proxies.

---

## Common Failure Modes

### 1. Connection Leak
**Scenario:** Client disconnects without sending close frame. Server never learns of disconnect. Holds dead connection. Memory leaks.  
**Fix:** Heartbeat/ping mechanism. If pong not received within timeout, server closes the connection.

### 2. WebSocket Server Memory Explosion
**Scenario:** 100,000 connections on one server. Each connection has receive/send buffers (default 4KB each). Plus connection state. Plus message queues. RAM exhausted.  
**Fix:** Profile memory per connection. Tune buffer sizes. Horizontal scale WebSocket servers. Use connection limits.

### 3. Load Balancer Without Sticky Sessions
**Scenario:** WebSocket upgrade request hits Server A. Subsequent HTTP request (e.g., REST API call) hits Server B. Fine for REST. Problem: if load balancer routes the WS reconnect to Server B, the user's "connection" is now on a different server. Session state lost.  
**Fix:** Use sticky sessions for WebSocket upgrade. Or ensure session state is external (Redis).

### 4. No Backpressure
**Scenario:** Server publishes 10,000 messages/second to a client that can only consume 100/second. Message queue fills. OOM. Connection drops.  
**Fix:** Implement backpressure. Monitor send queue size per connection. Drop old messages or disconnect slow clients.

---

## Staff-Level Thinking

WebSockets are stateful — each connection is pinned to a server. This breaks horizontal scaling assumptions.

**Operational question:** When you deploy a new version of your WebSocket server, what happens to existing connections? Options:
1. Close all connections on deploy (disruptive)
2. Graceful drain: stop accepting new connections, wait for existing to close naturally (slow)
3. Blue-green: new version on new servers, shift load balancer gradually (complex)

Staff Engineers design for *connection lifecycle* — not just "it works when connected."

**Alternative architecture for scale:** Rather than managing WebSocket servers yourself, use a managed WebSocket service (Pusher, Ably, AWS API Gateway WebSocket APIs). Trade operational control for operational simplicity. The right choice depends on scale, cost, and control requirements.
# Part 2: Databases

> Databases are where state lives. State is the hardest part of distributed systems. Master databases before mastering distribution.

---

# Chapter 9: Relational Databases — The Bedrock

## Interview Question

> *"When would you use a relational database vs NoSQL? What properties of relational databases make them uniquely suited for financial systems?"*

---

## Why Should You Care?

Relational databases have existed since the 1970s and remain the dominant choice for financial systems, IAM, e-commerce, and most stateful business applications. Understanding why helps you choose correctly — and defend your choice in interviews with reasoning, not fashion.

---

## Plain English Explanation

A relational database stores data in tables (rows and columns). Tables relate to each other via keys.

```
users table:
| id  | name    | email           |
| 001 | Alice   | alice@bank.com  |
| 002 | Bob     | bob@bank.com    |

accounts table:
| id  | user_id | balance | currency |
| A01 | 001     | 5000.00 | USD      |
| A02 | 002     | 3200.00 | USD      |

transactions table:
| id  | from_account | to_account | amount | timestamp           |
| T01 | A01          | A02        | 100.00 | 2026-05-31 10:00:00 |
```

The power is that you can **join** these tables: give me all transactions where the user's name is "Alice" and the amount > 500.

Relational databases also enforce **constraints**: you cannot insert a transaction where `from_account` doesn't exist in `accounts`. The database guarantees referential integrity.

---

## Internal Mechanics

### Tables, Rows, Columns

**Table:** A named collection of related data.  
**Row (tuple):** One record. All rows in a table have the same shape.  
**Column:** A named attribute with a defined type.

```sql
CREATE TABLE payments (
    id          UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     BIGINT        NOT NULL REFERENCES users(id),
    amount      DECIMAL(15,2) NOT NULL CHECK (amount > 0),
    currency    CHAR(3)       NOT NULL,
    status      VARCHAR(20)   NOT NULL DEFAULT 'pending',
    created_at  TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);
```

### Constraints — The Database's Promises

```
PRIMARY KEY     → Row uniqueness. No two rows have the same PK.
NOT NULL        → Column must always have a value.
UNIQUE          → Column value must be unique across all rows.
FOREIGN KEY     → References a row in another table (referential integrity).
CHECK           → Custom validation rule (e.g., amount > 0).
DEFAULT         → Value if not provided.
```

These are enforced by the database engine — not your application code. Even if your application has a bug, the database won't let you insert bad data.

### SQL — Structured Query Language

**The four fundamental operations:**

```sql
-- SELECT (Read)
SELECT u.name, p.amount, p.status
FROM payments p
JOIN users u ON p.user_id = u.id
WHERE p.status = 'pending'
  AND p.created_at > NOW() - INTERVAL '7 days'
ORDER BY p.created_at DESC
LIMIT 100;

-- INSERT (Create)
INSERT INTO payments (user_id, amount, currency, status)
VALUES (001, 500.00, 'USD', 'pending')
RETURNING id;

-- UPDATE (Modify)
UPDATE payments
SET status = 'succeeded', updated_at = NOW()
WHERE id = 'pay_123'
  AND status = 'pending';  -- optimistic concurrency control

-- DELETE (Remove)
DELETE FROM payments WHERE id = 'pay_123';
```

### Joins

**INNER JOIN:** Only rows that match in both tables.  
**LEFT JOIN:** All rows from left table; NULLs where no match in right.  
**RIGHT JOIN:** All rows from right table; NULLs where no match in left.  
**FULL OUTER JOIN:** All rows from both tables.

```sql
-- Users with their payment totals (including users with no payments)
SELECT u.id, u.name, COALESCE(SUM(p.amount), 0) as total_paid
FROM users u
LEFT JOIN payments p ON u.id = p.user_id
GROUP BY u.id, u.name
ORDER BY total_paid DESC;
```

---

## Production Example

### IAM: Storing Users, Roles, and Permissions

```sql
-- Core IAM schema
CREATE TABLE users (
    id          UUID PRIMARY KEY,
    email       VARCHAR(255) UNIQUE NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE roles (
    id   UUID PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL  -- 'admin', 'viewer', 'editor'
);

CREATE TABLE user_roles (
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
    granted_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, role_id)
);

CREATE TABLE permissions (
    id            UUID PRIMARY KEY,
    resource_type VARCHAR(100) NOT NULL,  -- 'document', 'payment'
    action        VARCHAR(50) NOT NULL,   -- 'read', 'write', 'delete'
    role_id       UUID REFERENCES roles(id)
);
```

**Why relational?**
- Referential integrity: Can't grant a role that doesn't exist.
- Joins: Get all permissions for a user in one query.
- ACID: Role changes are atomic. No half-states.
- Constraints: No duplicate user-role assignments.

---

## Common Failure Modes

### 1. N+1 Query Problem
**Scenario:** Load 100 users. For each user, fetch their accounts. 100 SELECT queries instead of 1 JOIN.  
**Fix:** JOIN or batch fetch. Use ORM's eager loading.

### 2. Missing Index on Foreign Key
**Scenario:** `payments.user_id` has no index. Query to get all payments for a user does full table scan. Works fine at 10K rows, catastrophic at 10M rows.  
**Fix:** Always index foreign keys. Monitor slow query logs.

### 3. Schema Migration Without Backwards Compatibility
**Scenario:** Rename column `amount` to `total_amount`. Deploy new code. Old code breaks immediately (reading old column name). Downtime.  
**Fix:** Always backwards-compatible migrations: add column first, backfill, migrate code, drop old column last.

### 4. Storing Sensitive Data Unencrypted
**Scenario:** `card_number` stored in plaintext in payments table. Database breach. PCI violation.  
**Fix:** Encrypt sensitive columns (pgcrypto, application-level encryption, Vault Transit). Tokenize card numbers.

---

## Staff-Level Thinking

Relational databases give you four things NoSQL doesn't easily provide:

1. **ACID transactions** — critical for financial correctness
2. **Referential integrity** — data consistency enforced at the engine level
3. **Joins** — complex relational queries without application-level assembly
4. **Schema enforcement** — contract for data shape

The cost: vertical scaling limits, schema rigidity, difficult sharding.

**Staff engineer question:** *"What's the write throughput requirement?"* If the answer is millions of writes per second across independent entities, NoSQL may be appropriate. If the answer involves multi-entity consistency (a payment debits one account and credits another atomically), relational + transactions is hard to replace.

---

# Chapter 10: Normalization and Denormalization — Organizing Truth

## Interview Question

> *"Your payment database is getting slow. An engineer suggests denormalizing the schema. What does that mean? When is it the right call? What are the risks?"*

---

## Why Should You Care?

Normalization is how you store truth once. Denormalization is the deliberate choice to store redundant copies for performance. Every database schema is a tradeoff between write complexity and read performance. Knowing where on this spectrum to be is a Staff Engineer skill.

---

## Plain English Explanation

**Normalization:** Each fact is stored exactly once. Want a user's name? It's in the `users` table. Always.

**Denormalization:** Repeat data in multiple places for faster reads. Store the user's name directly in the `payments` table too — so you don't need a JOIN to show "Alice paid $100."

```
Normalized:
  payments: {id, user_id, amount}    ← no user name here
  users:    {id, name, email}        ← name lives here only
  → Need JOIN to get name + amount together

Denormalized:
  payments: {id, user_id, user_name, amount}  ← name copied here too
  users:    {id, name, email}
  → Can read payments without JOIN
```

The cost: if Alice changes her name to "Alice Smith," you must update it in *both* places. Miss one → inconsistency.

---

## Internal Mechanics

### Normal Forms

**1NF (First Normal Form):**
- Each column contains atomic (indivisible) values
- No repeating groups

```sql
-- Violates 1NF (storing multiple values in one column):
| payment_id | tags                        |
| pay_123    | "urgent,high-value,fraud"   |  ← CSV in a column

-- 1NF compliant:
payment_tags table:
| payment_id | tag         |
| pay_123    | urgent      |
| pay_123    | high-value  |
| pay_123    | fraud       |
```

**2NF (Second Normal Form):**
- In 1NF
- Every non-key column depends on the *entire* primary key (no partial dependency)

```sql
-- Violates 2NF (user_name depends on user_id alone, not full PK):
| payment_id | user_id | user_name | amount |
-- user_name should be in users table, not here

-- 2NF compliant:
payments: {payment_id, user_id, amount}
users:    {user_id, user_name}
```

**3NF (Third Normal Form):**
- In 2NF
- No transitive dependencies (non-key columns don't depend on other non-key columns)

```sql
-- Violates 3NF:
| user_id | city_id | city_name | country |
-- city_name and country depend on city_id, not user_id

-- 3NF compliant:
users: {user_id, city_id}
cities: {city_id, city_name, country}
```

**BCNF (Boyce-Codd Normal Form):** Stricter 3NF. Handles edge cases with overlapping candidate keys.

### Practical Normalization in Production

Full 3NF is the standard for OLTP (Online Transaction Processing — your production database). It:
- Minimizes data duplication
- Makes updates simple (one place to change)
- Enables flexible queries
- Keeps write operations clean

Full normalization is *not* the target for OLAP (Online Analytical Processing — reporting and analytics). Analytical queries that join 10 tables are slow. That's why data warehouses use denormalized star schemas.

---

### When to Denormalize

**Decision framework:**

```
Is the read slow because of JOINs?  → Profile first. Often index is the fix.
Does the data change frequently?    → High change frequency makes denorm expensive.
Is consistency critical?            → Denorm creates consistency maintenance burden.
Is read:write ratio very high?      → Denorm is more attractive.
Can you tolerate eventual consistency? → Async denorm updates become possible.
```

**Denormalization patterns:**

**1. Materialized columns:**
```sql
-- Precompute account balance instead of summing transactions:
ALTER TABLE accounts ADD COLUMN balance DECIMAL(15,2) NOT NULL DEFAULT 0;
-- Update via triggers or application logic on every transaction
```

**2. Duplicate frequently-read columns:**
```sql
-- Copy merchant_name into transactions for fast display (name rarely changes):
ALTER TABLE transactions ADD COLUMN merchant_name VARCHAR(255);
```

**3. Precomputed aggregates:**
```sql
-- Daily spend summary instead of aggregating all transactions:
CREATE TABLE daily_spend_summary (
    user_id    UUID,
    date       DATE,
    total_spend DECIMAL(15,2),
    tx_count   INT
);
-- Maintained by batch job or triggers
```

---

## Production Example

### Payments: Normalized OLTP + Denormalized for Reporting

```
OLTP Database (normalized):
  users, accounts, transactions, merchants, fees
  → Used by payment processing. Writes heavy.
  → Joins on reads are acceptable (reads are fewer).

OLAP / Reporting Database (denormalized):
  fact_transactions (flat table with everything in it):
  | transaction_id | user_name | merchant_name | amount | fee |
  |                | category  | city | country | device_type |
  → ETL job copies from OLTP nightly
  → Analysts run complex aggregate queries
  → Slow joins would make analysts' queries painfully slow
  → Redundancy OK because this is read-only
```

This is the classic **Lambda Architecture** pattern: fast OLTP normalized writes, separate OLAP denormalized for reads.

---

## Common Failure Modes

### 1. Denormalizing Without Update Strategy
**Scenario:** Copy `merchant_name` into transactions. Merchant changes name. Now thousands of historical transactions show the old name. Inconsistency.  
**Fix:** Either don't denormalize mutable data, or build an update propagation mechanism.

### 2. Over-Normalizing for Analytical Workloads
**Scenario:** Reporting database is fully normalized (3NF). Analyst query joins 8 tables. Runs for 4 hours.  
**Fix:** Use separate data warehouse (Redshift, BigQuery, Snowflake) with denormalized star/snowflake schema.

### 3. Premature Denormalization
**Scenario:** Engineer denormalizes "for performance" before measuring. Added complexity. No actual benefit (missing index was the real problem).  
**Fix:** Measure first. Profile queries. Add indexes. Only denormalize as a last resort with clear benchmarks.

---

## Staff-Level Thinking

Normalization vs denormalization is a **read/write tradeoff**:
- Normalized: cheap writes, more expensive reads (joins)
- Denormalized: cheap reads, expensive writes (update propagation)

At Staff level, you think in *workload profiles*:
- Heavy write path (payment processing): normalize
- Heavy read path (user-facing dashboards): consider denormalization or CQRS
- Analytical path: separate store, denormalized

The word "denormalization" in an interview is a signal. It should trigger: *"What's the access pattern? What changes? What's the consistency requirement?"* — not a reflexive yes or no.

---

# Chapter 11: Transactions and ACID — Making Promises

## Interview Question

> *"Explain why ACID properties matter for a payment system. What would happen if you transferred money between accounts without transactions?"*

---

## Why Should You Care?

Transactions are the mechanism that makes databases trustworthy. Without them, any concurrent operation on shared data can leave the database in a corrupted state. Financial systems, IAM, and any multi-entity operation *require* transactions. This is the most interview-critical database concept.

---

## Plain English Explanation

Imagine transferring $100 from Alice's account to Bob's account.

**The naive approach:**
```
Step 1: Debit Alice: UPDATE accounts SET balance = balance - 100 WHERE id = 'alice'
Step 2: Credit Bob:  UPDATE accounts SET balance = balance + 100 WHERE id = 'bob'
```

What if the database crashes between Step 1 and Step 2?

- Alice's account: $100 gone
- Bob's account: nothing received
- $100 has vanished from the system

**A transaction wraps both steps:**
```
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 'alice';
  UPDATE accounts SET balance = balance + 100 WHERE id = 'bob';
COMMIT;
```

Now either both steps complete, or neither does. The $100 cannot vanish.

This is a **transaction**: an atomic unit of work that either fully succeeds or fully fails.

---

## Internal Mechanics

### ACID Properties

**A — Atomicity**
All operations in a transaction succeed together, or all are rolled back.

```
Transaction T:
  Operation 1: Debit Alice  → OK
  Operation 2: Credit Bob   → FAILS (constraint violation? disk error?)
  
  Result: Both operations ROLLED BACK.
  Alice's balance unchanged. Bob's balance unchanged.
  Database in consistent state.
```

**C — Consistency**
A transaction takes the database from one valid state to another valid state. All constraints are satisfied.

```
Constraint: balance >= 0
Transaction: Withdraw $5000 from account with $4000
  → Database REJECTS: violates CHECK (balance >= 0)
  → Transaction aborted
  → Database remains in consistent state
```

**I — Isolation**
Concurrent transactions appear to execute serially. One transaction cannot see another's uncommitted changes.

```
Time →
T1: BEGIN; UPDATE balance=balance-100; ...still running...  COMMIT
T2: BEGIN; SELECT balance; ...                              ...
            ↑
   T2 reads here. Does it see balance before T1's debit?
   After T1's debit but before commit?
   
   Answer depends on isolation level (Chapter 12).
```

**D — Durability**
A committed transaction is permanent. Even if the server crashes immediately after `COMMIT`, the data is not lost.

Achieved via Write-Ahead Log (WAL): changes are written to a log on disk *before* being applied to the actual data files. On crash recovery, WAL is replayed.

### How Transactions Work Internally

**Two-Phase Protocol:**

```
Phase 1: Acquire locks, execute operations
  BEGIN
  → Transaction acquires row-level locks on accessed rows
  → Writes go to:
    1. WAL (Write-Ahead Log) in memory
    2. Buffer pool (in-memory pages)
  
Phase 2: Commit or rollback
  COMMIT
  → WAL is flushed to disk (fsync) ← This is the Durability guarantee
  → Locks released
  → Buffer pool changes will be written to data files eventually
  
  ROLLBACK
  → WAL changes discarded
  → Buffer pool reverted
  → Locks released
```

**Write-Ahead Log (WAL):**
```
WAL File (sequential append, very fast):
  LSN 1001: BEGIN T1
  LSN 1002: T1 UPDATE accounts (alice): balance 5000→4900
  LSN 1003: T1 UPDATE accounts (bob):   balance 3000→3100
  LSN 1004: COMMIT T1
  
  If crash at LSN 1003:
    Recovery replays WAL from last checkpoint
    LSN 1003 not yet committed → rolled back
    
  If crash after LSN 1004:
    LSN 1004 = COMMIT → both changes applied on recovery
```

### Distributed Transactions (Two-Phase Commit)

When a transaction spans multiple databases (e.g., debit in Database A, credit in Database B):

```
Coordinator (orchestrates)
  │
  ├──── Participant A (accounts DB, shard 1)
  └──── Participant B (accounts DB, shard 2)

Phase 1 — PREPARE:
  Coordinator: "Can you commit?"
  Participant A: "Yes, I've written to WAL and locked rows"
  Participant B: "Yes, I've written to WAL and locked rows"

Phase 2 — COMMIT:
  Coordinator: "Commit!"
  Participant A: commits
  Participant B: commits
```

**Problem:** If coordinator crashes after Phase 1 but before Phase 2, participants are stuck in a "prepared" state with locks held. This is the 2PC problem — it blocks.

This is why distributed transactions are avoided at scale. Instead, use Saga patterns (Chapter 56).

---

## Production Example

### Payments: Account Transfer

```sql
BEGIN;

-- Check balance (with lock to prevent concurrent modification)
SELECT balance FROM accounts WHERE id = 'alice' FOR UPDATE;
-- Returns: 5000.00

-- Check sufficient funds
-- (Application code verifies: 5000 >= 1000 → proceed)

-- Debit source
UPDATE accounts 
SET balance = balance - 1000.00,
    updated_at = NOW()
WHERE id = 'alice'
  AND balance >= 1000.00;  -- Double-check in SQL

-- Verify debit succeeded
-- (Application checks rowsAffected = 1; if 0, Alice had insufficient funds)

-- Credit destination
UPDATE accounts
SET balance = balance + 1000.00,
    updated_at = NOW()
WHERE id = 'bob';

-- Record transaction
INSERT INTO transactions (from_account, to_account, amount, status)
VALUES ('alice', 'bob', 1000.00, 'completed');

COMMIT;
```

If anything between BEGIN and COMMIT fails (insufficient funds, constraint violation, network error), the entire transaction rolls back. Alice keeps her $1000. Bob receives nothing. The ledger balances.

### IAM: Atomic Role Grant

```sql
BEGIN;

-- Grant role (insert to user_roles)
INSERT INTO user_roles (user_id, role_id, granted_at, granted_by)
VALUES ('user_123', 'admin_role', NOW(), 'admin_456');

-- Audit log entry (must succeed with the grant)
INSERT INTO audit_log (action, actor, target, detail, timestamp)
VALUES ('ROLE_GRANTED', 'admin_456', 'user_123', 'admin_role', NOW());

-- Invalidate user's token cache (or record that it needs invalidation)
INSERT INTO token_invalidations (user_id, invalidated_at, reason)
VALUES ('user_123', NOW(), 'role_changed');

COMMIT;
```

All three operations succeed together, or none do. The audit log is always consistent with the actual role state.

---

## Common Failure Modes

### 1. Implicit Auto-Commit
**Scenario:** Developer forgets to wrap operations in a transaction. ORM has auto-commit enabled. Each SQL statement is its own transaction. A crash between statements leaves database inconsistent.  
**Fix:** Always explicit transactions for multi-statement operations. Disable auto-commit for payment operations.

### 2. Long-Running Transactions
**Scenario:** Transaction holds row locks for 30 seconds (doing external API call inside transaction). Other operations queue behind the lock. Performance degrades. Deadlock risk.  
**Fix:** Keep transactions short. Do external calls *outside* the transaction. Only open transaction when you're ready to commit.

```
BAD:
BEGIN;
  SELECT * FROM accounts WHERE id = 'alice' FOR UPDATE;
  call_external_fraud_api()  ← 500ms call INSIDE transaction
  UPDATE accounts...;
COMMIT;

GOOD:
  // Check fraud outside transaction
  fraud_result = call_external_fraud_api()
  if (fraud_result.ok) {
    BEGIN;
    // Fast database operations only
    COMMIT;
  }
```

### 3. Deadlock
**Scenario:**
```
T1: locks row A, then tries to lock row B
T2: locks row B, then tries to lock row A
→ T1 waits for T2. T2 waits for T1. Neither progresses.
```
Database detects deadlock, aborts one transaction.  
**Fix:** Always acquire locks in a consistent order. Keep transactions short. Catch deadlock errors and retry.

### 4. Lost Update
**Scenario:**
```
T1: read balance = 1000
T2: read balance = 1000
T1: write balance = 900 (deducted 100)
T2: write balance = 800 (deducted 200, based on stale read)
Final: 800 — T1's update lost. System lost $200.
```
**Fix:** Pessimistic locking (`SELECT FOR UPDATE`) or optimistic locking (version column check).

---

## Tradeoffs

| Consideration | Impact |
|--------------|--------|
| Atomic transactions | Correct, but slower than non-transactional writes |
| WAL fsync on commit | Durability, but disk I/O on every commit |
| Row-level locking | Better concurrency than table locks, but lock overhead |
| Long transactions | Hold locks, block other operations |
| Distributed transactions (2PC) | Correct across shards, but slow and blocking |

---

## Staff-Level Thinking

ACID is not free. `fsync` on every commit is the price of Durability. Row-level locks are the price of Isolation. Rollback capability requires undo logs.

**Performance tuning levers:**
- `synchronous_commit = off` in PostgreSQL: Asynchronous commit. Lose ~1 commit on crash. Gain 3-5x write throughput. Appropriate for some non-critical workloads. NEVER for payments.
- Group commit: Multiple transactions' WAL flushed in one fsync. Better throughput at the cost of slightly higher latency.

**Staff engineer question when reviewing a design:** *"Is this operation actually inside a transaction? What's the failure mode if the process crashes at line X? Can it leave orphan records?"*

---

# Chapter 12: Isolation Levels — How Much Do You Trust Your Neighbors?

## Interview Question

> *"What is a dirty read? A phantom read? When would you use SERIALIZABLE isolation, and what's the cost?"*

---

## Why Should You Care?

Isolation in ACID says concurrent transactions appear serial. But "appear serial" costs performance. Real databases offer a spectrum of isolation — from "see everything, including in-progress changes" to "perfect isolation." The wrong choice causes data corruption. The right choice is a performance/correctness tradeoff that every production engineer must understand.

---

## Plain English Explanation

If Alice and Bob are both modifying bank accounts simultaneously, isolation determines: **what does Alice see about Bob's changes while Bob is still making them?**

Four levels of answer, from weakest to strongest:
1. **Read Uncommitted:** Alice sees Bob's changes even before Bob commits them.
2. **Read Committed:** Alice only sees Bob's committed changes.
3. **Repeatable Read:** Alice sees a consistent snapshot as of when her transaction started.
4. **Serializable:** Alice and Bob's transactions appear to run one-at-a-time.

---

## Internal Mechanics

### Isolation Anomalies

**Dirty Read:** Reading uncommitted data from another transaction.

```
T1: BEGIN; UPDATE balance = 5000 WHERE id = 'alice';  (not committed)
T2: SELECT balance FROM accounts WHERE id = 'alice';
    → Returns 5000 (T1's uncommitted value)
T1: ROLLBACK;  ← T1 rolls back, balance reverted to 4000
T2 based a decision on data that never actually existed.
```

**Non-Repeatable Read:** Reading same row twice in same transaction yields different results.

```
T1: SELECT balance FROM accounts WHERE id = 'alice';  → 5000
    [T2 commits: UPDATE balance = 4500 WHERE id = 'alice']
T1: SELECT balance FROM accounts WHERE id = 'alice';  → 4500
    ← Different result for same query in same transaction!
```

**Phantom Read:** A query run twice returns different *sets* of rows.

```
T1: SELECT COUNT(*) FROM payments WHERE status = 'pending';  → 10
    [T2 inserts a new pending payment and commits]
T1: SELECT COUNT(*) FROM payments WHERE status = 'pending';  → 11
    ← Different set of rows — the "phantom" row appeared!
```

**Write Skew (not always listed, but critical):**

```
Two doctors both check: "Is there at least one doctor on call?"
Both read: [Dr. Alice on-call, Dr. Bob on-call] → 2 doctors
Both decide: "I can take myself off call"
Both update: themselves to off-call
Result: Zero doctors on call. Neither transaction saw the other's write.
```

### Isolation Levels vs Anomalies

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Write Skew |
|----------------|-----------|---------------------|-------------|-----------|
| Read Uncommitted | Possible | Possible | Possible | Possible |
| Read Committed | Prevented | Possible | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Possible* | Possible |
| Serializable | Prevented | Prevented | Prevented | Prevented |

*PostgreSQL's Repeatable Read uses MVCC and also prevents phantoms in practice.

### MVCC — Multi-Version Concurrency Control

Most modern databases (PostgreSQL, MySQL InnoDB) use MVCC instead of read locks.

**The idea:** Keep multiple versions of each row. Each transaction sees a snapshot of the database as of its start time.

```
Row: accounts(alice)
Version history:
  T_100: balance=5000  ← created at transaction 100
  T_150: balance=4900  ← updated at transaction 150 (committed)
  T_200: balance=4800  ← updated at transaction 200 (in-progress, uncommitted)

Transaction T_160 (started after T_150 committed, before T_200 committed):
  Reads accounts(alice) → sees T_150 version → balance=4900
  T_200 is invisible (not yet committed when T_160 started)
```

MVCC allows readers to never block writers, and writers to never block readers. This is why PostgreSQL read performance is so good under concurrent writes.

**Cost:** Old row versions (called "dead tuples" in PostgreSQL) accumulate. `VACUUM` process reclaims space. If VACUUM can't keep up, table bloat and performance degradation occur.

---

## Production Example

### Payments: Choosing Isolation Level

**Read Committed (PostgreSQL default):**
- Each statement sees the latest committed data
- Sufficient for most read-only queries
- Risky for multi-statement update sequences

**Repeatable Read:**
- Transaction sees consistent snapshot from its start
- Safe for account balance checks + update sequences
- Default choice for payment transactions in PostgreSQL

```sql
-- Payment transfer with Repeatable Read
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Read Alice's balance
SELECT balance FROM accounts WHERE id = 'alice' FOR UPDATE;
-- Returns: 5000 (snapshot from transaction start, locked for update)

-- Even if another transaction commits a change to alice's balance
-- between this SELECT and our UPDATE, our FOR UPDATE lock prevents it.

UPDATE accounts SET balance = balance - 1000 WHERE id = 'alice';
COMMIT;
```

**Serializable:**
- Most expensive. Uses predicate locks (or SSI — Serializable Snapshot Isolation in PostgreSQL).
- Appropriate for: inventory allocation, seat booking, complex invariants
- Automatic retry on serialization failures

```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- All reads and writes participate in conflict detection
-- If PostgreSQL detects a serialization anomaly, it aborts with:
-- ERROR: could not serialize access due to concurrent update
-- Application must catch and retry.
```

### IAM: Token Validation

Token introspection is high-volume and read-only. Read Committed is sufficient:

```sql
-- Token lookup: Read Committed is fine
-- We want the most current committed state
SELECT u.id, u.status, r.name as role
FROM tokens t
JOIN users u ON t.user_id = u.id
JOIN user_roles ur ON u.id = ur.user_id
JOIN roles r ON ur.role_id = r.id
WHERE t.token_hash = $1
  AND t.expires_at > NOW()
  AND u.status = 'active';
```

If a user is deactivated between two reads of the same token, Read Committed ensures the second read sees the deactivation. That's the correct behavior — revocation should take effect immediately.

---

## Common Failure Modes

### 1. Using Read Committed for Balance Transfers
**Scenario:** Two transfers read Alice's balance simultaneously (both see $1000). Both deduct $1000. Both write $0. Correct final balance: -$1000 (double spend). Read Committed didn't protect against this non-repeatable read.  
**Fix:** Use `SELECT FOR UPDATE` or Repeatable Read isolation.

### 2. Write Skew in Booking Systems
**Scenario:** Seat booking: two users both check "is seat 14A available?" Both see "yes." Both book. Two people booked the same seat.  
**Fix:** Serializable isolation. Or explicit lock: `SELECT ... FOR UPDATE`.

### 3. Long VACUUM Delays (PostgreSQL)
**Scenario:** Long-running transactions prevent VACUUM from reclaiming dead tuples. Table grows to 100GB of garbage. All queries slow down.  
**Fix:** Avoid long-running read transactions. Monitor `pg_stat_user_tables.n_dead_tup`. Set statement_timeout.

---

## Staff-Level Thinking

**Default to Read Committed for reads, Repeatable Read for updates, Serializable for complex invariants.** But understand the performance cost of each.

**Serializable Snapshot Isolation (SSI)** in PostgreSQL 9.1+ is a significant advancement — it detects serialization conflicts without true serialization overhead most of the time. But the application must handle retry logic on serialization errors. This is non-trivial to implement correctly.

**Key interview signal:** A Staff Engineer knows that isolation is not binary. They can explain which specific anomaly they're protecting against for each transaction in their design, and justify the isolation level they chose.

---

# Chapter 13: Indexes — The Art of Finding Fast

## Interview Question

> *"Explain how a B-Tree index works. Why does `SELECT * FROM users WHERE last_name = 'Smith'` run in O(log n) with an index but O(n) without? What's the cost of having too many indexes?"*

---

## Why Should You Care?

Indexes are the single most impactful performance tool in a relational database. Missing an index turns a 10ms query into a 10-second full table scan. The wrong index wastes write performance. Understanding indexes separates engineers who "add an index" from engineers who understand *why* and *when*.

---

## Plain English Explanation

A database without indexes is like a phone book with no alphabetical order. To find "Smith," you read every single entry from page 1.

An index is a sorted copy of a column (or columns), with pointers to the full row. Like the phone book's alphabetical order: jump to "S," then "Sm," then find "Smith" in O(log n) steps.

---

## Internal Mechanics

### B-Tree Index (Default)

The most common index type. A balanced tree of sorted values.

```
B-Tree for payments.user_id:

                    [500]
                   /     \
             [250]         [750]
            /    \         /    \
         [100] [400]    [600]  [900]
         / \   / \      / \    / \
       [..][..][..][..][..][..][..][..]
       Leaf nodes: contain (value, row_pointer) pairs

Leaf nodes are linked: [100]→[250]→[400]→[500]→[600]→[750]→[900]
↑ enables range scans efficiently
```

**Lookup: WHERE user_id = 500**
1. Start at root: 500 = root value
2. Found! Follow pointer to data row.
3. Cost: O(log n) — typically 3-4 comparisons for a million rows

**Range Scan: WHERE user_id BETWEEN 400 AND 600**
1. Find 400 in tree (log n)
2. Follow linked leaf nodes: 400→500→600
3. Collect all matching rows
4. Cost: O(log n + k) where k = matching rows

**Full Table Scan (no index): WHERE email = 'alice@bank.com'**
1. Read every row. Compare email.
2. Cost: O(n) — for 10 million rows, 10 million comparisons.

### Index Types in PostgreSQL

```
B-Tree:    Default. Equality, range, ORDER BY. Works for most cases.
Hash:      Equality only. Faster lookups than B-Tree for exact match.
           Not crash-safe before PostgreSQL 10. Rarely used.
GIN:       Full-text search, arrays, JSONB. Multi-value columns.
GiST:      Geometric data, range types, full-text. Generalized Search Tree.
BRIN:      Block Range INdex. Huge sequential tables (time-series).
           Very small index, good for append-only data sorted by time.
Partial:   Index only a subset of rows.
Expression: Index on a function result.
```

### Composite Indexes (Multi-Column)

```sql
CREATE INDEX idx_payments_user_status ON payments (user_id, status);
```

**Left-prefix rule:** The composite index is usable only if the query references columns from the *left* side of the index.

```sql
-- Uses index (both columns):
WHERE user_id = 123 AND status = 'pending'

-- Uses index (left prefix):
WHERE user_id = 123

-- Does NOT use index (skips user_id):
WHERE status = 'pending'

-- Range on first column; can't use index for second:
WHERE user_id > 100 AND status = 'pending'
-- Only uses index for user_id > 100; filters status in memory
```

### Covering Indexes (Index-Only Scans)

If an index contains *all* columns needed by a query, PostgreSQL can answer the query from the index alone — never touching the actual table.

```sql
-- Frequent query pattern:
SELECT user_id, status, created_at FROM payments WHERE user_id = 123;

-- Covering index:
CREATE INDEX idx_payments_covering ON payments (user_id, status, created_at);

-- Now: SELECT reads only the index, not the table. Faster.
```

### Partial Indexes

Index only the rows you actually query:

```sql
-- 99% of payments are 'completed'. You only query 'pending'.
-- Full index has 10M rows. Only 100K are 'pending'.

CREATE INDEX idx_payments_pending ON payments (user_id)
WHERE status = 'pending';
-- Index has only 100K entries. Much smaller, much faster.
```

### Expression Indexes

```sql
-- Frequently search by lowercase email (case-insensitive login):
CREATE INDEX idx_users_lower_email ON users (LOWER(email));

-- Now this uses the index:
WHERE LOWER(email) = 'alice@bank.com'

-- Without it, this wouldn't use a regular email index:
CREATE INDEX idx_users_email ON users (email);
WHERE LOWER(email) = 'alice@bank.com'  -- function = can't use index
```

---

## Production Example

### IAM: Token Lookup Index Strategy

Token validation is on the critical path of every API request. It must be fast.

```sql
-- Token storage
CREATE TABLE access_tokens (
    id          UUID PRIMARY KEY,
    token_hash  BYTEA        NOT NULL,  -- hashed value
    user_id     UUID         NOT NULL,
    expires_at  TIMESTAMPTZ  NOT NULL,
    scopes      TEXT[]       NOT NULL,
    revoked_at  TIMESTAMPTZ            -- NULL = not revoked
);

-- Index for lookup by token hash
CREATE UNIQUE INDEX idx_tokens_hash ON access_tokens (token_hash);

-- Partial index: only active (non-expired, non-revoked) tokens
CREATE INDEX idx_tokens_active ON access_tokens (user_id, expires_at)
WHERE revoked_at IS NULL;

-- Token cleanup: find all expired tokens
CREATE INDEX idx_tokens_expires ON access_tokens (expires_at)
WHERE expires_at < NOW();
-- Better: BRIN index if tokens are inserted in time order
CREATE INDEX idx_tokens_expires_brin ON access_tokens USING BRIN (expires_at);
```

### Payments: Composite Index Design

```sql
-- Common query: user's recent payments, by status
SELECT * FROM payments
WHERE user_id = 123
  AND status = 'pending'
  AND created_at > NOW() - INTERVAL '30 days'
ORDER BY created_at DESC
LIMIT 20;

-- Optimal index:
CREATE INDEX idx_payments_user_status_date
ON payments (user_id, status, created_at DESC);
-- user_id (equality) → status (equality) → created_at (range + sort)
-- All in one index. No sort needed. Covers the ORDER BY.
```

---

## Index Costs — Why You Can't Just Index Everything

**Write overhead:** Every INSERT, UPDATE, DELETE must update all affected indexes. 10 indexes = 10 index tree updates per write. Writes slow down proportionally.

```
Table with 1 index:  INSERT costs 1 tree update
Table with 10 indexes: INSERT costs 10 tree updates
→ 10x write overhead for excessive indexing
```

**Storage:** Indexes take disk space. A heavily indexed table can have indexes larger than the table itself.

**Planner confusion:** Too many indexes confuse the query planner. It may choose a suboptimal index or spend time comparing many options.

**Rule:** Only create indexes that serve specific, measured, important queries.

---

## Common Failure Modes

### 1. Missing Index on High-Cardinality Foreign Key
**Scenario:** `payments.user_id` references `users.id`. No index on `user_id`. Query for all payments for user 123: full table scan of 50M rows.  
**Fix:** Always index foreign keys. They're almost always queried.

### 2. Low-Selectivity Index (Waste of Space)
**Scenario:** Index on `payments.currency` column. Only 3 distinct values (USD, EUR, GBP). Index for `WHERE currency = 'USD'` returns 70% of rows. Full table scan would be faster (less I/O overhead).  
**Fix:** Indexes work best on high-cardinality columns. Skip low-cardinality ones.

### 3. Index Invalidated by Function in WHERE Clause
**Scenario:** `CREATE INDEX idx_email ON users(email)`. Query: `WHERE LOWER(email) = 'alice@bank.com'`. Index not used — function applied to indexed column.  
**Fix:** Create expression index on `LOWER(email)`.

### 4. Bloated Indexes After Many Deletes
**Scenario:** Millions of rows deleted. B-Tree pages become partially empty. Index is large but sparse. Query performance degrades.  
**Fix:** `REINDEX CONCURRENTLY` (PostgreSQL 12+). Or `VACUUM` with index cleanup.

---

## Staff-Level Thinking

Index design is not a one-time activity. It's a feedback loop:

1. Identify slow queries via slow query log or `pg_stat_statements`
2. `EXPLAIN ANALYZE` to see what the planner does
3. Identify missing indexes or bad plans
4. Create or drop indexes
5. Monitor write performance impact
6. Repeat

**Staff engineer heuristic:** The question isn't "should I add an index?" It's "what's the query pattern, what's the cardinality, what's the read:write ratio, and what's the cost of this index on write throughput?"

---

# Chapter 14: Query Optimization — Teaching the Database to Think

## Interview Question

> *"Your query returns correct results but takes 30 seconds. Walk me through how you would diagnose and fix it."*

---

## Why Should You Care?

Writing SQL is easy. Writing *fast* SQL is hard. Queries that work fine in development take minutes in production (10M rows). Understanding query optimization lets you diagnose and fix these problems systematically.

---

## Plain English Explanation

When you run a SQL query, the database doesn't execute it literally. A component called the **query planner** (or query optimizer) decides the best way to execute it. It considers:

- Which indexes exist?
- How many rows are in each table?
- What does the WHERE clause filter to?
- Is it cheaper to scan the table or use an index?

The planner makes a **query plan**: a tree of operations describing how to get your data. Your job is to make the planner's choices good.

---

## Internal Mechanics

### EXPLAIN and EXPLAIN ANALYZE

```sql
EXPLAIN SELECT * FROM payments WHERE user_id = 123 AND status = 'pending';

-- Output:
Index Scan using idx_payments_user_status on payments
    (cost=0.43..12.50 rows=5 width=120)
  Index Cond: ((user_id = 123) AND (status = 'pending'))
```

**EXPLAIN** shows the *planned* execution. No query is actually run.  
**EXPLAIN ANALYZE** runs the query and shows *actual* results alongside estimates:

```sql
EXPLAIN ANALYZE SELECT * FROM payments WHERE user_id = 123 AND status = 'pending';

-- Output:
Index Scan using idx_payments_user_status on payments
    (cost=0.43..12.50 rows=5 width=120)
    (actual time=0.082..0.094 rows=3 loops=1)
  Index Cond: ((user_id = 123) AND (status = 'pending'))
Planning Time: 0.234 ms
Execution Time: 0.118 ms
```

**Key numbers to watch:**
- `cost`: planner's estimate (relative units, not seconds)
- `rows`: estimated vs actual — large discrepancy = stale statistics
- `actual time`: real milliseconds per operation

### Reading a Query Plan

```
Nested Loop  (cost=0..500)
  → Hash Join  (cost=0..200)
       → Seq Scan on users  (cost=0..100)
       → Hash  (cost=0..50)
            → Index Scan on accounts  (cost=0..50)
  → Seq Scan on payments  (cost=0..100)
```

Read from innermost to outermost. The database executes from bottom to top.

### Seq Scan vs Index Scan

```
Seq Scan (Sequential Scan):
  → Reads every row in the table.
  → Fast when fetching large fraction of table (>10-20% of rows).
  → Slow for selective queries on large tables.

Index Scan:
  → Reads index, then fetches individual rows.
  → Fast when fetching small fraction of table.
  → Slow when fetching many rows (random I/O worse than sequential).

Bitmap Index Scan:
  → Reads index to build bitmap of matching row positions.
  → Then reads table in sequential order.
  → Best for medium selectivity (fetching 1-10% of table).
```

### Statistics and the Planner

The planner relies on statistics to estimate row counts. Statistics are maintained by `ANALYZE` (run by `autovacuum` automatically).

```sql
-- View table statistics
SELECT tablename, n_live_tup, n_dead_tup, last_analyze
FROM pg_stat_user_tables;

-- Manually update statistics
ANALYZE payments;

-- Adjust statistics detail (default: 100 most common values)
ALTER TABLE payments ALTER COLUMN status SET STATISTICS 500;
```

**Stale statistics = bad plans.** If the planner thinks `status = 'pending'` returns 5 rows but actually returns 500,000, it may choose an index scan when a seq scan would be better.

---

## Production Example

### Diagnosing a Slow Payment Query

```sql
-- Complaint: "User payment history page takes 30 seconds"
-- Query:
SELECT p.id, p.amount, p.status, p.created_at, m.name as merchant_name
FROM payments p
JOIN merchants m ON p.merchant_id = m.id
WHERE p.user_id = 12345
ORDER BY p.created_at DESC
LIMIT 20;
```

**Step 1: EXPLAIN ANALYZE**
```
Sort (cost=50000..50010 rows=10 width=200)
    (actual time=28500..28510 rows=20 loops=1)
  Sort Key: p.created_at DESC
  → Seq Scan on payments  (cost=0..49000 rows=10000 width=180)
       (actual time=0.1..27000 rows=8000 loops=1)
    Filter: (user_id = 12345)
    Rows Removed by Filter: 9992000
  → Hash Join
       → Seq Scan on merchants
```

**Diagnosis:**
- `Seq Scan on payments` reading 10M rows, filtering to 8000
- Estimated rows = 10000, actual = 8000 (planner thought only 10, not 8000 → stale stats)
- No index on `user_id`
- No index on `created_at` for ORDER BY

**Fix 1: Add index**
```sql
CREATE INDEX idx_payments_user_date ON payments (user_id, created_at DESC);
ANALYZE payments;
```

**After fix, EXPLAIN ANALYZE:**
```
Limit (cost=0.56..12 rows=20 width=200)
    (actual time=0.3..0.8 rows=20 loops=1)
  → Index Scan Backward using idx_payments_user_date on payments
       (cost=0.56..5000 rows=8000 width=180)
       (actual time=0.2..0.5 rows=20 loops=1)
    Index Cond: (user_id = 12345)
  → Hash Join...
Execution Time: 1.2 ms
```

30 seconds → 1.2 milliseconds. From index.

### Common Query Optimization Patterns

```sql
-- 1. Use LIMIT aggressively for paginated queries
SELECT ... ORDER BY created_at DESC LIMIT 20 OFFSET 0;
-- Not: SELECT * then paginate in application code

-- 2. Filter early with WHERE, let JOIN on smaller result
SELECT ...
FROM payments p
WHERE p.user_id = 123    -- filter before join
JOIN merchants m ON p.merchant_id = m.id;

-- 3. Avoid functions on indexed columns in WHERE
-- Bad (index not used):
WHERE DATE(created_at) = '2026-01-01'
-- Good (index used):
WHERE created_at >= '2026-01-01' AND created_at < '2026-01-02'

-- 4. Use EXISTS instead of COUNT for existence checks
-- Bad:
SELECT COUNT(*) FROM payments WHERE user_id = 123 HAVING COUNT(*) > 0;
-- Good:
SELECT EXISTS(SELECT 1 FROM payments WHERE user_id = 123);

-- 5. Avoid SELECT * in production
-- Bad: returns all columns, wider rows, more I/O
SELECT * FROM payments WHERE user_id = 123;
-- Good: only needed columns
SELECT id, amount, status, created_at FROM payments WHERE user_id = 123;
```

---

## Staff-Level Thinking

Query optimization is not a one-time fix. It's an ongoing operational practice:

**Tooling:**
- `pg_stat_statements`: Top slow queries across all connections
- `auto_explain`: Automatically log plans for slow queries
- Datadog/Prometheus: Database query latency percentiles
- pgBadger: Log analysis and visualization

**The three causes of slow queries (in order of frequency):**
1. Missing index (80% of cases)
2. Stale statistics causing bad plan
3. Genuinely expensive query that needs redesign

**When a query is genuinely expensive:**
- Caching: Can this result be cached? (Chapter 21)
- CQRS: Is this a read that should be served from a read replica or a separate read model? (Chapter 55)
- Denormalization: Should a precomputed value be stored? (Chapter 10)
- Batch: Does this need to be real-time, or can it be a background job?

A Staff Engineer doesn't just optimize queries. They question whether the query should exist at all.
# Chapter 15: Partitioning — Cutting the Table

## Interview Question

> *"Your payments table has 2 billion rows and queries are slow despite good indexes. What is table partitioning and when does it help?"*

---

## Why Should You Care?

At scale, even indexed tables hit limits. A 2-billion-row table with a 50GB index takes minutes to scan a date range. Partitioning splits a large table into smaller physical pieces, allowing the database to skip entire partitions for queries that filter on the partition key.

---

## Plain English Explanation

Imagine a filing cabinet with 10 million documents in one drawer. Finding documents from 2025 requires searching every document.

Partitioning gives each year its own drawer: 2023 drawer, 2024 drawer, 2025 drawer. To find 2025 documents, open the 2025 drawer only. The other drawers are never touched.

The table looks like one table to the application. Internally, it's multiple physical storage units.

---

## Internal Mechanics

### Partitioning Types

**Range Partitioning (most common for time-series)**

```sql
-- Create partitioned payments table
CREATE TABLE payments (
    id          UUID        NOT NULL,
    user_id     BIGINT      NOT NULL,
    amount      DECIMAL(15,2) NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE payments_2026_01 PARTITION OF payments
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE payments_2026_02 PARTITION OF payments
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
CREATE TABLE payments_2026_03 PARTITION OF payments
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
-- ... and so on

-- Query automatically routes to correct partition(s):
SELECT * FROM payments
WHERE created_at >= '2026-01-01'
  AND created_at < '2026-02-01';
-- Only scans payments_2026_01 — partition pruning!
```

**List Partitioning (by discrete values)**

```sql
CREATE TABLE payments PARTITION BY LIST (currency);

CREATE TABLE payments_usd PARTITION OF payments
    FOR VALUES IN ('USD');
CREATE TABLE payments_eur PARTITION OF payments
    FOR VALUES IN ('EUR');
CREATE TABLE payments_gbp PARTITION OF payments
    FOR VALUES IN ('GBP');
```

**Hash Partitioning (by hash of column, for even distribution)**

```sql
CREATE TABLE payments PARTITION BY HASH (user_id);

-- 4 partitions, each gets ~25% of users
CREATE TABLE payments_0 PARTITION OF payments
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE payments_1 PARTITION OF payments
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);
-- ...
```

### Partition Pruning

The key benefit: the query planner recognizes when a WHERE clause restricts results to specific partitions. Non-matching partitions are completely skipped.

```sql
EXPLAIN SELECT * FROM payments
WHERE created_at >= '2026-01-01'
  AND created_at < '2026-02-01';

-- Output:
Append
  → Seq Scan on payments_2026_01
  (Partitions payments_2026_02, 2026_03... excluded by pruning)
```

Without partitioning: scan all 2 billion rows.  
With partitioning: scan 1 month's partition (~5-10 million rows).

### Partition Maintenance

```sql
-- Drop old data by dropping entire partition (instant, no VACUUM needed):
DROP TABLE payments_2022_01;
-- vs: DELETE FROM payments WHERE created_at < '2022-02-01'; (slow, leaves dead tuples)

-- Add future partition before data arrives:
CREATE TABLE payments_2027_01 PARTITION OF payments
    FOR VALUES FROM ('2027-01-01') TO ('2027-02-01');

-- Automate with pg_partman extension
```

---

## Production Example

### Payments: Time-Based Partitioning

A payment processor stores 2 billion transactions going back 5 years. Monthly partitioning:
- Each partition: ~33M rows (2B / 60 months)
- Queries for "last 30 days" scan only 1-2 partitions
- Old data (>2 years) can be archived by dropping partitions
- VACUUM runs per-partition (parallelized)

**Global indexes on partitioned tables:** PostgreSQL supports global unique indexes on partitioned tables from v11+. Critical for UUID primary keys — uniqueness must be enforced across all partitions.

### IAM: Audit Log Partitioning

Audit logs are write-heavy, read rarely (only during investigation). Partition by month:

```sql
-- Compliance: keep 7 years of audit logs
-- Queries mostly last 90 days
-- Old data: compressed or archived to cold storage

CREATE TABLE audit_log PARTITION BY RANGE (created_at);
-- Monthly partitions
-- Old partitions: pg_partman moves to tablespace on cheap disks
-- Very old partitions: export to S3, drop table
```

---

## Common Failure Modes

### 1. Partition Key Not in WHERE Clause
**Scenario:** Partitioned on `created_at` but query `WHERE user_id = 123` doesn't include `created_at`. All partitions scanned. No benefit, extra overhead.  
**Fix:** Partition on the column(s) actually used in WHERE clauses.

### 2. Missing Partition Before Data Arrives
**Scenario:** Monthly partitions. January 31 passes. No February partition exists. February 1st inserts fail.  
**Fix:** Automate partition creation. Create next month's partition at start of current month.

### 3. Cross-Partition Joins
**Scenario:** JOIN between partitioned and non-partitioned table without partition key. Planners scans all partitions in nested loop.  
**Fix:** Include partition key in JOIN conditions where possible. Or denormalize.

---

## Staff-Level Thinking

Partitioning is *horizontal* splitting of a table within one database. Sharding (Chapter 17) is splitting a table *across multiple databases*. 

**Choose partitioning when:**
- Table is very large (>100GB, >500M rows)
- Query patterns have a clear range filter (time is the most common)
- You need cheap bulk deletes of old data
- VACUUM is struggling (partitions can be vacuumed independently)

**Don't partition when:**
- Table is medium-sized (index is the right fix)
- Query patterns don't match the partition key
- You have complex cross-partition foreign keys

---

# Chapter 16: Replication — Keeping Copies

## Interview Question

> *"Your payment database goes down. What happens? How do you design for database high availability? What does synchronous vs asynchronous replication mean for your system's guarantees?"*

---

## Why Should You Care?

A database without replication is a single point of failure. Replication solves two problems: high availability (survive a failure) and read scalability (distribute read load). But replication introduces its own complexity: consistency lag, failover management, and split-brain scenarios.

---

## Plain English Explanation

Replication means keeping copies of your database on multiple servers, constantly synchronized.

When the primary database fails, one of the copies (replicas) can take over. This is high availability.

Read queries can be served from replicas — if you can tolerate reads that might be slightly behind the primary. This is read scalability.

The tradeoff: copies are never perfectly synchronized in real time. Some lag exists. How much lag you can tolerate defines which replication mode you use.

---

## Internal Mechanics

### Streaming Replication (PostgreSQL)

```
Primary Database
  ↓  WAL (Write-Ahead Log) stream
  ├──→ Replica 1 (sync:  primary waits for ACK before confirming write)
  ├──→ Replica 2 (async: primary doesn't wait; replica may lag)
  └──→ Replica 3 (async)

WAL stream:
  Primary writes WAL → sends to replicas → replicas replay WAL → replica catches up
```

**Physical replication:** Exact byte-for-byte copy. No transformation. Simplest.  
**Logical replication:** Replicate specific tables or transformed data. Cross-version. More flexible.

### Synchronous vs Asynchronous Replication

**Asynchronous (default):**
```
Client → Primary: INSERT payment (100ms)
         Primary: WAL written, commit confirmed ← response to client here
         Primary → Replica: WAL sent (background, ~milliseconds later)
```

- Client gets confirmation immediately after primary commits
- Replica may lag by milliseconds to seconds
- If primary crashes before WAL is sent, replica doesn't have that data
- **Risk:** Data loss window = replication lag at moment of failure

**Synchronous:**
```
Client → Primary: INSERT payment
         Primary: WAL written
         Primary → Replica: WAL sent, waiting...
         Replica: WAL received, ACK sent
         Primary: commit confirmed ← response to client here
```

- Client waits for at least one replica to acknowledge
- No data loss on primary failure (replica has the data)
- **Cost:** Write latency = primary commit time + replica ACK round trip

**PostgreSQL configuration:**
```
# Require 1 synchronous replica:
synchronous_standby_names = 'FIRST 1 (replica1, replica2)'

# Require 2 synchronous replicas:
synchronous_standby_names = 'FIRST 2 (replica1, replica2, replica3)'
```

### Replication Lag

```sql
-- Check replication lag on primary:
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    (sent_lsn - replay_lsn) AS bytes_lag,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```

**Lag causes:** Replica overloaded, network slow, replay bottleneck (single-threaded in older PG), long transactions on replica.

### Failover

When the primary fails:

```
1. Detect failure (health check, timeout)
2. Choose new primary (highest LSN replica = least data loss)
3. Promote chosen replica to primary
4. Update application connection strings
5. Other replicas repoint to new primary
```

**Automatic failover tools:**
- **Patroni** (most common): Leader election via etcd/Consul/ZooKeeper. Automatic promotion.
- **pg_auto_failover**
- **AWS RDS Multi-AZ**: Managed automatic failover (~30-60 seconds)

```
Normal:
  App → HAProxy/pgBouncer → Primary (us-east-1a)
                           ↓ sync replication
                         Replica (us-east-1b)

Failure (Primary crashes):
  HAProxy health check fails on primary
  Patroni detects: no heartbeat from primary
  Patroni promotes Replica as new Primary
  HAProxy updated: route to new Primary
  App connects to new Primary
  
  Downtime: ~10-30 seconds (detection + promotion)
```

---

## Production Example

### Payments: Replication Topology

```
           ┌─────────────────────────┐
           │   Primary (writer)       │
           │   us-east-1a             │
           └────────┬────────────────┘
                    │
         ┌──────────┴──────────┐
         │ sync replication     │ async replication
         ▼                      ▼
  ┌──────────────┐    ┌──────────────────┐
  │ Sync Replica │    │ Async Replica    │
  │ us-east-1b   │    │ us-west-2        │
  │ (HA standby) │    │ (disaster recov) │
  └──────────────┘    │ (read traffic)   │
                      └──────────────────┘
                               │
                      ┌────────┴────────┐
                      │ Async Replica   │
                      │ us-west-2b      │
                      │ (analytics)     │
                      └─────────────────┘
```

- **Primary + Sync Replica:** Zero data loss failover within AZ
- **Async us-west-2:** Cross-region disaster recovery (lag: 50-200ms)
- **Async analytics:** Heavy reporting queries routed here, don't impact primary

### IAM: Read-Heavy Replication

Token introspection is read-heavy (millions of reads, few writes). Route reads to replicas:

```java
// DataSource routing in Spring Boot IAM service
@Bean
public DataSource routingDataSource() {
    Map<Object, Object> dataSources = new HashMap<>();
    dataSources.put("PRIMARY", primaryDataSource());
    dataSources.put("REPLICA", replicaDataSource());
    
    ReplicaAwareDataSource ds = new ReplicaAwareDataSource();
    ds.setTargetDataSources(dataSources);
    ds.setDefaultTargetDataSource(primaryDataSource());
    return ds;
}

// Route token lookups to replica (read-only, stale tolerance)
@Transactional(readOnly = true)
public TokenInfo introspect(String token) {
    // Spring routes to REPLICA based on @Transactional(readOnly=true)
    return tokenRepository.findByHash(hash(token));
}

// Route token writes to primary
@Transactional
public void revokeToken(String tokenId) {
    tokenRepository.revoke(tokenId);  // Goes to PRIMARY
}
```

**Caveat:** Read-your-writes consistency. If a user just changed their password, the revoke goes to primary. The next token introspection (from replica) must NOT serve a cached valid token. Solution: route reads to primary for N seconds after a write that affects this user.

---

## Common Failure Modes

### 1. Replica Lag During High Write Load
**Scenario:** Primary handles 10,000 writes/second. Replica replay is single-threaded (pre-PG13). Replica falls 5 seconds behind. Application reads stale data.  
**Fix:** Parallel logical replication (PG13+). Or accept eventual consistency on reads and design accordingly.

### 2. Split Brain
**Scenario:** Network partition. Both primary and replica think they're the primary. Both accept writes. Writes diverge.  
**Fix:** Quorum-based promotion. Only promote if majority of nodes agree. Use Patroni + DCS (Distributed Configuration Store like etcd).

### 3. Failover Too Slow
**Scenario:** Primary crashes. Health check interval is 30 seconds. Promotion takes 30 seconds. 60 seconds of downtime for a payments system.  
**Fix:** Tune health check intervals (5-10 seconds). Pre-promote standby to "hot standby" state.

### 4. Cascading Replica Lag
**Scenario:** One slow query on primary holds locks. Replica must replay in same order. Replay blocked on same lock. All replica traffic stacked.  
**Fix:** Identify and kill slow queries on primary. Monitor lock waits on replica.

---

## Staff-Level Thinking

**RPO (Recovery Point Objective):** Maximum data loss acceptable.
- Async replication: RPO = replication lag at failure time (milliseconds to seconds)
- Sync replication: RPO ≈ 0 (data loss only if both primary and sync replica fail simultaneously)

**RTO (Recovery Time Objective):** Maximum downtime acceptable.
- Manual failover: minutes to hours (unacceptable for payments)
- Automatic failover (Patroni): 10-60 seconds
- Managed (RDS Multi-AZ): 30-60 seconds

**Payments require:** RPO = 0, RTO < 30 seconds.  
**This demands:** Synchronous replication to at least one replica. Automatic failover.

---

# Chapter 17: Sharding — Splitting the Database Itself

## Interview Question

> *"Your payment database can't scale vertically anymore. You have 50TB of data and 100,000 writes per second. How do you shard? What problems does sharding introduce?"*

---

## Why Should You Care?

Partitioning splits a table within one database. Sharding splits data *across multiple independent databases*. It's the solution when one machine cannot hold all your data or handle all your write load. It's also one of the most operationally complex things you can do to a database.

---

## Plain English Explanation

Imagine a single bank branch handling 1 million customers. Lines are long. Solution: open 10 branches. Customers A-J go to branch 1. K-T to branch 2. U-Z to branch 3.

Each branch is independent. Has its own staff (database server). Serves its customers faster.

Problem: customer wanting a joint account needs to visit two branches. Complex coordination.

Sharding is the same idea: split customers across multiple databases. Each database (shard) serves its subset. But cross-shard operations become very complex.

---

## Internal Mechanics

### Sharding Strategies

**Range-based sharding:**
```
Shard 1: user_id 1 - 10,000,000
Shard 2: user_id 10,000,001 - 20,000,000
Shard 3: user_id 20,000,001 - 30,000,000
```

- Simple to understand and route
- Uneven distribution possible (hot shards)
- Range queries (user 1-1000) stay on one shard

**Hash-based sharding:**
```
shard_id = hash(user_id) % num_shards

user_id=123  → hash=abc → abc % 4 = 1 → Shard 1
user_id=456  → hash=def → def % 4 = 2 → Shard 2
user_id=789  → hash=ghi → ghi % 4 = 3 → Shard 3
```

- Even distribution (hash distributes uniformly)
- Range queries span all shards (bad for "users 1-1000")
- Resharding (changing num_shards) requires rehashing all keys

**Consistent hashing (Chapter 41):**
- Minimizes data movement when shards are added/removed
- Used by Cassandra, DynamoDB

**Directory-based sharding:**
```
Routing table (in Redis or database):
  user_id 123 → Shard 2
  user_id 456 → Shard 1
  user_id 789 → Shard 4
```

- Maximum flexibility (move any user to any shard)
- Routing table becomes critical bottleneck/single point of failure

### Sharding Architecture

```
Application Server
       │
       │ "I need user 12345's payments"
       ▼
Shard Router / Query Router
       │
       │ hash(12345) % 4 = 1 → Shard 1
       ▼
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│  Shard 1 │  │  Shard 2 │  │  Shard 3 │  │  Shard 4 │
│ users    │  │ users    │  │ users    │  │ users    │
│ 0-25%    │  │ 25-50%   │  │ 50-75%   │  │ 75-100%  │
└──────────┘  └──────────┘  └──────────┘  └──────────┘
```

Each shard is a complete, independent database with its own primary/replica setup.

### Cross-Shard Queries — The Hard Problem

```
Single-shard query (easy):
  "Get user 123's last 20 payments"
  → hash(123) = shard 1
  → Query shard 1 only

Cross-shard query (hard):
  "Get all pending payments across all users for last hour"
  → Must query ALL shards
  → Aggregate results in application
  → Can't use SQL ORDER BY / LIMIT efficiently
```

Cross-shard queries require scatter-gather: send query to all shards, collect results, sort/merge in application.

**Scatter-Gather:**
```
App Server
  ├──→ Shard 1: "pending payments last hour"
  ├──→ Shard 2: "pending payments last hour"
  ├──→ Shard 3: "pending payments last hour"
  └──→ Shard 4: "pending payments last hour"
  ↓
  Wait for all responses
  Merge: combine all results
  Sort: sort combined list by time
  Limit: take top 100
```

Latency = slowest shard (not average). If one shard is slow or unavailable, the entire query is delayed.

### Resharding

Adding a new shard requires redistributing data:

```
Before: 4 shards, hash % 4
After:  5 shards, hash % 5

All keys must be re-evaluated:
  key that was in shard 1 (hash % 4 = 1) might now be shard 3 (hash % 5 = 3)
  Must move key from shard 1 to shard 3

Resharding 1TB: may take hours/days with online migration
```

**Consistent hashing** minimizes this: only 1/N keys move when adding the N+1th shard.

---

## Production Example

### Payments: Sharding by User ID

User ID is the natural shard key for payments. All payments for a user are on the same shard. Most queries are "payments for user X" — single-shard.

```
Shard key: user_id
Num shards: 16 (fixed, with enough headroom)
Routing: shard_id = murmur3(user_id) % 16

Each shard:
  Primary: 32-core, 256GB RAM, 10TB NVMe SSD
  Replicas: 2 async + 1 sync (Chapter 16)
  Capacity per shard: ~3TB data, ~6000 writes/second

Total: 16 × 10TB = 160TB capacity, 96,000 writes/second
```

**What's hard:**
- "Total revenue today": scatter-gather across 16 shards
- "Fraud detection": user A paid merchant X — is merchant X's total >$10K across all users? Cross-shard aggregation
- Solution: Move analytics to a separate non-sharded read model (Kafka → data warehouse)

### IAM: Why Sharding Is Often Avoided

IAM data volumes are typically much smaller than payment transaction volumes. A company with 10 million users has:
- Users table: 10M rows × 500 bytes = 5GB
- Roles/permissions: megabytes

5GB fits on a single Postgres instance with replicas. No sharding needed. The complexity of sharding is unwarranted until you genuinely can't fit on one machine.

**Staff Engineer principle:** Never shard until you absolutely must. The operational complexity is enormous.

---

## Common Failure Modes

### 1. Hot Shard
**Scenario:** Hash function distributes evenly. But some users are much more active than others (celebrities, businesses). Shard containing "amazon_account" gets 10x normal traffic.  
**Fix:** Application-level request throttling per shard. Or split hot accounts to dedicated shards (directory-based routing).

### 2. Cross-Shard Transaction Nightmare
**Scenario:** Transfer money from user A (shard 1) to user B (shard 3). Must debit shard 1 and credit shard 3 atomically. No ACID guarantee across shards.  
**Fix:** Saga pattern (Chapter 56). Two-phase commit (risky). Or design to avoid cross-shard transactions (keep related data together).

### 3. Schema Changes Across All Shards
**Scenario:** Add column to `payments` table. Must run migration on 16 shards. One shard fails midway. Application code now running against inconsistent schemas.  
**Fix:** Backwards-compatible migrations. Feature flags. Schema migration tooling that handles partial failures.

---

## Staff-Level Thinking

**Alternatives to sharding (try these first):**
1. Vertical scale (bigger machine)
2. Read replicas (scale reads)
3. Caching (reduce database load)
4. Partitioning (within one database)
5. Archiving old data (reduce table size)

**Only shard when you've exhausted the above.**

**If you must shard:**
- Choose shard key carefully (data colocated on same shard for common queries)
- Avoid cross-shard operations in hot paths
- Build scatter-gather for analytics queries
- Plan for resharding from day one (consistent hashing or overshard and subdivide)
- Instrument per-shard metrics separately

---

# Chapter 18: PostgreSQL Fundamentals

## Why Should You Care?

PostgreSQL is the default choice for relational workloads at most modern tech companies. Understanding its specific features and tuning knobs is essential for any system design involving a relational database.

---

## What Makes PostgreSQL Different

**MVCC (Multi-Version Concurrency Control):** Readers never block writers. Writers never block readers. This is PostgreSQL's defining characteristic for concurrency.

**Extensibility:** Custom data types, custom operators, extensions (PostGIS, pg_vector, TimescaleDB, Citus). PostgreSQL is a platform, not just a database.

**ACID Compliance:** Full ACID with WAL-based durability. No compromises.

**Rich SQL:** Window functions, CTEs (Common Table Expressions), JSON/JSONB support, array types, full-text search, lateral joins.

### Key PostgreSQL Features for System Design

**JSONB — Flexible Schema Within Relational**
```sql
-- Store flexible metadata in JSONB:
CREATE TABLE payments (
    id       UUID PRIMARY KEY,
    amount   DECIMAL(15,2),
    metadata JSONB          -- flexible structure
);

-- Query inside JSONB:
SELECT * FROM payments
WHERE metadata->>'merchant_category' = 'restaurants'
  AND (metadata->>'fraud_score')::float < 0.5;

-- Index JSONB field:
CREATE INDEX idx_payments_merchant_cat
ON payments ((metadata->>'merchant_category'));
```

**CTEs and Window Functions**
```sql
-- Running balance (window function):
SELECT
    id,
    amount,
    SUM(amount) OVER (
        PARTITION BY user_id
        ORDER BY created_at
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_balance
FROM transactions
WHERE user_id = 123
ORDER BY created_at;

-- Recursive CTE (hierarchical permission inheritance):
WITH RECURSIVE permission_tree AS (
    -- Base case: direct permissions
    SELECT resource_id, permission, user_id
    FROM permissions WHERE user_id = 123

    UNION ALL

    -- Recursive: inherit from parent roles
    SELECT p.resource_id, p.permission, pt.user_id
    FROM permissions p
    JOIN user_roles ur ON p.role_id = ur.role_id
    JOIN permission_tree pt ON ur.user_id = pt.user_id
)
SELECT DISTINCT resource_id, permission FROM permission_tree;
```

**Logical Replication**
```sql
-- On publisher (source):
CREATE PUBLICATION payments_pub FOR TABLE payments;

-- On subscriber (replica, different version or cluster):
CREATE SUBSCRIPTION payments_sub
  CONNECTION 'host=primary dbname=payments'
  PUBLICATION payments_pub;
```

Enables: cross-version upgrades with near-zero downtime, selective table replication, feeding Kafka via Debezium CDC.

### Critical PostgreSQL Tuning Parameters

```
# Memory
shared_buffers = 25% of RAM        # Main buffer pool (e.g., 64GB RAM → 16GB)
work_mem = 64MB                     # Per-sort/hash operation (careful: per-connection)
maintenance_work_mem = 1GB         # For VACUUM, CREATE INDEX

# WAL
wal_level = replica                # Required for streaming replication
max_wal_senders = 10               # Max replica connections
wal_buffers = 64MB                 # WAL write buffer

# Autovacuum (crucial for MVCC cleanup)
autovacuum_vacuum_scale_factor = 0.01  # Vacuum when 1% of table is dead (aggressive)
autovacuum_analyze_scale_factor = 0.01

# Query Planning
random_page_cost = 1.1             # SSD: set near 1.0 (vs default 4.0 for HDD)
effective_cache_size = 75% of RAM  # Hint for planner (OS + shared_buffers)

# Connections
max_connections = 200              # Use connection pooler (pgBouncer) instead
```

### pgBouncer — Connection Pooling

```
Problem:
  PostgreSQL processes one connection per backend process.
  10,000 concurrent connections = 10,000 processes.
  Each process = ~5-10MB RAM. 10,000 = 50-100GB RAM. Unworkable.

Solution: pgBouncer
  App → pgBouncer (1000 connections) → PostgreSQL (50 connections)
  pgBouncer multiplexes 1000 app connections over 50 database connections.

Pool modes:
  Session pooling:     One backend for entire session. Safe but few clients served.
  Transaction pooling: One backend for one transaction. More efficient. Default.
  Statement pooling:   One backend per statement. Most efficient. Breaks multi-statement transactions.
```

---

# Chapter 19: MySQL Fundamentals

## Why Should You Care?

MySQL (and its fork MariaDB) is the other dominant relational database. Many organizations use MySQL for its performance at scale and because it's what Facebook, Twitter, Shopify, and many others chose historically. Understanding its differences from PostgreSQL is important for comparing tradeoffs.

---

## What Makes MySQL Different

**InnoDB storage engine:** MySQL's primary ACID-compliant engine. Buffer pool, clustered indexes, MVCC — similar to PostgreSQL in fundamentals.

**Clustered Primary Key:** InnoDB organizes table data *by primary key*. The primary key IS the table. This changes performance characteristics significantly.

```
PostgreSQL: Heap-organized. Rows stored in pages regardless of PK order.
            Index points to row via heap tuple ID (ctid).

MySQL/InnoDB: Clustered index. Rows stored IN ORDER of primary key.
              Range scan by PK = sequential I/O.
              Non-clustered indexes store PK value (not row pointer).
```

**Implication for design:** Choose UUID vs AUTO_INCREMENT primary keys carefully in MySQL:
- `BIGINT AUTO_INCREMENT`: Sequential inserts → no page splits → fast inserts
- `UUID`: Random inserts → page fragmentation → slower inserts, larger indexes
- UUIDs are preferred for distributed systems (globally unique without coordination)

**MySQL-specific features:**

```sql
-- Semi-sync replication (between async and sync):
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET GLOBAL rpl_semi_sync_master_enabled = ON;
-- Primary waits for at least one replica to ACK before confirming commit
-- Falls back to async if no replica responds within timeout

-- MySQL Group Replication (built-in Paxos-based HA):
-- All nodes can accept writes. Conflict detection built-in.
-- Used by PlanetScale, Vitess (YouTube's MySQL sharding layer)
```

**Connection to sharding:** Vitess (used at YouTube, PlanetScale) is a sharding middleware specifically for MySQL. It provides connection pooling, query routing, and online resharding. MySQL's clustering story at extreme scale relies on Vitess.

---

## PostgreSQL vs MySQL: When to Choose Which

| Factor | PostgreSQL | MySQL |
|--------|-----------|-------|
| JSON support | JSONB (full indexing) | JSON (limited index support) |
| Complex queries | Superior (window functions, CTEs, lateral joins) | Decent but historically behind |
| Full-text search | Built-in (tsvector) | Built-in but limited |
| Extensions | Rich ecosystem | More limited |
| Sharding | Citus, manual | Vitess (mature, battle-tested) |
| Replication | Streaming + logical | Binary log replication + Group Replication |
| Financial systems | Preferred | Common (esp. MySQL 8.0+) |
| Web applications | Preferred modern choice | Historically dominant (LAMP stack) |

**In practice:** Choose what your team knows. Both are excellent for 99% of use cases.

---

# Chapter 20: NoSQL Databases — When Relational Isn't Enough

## Interview Question

> *"Design the database layer for a system that needs to store 100 billion IoT sensor readings per year, queried by device and time range. Would you use PostgreSQL or Cassandra? Why?"*

---

## Why Should You Care?

NoSQL databases are not "better than" relational databases. They're *different tradeoffs*. Understanding when each NoSQL type is appropriate — and why — is one of the most common interview topics.

---

## The Four NoSQL Types

### 1. Key-Value Stores (Redis, DynamoDB)

```
Storage: map of key → value
  "user:123:session" → "{token: ..., expires: ...}"
  "rate_limit:ip:192.168.1.1:2026-05-31-10" → "42"
  "cache:product:456" → "{name: ..., price: ...}"

Operations:
  GET key
  SET key value [EX seconds]
  DEL key
  INCR key  (atomic counter)
```

**When to use:**
- Session storage
- Cache (Chapter 21+)
- Rate limiting counters
- Distributed locks
- Leaderboards (sorted sets in Redis)

**What it can't do:**
- Complex queries
- Relationships
- Arbitrary access patterns

**Redis specifics:**
```
Data structures:
  String:      Simple value
  List:        Ordered sequence (queue, stack)
  Set:         Unordered unique values
  Sorted Set:  Unique values with score (leaderboard)
  Hash:        Field-value pairs (mini-map)
  Stream:      Append-only log (Kafka-lite)

Redis Cluster:
  Sharding built-in via consistent hashing
  16384 hash slots distributed across nodes
```

### 2. Document Stores (MongoDB, Firestore)

```json
// Document = self-contained JSON object
{
  "_id": "payment_xyz123",
  "user": {
    "id": "usr_456",
    "name": "Alice",
    "email": "alice@bank.com"
  },
  "amount": 5000.00,
  "currency": "USD",
  "line_items": [
    {"product": "Widget A", "qty": 2, "price": 50.00},
    {"product": "Widget B", "qty": 1, "price": 4900.00}
  ],
  "metadata": {"ip": "192.168.1.1", "device": "iPhone"}
}
```

**When to use:**
- Documents with variable schema
- Hierarchical/nested data that belongs together
- Content management systems
- Product catalogs (each product type has different attributes)

**What it can't do:**
- Multi-document transactions (improved in MongoDB 4.0+ but not perfect)
- Efficient cross-collection joins
- Complex relational queries

**MongoDB specifics:**
- Documents up to 16MB
- Rich query language (but not SQL)
- Aggregation pipeline for analytics
- Change streams for CDC (like Postgres logical replication)
- Atlas Search (full-text) built in

### 3. Wide-Column Stores (Cassandra, HBase)

```
Table structure (looks like relational but isn't):

Cassandra "payments" table:
Partition Key (hash sharding): user_id
Clustering Columns (sort within partition): created_at DESC, payment_id

| user_id | created_at       | payment_id | amount | status    |
|---------|------------------|------------|--------|-----------|
| user_1  | 2026-05-31 10:00 | pay_001    | 100    | succeeded |
| user_1  | 2026-05-30 09:00 | pay_002    | 200    | pending   |
| user_1  | 2026-05-29 08:00 | pay_003    | 300    | failed    |
| user_2  | 2026-05-31 11:00 | pay_004    | 150    | succeeded |
```

Data is physically stored sorted by partition key, then by clustering columns within the partition. This makes range queries within a partition *extremely* fast.

**When to use:**
- Time-series data (IoT, metrics, events)
- Write-heavy workloads (Cassandra write performance is exceptional)
- Data naturally keyed by a partition key and sorted by a range key
- Geographic distribution (Cassandra is multi-region natively)

**What it can't do:**
- Arbitrary queries (queries must match the partition + clustering key)
- Joins
- Multi-row transactions (lightweight transactions exist but are limited)

**Cassandra specifics:**
```
CQL (Cassandra Query Language):
CREATE TABLE payments_by_user (
    user_id    UUID,
    created_at TIMESTAMP,
    payment_id UUID,
    amount     DECIMAL,
    status     TEXT,
    PRIMARY KEY (user_id, created_at, payment_id)
) WITH CLUSTERING ORDER BY (created_at DESC, payment_id ASC);

-- Fast (single partition):
SELECT * FROM payments_by_user
WHERE user_id = ?
  AND created_at >= ? AND created_at <= ?
LIMIT 20;

-- Impossible (no partition key):
SELECT * FROM payments_by_user WHERE status = 'pending';
-- Would require ALLOW FILTERING (full table scan across all nodes — terrible)
```

**Consistency in Cassandra (Tunable):**
```
Quorum writes: write to majority of replica nodes before ACK
Quorum reads:  read from majority of replica nodes

If W + R > N (where N = total replicas):
  Read-your-writes guaranteed (quorum read sees quorum write)

LOCAL_QUORUM: quorum within a data center (avoid cross-DC latency)
```

### 4. Graph Databases (Neo4j, Amazon Neptune, JanusGraph)

Data is represented as nodes (entities) and edges (relationships).

```
Nodes:
  (Alice) [User]
  (Bob)   [User]
  (Doc123) [Document]
  (Org456) [Organization]

Edges:
  (Alice)-[:MEMBER_OF]->(Org456)
  (Alice)-[:OWNS]->(Doc123)
  (Bob)-[:MEMBER_OF]->(Org456)
  (Org456)-[:HAS_ACCESS_TO]->(Doc123)

Query (Cypher language for Neo4j):
  "Who can access Doc123?"
  MATCH (u:User)-[:MEMBER_OF]->(o:Organization)-[:HAS_ACCESS_TO]->(d:Document)
  WHERE d.id = 'Doc123'
  RETURN u.name
```

**When to use:**
- Authorization with deep hierarchical permissions (ReBAC — Relationship-Based Access Control)
- Social networks (friends, connections)
- Fraud detection (rings of related accounts)
- Recommendation systems (users who bought X also bought Y)

**In IAM:** Graph databases are increasingly used for fine-grained authorization. Zanzibar (Google's authorization system) uses a relationship graph. OpenFGA and SpiceDB implement Zanzibar-style graph authorization.

---

## NoSQL vs Relational: Decision Framework

```
Start with these questions:

1. Do you need ACID transactions across multiple entities?
   YES → Relational database

2. Do you need arbitrary ad-hoc queries (SQL)?
   YES → Relational database

3. Is your schema fixed and well-understood?
   YES → Lean toward Relational

4. Do you need to store 100 billion rows with write throughput > 100K/sec?
   YES → Consider Cassandra or similar

5. Is your data naturally document-shaped (variable, hierarchical)?
   YES → Consider document store (MongoDB)

6. Do you need sub-millisecond reads for key lookups?
   YES → Consider Redis / key-value store

7. Is your authorization model deeply hierarchical and relationship-based?
   YES → Consider graph database

8. Everything else → Relational database (it handles most use cases well)
```

---

## Production Example

### Choosing a Database for IoT Sensor Data

**Requirements:**
- 100 billion readings/year ≈ 3,170 reads/second per device, thousands of devices
- Query: "Give me device X's readings for the last 24 hours"
- No complex multi-table joins
- High write throughput
- Time-series queries

**Analysis:**
- PostgreSQL + TimescaleDB: Works well. TimescaleDB extends PG with time-series partitioning, compression, and continuous aggregates. Best choice if you need SQL and moderate scale.
- Cassandra: Excellent for pure time-series at extreme scale. Partition key = device_id, clustering = timestamp. No SQL though — limited to modeled query patterns.
- InfluxDB: Purpose-built time-series. Best time-series compression and query language (Flux). Limited to time-series workloads.

**IAM Platform database choice:**
- Core IAM data (users, roles, permissions): PostgreSQL — ACID, relational joins, moderate scale
- Token cache: Redis — sub-millisecond key lookups, TTL-based expiry
- Audit logs: Cassandra or TimescaleDB — high write volume, time-range queries
- ReBAC policy evaluation: OpenFGA with Redis cache — graph traversal

---

## Staff-Level Thinking

**The polyglot persistence reality:** Large systems don't use one database. They use several, each for what it's best at:

```
IAM Platform architecture:

  Users, Roles, Permissions → PostgreSQL (ACID, relational)
  Access Tokens             → Redis (sub-ms lookup, TTL)
  Audit Events              → Cassandra (write-heavy, time-series)
  Authorization Policies    → OpenFGA (graph, ReBAC)
  Analytics                 → Redshift / BigQuery (OLAP, denormalized)
```

**The trap:** Choosing NoSQL because it "scales better" without understanding the access patterns. NoSQL forces you to model your queries upfront. SQL gives you flexibility to query however you want. Flexibility has value, especially early in a product's life when access patterns are unknown.

**Staff Engineer heuristic:** Default to PostgreSQL. Switch to NoSQL only when you have a concrete, measured problem that PostgreSQL can't solve. The operational simplicity of one database is worth significant performance headroom.
# Part 3: Caching

> Caching is the art of trading money for time, and staleness for speed. Every cache is a consistency tradeoff. Know what you're accepting before you cache.

---

# Chapter 21: Why Caching Exists

## Interview Question

> *"Your payment API handles 50,000 requests per second. Your database can handle 5,000 queries per second. How do you bridge this gap without scaling the database?"*

---

## Why Should You Care?

Caching is the single most effective performance optimization in distributed systems. It reduces database load, cuts latency, and enables systems to serve orders of magnitude more traffic than their backing stores could alone. But caching is also responsible for some of the worst data correctness bugs in production.

---

## Plain English Explanation

Databases are slow relative to memory. An in-memory read takes nanoseconds. A database query takes milliseconds. 1,000,000x slower.

Caching stores frequently-accessed data in memory, close to the application. When the application needs data:
1. Check the cache first (fast)
2. If found (cache hit): return immediately
3. If not found (cache miss): fetch from database, store in cache, return

For data that many users request (exchange rates, product prices, user profiles), the database is queried once and the result is served to thousands of subsequent requesters from the cache.

---

## Internal Mechanics

### The Cache Stack

```
Request for "EUR/USD exchange rate"
  │
  ▼
L1: In-Process Cache (application memory)
  Hash map in JVM heap. Sub-microsecond. Evicted on deploy.
  │
  ▼ (miss)
L2: Distributed Cache (Redis)
  Shared across instances. ~0.5ms. Survives deploy.
  │
  ▼ (miss)
L3: Database
  PostgreSQL. ~5-20ms. Source of truth.
  │
  Fill L2 cache with result, TTL=300s
  Fill L1 cache with result, TTL=60s
```

### Cache Hit Ratio

```
Cache Hit Ratio = cache_hits / (cache_hits + cache_misses)

90% hit rate: 10% of requests hit the database
95% hit rate: 5% hit database
99% hit rate: 1% hit database

Impact:
  1000 req/sec, 99% cache: 10 req/sec to DB
  1000 req/sec, 90% cache: 100 req/sec to DB
  1000 req/sec, 50% cache: 500 req/sec to DB

For most read-heavy systems, 95-99% hit rate is achievable for 
frequently-accessed data.
```

### Eviction Policies

When cache is full, what do you evict?

```
LRU (Least Recently Used):     Evict the item accessed least recently.
                                Good for temporal locality (recent = more likely needed again).

LFU (Least Frequently Used):   Evict the item accessed least often.
                                Good for stable popular items. Harder to implement.

FIFO:                           Evict oldest item.
                                Simple. Poor hit rate for most workloads.

Random:                         Evict randomly.
                                Surprisingly effective for large caches. Simpler than LRU.

TTL-based expiry:               Evict items after a time. Combines with above.
                                Most practical for cache consistency.
```

**Redis default eviction policy:** `noeviction` (reject new writes when full). Change to `allkeys-lru` for a cache use case.

---

## Production Example

### IAM: Caching Permission Evaluations

Permission evaluation is expensive: DB query + RBAC/ABAC policy evaluation. But for most users, permissions don't change often.

```
Request: "Can user 123 read document 456?"

Without cache:
  → DB query for user's roles: 5ms
  → DB query for role's permissions: 3ms
  → Policy evaluation: 1ms
  → Total: 9ms, every request

With Redis cache:
  → Cache key: "authz:user_123:doc_456:read"
  → Cache value: "PERMIT" or "DENY"
  → Cache TTL: 60 seconds

  Cache hit: 0.5ms
  Cache miss (first time or after TTL): 9ms + 0.5ms write

At 10,000 authorization checks/second with 95% hit rate:
  → 500 DB queries/second (vs 10,000 without cache)
  → Database stays healthy
```

**Cache invalidation on permission change:**
```
When admin revokes user_123's access to document_456:
  1. Update database
  2. DELETE "authz:user_123:doc_456:*" from Redis (pattern delete)
  3. OR: Wait for TTL expiry (eventual consistency — max 60s delay)

Financial/security context: Option 2 is usually unacceptable.
Use option 1: active invalidation on permission change.
```

---

## The Two Hard Problems of Caching

**1. Cache Invalidation:** When does cached data become stale? How do you remove it?  
**2. Cache Stampede (Thundering Herd):** When a cached item expires, many concurrent requests miss simultaneously and hammer the database. (Chapter 29)

These are covered in depth in subsequent chapters. Remember them — they're the exam questions of caching.

---

# Chapter 22: Cache-Aside

## Interview Question

> *"Describe the Cache-Aside pattern. What failure modes exist when the database and cache get out of sync?"*

---

## Plain English Explanation

Cache-Aside (also called Lazy Loading) is the most common caching pattern. The application is responsible for managing the cache:

1. **Read:** Check cache → cache hit: return. Cache miss: read DB, write to cache, return.
2. **Write:** Write to DB → invalidate cache (or update it).

The cache sits "aside" — the application explicitly decides what to cache and when.

---

## Internal Mechanics

```
READ path:
  value = cache.get(key)
  if value is null:
    value = db.query(...)
    cache.set(key, value, ttl=300)
  return value

WRITE path:
  db.update(...)
  cache.delete(key)          ← Invalidate (simplest)
  OR
  cache.set(key, new_value)  ← Update (risk: inconsistency if done wrong)
```

### The Race Condition

```
Time →
T1: App1 reads cache: MISS
T2: App1 queries DB: value = "old"
T3: App2 writes new value to DB
T4: App2 invalidates cache (DELETE key)
T5: App1 writes "old" value to cache ← STALE DATA WRITTEN

Now cache has "old" value. DB has "new" value. Cache is inconsistent.
This lasts until TTL expiry.
```

**Mitigation:** Short TTL. Accept eventual consistency. For payments/IAM, use write-through or invalidation-on-write with careful ordering.

---

## Production Example

### Payments: Product Catalog Caching

```java
public Product getProduct(String productId) {
    String cacheKey = "product:" + productId;
    
    // Try cache first
    Product cached = redis.get(cacheKey, Product.class);
    if (cached != null) {
        return cached;  // Cache hit: ~0.5ms
    }
    
    // Cache miss: query DB
    Product product = db.findProduct(productId);  // ~10ms
    
    // Store in cache
    redis.set(cacheKey, product, Duration.ofMinutes(10));
    
    return product;
}

public void updateProduct(String productId, ProductUpdate update) {
    // Write to database first
    db.updateProduct(productId, update);
    
    // Invalidate cache
    redis.delete("product:" + productId);
    
    // Next read will repopulate from DB
}
```

**Benefit:** Product reads are cached. 99% hit rate for popular products. DB load: 1% of read requests.  
**Risk:** After update, cache is invalid. Next read gets fresh DB value. Low risk — TTL handles stragglers.

---

## Tradeoffs

| Aspect | Cache-Aside | Notes |
|--------|------------|-------|
| Cache miss penalty | High (DB call) | First caller pays; subsequent callers cached |
| Consistency | Eventual | Race condition possible; TTL is safety net |
| Complexity | Medium | Application handles all cache logic |
| Stale data | Possible | Controlled by TTL and invalidation |
| Cold start | Slow | Cache empty; all misses; database hammered |

---

# Chapter 23: Write-Through

## Plain English Explanation

Write-Through: every write goes to the cache and the database simultaneously (or cache first, then DB). Cache is always up-to-date after writes.

```
WRITE path:
  cache.set(key, value)  ← write to cache
  db.write(key, value)   ← write to DB
  (both succeed, or roll back)

READ path:
  value = cache.get(key)
  if null:
    value = db.read(key)  ← cold start only; should rarely miss
  return value
```

**Benefit:** Cache is always consistent after writes. No stale data from write race conditions.

**Cost:** Every write touches both cache and DB. Write latency = cache_write + db_write (sequential) or max(cache_write, db_write) (parallel). Unused data gets cached (writes data that may never be read).

**When to use:** Read-heavy workloads where data written is almost certainly going to be read soon. User profiles, session data, product details.

**When not to use:** Write-heavy workloads where most written data is never read (event logs, audit trails). Cache fills with useless data.

---

# Chapter 24: Write-Back (Write-Behind)

## Plain English Explanation

Write-Back: write to cache first, return success to client. The cache asynchronously writes to the database in the background.

```
WRITE path:
  cache.set(key, value)  ← write to cache only
  return success          ← client gets response immediately
  [background: cache → db write, batched or delayed]

READ path:
  value = cache.get(key)  ← always fast; cache is the source
  return value
```

**Benefit:** Extremely fast writes. Client doesn't wait for DB. DB writes can be batched (10 cache writes become 1 DB write).

**Risk:** If cache dies before writing to DB, **data is lost**. Cache is now the source of truth, not the DB.

**When to use:**
- High-frequency writes where batching saves cost (metric aggregation, view counters, leaderboard scores)
- Non-critical data where some loss is acceptable
- Gaming: player position updates (lose 5 seconds of movement on crash — acceptable)

**When NOT to use:**
- Financial transactions: never. A payment that exists in cache but not DB is a disaster.
- User authentication state
- Any data where loss would cause inconsistency or financial impact

---

## Production Example: Write-Back in Payments (Wrong!)

```
BAD DESIGN:
  User initiates payment
  → Write to Redis: "payment_xyz = $100, pending"
  → Return "Payment initiated!" to user
  → Background job writes to DB... but Redis crashes before job runs
  → Payment exists nowhere
  → Customer was charged (card network approved), but no record in our DB
  → Financial discrepancy
```

**Correct approach for payments:** Write-Through or CQRS with event sourcing. The DB is always the authoritative record.

---

# Chapter 25: Read-Through

## Plain English Explanation

Read-Through: the cache is the primary read interface. On cache miss, the cache itself fetches from the database and caches the result. The application only talks to the cache.

```
Application → Cache (always)
                │
                │ miss? → DB
                │
                └─ populate cache, return value

Application never directly reads from DB.
Cache manages its own population.
```

**Difference from Cache-Aside:**
- Cache-Aside: Application manages DB reads on miss
- Read-Through: Cache manages DB reads on miss (application just asks cache)

**When used:** Managed caching layers (Memcached as read-through, AWS ElastiCache with read-through configured). Reduces boilerplate in application code.

**Same consistency risks as Cache-Aside** — the stale data and race condition problems are identical. The pattern just moves the cache-population logic.

---

# Chapter 26: Cache Invalidation — The Hard Problem

## Interview Question

> *"Phil Karlton said 'There are only two hard things in computer science: cache invalidation and naming things.' Why is cache invalidation hard? How do you invalidate a Redis cache when you have distributed services?"*

---

## Why Should You Care?

Cache invalidation is where caching goes wrong in production. Stale data. Incorrect displays. Security holes (cached permissions after revocation). Invalidating the right keys at the right time is an unsolved problem in the general case — it requires architectural discipline.

---

## Plain English Explanation

You've cached something. Now the underlying data changed. How do you tell every cache in the system to throw away the old copy?

**The problem:**
1. Multiple cache layers (L1, L2, regional caches, CDN)
2. Multiple services may have cached the same data
3. Distributed systems have no single point of authority for "is this stale?"

---

## Invalidation Strategies

### 1. TTL-Only (Time-Based Expiry)

```
Cache entry expires after N seconds. No explicit invalidation.

Pros: Simple. Self-healing.
Cons: Stale data persists up to TTL duration.

Use when: Data can tolerate eventual consistency. Short TTL acceptable.
Example: Exchange rates (5-minute TTL), product catalog (10-minute TTL)
```

### 2. Active Invalidation (Delete on Write)

```
On update to data X:
  1. Write new value to DB
  2. DELETE cache key for X from Redis

Pros: Immediate consistency after writes.
Cons: Cache miss on next read (must go to DB). Requires knowledge of which keys to invalidate.

Use when: Data consistency is critical.
Example: User permissions (revoked access = instant invalidation)
```

### 3. Event-Driven Invalidation

```
Database change → event → message bus → cache invalidators → delete cache keys

Change Data Capture (CDC):
  Postgres WAL → Debezium → Kafka → Cache Invalidator Service → Redis DEL

Pros: Decoupled. Works across services. DB is authoritative trigger.
Cons: Async (slight delay). Complex infrastructure.

Example:
  User role changed in DB
  → WAL change captured by Debezium
  → Published to Kafka "user-changes" topic
  → Cache invalidator consumes event
  → Deletes Redis key "user:123:permissions"
  → Deletes Redis key "authz:user_123:*"
```

```
DB → Debezium → Kafka
                  │
                  ├── IAM Cache Invalidator (deletes permission cache)
                  ├── Session Cache Invalidator (marks sessions for re-auth)
                  └── API Response Cache Invalidator (clears cached API responses)
```

### 4. Cache Versioning (Namespace Invalidation)

```
Instead of deleting keys, change the namespace prefix:

Current:  cache_v1:user:123:permissions
On global invalidation: bump version to v2
New reads: cache_v2:user:123:permissions → miss → fetches from DB

Old keys (cache_v1:*) become unreachable, expire naturally via TTL.
```

**Use case:** System-wide permission model change. Instead of deleting millions of keys, bump the version. Instant invalidation of all cached permissions.

### 5. Distributed Cache Invalidation (Hard)

```
Service A caches user permissions in its local Redis.
Service B caches user permissions in its local Redis.

User's permission changes.
Service A invalidates its Redis. ✓
Service B is not notified. Service B still serves stale permissions. ✗

Solutions:
  a) Shared Redis cluster (both services use same Redis)
  b) Event-driven: Both services subscribe to user-change events
  c) Short TTL: Accept eventual consistency
  d) No local caches: Only shared distributed cache
```

---

## Production Example

### IAM: Permission Cache Invalidation on Revocation

```
Scenario: Admin revokes user_123's admin role at 10:00:00.

Without invalidation:
  10:00:00 - Role revoked in DB
  10:00:30 - User makes admin API call
  Cache hit: "user_123:admin_permissions" still cached
  Cache TTL = 5 minutes
  10:05:00 - Cache expires
  User retries admin call
  Cache miss, fetches from DB → denied
  
  5 minutes of unauthorized access after revocation!

With active invalidation:
  10:00:00 - Role revoked in DB
  10:00:00 - Cache key deleted: "user_123:admin_permissions"
  10:00:30 - User makes admin API call
  Cache miss, fetches from DB → denied
  
  Immediate revocation effect!
```

### Tag-Based Invalidation

For complex cache dependencies, use tags:

```
Cache "product:123:detail" → tagged with ["product:123", "category:electronics"]
Cache "category:electronics:list" → tagged with ["category:electronics"]

When product 123 is updated:
  Invalidate all cache entries tagged "product:123"
  → Deletes "product:123:detail"

When electronics category is updated:
  Invalidate all entries tagged "category:electronics"
  → Deletes "product:123:detail" AND "category:electronics:list"
```

Redis doesn't support this natively. Implement with sets:
```redis
SADD tag:product:123 "product:123:detail"
SADD tag:category:electronics "product:123:detail" "category:electronics:list"

# On invalidation:
SMEMBERS tag:product:123 → ["product:123:detail"]
DEL product:123:detail
DEL tag:product:123
```

---

## Staff-Level Thinking

Cache invalidation is fundamentally an event propagation problem. The more distributed your cache, the harder invalidation becomes.

**Design principle:** Minimize cache layers. Each additional layer is another place to propagate invalidations.

**Security implication:** For permission caches, active invalidation is not optional. Stale permission cache = potential unauthorized access. TTL-only is unacceptable for security-sensitive caches.

---

# Chapter 27: TTL — Time as a Consistency Tool

## Why Should You Care?

TTL (Time To Live) is the simplest cache consistency mechanism. Set a number. Data expires. Fresh data fetched. It's blunt but reliable. Understanding TTL tradeoffs helps you choose the right TTL for each cached item.

---

## TTL Design Guide

```
TTL Selection Framework:

How often does this data change?
  Changes rarely (hours/days):     TTL = hours
  Changes occasionally (minutes):  TTL = minutes
  Changes frequently (seconds):    TTL = seconds, or don't cache

How harmful is stale data?
  Financial consequence:           Short TTL + active invalidation
  UX degradation only:             Longer TTL acceptable
  Cosmetic only (avatar, theme):   Very long TTL fine

What's the read:write ratio?
  High read, low write:            Longer TTL (cache is effective)
  Equal read/write:                Short TTL or write-through
  Write heavy:                     Don't cache writes

Examples:
  Exchange rates:                  TTL = 1-5 minutes (changes rarely, small $$ impact)
  User profile (name, avatar):     TTL = 10-60 minutes (changes rarely, cosmetic)
  Permission evaluation:           TTL = 30-60 seconds + active invalidation on change
  Session validation:              TTL = token expiry (match with JWT exp)
  Product prices:                  TTL = 5 minutes (significant $$ impact, changes infrequently)
  Fraud scores:                    TTL = 30 seconds (must be fresh)
  Rate limit counters:             TTL = window size (e.g., 1 minute)
```

### Sliding vs Fixed TTL

```
Fixed TTL:
  Key cached at T=0, TTL=60s
  Key expires at T=60, regardless of access
  Access at T=55 → key expires at T=60 anyway

Sliding TTL (refresh on read):
  Key cached at T=0, TTL=60s
  Access at T=55 → TTL reset to 60s, expires at T=115
  Key never expires if accessed frequently enough

Redis sliding TTL:
  redis.get(key)  ← read
  redis.expire(key, 60)  ← reset TTL (not atomic — race condition possible)

Use fixed TTL for: Data that must be periodically refreshed regardless of access
Use sliding TTL for: Session data that should expire if user becomes inactive
```

---

# Chapter 28: Hot Keys — When One Key Breaks Everything

## Interview Question

> *"You have Redis with 100 shards. One key — the top trending product — receives 500,000 requests per second. That Redis node becomes the bottleneck. How do you handle hot keys?"*

---

## Why Should You Care?

Even perfectly distributed systems can have hot keys: cache keys accessed so frequently that a single Redis node becomes a bottleneck. This is a subtle but devastating failure mode that looks like "Redis is slow" but is actually "one Redis key is being hammered."

---

## Plain English Explanation

Imagine 100 bank branches (Redis shards). 99 branches handle 100 customers/hour. One branch handles 50,000 customers/hour because it's next to the stadium on game day. That branch melts.

Hot keys are the cache equivalent.

---

## Hot Key Patterns

**Celebrity problem in IAM:** If a super-admin's permissions are cached as a single key, every API request across the system hits that one key. That Redis shard gets all the load.

**Viral content in payments:** Payment page for a viral product launch. Millions of users simultaneously view the product — all hitting the same cache key for product details.

---

## Solutions

### 1. Key Duplication (Replicate Hot Key)

```
Instead of one key "product:viral_item", create N copies:
  "product:viral_item:1"
  "product:viral_item:2"
  ...
  "product:viral_item:10"

On read: randomly select a copy
  shard_id = random(1, 10)
  redis.get("product:viral_item:" + shard_id)

Load is distributed across N Redis nodes.
On update: update all N copies (or invalidate all).
```

### 2. Local Cache (L1 Cache)

```java
// Application-level in-process cache for hot items
private final Cache<String, Product> localCache = CaffeineCache.newBuilder()
    .maximumSize(1000)
    .expireAfterWrite(10, TimeUnit.SECONDS)
    .build();

public Product getProduct(String id) {
    // L1: Local JVM cache (nanoseconds)
    Product local = localCache.getIfPresent(id);
    if (local != null) return local;
    
    // L2: Redis (0.5ms)
    Product cached = redis.get("product:" + id, Product.class);
    if (cached != null) {
        localCache.put(id, cached);
        return cached;
    }
    
    // L3: Database (10ms)
    Product db = database.findProduct(id);
    redis.set("product:" + id, db, Duration.ofMinutes(10));
    localCache.put(id, db);
    return db;
}
```

100 application instances each with 10-second local cache:
→ Each instance needs only 1 Redis fetch per 10 seconds per hot key
→ 100 instances × 1 fetch / 10s = 10 Redis requests/second instead of 500,000

### 3. Read-Through with In-Process Deduplication

```java
// Only one goroutine/thread fetches from DB; others wait
private final Map<String, CompletableFuture<Product>> inFlight = new ConcurrentHashMap<>();

public Product getProduct(String id) {
    CompletableFuture<Product> existing = inFlight.get(id);
    if (existing != null) return existing.get();  // Wait for in-flight request
    
    CompletableFuture<Product> future = new CompletableFuture<>();
    inFlight.put(id, future);
    
    Product value = fetchFromCacheOrDB(id);
    future.complete(value);
    inFlight.remove(id);
    return value;
}
```

### 4. CDN for Read-Heavy Data

Push product catalog, exchange rates, and other hot data to CDN edges. Requests served from edge servers, never reaching Redis or your application.

---

# Chapter 29: Cache Stampede — The Self-Inflicted DDoS

## Interview Question

> *"Your Redis cache has a TTL of 5 minutes on product prices. At exactly 5 minutes, the entry expires. 10,000 concurrent users all get a cache miss simultaneously. What happens? How do you prevent it?"*

---

## Why Should You Care?

Cache stampede (also called Thundering Herd or Dog-Piling) is one of the most common and damaging failure modes in caching. It turns a cache expiry into a self-inflicted denial of service on your database. Under high traffic, it can bring down production systems.

---

## Plain English Explanation

Your cache has a popular item. TTL expires. At the exact moment of expiry, 10,000 users simultaneously request it. They all get a cache miss. All 10,000 race to the database to fetch the fresh value.

Your database suddenly receives 10,000 concurrent queries for the same thing. It buckles. Latency spikes. Other queries queue up. System degrades.

The cache that was supposed to protect the database just created its biggest attack.

---

## Solutions

### 1. Cache Lock (Mutex Lock)

```python
def get_with_lock(key, fetch_fn, ttl=300):
    # Try cache first
    value = redis.get(key)
    if value:
        return value
    
    # Miss: try to acquire lock
    lock_key = key + ":lock"
    lock_acquired = redis.set(lock_key, "1", nx=True, ex=10)  # nx = set if not exists
    
    if lock_acquired:
        # We got the lock: fetch and cache
        value = fetch_fn()
        redis.set(key, value, ex=ttl)
        redis.delete(lock_key)
        return value
    else:
        # Another thread has the lock: wait and retry
        time.sleep(0.1)
        return redis.get(key)  # By now, the locking thread should have set it
```

**Problem:** Threads that don't get the lock spin-wait or get None. Not ideal.

### 2. Probabilistic Early Recomputation (XFetch / PER)

```
Brilliant approach: Recompute the cache entry BEFORE it expires, probabilistically.

When reading cached value:
  time_remaining = expires_at - now
  beta = 1.0  (tunable)
  
  if -beta * log(random()) >= time_remaining:
    # Randomly triggered early recomputation
    fetch from DB and recache
    # This happens before expiry, avoiding the stampede
```

The probability of early recomputation increases as expiry approaches. Expensive items (long fetch time) should recompute earlier. Works naturally.

### 3. Stale-While-Revalidate

```
Return stale data immediately. Asynchronously refresh in background.

def get_with_stale(key, fetch_fn):
    value, expires_at = redis.get_with_ttl(key)
    
    if value and expires_at > now + grace_period:
        return value  # Fresh: serve immediately
    
    elif value and expires_at > now:
        # Stale but not expired: serve stale, refresh async
        if not is_refreshing(key):
            async_refresh(key, fetch_fn)
        return value  # Return stale immediately — no latency hit
    
    else:
        # Expired: synchronous fetch (cache miss)
        value = fetch_fn()
        redis.set(key, value)
        return value
```

**User experience:** User gets slightly stale data for one request. Immediately after, fresh data available. No latency spike. No database overload.

**HTTP equivalent:** `Cache-Control: stale-while-revalidate=60`
Browser serves stale for up to 60 seconds while fetching fresh in background.

### 4. Cache Key Jitter (Prevent Synchronized Expiry)

```
# Don't cache everything with exactly the same TTL
# If 1000 products all have TTL=300s, they all expire at the same time

# Add random jitter:
ttl = 300 + random.randint(-30, 30)  # 270-330 seconds
redis.set(key, value, ex=ttl)

# Now expirations are spread over time. No synchronized stampede.
```

---

## Production Example

### Payments: Exchange Rate Cache Stampede

Exchange rates change every minute. You cache them for 60 seconds.

At 10:00:00 UTC, the cache expires. Traffic: 50,000 req/sec. All get cache miss. All query the exchange rate API or DB.

```
With Stale-While-Revalidate:
  10:00:00 - Cache TTL ends, stale-while-revalidate = 10s window
  10:00:00 to 10:00:10 - Serve stale rate, one background refresh triggered
  10:00:01 - Background fetch completes, cache updated
  10:00:01+ - All requests see fresh rate
  
  Database received: 1 query (background refresh)
  Not: 50,000 concurrent queries
```

### IAM: Distributed Permission Cache with Jitter

```java
// Instead of fixed 60-second TTL for all permissions
int baseTTL = 60;
int jitter = new Random().nextInt(20);  // 0-20 seconds
redis.set(cacheKey, permissionResult, Duration.ofSeconds(baseTTL + jitter));

// Permission cache for user 1 expires at T+72s
// Permission cache for user 2 expires at T+65s
// Permission cache for user 3 expires at T+78s
// Expirations spread out. No stampede.
```

---

## Summary: Cache Pattern Selection Guide

```
Workload / Need                   → Recommended Pattern
─────────────────────────────────────────────────────
Read-heavy, occasional writes     → Cache-Aside + active invalidation
Read+Write, strong consistency    → Write-Through
Write-heavy, eventual consistency → Write-Back (non-financial only)
Simple, tolerate some staleness   → Cache-Aside + TTL only
Security-critical permissions     → Cache-Aside + immediate invalidation on change
High-traffic, popular items       → Local L1 cache + Redis L2 + TTL jitter
Large-scale reads (millions)      → CDN + Redis + L1
```
# Part 4: Messaging Systems

> Messaging systems decouple producers from consumers. This decoupling is what makes large-scale distributed systems composable and resilient. Without it, every service is a synchronous dependency on every other service — a recipe for cascade failures.

---

# Chapter 30: Queues — Work Buffering

## Interview Question

> *"Your payment processor receives 50,000 payment requests per second but can only complete 5,000 per second. How do you handle the burst without losing requests or making users wait?"*

---

## Why Should You Care?

Queues buffer work between producers (who generate tasks) and consumers (who process them). They absorb traffic spikes, enable async processing, and decouple component speeds. Almost every production system uses queues in some form.

---

## Plain English Explanation

Imagine a restaurant. 50 customers walk in at 7 PM. The kitchen can cook 5 meals at a time. Without a queue, 45 customers leave in frustration. With a queue (a waitlist), customers wait but are served eventually. The kitchen processes at its own pace.

Queues work the same way: producers add work to the queue, consumers pull work from the queue at their own pace. Neither is blocked by the other.

---

## Internal Mechanics

### Queue Operations

```
Producer                Queue                Consumer
  │                       │                     │
  │──ENQUEUE(msg)─────→  │  [msg3][msg2][msg1] │
  │                       │───DEQUEUE()────────→│
  │                       │                     │ process msg1
  │──ENQUEUE(msg)─────→  │  [msg4][msg3][msg2] │
  │                       │───DEQUEUE()────────→│
                          │                     │ process msg2
```

**FIFO (First In, First Out):** Messages processed in order they were added. Standard behavior.

**Priority Queue:** Higher-priority messages processed first. SQS, RabbitMQ support this.

### Queue Properties

**At-Least-Once Delivery:**
Message is guaranteed to be delivered at least once. May be delivered multiple times (duplicates possible).
- Consumer must be idempotent (handle duplicates safely)
- Most queue systems guarantee this

**At-Most-Once Delivery:**
Message delivered zero or one times. May be lost but never duplicated.
- Low overhead
- Only acceptable where loss is tolerable (metrics, logs)

**Exactly-Once Delivery:**
Message delivered exactly once. The holy grail. Very hard to achieve in distributed systems.
- Requires coordination between queue and consumer (distributed transaction or idempotency infrastructure)
- Kafka Transactions approach this (but requires careful implementation)

### Visibility Timeout

```
Consumer fetches message from queue
  → Message becomes invisible to other consumers for N seconds (visibility timeout)
  → Consumer processes message
  → If done: Consumer deletes message from queue
  → If not done in N seconds: Message becomes visible again → redelivered to another consumer
  
This enables:
  - Fault tolerance: failed consumer → message redelivered
  - Parallelism: multiple consumers each process different messages
  - Safety: message not lost if consumer crashes
```

---

## Production Example

### Payments: Async Payment Processing

```
User submits payment request
    ↓
API Server:
  1. Validate request (fast, synchronous)
  2. Store payment in DB: status = "pending"
  3. Enqueue to payment-processing queue
  4. Return HTTP 202 Accepted: {payment_id: "pay_123", status: "pending"}
    ↓
Payment Queue: [pay_123] [pay_456] [pay_789]
    ↓
Payment Processor (consumes from queue):
  1. Fetch payment_id from queue
  2. Call payment gateway (500ms - 2s)
  3. Update payment status in DB: "succeeded" or "failed"
  4. Publish payment-completed event
  5. Delete message from queue (or ACK)
    ↓
User polls GET /payments/pay_123 until terminal status
  OR WebSocket push on completion
```

**Why async?** Payment gateway calls take up to 2 seconds. Holding an HTTP connection open for 2 seconds × 50,000 concurrent requests = 100,000 seconds of connection time. Unscalable. Queue-based async processing is the correct architecture.

---

# Chapter 31: Pub/Sub — Decoupled Broadcasting

## Interview Question

> *"How does pub/sub differ from point-to-point queuing? When would you use each?"*

---

## Plain English Explanation

**Point-to-point queue (Chapter 30):** One producer, one consumer (per message). Like sending a letter — one recipient gets it, then it's gone.

**Pub/Sub:** One producer (publisher), multiple consumers (subscribers). Like a newspaper — publisher prints once, all subscribers receive a copy.

In pub/sub, publishers don't know who subscribes. Subscribers don't know who publishes. Complete decoupling.

---

## Internal Mechanics

```
Publisher                Topic/Channel             Subscribers
  │                          │                          │
  │──publish(event)─────→   │  ┌───→ Subscriber A ────→│ receives copy
  │                          │  ├───→ Subscriber B ────→│ receives copy
  │                          │  └───→ Subscriber C ────→│ receives copy
```

### Point-to-Point vs Pub/Sub

| Aspect | Point-to-Point Queue | Pub/Sub |
|--------|---------------------|---------|
| Consumers per message | One | Many |
| Producer awareness | Knows queue exists | Only knows topic name |
| Consumer awareness | Knows queue exists | Only knows topic name |
| Message delivery | Consumed once | Copy per subscriber |
| Use case | Work distribution | Event broadcast |

---

## Production Example

### IAM: User Lifecycle Events

```
User created event published to topic "user-lifecycle":
  {event: "USER_CREATED", user_id: "usr_123", email: "alice@bank.com"}

Subscribers:
  Email Service → sends welcome email
  Onboarding Service → creates default workspace
  Audit Service → logs creation event
  Analytics Service → increments new-user counter
  Fraud Service → initializes fraud profile

Publisher (user service) doesn't know about any of these subscribers.
New subscribers added without touching user service code.
This is the extension without modification principle.
```

### Payments: Event Fan-Out

```
PaymentCompleted event published:
  {payment_id: "pay_123", amount: 100, user_id: "usr_456", merchant_id: "mer_789"}

Subscribers:
  Notification Service → sends push notification to user
  Loyalty Service → awards points
  Reconciliation Service → updates merchant ledger
  Fraud Service → updates fraud model with completed transaction
  Analytics Service → updates real-time revenue dashboard
  Tax Service → records taxable event
```

Each subscriber is independent. Failure in one doesn't affect others.

---

# Chapter 32: Event-Driven Architecture

## Interview Question

> *"What is event-driven architecture? What are the benefits and risks compared to direct API calls between services?"*

---

## Why Should You Care?

Event-driven architecture is the foundation of modern scalable systems. Instead of services calling each other (synchronous coupling), services emit events and react to events (async decoupling). This is what enables horizontal scaling, resilience, and independent deployability.

---

## Plain English Explanation

**Synchronous (RPC/REST):**
Service A calls Service B. Service A waits. If Service B is slow, Service A is slow. If Service B is down, Service A fails.

**Event-Driven:**
Service A emits an event. Service B (and C, D, E...) react to it when ready. Service A doesn't wait. Service A doesn't know or care who reacts.

---

## Internal Mechanics

### Event Types

**Domain Events:** Something that happened in the business domain.
```
PaymentCompleted{amount: 100, user_id: 123, timestamp: ...}
UserRegistered{email: "alice@bank.com", ...}
OrderShipped{order_id: 456, tracking: "UPS123"}
```

**Commands (sent to queue, not pub/sub):** A request for something to happen. Directed to a specific handler.
```
ProcessPayment{payment_id: 123, amount: 100}
SendEmail{template: "welcome", to: "alice@bank.com"}
```

**Queries:** Requests for information. In event-driven systems, often served from read models (CQRS, Chapter 55).

### Event Storming — The Design Process

Before building, map the domain:
```
1. Identify Domain Events (orange stickies): "What happened?"
2. Identify Commands (blue stickies): "What triggered it?"
3. Identify Aggregates (yellow stickies): "What owns this state?"
4. Identify Policies (purple stickies): "When X happens, do Y"

PaymentSubmitted → [Payment Aggregate] → PaymentValidated
PaymentValidated → [Payment Aggregate] → GatewayCallInitiated
GatewayCallInitiated → [Gateway] → PaymentAuthorized
PaymentAuthorized → [Payment Aggregate] → PaymentCompleted
PaymentCompleted → [Policy: "On payment complete, notify user"] → NotificationSent
PaymentCompleted → [Policy: "On payment complete, award loyalty"] → LoyaltyAwarded
```

### Benefits and Risks

```
Benefits:
  ✓ Decoupling: Services don't know about each other
  ✓ Scalability: Consumers scale independently
  ✓ Resilience: Consumer failure doesn't affect producer
  ✓ Extensibility: Add new consumers without changing producers
  ✓ Temporal decoupling: Consumer processes when ready, not when producer calls

Risks:
  ✗ Eventual consistency: Consumers process asynchronously → delay in side effects
  ✗ Hard to trace: No single call stack; events flow across services
  ✗ Hard to test: Must simulate entire event chain in tests
  ✗ Ordering: Events may arrive out of order across consumers
  ✗ Schema evolution: Events must be backwards compatible (old consumers process new events)
  ✗ Debugging: "Why didn't the notification send?" requires tracing events across systems
```

---

# Chapter 33: Producers and Consumers

## Plain English Explanation

**Producer:** The service that creates and publishes events or messages.  
**Consumer:** The service that reads and processes events or messages.

The decoupling between them is the point.

## Consumer Patterns

**Competing Consumers:**
```
Queue: [msg1][msg2][msg3][msg4][msg5]
  ├──→ Consumer 1: processes msg1
  ├──→ Consumer 2: processes msg2
  └──→ Consumer 3: processes msg3
  
Work distributed. Parallel processing. Scale consumers by adding more.
If Consumer 2 crashes, msg2 becomes visible again. Consumer 1 or 3 picks it up.
```

**Dedicated Consumer (per topic):**
```
Topic: user-events
  ├──→ Email Service Consumer: sends emails for all user events
  ├──→ Analytics Consumer: logs all user events
  └──→ Fraud Consumer: analyzes all user events
  
Each subscriber gets all events. Subscribers don't compete.
(Pub/sub model, Chapter 31)
```

**Consumer Group (Kafka-specific, Chapter 34):**
Combines competing consumers within a topic. Multiple instances of the same service share work. Multiple services (groups) each get all events.

---

## Backpressure

Backpressure occurs when consumers can't keep up with producers. Queue fills. Memory exhausted. System crashes.

```
Producer: 10,000 events/sec
Consumer: 1,000 events/sec
Queue fills at: 9,000 events/sec
After 1 hour: queue has 32 million events
Memory: exhausted → crash

Handling backpressure:
  1. Scale consumers (add more consumer instances)
  2. Slow down producer (reactive systems — producer pauses when queue is full)
  3. Drop messages (acceptable for non-critical data like metrics)
  4. Alert and page on-call when queue depth exceeds threshold
```

---

# Chapter 34: Consumer Groups

## Interview Question

> *"In Kafka, what is a consumer group? How does Kafka ensure that each message is processed exactly once per consumer group while allowing multiple groups to independently consume all messages?"*

---

## Why Should You Care?

Consumer groups are how Kafka scales consumption across multiple instances while maintaining order guarantees. They're also how multiple independent services can each consume all events from the same topic. This is fundamental to Kafka-based architectures.

---

## Internal Mechanics

### Partitions and Consumer Groups

```
Kafka Topic "payments" with 6 partitions:

Partitions:  [P0] [P1] [P2] [P3] [P4] [P5]

Consumer Group A (Payment Processor Service, 3 instances):
  Instance 1: P0, P1
  Instance 2: P2, P3
  Instance 3: P4, P5
  → Each partition assigned to exactly one consumer in the group
  → Work is distributed among 3 instances

Consumer Group B (Fraud Detection Service, 2 instances):
  Instance 1: P0, P1, P2
  Instance 2: P3, P4, P5
  → Independently consumes ALL partitions
  → Independent cursor (offset) tracking

Consumer Group C (Audit Service, 1 instance):
  Instance 1: P0, P1, P2, P3, P4, P5
  → One instance consumes all partitions sequentially
```

**Key insight:** Consumer groups are independent. Group A's offset (position) doesn't affect Group B's offset. Both read all messages.

### Ordering Guarantee

Kafka guarantees ordering **within a partition**.

```
Partition 0: [payment:user_1:T1] [payment:user_1:T2] [payment:user_1:T3]
             ← All user_1 payments in order within P0

To guarantee all events for user_1 are processed in order:
  Use user_id as partition key:
  partition = hash(user_id) % num_partitions
  → All user_1 events go to same partition
  → Same consumer processes user_1 events in order

Cross-partition order: NOT guaranteed.
  user_1's events may arrive before or after user_2's events across partitions.
```

### Rebalancing

When consumers join or leave a group, Kafka rebalances partition assignments:

```
Before: 3 consumers, 6 partitions
  C1: P0, P1
  C2: P2, P3
  C3: P4, P5

Consumer C2 crashes:
  Rebalance triggered
  C1: P0, P1, P2, P3
  C3: P4, P5

Consumer C4 joins:
  Rebalance triggered
  C1: P0, P1
  C3: P2, P3
  C4: P4, P5

During rebalance: brief pause in consumption (stop-the-world).
Incremental Cooperative Rebalancing (Kafka 2.4+): avoids full pause.
```

---

# Chapter 35: Ordering — The Hardest Guarantee

## Interview Question

> *"You need to process payments for a user in order: pay, refund, dispute. How do you guarantee order in a distributed messaging system?"*

---

## Why Should You Care?

Order matters in financial systems: a refund before a payment is meaningless. A state machine must transition in sequence. Getting ordering wrong produces corrupted data.

---

## Ordering Challenges

### Why Order is Hard

```
Producer sends: [event_1] [event_2] [event_3]

Network delivers:
  Consumer A receives: [event_1] [event_3] [event_2]  ← out of order!
  
  Or:
  Consumer A receives: [event_1]
  Consumer B receives: [event_3] [event_2]  ← split across consumers!
```

### Solutions

**1. Partition Key (Kafka):**
Events for the same entity go to the same partition, in order.
```
partitionKey = user_id  → All events for user 123 → Partition 2 (always)
One consumer handles Partition 2 → processes user 123 events in order
```

**2. Sequence Numbers (Application Level):**
```json
{
  "event_type": "payment_updated",
  "payment_id": "pay_123",
  "sequence": 5,
  "data": {...}
}
```
Consumer sorts by sequence before processing. Reorders out-of-order events.

**3. Database Ordering (append-only log):**
All events for an entity written to database in order. Consumers read from DB in sequence order.

**4. Inbox Pattern:**
Events stored in an "inbox" table with sequence. Processed in-order from the table.

---

# Chapter 36: Retries — When to Try Again

## Interview Question

> *"Your payment notification service fails intermittently because the email provider is slow. How do you implement retries safely without sending duplicate emails?"*

---

## Why Should You Care?

In distributed systems, failures are normal. Networks drop packets. Services restart. Databases hiccup. Retries make your system resilient. But naive retries create new problems: duplicate processing, overload, and cascading failures.

---

## Retry Strategies

### Simple Retry (Wrong)

```python
for attempt in range(5):
    try:
        send_email(user, template)
        break
    except Exception:
        time.sleep(1)  # Wait 1 second then retry
```

**Problem:** If 1000 consumers all fail and retry at the same time, the email service gets 1000 simultaneous retries → overload → more failures → more retries. **Retry storm.**

### Exponential Backoff (Better)

```python
def send_with_backoff(user, template, max_attempts=5):
    for attempt in range(max_attempts):
        try:
            send_email(user, template)
            return  # Success
        except TransientError:
            if attempt == max_attempts - 1:
                raise  # Final attempt failed
            
            wait_seconds = (2 ** attempt) * (0.5 + random.random() * 0.5)
            # Attempt 0: wait 0.5-1s
            # Attempt 1: wait 1-2s
            # Attempt 2: wait 2-4s
            # Attempt 3: wait 4-8s
            # Attempt 4: wait 8-16s
            time.sleep(wait_seconds)
```

**Jitter** (random variation) prevents synchronized retries from different consumers.

### Idempotency Keys for Safe Retries

```
Problem: Email retried. Sent twice.
Solution: Each notification has a unique idempotency key.

send_email(
    to=user.email,
    template="payment_confirmed",
    idempotency_key=f"payment:{payment_id}:notification:v1"
)

Email service: tracks sent idempotency keys.
If key already sent → return success (don't resend).
Result: Email sent exactly once regardless of retries.
```

### What to Retry vs What Not to Retry

```
RETRY (transient errors):
  503 Service Unavailable
  504 Gateway Timeout
  429 Too Many Requests (with respect to Retry-After header)
  Network timeouts
  Database connection errors

DON'T RETRY (permanent errors):
  400 Bad Request (request is wrong, retrying won't help)
  401 Unauthorized (credentials invalid)
  403 Forbidden (not allowed, retrying won't change permissions)
  404 Not Found (resource doesn't exist)
  422 Unprocessable Entity (invalid data)
```

---

# Chapter 37: Dead Letter Queues — The Poison Pill Handler

## Interview Question

> *"A payment message in your queue keeps failing and being retried indefinitely. The consumer is crashing every time it processes it. What happens to your queue? How do you handle this?"*

---

## Why Should You Care?

Some messages are permanently broken — bad data, bugs in processing logic, corrupt payloads. If you retry them forever, they block the queue, waste resources, and prevent other messages from being processed.

---

## Plain English Explanation

A Dead Letter Queue (DLQ) is a separate queue where messages go after they've failed processing N times.

```
Normal Queue: [msg_A][msg_B][msg_BROKEN][msg_C][msg_D]

Processing:
  msg_A: success → consumed
  msg_B: success → consumed
  msg_BROKEN: fails 3 times → moved to DLQ
  msg_C: success → consumed

Dead Letter Queue: [msg_BROKEN]
  → Engineers investigate why msg_BROKEN is failing
  → Fix bug or bad data
  → Optionally replay from DLQ back to main queue
```

---

## Production Example

### Payments: DLQ for Failed Processing

```
SQS Payment Queue ← Payment Processor consumes
  
  MaxReceiveCount = 3 (retry 3 times)
  After 3 failures → moved to SQS DLQ: payment-processor-dlq

DLQ monitor:
  CloudWatch alert: DLQ depth > 0 → page on-call engineer
  
Investigation:
  Engineer reads failed message from DLQ
  Identifies root cause: malformed merchant_id in message
  Fixes upstream data pipeline
  Reprocesses valid messages from DLQ manually
  Discards truly broken messages after investigation
```

### DLQ Metrics to Monitor

```
Queue depth (DLQ): Alert if > 0 (investigate immediately)
DLQ message age: Alert if > 1 hour (messages stagnating — nobody processing)
DLQ consumer errors: Who is reading the DLQ? If nobody, messages accumulate forever.
```

### Poison Pill Pattern

A poison pill is a special message that signals consumers to shut down. In queue systems, "poison pill" also colloquially refers to messages that crash consumers.

```
Actual Poison: Message with corrupt data that causes consumer NullPointerException
  → Consumer crashes
  → Message becomes visible again after visibility timeout
  → Next consumer picks it up, crashes
  → All consumers cycling through crashes
  → Queue processing halted

DLQ breaks this cycle: after N retries, the message is removed to DLQ.
Other messages can be processed normally.
```

---

# Chapter 38: Kafka Fundamentals

## Interview Question

> *"Describe Kafka's architecture. How does Kafka achieve high throughput? What is the difference between Kafka and a traditional message queue like RabbitMQ?"*

---

## Why Should You Care?

Kafka is the backbone of event-driven architecture at scale. Uber, Airbnb, LinkedIn, Netflix — all use Kafka as their central event streaming platform. It handles millions of events per second with low latency and durable storage.

---

## Plain English Explanation

Traditional queues: messages consumed and deleted. Like a to-do list — done item removed.

Kafka: messages stored in an immutable log. Multiple consumers read independently. Like a newspaper archive — reading it doesn't remove it. Different readers (groups) can be at different points in the archive.

This makes Kafka a **distributed append-only log** — not just a queue.

---

## Internal Mechanics

### Kafka Components

```
┌──────────────────────────────────────────────────────┐
│                 Kafka Cluster                         │
│                                                       │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│   │ Broker 1 │  │ Broker 2 │  │ Broker 3 │          │
│   │          │  │          │  │          │          │
│   │ Topic:   │  │ Topic:   │  │ Topic:   │          │
│   │  P0(L)   │  │  P1(L)   │  │  P2(L)   │          │
│   │  P1(F)   │  │  P2(F)   │  │  P0(F)   │          │
│   │  P2(F)   │  │  P0(F)   │  │  P1(F)   │          │
│   └──────────┘  └──────────┘  └──────────┘          │
│        L = Leader,  F = Follower                     │
│                                                       │
│   ZooKeeper / KRaft: Cluster metadata, leader election│
└──────────────────────────────────────────────────────┘

Producer → Broker (Leader of target partition) → replicate to Followers
Consumer → Broker (Leader of target partition) → read from offset
```

### Topic → Partitions → Segments

```
Topic "payments":
  Partition 0: [offset 0][offset 1][offset 2]...[offset N]
  Partition 1: [offset 0][offset 1]...[offset M]
  Partition 2: [offset 0][offset 1]...[offset K]

Each partition:
  = Ordered, immutable sequence of records
  = Stored as log segments on disk

Segment file (on broker disk):
  /kafka/data/payments-0/00000000000000000000.log
  /kafka/data/payments-0/00000000000000001000.log
  → Segments rolled when size or time threshold exceeded
  → Old segments deleted or compacted based on retention policy
```

### Why Kafka is Fast

**Sequential disk writes:** Kafka writes to disk sequentially (append-only). Sequential I/O is orders of magnitude faster than random I/O. A modern disk can do 600 MB/s sequential, vs 150 MB/s random.

**Zero-copy:** Kafka uses `sendfile()` system call to transfer data from disk to network without copying through user space. Data goes: disk → kernel buffer → NIC. No intermediate copy.

**Batching:** Producers batch multiple messages into one network packet. `batch.size` and `linger.ms` control batching aggressiveness.

**Compression:** Messages compressed (snappy, lz4, zstd) in batches. Better compression ratio than per-message.

### Consumer Offset

```
Partition 0: [msg0][msg1][msg2][msg3][msg4][msg5]

Consumer Group A offset: 3 (processed through msg2, next to process: msg3)
Consumer Group B offset: 5 (processed through msg4, next to process: msg5)

Offsets stored in Kafka's internal topic: __consumer_offsets

On consumer restart:
  Consumer asks Kafka: "What's my offset for partition 0?"
  Kafka returns: 3
  Consumer resumes from msg3
```

### Key Configuration

```
Producer:
  acks = all          # Wait for all replicas to ACK (durability, slower)
  acks = 1            # Wait for leader only (faster, lose data if leader crashes)
  acks = 0            # Fire and forget (fastest, may lose data)
  
  retries = 2147483647    # Retry on failure
  enable.idempotence = true  # Exactly-once producer semantics
  
Consumer:
  auto.offset.reset = earliest   # Start from beginning of topic (good for new groups)
  auto.offset.reset = latest     # Only new messages (good for live consumers)
  enable.auto.commit = false     # Manually commit offsets (don't lose messages)
  max.poll.records = 500         # Batch size per poll
  
Broker (retention):
  log.retention.hours = 168      # Keep 7 days of data
  log.retention.bytes = -1       # No size limit (size-based retention disabled)
  log.cleanup.policy = delete    # Delete old segments (vs compact)
```

### Log Compaction

```
Default retention: delete old segments after N days
Log compaction: keep only the LATEST value for each key

Topic "user-preferences" with compaction:
  [key:user_1, value:{theme:dark}]
  [key:user_2, value:{theme:light}]
  [key:user_1, value:{theme:light}]  ← user_1 changed theme

After compaction:
  [key:user_1, value:{theme:light}]  ← only latest
  [key:user_2, value:{theme:light}]

Use case: Change Data Capture (latest value per entity)
         Config/settings topics (latest config per key)
```

---

## Production Example

### IAM Platform: Kafka as Event Backbone

```
Event flow:
  User created in DB
  → UserCreatedEvent published to Kafka: "iam-user-events"
  → Consumer Group: email-service → sends welcome email
  → Consumer Group: onboarding-service → creates workspace
  → Consumer Group: audit-service → logs to audit trail
  → Consumer Group: analytics-service → increments user count

Permission changed in DB
  → PermissionChangedEvent to "iam-permission-events"
  → Consumer Group: cache-invalidator → deletes Redis cache keys
  → Consumer Group: audit-service → logs permission change
  
All events retained 7 days:
  → New consumer group can "replay" last 7 days to catch up
  → During incident investigation, replay events to understand sequence
```

---

# Chapter 39: RabbitMQ Fundamentals

## Why Should You Care?

RabbitMQ is the alternative messaging system when Kafka's complexity is overkill. For task queues, job processing, and traditional messaging patterns, RabbitMQ is simpler and more feature-rich for routing and filtering.

---

## RabbitMQ vs Kafka

| Aspect | RabbitMQ | Kafka |
|--------|---------|-------|
| Model | Smart broker, dumb consumer | Dumb broker, smart consumer |
| Message retention | Deleted after consumption | Retained for configured period |
| Consumer tracking | Broker tracks | Consumer tracks (offsets) |
| Ordering | Per-queue | Per-partition |
| Throughput | ~50K msg/sec | ~1M msg/sec |
| Routing | Rich (exchanges, bindings) | Simple (topic, partition key) |
| Replay | No (deleted on consume) | Yes (offset-based) |
| Complexity | Lower | Higher |
| Use case | Task queues, job processing | Event streaming, event sourcing |

### RabbitMQ Concepts

```
Producer → Exchange → Binding → Queue → Consumer

Exchange types:
  Direct: Route to queue by exact routing key match
  Topic: Route by pattern match ("payments.*", "*.completed")
  Fanout: Route to ALL bound queues (broadcast)
  Headers: Route by message headers

Example:
  Producer publishes: routing_key="payment.completed"
  
  Exchange routes to:
    Queue "email-notifications" (bound: "payment.*")
    Queue "analytics" (bound: "#.completed")
    Queue "audit" (bound: "#")
```

### When to Choose RabbitMQ vs Kafka

**Choose RabbitMQ when:**
- You need complex routing/filtering logic
- Message replay is not required
- You want simple task queue (celery-style job processing)
- Message lifecycle is simple (process and delete)
- Lower operational complexity matters

**Choose Kafka when:**
- You need message replay
- High throughput (>100K messages/second)
- Multiple consumer groups (pub/sub + point-to-point)
- Event sourcing / CQRS
- Stream processing (Kafka Streams, Flink)
- Cross-datacenter replication

**In practice:** Most modern platforms use Kafka for event streaming and Redis (as a lightweight queue) for simple task queuing. RabbitMQ is common in enterprise environments (Spring AMQP, .NET), while Kafka dominates greenfield high-volume systems.
# Part 5: Distributed Systems

> "The hard problems in distributed systems are not technical. They are about reasoning under uncertainty, partial failure, and unbounded latency."

---

## Chapter 40: Scalability

### Interview Question

> "Your service handles 1,000 requests per second today. In 18 months you expect 100x growth. Walk me through how you'd design for that."

---

### Why Should You Care?

Every system starts small. Every system that succeeds eventually confronts scale. Scalability is not a feature you add later — it is a dimension you design for from the beginning, even if you don't implement it immediately.

The engineer who can reason about *where* a system breaks at 10x, 100x, and 1000x load — and explain the tradeoffs of each approach — is worth far more than the engineer who can only describe the current architecture.

---

### Plain English Explanation

Scalability means: when demand increases, can your system handle it — and at what cost?

Two fundamental strategies exist:

**Vertical scaling** (scale up): Give one machine more power. More CPU cores. More RAM. Faster disks. Bigger network interface.

**Horizontal scaling** (scale out): Add more machines. Run many copies of your service, and distribute the load across them.

These are not equivalent. They have different cost curves, failure profiles, complexity tradeoffs, and ceilings.

---

### Internal Mechanics

#### Vertical Scaling

```text
Before:                    After:
┌─────────────┐           ┌─────────────────────┐
│  4 cores    │   -->     │  64 cores           │
│  16 GB RAM  │           │  512 GB RAM         │
│  500 GB SSD │           │  10 TB NVMe         │
│  1 Gbps NIC │           │  100 Gbps NIC       │
└─────────────┘           └─────────────────────┘
```

Advantages:
- No code changes required
- No distributed system complexity
- Low operational overhead
- Strong consistency — single machine, no network
- Fast: no inter-node communication

Disadvantages:
- **Hard ceiling**: the largest machine in existence has finite specs
- **Single point of failure**: one machine goes down, everything goes down
- **Cost curve is exponential**: doubling RAM costs 4–10x, not 2x
- **Downtime to upgrade**: typically requires reboot/migration
- **No geographic distribution**: all users pay latency to one location

When to use vertical scaling:
- Databases with complex transactions (PostgreSQL, MySQL) — often scale up before out
- Caches (Redis single-node fits in memory)
- Message brokers needing low-latency coordination
- ML inference servers needing GPU co-location
- When the complexity of horizontal scaling isn't worth it yet

#### Horizontal Scaling

```text
Before:                    After:
                          ┌──────────────┐
                          │ Load Balancer│
                          └──────┬───────┘
┌─────────────┐                  │
│  Server A   │     ┌────────────┼────────────┐
│  4 cores    │ --> │            │            │
│  16 GB RAM  │  Server A    Server B     Server C
└─────────────┘   4 cores    4 cores     4 cores
                  16 GB      16 GB       16 GB
```

Advantages:
- Linear cost scaling (add one machine, get one unit more capacity)
- No theoretical ceiling — add machines indefinitely
- Fault tolerance: one machine dies, others absorb load
- Geographic distribution: place nodes near users
- Rolling upgrades without downtime

Disadvantages:
- **Distributed systems complexity**: partial failures, network partitions, clock skew
- **Stateful services are hard**: sessions, caches, sticky state
- **Consistency challenges**: distributed transactions, eventual consistency
- **Operational overhead**: service discovery, load balancing, health checks
- **Data distribution**: sharding, replication, cross-shard queries

#### The Stateless vs Stateful Problem

The root challenge of horizontal scaling is **state**.

Stateless services (API servers, rendering services) scale horizontally trivially — any request can go to any node. Add nodes, add capacity.

Stateful services (databases, caches, sessions) are hard to scale horizontally because:
- Data must be partitioned across nodes
- Reads and writes must be routed to the correct node
- Node failures require data migration and failover
- Distributed transactions span nodes

This is why the standard architecture is:
- Stateless application tier: horizontally scaled freely
- Stateful data tier: vertically scaled first, then carefully partitioned horizontally

```text
┌──────────────────────────────────────────────────┐
│            STATELESS TIER (scale freely)         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ API Node │  │ API Node │  │ API Node │       │
│  └──────────┘  └──────────┘  └──────────┘       │
└─────────────────────┬────────────────────────────┘
                      │
┌─────────────────────┴────────────────────────────┐
│            STATEFUL TIER (scale carefully)       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │  Shard 1 │  │  Shard 2 │  │  Shard 3 │       │
│  │(Primary) │  │(Primary) │  │(Primary) │       │
│  └──────────┘  └──────────┘  └──────────┘       │
└──────────────────────────────────────────────────┘
```

#### Scalability Dimensions

Requests per second (RPS) is one dimension. Others matter equally:

| Dimension | What it means | Bottleneck |
|---|---|---|
| Throughput | Requests/sec processed | CPU, network bandwidth |
| Latency | Time per individual request | DB queries, serialization, GC |
| Storage | Total data volume | Disk, memory |
| Connections | Concurrent open connections | File descriptors, RAM |
| Bandwidth | Data in/out per second | NIC, CDN egress |
| Read/Write ratio | Read vs write operations | Cache effectiveness, replication lag |

A system might scale perfectly for throughput but collapse under storage growth. Design for the dimension that will actually be your bottleneck.

#### The Scalability Ceiling

Every scaling decision has a ceiling. Staff engineers think about *when* they'll hit it:

- Vertical: limited by hardware
- Stateless horizontal: limited by shared state (database)
- Database vertical: limited by hardware
- Database horizontal (sharding): limited by cross-shard operations
- CDN: limited by origin capacity
- Message queue: limited by partition count and broker capacity

The ceiling for most web applications is the database. Everything else scales; the database does not.

---

### Production Example: IAM Platform

An IAM platform at a bank handles:
- Authentication (login): 5,000 requests/sec at peak
- Token validation (every API call): 500,000 requests/sec
- Permission checks (every authorization decision): 1,000,000 req/sec

Three different scaling strategies:

**Authentication service** (stateless after session issuance):
Scale horizontally. Each auth node handles requests independently. Session tokens stored in Redis cluster.

**Token validation** (pure computation — validate JWT signature):
Scale horizontally, with in-process caching of JWK sets. No database hit required per request.

**Permission checks** (RBAC lookup — is user X allowed to do Y on resource Z?):
The bottleneck. Options:
1. Cache permissions at application tier (fast, but stale)
2. Materialize permissions into Redis (fast, requires cache invalidation)
3. Use a dedicated policy engine (OPA) with local policy cache
4. Eventually: read replicas of the IAM database, with 1-2 second consistency tolerance

The permission check cannot scale infinitely by adding API nodes — the data tier becomes the limit.

---

### ASCII Architecture Diagram

```text
                    SCALABILITY EVOLUTION

Phase 1: Monolith (1-100k users)
┌─────────────────────────────┐
│    Single VM                │
│  ┌──────┐  ┌─────────────┐  │
│  │ App  │  │  PostgreSQL │  │
│  └──────┘  └─────────────┘  │
└─────────────────────────────┘

Phase 2: App/DB Separation (100k-1M users)
┌──────────────┐     ┌──────────────────┐
│  App Server  │ --> │  PostgreSQL       │
│  (2 nodes)   │     │  (Primary + RR)  │
└──────────────┘     └──────────────────┘

Phase 3: Caching + LB (1M-10M users)
                ┌─────────────────┐
Clients -->     │  Load Balancer  │
                └────────┬────────┘
                         │
            ┌────────────┼────────────┐
         App Node 1   App Node 2   App Node 3
                         │
              ┌──────────┼──────────┐
           Redis      PostgreSQL    PostgreSQL
           Cluster    Primary       Read Replica

Phase 4: Sharded (10M-1B users)
                ┌──────────────────┐
Clients -->     │   API Gateway    │
                └────────┬─────────┘
                         │
           ┌─────────────┼─────────────┐
        App Node N    App Node N    App Node N
                         │
          ┌──────────────┼──────────────┐
       DB Shard 1    DB Shard 2    DB Shard 3
      (users 0-33%)  (33-66%)     (66-100%)
```

---

### Common Failure Modes

**1. Scaling the wrong tier**: Adding API nodes doesn't help if the database is saturated. Always instrument and find the real bottleneck before scaling.

**2. Stateful application tier**: Engineers store session state in local memory (HashMaps, Caffeine) in API nodes, then add a load balancer. Requests start failing because node 2 doesn't have the session that node 1 created. Fix: externalize all state to Redis or a database.

**3. Thundering herd after scaling**: A new node joins the pool cold — empty cache. The load balancer sends it full traffic. It saturates its database connections while warming up. Fix: canary deployments, slow traffic ramp, pre-warming.

**4. Database connection exhaustion**: 50 API nodes × 20 connections each = 1,000 PostgreSQL connections. PostgreSQL struggles beyond ~500 connections. Fix: PgBouncer connection pooling in front of every database.

**5. Ignoring the read/write ratio**: A system with 95% reads can scale dramatically with read replicas. A system with 50% writes cannot — those writes still go to the primary. Understand your workload before choosing your scaling strategy.

---

### Tradeoffs

| Strategy | Speed to implement | Cost | Operational complexity | Ceiling |
|---|---|---|---|---|
| Vertical scale | Hours | Exponential | Low | Hardware limit |
| Stateless horizontal | Days | Linear | Medium | DB |
| Read replicas | Days | Linear | Medium | Primary write throughput |
| Caching | Days | Low | Medium | Cache invalidation complexity |
| Sharding | Weeks-months | Linear | High | Cross-shard operations |
| Multi-region | Months | High | Very high | Consistency model |

---

### Interview Discussion

Interviewers want to hear:
1. Where is the bottleneck at each stage of growth?
2. What changes as you scale from 1x to 10x to 100x?
3. What is stateful vs stateless in your system?
4. What are the consistency tradeoffs of your scaling approach?

Common mistakes:
- Jumping straight to "add more servers" without identifying the bottleneck
- Forgetting the database is almost always the bottleneck
- Ignoring connection pooling
- Not considering the operational cost of horizontal complexity

---

### Staff-Level Thinking

A staff engineer asks:

> "We can scale this today with vertical scale — it's fast and simple. We'll need to revisit in 12 months when we hit the hardware ceiling. At that point we'll need connection pooling and read replicas. Sharding is a 6-month project and we don't need it for 3 years at current growth. Let's not overengineer."

Scalability decisions are *timing* decisions as much as technical ones. Premature horizontal scaling adds complexity that kills velocity. Too-late scaling causes incidents that kill users. The staff engineer knows where the ceiling is, and plans the transition *before* it hits, not after.

---

## Chapter 41: Load Balancing

### Interview Question

> "Explain how you'd design the load balancing layer for a payment processing service. What algorithm would you use and why?"

---

### Why Should You Care?

Load balancing is the entry point of every distributed system. It is the first thing a request hits and the last point of centralized control before your system fragments into many nodes. A misunderstood load balancer is the source of: uneven traffic distribution, sticky session bugs, connection exhaustion, thundering herds, and cascading failures.

---

### Plain English Explanation

A load balancer is a traffic director. It sits between your clients and your servers, receives all incoming requests, and decides which backend server should handle each one.

The goal: no single server gets overwhelmed while others sit idle. Without a load balancer, your system can only scale to a single server's capacity.

---

### Internal Mechanics

#### L4 vs L7 Load Balancing

This is the most important distinction in load balancing.

**Layer 4 (Transport Layer) Load Balancing**

Operates at the TCP/UDP level. The load balancer sees source IP, destination IP, and port — but not the HTTP payload.

```text
Client: TCP SYN --> Load Balancer --> Backend
                    (routes by IP/port, not content)
```

Characteristics:
- Extremely fast (no protocol parsing)
- Cannot inspect HTTP headers, paths, or bodies
- One TCP connection per backend routing decision
- Lower latency overhead (microseconds)
- Examples: AWS NLB, HAProxy in TCP mode, hardware load balancers

When to use L4:
- Non-HTTP traffic (raw TCP, UDP, gRPC)
- Extreme low-latency requirements
- When you don't need content-based routing
- As the outer layer in a two-tier LB setup

**Layer 7 (Application Layer) Load Balancing**

Terminates the incoming connection, inspects the HTTP request, then makes a new connection to the backend.

```text
Client --> LB (terminates TLS, reads HTTP) --> Backend
           (routes by path, header, cookie, body)
```

Characteristics:
- Slower (must parse HTTP, terminate TLS)
- Can route by URL path, headers, cookies, query params
- Can inject headers (X-Forwarded-For, request IDs)
- Can do A/B testing, canary deployments
- Can perform SSL offloading (terminate TLS at LB, plain HTTP to backends)
- Examples: AWS ALB, nginx, Envoy, HAProxy in HTTP mode, Traefik

When to use L7:
- HTTP/HTTPS services (almost always)
- Path-based routing to microservices (`/payments → payment-service`, `/users → user-service`)
- Sticky sessions based on cookies
- SSL termination

#### Load Balancing Algorithms

**Round Robin**

Requests go to servers in sequence: Server 1, Server 2, Server 3, Server 1, Server 2, ...

```text
Request 1 --> Server A
Request 2 --> Server B
Request 3 --> Server C
Request 4 --> Server A  (cycle repeats)
```

Pros: Simple, even distribution when requests are homogeneous.
Cons: Doesn't account for server capacity or current load. A slow request on Server A doesn't prevent more requests being sent there.

**Weighted Round Robin**

Like round robin but servers have weights. A server with weight 3 gets 3x more traffic than a server with weight 1.

Use case: heterogeneous backends (some nodes are larger/faster than others).

**Least Connections**

Send each new request to the server with the fewest active connections.

```text
Server A: 50 active connections
Server B: 12 active connections  <-- new request goes here
Server C: 30 active connections
```

Pros: Adapts to slow requests. A server handling a long-running request naturally receives fewer new ones.
Cons: Connection count isn't always a proxy for load. A server may have 5 connections each doing heavy CPU work.

**Least Response Time**

Send each request to the server with the lowest average response time. More accurate than connection count.

**IP Hash**

Hash the client's IP address to determine the server. Same client always goes to same server.

```text
SHA256(client_ip) % num_servers = server_index
```

Pros: "Sticky" routing — a client always hits the same server. Useful for stateful applications where server-local caching matters.
Cons: Uneven distribution if IP address space isn't uniform. Client IP changes (mobile networks, VPNs) break stickiness.

**Cookie-Based Sticky Sessions**

L7 load balancers can insert a cookie that identifies the backend server. Subsequent requests with that cookie always go to the same server.

Pros: More reliable stickiness than IP hash.
Cons: Doesn't help with server failures (the sticky server dies, session is lost unless it was externalized).

**Consistent Hashing**

Hash both servers and requests onto a ring. A request goes to the first server clockwise from its position on the ring.

```text
          0
    330       30
  300           60
 270      Ring   90
  240           120
    210       150
          180

Servers: A @ 60, B @ 180, C @ 300
Request X @ 40 --> goes to A (next clockwise)
Request Y @ 200 --> goes to C (next clockwise)
```

Pros: When a server is added or removed, only a fraction (1/N) of requests reroute. Critical for distributed caches.
Cons: Requires virtual nodes (vnodes) for even distribution; adds complexity.

#### Health Checks

Load balancers continuously probe backends:

```text
LB --> GET /health HTTP/1.1 --> Server A
LB <-- 200 OK               <-- Server A   (healthy)

LB --> GET /health HTTP/1.1 --> Server B
LB <-- timeout                             (unhealthy: remove from rotation)
```

Types:
- **Active**: LB sends probes every N seconds
- **Passive**: LB monitors real traffic for failure signals

When a backend fails health checks, it is removed from the rotation. When it recovers, it is added back. This is the foundation of automatic failover.

#### Connection Draining (Graceful Shutdown)

When a backend is removed (deployment, scale-down, failure), in-flight requests must not be killed. Connection draining:

1. Backend signals "shutting down"
2. Load balancer stops sending new requests to this backend
3. Existing in-flight requests complete (up to a timeout, e.g., 30s)
4. Backend shuts down cleanly

Without draining: users mid-payment see errors when a deployment happens.

#### SSL Termination

```text
Client --> [HTTPS] --> Load Balancer --> [HTTP] --> Backend
                       (terminates TLS)   (plain text within VPC)
```

Pros: Backends don't need TLS certs or TLS processing overhead. TLS config managed centrally.
Cons: Traffic within VPC is unencrypted (mitigated by network isolation, or use mTLS internally).

---

### Production Example: IAM Platform

An IAM platform has three backend services:
- `/auth/*` → Authentication service
- `/token/*` → Token service
- `/admin/*` → Admin service

L7 load balancer (nginx/Envoy) configuration:

```text
Client Request
      │
      ▼
┌─────────────────────────────────────────────┐
│           L7 Load Balancer (Envoy)          │
│                                             │
│  Path: /auth/*   --> Auth Service Pool      │
│  Path: /token/*  --> Token Service Pool     │
│  Path: /admin/*  --> Admin Service Pool     │
│                                             │
│  Algorithm: Least Connection                │
│  Health: GET /health every 5s               │
│  Drain: 30s graceful on deployment          │
└─────────────────────────────────────────────┘
      │              │              │
      ▼              ▼              ▼
Auth Nodes x3    Token Nodes x5   Admin Nodes x2
(stateless)      (stateless)      (stateless)
      │              │              │
      └──────────────┴──────────────┘
                     │
             ┌───────┴───────┐
         PostgreSQL       Redis
         (Primary)        Cluster
```

Token service gets 5 nodes because token validation is called on every API request — far higher traffic than auth or admin. The load balancer uses path-based routing to direct token requests to the larger pool.

For the token service, **Least Connection** works well because token validation response time varies based on whether the JWK set is cached (fast) or must be fetched (slow). The algorithm naturally avoids sending new requests to nodes currently doing a slow JWK refresh.

---

### Common Failure Modes

**1. Health check mismatch**: Health check passes (returns 200) but service is actually degraded (slow database queries, full connection pool). The LB thinks the node is healthy and sends full traffic to a struggling node. Fix: health checks that verify actual downstream dependencies, not just "am I running."

**2. Sticky session trap**: Using IP hash or cookies for stickiness because the app stores state in memory. A node dies, and those users' state is lost. Worse: one node gets disproportionate traffic from a high-volume IP (e.g., all traffic from a corporate NAT). Fix: externalize state; use stickiness only as a cache optimization, not a requirement.

**3. Connection exhaustion at LB**: Each backend keeps a pool of connections to the database. With many API nodes behind an LB, total connections = (nodes × pool_size). 100 nodes × 20 connections = 2,000 DB connections. PostgreSQL dies. Fix: PgBouncer in front of Postgres; or reduce pool sizes.

**4. Undetected slow nodes**: A node has a memory leak and responds slowly to real requests but still passes health checks. Round robin keeps sending traffic. Least Response Time or least connection algorithms handle this better.

**5. Split-brain at LB layer**: Two load balancers running but out of sync about which backends are healthy. Requests go to different backends based on which LB they hit. Fix: use managed LB services (AWS ALB), not DIY pairs; use active-passive failover not active-active.

---

### Tradeoffs

| Algorithm | Best for | Worst for |
|---|---|---|
| Round Robin | Homogeneous requests, stateless services | Variable request duration |
| Least Connections | Variable request duration | Not accounting for CPU load |
| IP Hash | Stateful apps needing stickiness | Uneven IP distribution |
| Consistent Hashing | Distributed caches | Stateful sessions (need vnodes) |
| Weighted Round Robin | Mixed capacity nodes | Real-time load adaptation |

---

### Interview Discussion

A strong answer for the payment processing load balancer question covers:
- L7 for content-based routing and TLS termination
- Least connections (payments vary wildly in duration: card auth is fast, 3DS is slow)
- Connection draining (never kill a payment mid-flight on deployment)
- Health checks that verify downstream DB connectivity
- No sticky sessions — stateless payment nodes, state in DB

---

### Staff-Level Thinking

A staff engineer thinks about the load balancer as a **reliability boundary**: it is the last component that can make a holistic decision before requests fragment into independent backend servers.

This means: implement request ID injection at the LB, log timing at the LB (Time to First Byte, upstream response time), and use the LB's access logs as the ground truth for latency SLO measurement. Everything downstream is harder to aggregate.

---

## Chapter 42: Consistency

### Interview Question

> "Your payment service is replicated across three data centers. A user submits a payment. What consistency guarantees does the system need, and how do you achieve them?"

---

### Why Should You Care?

Consistency is the most misunderstood concept in distributed systems. Most engineers know "eventual consistency" but cannot explain when it is safe to use, what anomalies it permits, or how it differs from "strong consistency" in practice. Getting this wrong causes data loss, duplicate charges, and security vulnerabilities.

---

### Plain English Explanation

In a distributed system, the same data lives on multiple machines (replicas). When you update data on one machine, the other machines must eventually reflect that update.

**The question is**: how long does "eventually" take, and what happens if someone reads from a replica that hasn't received the update yet?

Consistency models define the rules about what a reader can observe.

---

### Internal Mechanics

#### Strong Consistency (Linearizability)

Every read sees the most recent write. The system behaves as if there is only one copy of the data.

```text
Time -->
Client A: write(x=5)    [completes]
Client B:                           read(x) --> 5  (always)

Even if B reads from a different replica than A wrote to,
B will not see a stale value.
```

How it works: The write is not acknowledged until all replicas (or a quorum) have received and applied it. Any subsequent read on any replica returns the latest value.

Cost: Write latency increases because you must wait for all replicas to confirm. If any replica is unreachable, the write blocks or fails.

Use cases: Financial balances, inventory counts, distributed locks, permission changes (security-sensitive).

#### Eventual Consistency

All replicas will converge to the same value *eventually*, but a read might return a stale value during the convergence window.

```text
Time -->
Client A: write(x=5)   --> Replica 1 [ack to A]
                       --> Replica 2 [propagating...]
                       --> Replica 3 [propagating...]

Client B:  read(x) --> might see 3 (old value) if B hits Replica 2
           read(x) --> might see 5 (new value) if B hits Replica 1
           read(x) --> will eventually see 5 when Replica 2 is updated
```

How it works: Write is acknowledged after the primary receives it. Replication to other replicas happens asynchronously. Reads may return stale data.

Cost: Reads can be stale. Applications must be designed to tolerate this.

Use cases: Shopping cart, social media likes, DNS propagation, CDN cache, product catalog.

#### Read-Your-Writes Consistency

After a user writes data, their subsequent reads will see that write — even if other users might see stale data.

```text
User A: write(profile_name="Alice")
User A: read(profile_name) --> "Alice"  (guaranteed)

User B: read(profile_name) --> might see "Bob" (old value) temporarily
```

How it works: Route the user's reads to the same replica they wrote to (sticky routing), or track the write's replication sequence number and ensure the read is served by a replica that has caught up to that point.

Use cases: Any update-then-view flow. User updates their profile, immediately views it.

#### Monotonic Reads

Once a user reads a value, subsequent reads will not return an older value.

```text
Without Monotonic Reads:
User reads post (at t=10) --> sees 150 likes
User reads post (at t=11) --> sees 140 likes   (stale replica!)
User reads post (at t=12) --> sees 150 likes

With Monotonic Reads:
User reads post (at t=10) --> sees 150 likes
User reads post (at t=11) --> sees 150 or more likes (never less)
```

How it works: Pin a user's reads to a specific replica for the duration of a session.

**Causal Consistency**

If event A caused event B, any reader who sees B will also see A.

```text
Comment posted: "Great post, Alice!"
Reply posted:   "Thanks, Bob!"

A reader who sees the reply will also see the original comment.
(Without causal consistency: you might see "Thanks, Bob!" with no context.)
```

#### Quorum Reads and Writes

For a cluster of N replicas:
- **W** = number of replicas that must confirm a write
- **R** = number of replicas that must respond to a read
- **Consistency guarantee**: if W + R > N, reads always overlap with at least one replica that received the latest write

```text
N=3 replicas:

W=3, R=1: Strong consistency on writes. Fast reads. Slow writes (must wait for all 3).
W=1, R=3: Fast writes. Strong reads. Any single-replica write is immediately visible.
W=2, R=2: Balanced. W+R=4 > N=3. Overlaps by at least 1. Strong consistency.
W=1, R=1: Fast everything. W+R=2 = N=2 < 3+1. No overlap guaranteed. Eventual consistency.
```

This is the fundamental mechanic behind Cassandra's tunable consistency, DynamoDB's strongly consistent reads, and distributed databases generally.

---

### Production Example: IAM Platform

**Token issuance** (strong consistency required):
When a token is issued and the user's permissions are changed (role revoked), the next authentication check must reflect the revocation. Using eventual consistency here would allow a revoked user to continue operating for seconds or minutes after revocation.

Solution: Write permission changes to the primary with W=quorum, read with R=quorum. For a 3-node cluster: W=2, R=2 guarantees overlap.

**Audit logging** (eventual consistency acceptable):
Audit logs are append-only. A 2-second replication lag means the audit log UI might show a recent event with slight delay. This is acceptable — the event is not lost, just delayed in appearing. Use W=1 (fast write), and the logs converge quickly.

**Session cache in Redis** (read-your-writes required):
User logs in → session is written to Redis primary. Immediate redirect shows the dashboard, which reads the session. If this read hits a replica that hasn't received the write yet, the user appears logged out immediately after logging in.

Solution: Write to primary, read from primary (or use `WAIT` command to ensure replication before redirecting).

---

### ASCII Diagram: Consistency Spectrum

```text
CONSISTENCY SPECTRUM
Stronger ←─────────────────────────────────────────→ Weaker

┌──────────────┬──────────────┬──────────────┬──────────────┐
│ Linearizable │   Sequential │    Causal    │  Eventual    │
│              │  Consistency │  Consistency │  Consistency │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ Behaves as   │ Operations   │ Cause-effect │ Replicas     │
│ single copy  │ appear in    │ ordering     │ converge     │
│ of data      │ some total   │ preserved    │ eventually   │
│              │ order        │              │              │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ Highest      │ High latency │ Medium       │ Lowest       │
│ latency      │              │ latency      │ latency      │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ Payments     │ Distributed  │ Social feed  │ DNS, CDN,    │
│ Locks        │ coordination │ Comments     │ Likes, Cart  │
│ Permissions  │              │              │              │
└──────────────┴──────────────┴──────────────┴──────────────┘
```

---

### Common Failure Modes

**1. Using eventual consistency for security-sensitive data**: A role is revoked, but the service reads from an eventually consistent cache. The revoked user continues to operate for 30 seconds. This is a security incident.

**2. Read-after-write failure**: User updates their password, is redirected to the dashboard, dashboard reads from a read replica, sees the old password, shows "incorrect password" on the session check. User experiences a phantom logout.

**3. Monotonic read violation causing confusion**: User views their order history (150 items), refreshes (140 items — different replica), refreshes again (150 items). User calls support believing items were deleted.

**4. Quorum misconfiguration**: N=3, W=1, R=1. Engineers believe the system is "replicated." A network partition isolates one replica. Two replicas receive writes. The isolated replica diverges. After partition heals, last-write-wins resolution silently overwrites data. The system appeared to work fine during the partition.

---

### Tradeoffs

| Model | Latency | Availability | Anomalies possible |
|---|---|---|---|
| Linearizable | Highest | Lowest | None |
| Read-Your-Writes | Low (with routing) | High | Other users see stale |
| Monotonic Reads | Low | High | Cross-user stale reads |
| Eventual | Lowest | Highest | All anomalies possible |

---

### Interview Discussion

For the payment consistency question, a strong answer:
1. Identifies that payment submission needs strong consistency (no duplicate processing)
2. Identifies that payment history reads can tolerate slight staleness (eventual OK)
3. Explains quorum mechanics
4. Mentions idempotency keys as a complement to consistency (Chapter 51)
5. Explains the latency cost tradeoff of strong consistency

---

### Staff-Level Thinking

Consistency is not binary. The mistake is choosing "strong" or "eventual" for an entire system. A staff engineer reasons about **consistency per operation**:
- Write the money transfer record: linearizable
- Read the recent transaction list: eventual (with monotonic reads)
- Check if user has a permission: read-your-writes (after permission change)
- Serve the landing page product catalog: eventual

The right model is the weakest one you can use safely for each specific operation. Weaker consistency = lower latency = higher availability = lower cost.

---

## Chapter 43: CAP Theorem

### Interview Question

> "You're designing a distributed database for a payment system. How does CAP theorem influence your architecture decisions?"

---

### Why Should You Care?

CAP theorem is one of the most cited and most misunderstood theorems in distributed systems. Engineers either apply it dogmatically ("we chose AP because we need availability") or dismiss it ("CAP is outdated"). Both are wrong. CAP captures a fundamental truth about network partitions that shapes every distributed database design.

---

### Plain English Explanation

CAP theorem states: a distributed system can guarantee at most **two** of three properties simultaneously:

- **C**onsistency: every read receives the most recent write (or an error)
- **A**vailability: every request receives a response (not an error) — though it might be stale
- **P**artition tolerance: the system continues operating despite network partitions (messages between nodes being dropped or delayed)

The crucial insight: **P is not optional**. Network partitions happen. Networks fail, packets get dropped, a switch misconfigures, a data center loses connectivity. You cannot choose "not to tolerate partitions" in a real distributed system.

Therefore, the real choice is: when a partition occurs, do you sacrifice **C** (allow stale reads) or **A** (return errors)?

---

### Internal Mechanics

#### What is a Partition?

```text
Data Center A ════════════════ Data Center B
    Node 1          X             Node 2
                  (partition:
                  messages not
                  getting through)
```

A partition means nodes cannot communicate. A node doesn't know if the remote node is slow, dead, or unreachable due to a network issue.

#### The Fundamental Dilemma

During a partition, if a client sends a write to Node 1:
- Node 1 cannot propagate the write to Node 2
- Node 2 doesn't know about the write
- A client reading from Node 2 will see stale data

**Option 1: CP — Sacrifice Availability for Consistency**

Node 1 refuses the write (returns an error) until it can confirm Node 2 received it. During the partition, no writes succeed. The system remains consistent but unavailable.

```text
Client: write(x=5) --> Node 1
Node 1: "I can't reach Node 2. I will not accept this write."
Node 1 --> Client: ERROR 503 Service Unavailable

Consistency preserved ✓
Availability violated ✗
```

Examples: ZooKeeper, HBase, Consul (default), etcd, traditional relational databases with synchronous replication.

**Option 2: AP — Sacrifice Consistency for Availability**

Node 1 accepts the write and acknowledges it, even though Node 2 doesn't know about it yet. Clients reading from Node 2 during the partition will see stale data. After the partition heals, data is reconciled (possibly with conflicts).

```text
Client: write(x=5) --> Node 1
Node 1: "I'll accept this even though Node 2 is unreachable."
Node 1 --> Client: 200 OK

Client: read(x)    --> Node 2
Node 2: returns x=3 (old value, hasn't received the write yet)

Availability preserved ✓
Consistency violated ✗
```

Examples: Cassandra, DynamoDB, CouchDB, Riak, DNS.

#### The Reconciliation Problem

AP systems must reconcile divergent state after a partition heals. Strategies:

**Last-Write-Wins (LWW)**: The write with the latest timestamp wins. Simple but dangerous — clocks are not perfectly synchronized in distributed systems. A write from the future (clock skew) could win.

**Vector Clocks**: Track the causal history of each value. Detect conflicts (both nodes modified the same value). Present conflicting versions to the application to resolve. Used by Riak, Amazon's DynamoDB internally.

**CRDT (Conflict-Free Replicated Data Types)**: Data structures designed so that all merge operations are commutative, associative, and idempotent. Merging always produces a consistent result, regardless of order. Examples: counters, sets, maps. Used by Redis CRDT, Riak, Apple's Notes.

---

### Production Example: Payment System

**The payment system choice:**

Consider a payment processing service with two data centers:

```text
DC East: Node A                      DC West: Node B
(Processes East Coast payments)      (Processes West Coast payments)
            │                                    │
            └──────── Network Partition ─────────┘
```

During the partition:

**If CP (consistent, sacrifices availability):**
- No writes accepted until partition heals
- East Coast users cannot make payments for the duration of the partition
- 0% data inconsistency, but 100% of users are impacted during the outage

**If AP (available, sacrifices consistency):**
- Both nodes accept payments independently
- After partition heals: what if a user's balance was decremented on both sides for the same transaction?
- Risk: double-spend. For payments, this is catastrophic.

**The right answer: CP for core payment state.**

However, this doesn't mean the entire payment system is CP. A nuanced design:
- Balance deductions: CP (must be consistent, cannot allow double-spend)
- Payment confirmations/receipts: eventually consistent (acceptable to see delay)
- Analytics/reporting: eventually consistent (read from replicas)
- Fraud signals: eventually consistent (near-real-time is fine)

---

### ASCII Diagram

```text
CAP THEOREM DURING NETWORK PARTITION

        Consistency
             │
    CP       │       (not possible: need P)
  systems    │
    ZK,etcd  │
    RDBMS    │
             │
─────────────┼────────────────── Partition
             │              Tolerance (required)
    CA        │       AP
  (single node│    systems
  not distributed│   Cassandra,
  systems)    │    DynamoDB,
             │    CouchDB
           Availability

PRACTICAL CHOICE DURING PARTITION:
┌─────────────────────────────────────────────────────┐
│                                                     │
│   Should this node accept writes during partition?  │
│                                                     │
│   YES (AP): serve users, risk stale/conflicting     │
│             data after partition heals              │
│                                                     │
│   NO  (CP): return errors, users can't proceed,    │
│             but data is never inconsistent          │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

### Common Failure Modes

**1. CP system in a partition causes extended outage**: Engineers choose CP for consistency but don't have a plan for what users experience during the partition. If the partition lasts 10 minutes, the system is down for 10 minutes.

**2. AP system with financial data**: Engineering chooses "high availability" for a financial system without thinking through what consistency violations mean. Result: duplicate debits, inconsistent balances.

**3. Misidentifying the partition**: Engineers say "we chose AP" but they've actually configured synchronous replication. During a partition, their system blocks (not AP — it's effectively CP). Understand what your system actually does.

**4. Ignoring partition recovery**: AP systems diverge during partitions. If the team hasn't implemented conflict resolution, partition recovery causes silent data corruption.

---

### Tradeoffs

| System type | During partition | After partition | Example use cases |
|---|---|---|---|
| CP | Returns errors | No reconciliation needed | Financial balances, distributed locks |
| AP | Accepts requests (stale responses) | Must reconcile | Product catalog, user preferences, DNS |

---

### Interview Discussion

Red flags:
- Saying "we chose CA" (impossible for distributed systems under partition)
- Not knowing what "partition" means
- Not understanding that the choice is AP vs CP *during a partition*

Green flags:
- Explaining the payment system needs CP for balance state
- Explaining that other parts of the system can be AP
- Understanding conflict resolution strategies for AP

---

### Staff-Level Thinking

CAP is not a shopping list. It is a tool for reasoning about **what your system must do when things go wrong**. A staff engineer defines the partition behavior *before* a partition occurs.

For each data type in your system, ask: "If a network partition isolates two of our nodes for 10 minutes, what should happen?" That answer — "serve stale data" or "return errors" — defines your choice. Then verify your database and configuration actually implement that choice.

---

## Chapter 44: PACELC

### Interview Question

> "CAP theorem says you can have consistency or availability during a partition. Is that the whole picture? What happens when there's no partition?"

---

### Why Should You Care?

CAP theorem only describes behavior *during partitions*. But partitions are rare in well-operated systems. Most of the time, your system is running normally — and even then, there's a fundamental tradeoff: **latency vs consistency**. PACELC captures this.

---

### Plain English Explanation

PACELC extends CAP:

> If there is a **P**artition, choose between **A**vailability and **C**onsistency (same as CAP).
> **E**lse (no partition), choose between **L**atency and **C**onsistency.

The "else" is the key insight. Even in normal operation, achieving strong consistency requires coordinating multiple replicas — which takes time (latency). Relaxing consistency allows faster responses.

---

### Internal Mechanics

#### The Latency-Consistency Tradeoff (No Partition)

```text
Strong Consistency Read (no partition):
  Client → Replica 1 → Replica 2 → Replica 3
                   ↓ quorum achieved ↓
                        Response

Latency: sum of network round trips to coordinate quorum
Typical: 5-15ms per quorum read in same DC, 50-200ms cross-DC

Eventual Consistency Read (no partition):
  Client → Replica 1 → Response (immediately, no coordination)

Latency: one network hop to nearest replica
Typical: <1ms in same DC
```

To get a consistent read, you must wait for multiple replicas to agree. This adds latency proportional to the number of replicas and their geographic distance.

#### Database PACELC Classifications

| Database | Partition behavior | Else (no partition) behavior |
|---|---|---|
| Dynamo/Cassandra | PA (available during partition) | EL (low latency, eventual consistency) |
| ZooKeeper/etcd | PC (consistent during partition) | EC (consistent, higher latency) |
| PostgreSQL (sync replication) | PC | EC |
| MongoDB (default) | PA | EL |
| DynamoDB (strong reads) | PC | EC |
| DynamoDB (eventual reads) | PA | EL |

#### Tunable Consistency

Some systems (Cassandra, DynamoDB) let you choose the tradeoff *per request*:

```text
// Cassandra: choose consistency level per query
SELECT * FROM payments WHERE id=123
  WITH CONSISTENCY QUORUM;   -- EC: latency cost, strong consistency

SELECT * FROM product_catalog WHERE id=456
  WITH CONSISTENCY ONE;      -- EL: fast, eventual consistency
```

This is powerful: use strong consistency where it matters (financial data), eventual consistency where it doesn't (product catalog).

---

### Production Example: IAM Platform

**Login flow (EC, higher latency acceptable)**:
User login touches the credential database. We want consistency — an incorrect password should not succeed because of stale data, and a revoked account should not authenticate. We pay the latency cost (quorum read, ~5ms) and accept this.

**Token introspection (EL, latency critical)**:
Every API call introspects the access token. At 500,000 req/sec, each millisecond of additional latency means 500 additional seconds of cumulative wait per second of traffic. Here, eventual consistency (serve from in-memory cache, refresh every 30s) is worth the tradeoff. The risk: a revoked token may work for up to 30 seconds. This is a *designed*, *documented* risk with an acceptable window.

The PACELC framework makes this explicit: for token introspection, we choose EL (latency over consistency) in the else branch. We document the staleness window (30s) as a known characteristic.

---

### Staff-Level Thinking

PACELC forces engineers to reason about normal-case behavior, not just failure scenarios. Ask for every operation:

1. Is there a partition? → AP or CP?
2. No partition? → Do we want fast (EL) or consistent (EC)?
3. What is the acceptable staleness window if we choose EL?
4. Who is harmed by stale data in this specific case?

The answers to these questions are architecture documentation. A staff engineer writes them down.

---

## Chapter 45: Replication Models

### Interview Question

> "Walk me through the tradeoffs of Leader-Follower vs Multi-Leader vs Leaderless replication. When would you choose each?"

---

### Why Should You Care?

Replication is how data survives failures. Every database you use replicates data — but the *model* of that replication determines: how consistent the data is, how available the system is, and what anomalies are possible. Engineers who don't understand replication models are surprised when their "highly available" database loses data on failover.

---

### Plain English Explanation

In a replicated database, the same data lives on multiple nodes. The replication model defines: **who is allowed to accept writes, and how do writes propagate to other nodes?**

Three fundamental models:
1. **Leader-Follower**: One node is the writer, others are readers
2. **Multi-Leader**: Multiple nodes accept writes, must reconcile
3. **Leaderless**: Any node accepts writes, quorum-based

---

### Internal Mechanics

#### Leader-Follower (Primary-Replica)

```text
                 ┌─────────────┐
Writes only -->  │    Leader   │
                 └──────┬──────┘
                        │ replication stream
            ┌───────────┼───────────┐
            ▼           ▼           ▼
         Follower 1  Follower 2  Follower 3
         (Read OK)   (Read OK)   (Read OK)
```

**Write path**: All writes go to the leader. Leader writes to its WAL (Write-Ahead Log). Replication stream propagates to followers.

**Read path**: Reads can go to any node (leader or followers for eventual consistency; leader only for strong consistency).

**Synchronous vs Asynchronous Replication**:

*Synchronous*: Leader waits for at least one follower to confirm receipt before acknowledging the write to the client.
- Pro: No data loss on leader failure (follower has the write)
- Con: Write latency increases; if the follower is slow, writes are slow

*Asynchronous*: Leader acknowledges the write immediately after writing locally. Replication happens in the background.
- Pro: Fast writes (leader doesn't wait)
- Con: If the leader crashes before replication completes, the write is lost

**Failover**: When the leader fails, one follower is promoted to leader. This is the hard part:
- How is the new leader chosen? (Consensus algorithm, or manual intervention)
- What happens to writes the old leader acknowledged but hadn't replicated yet? (Lost)
- What if the old leader comes back and thinks it's still the leader? (Split-brain)

Failover bugs cause more data loss incidents than almost any other single issue. Chapter 16 covers this in depth.

**Replication Lag**: Asynchronous followers can fall behind the leader. During lag:
- Reads from followers may be stale
- Lag can grow during write spikes
- If a follower restarts, it must catch up from its last known position

**Leader-Follower Use Cases**:
- PostgreSQL, MySQL, MongoDB: standard relational/document database setup
- Redis: primary-replica replication
- Most databases default to this model

Pros: Simple writes (no conflict resolution), strong consistency reads available from leader, easy to reason about.
Cons: Leader is a bottleneck for writes, single point of failure (though mitigated by failover), read replicas can serve stale data.

#### Multi-Leader (Multi-Primary, Active-Active)

```text
                 ┌─────────────┐        ┌─────────────┐
Writes OK -->    │  Leader 1   │◄──────►│  Leader 2   │  <-- Writes OK
                 │  (DC East)  │  sync  │  (DC West)  │
                 └──────┬──────┘        └──────┬──────┘
                        │                      │
                 Followers           Followers
                 in DC East          in DC West
```

Both leaders accept writes. They replicate to each other (and to their own followers).

**The central problem: write conflicts.**

```text
Timeline:
T=1: Leader 1 sets user_name = "alice"
T=1: Leader 2 sets user_name = "bob"   (concurrent)
T=2: Leaders replicate to each other
     Leader 1 receives: "set user_name=bob"
     Leader 2 receives: "set user_name=alice"
CONFLICT: What should user_name be?
```

Conflict resolution strategies:
- **Last Write Wins (LWW)**: Highest timestamp wins. Simple, but clock skew causes issues.
- **Application-level resolution**: Surface conflicts to the application to resolve.
- **CRDT**: Use data structures that merge automatically without conflicts.

**Multi-Leader Use Cases**:
- Multi-datacenter deployments (each DC has a local leader for low-latency writes)
- Offline-capable applications (each device is a "leader"; syncs when connected)
- Collaborative editing (Google Docs; each client applies edits locally, syncs to server)

Pros: Writes accepted locally in each DC (low latency for geographically distributed users), survives single-DC failure.
Cons: Conflict resolution is complex and error-prone, not appropriate for data requiring strong consistency.

**Never use multi-leader for financial data without extremely careful conflict resolution.** A bank balance written concurrently on two leaders with LWW reconciliation will silently lose writes.

#### Leaderless (Dynamo-Style)

```text
                         Client
                        /  |  \
                       /   |   \
                 Node A  Node B  Node C
                 (writes  (writes  (writes
                  OK)      OK)      OK)

Write: send to W nodes, wait for W acknowledgements
Read:  send to R nodes, take newest value (by version clock)
```

Any node accepts reads and writes. Consistency is achieved by quorum: if W nodes must confirm a write, and R nodes must respond to a read, then with W + R > N, reads always see the latest write.

**Read Repair**: When a read is served by R nodes, if any node has a stale value, it is updated in the background.

**Anti-Entropy**: Background process continuously compares replicas and brings stale ones up to date.

**Leaderless Use Cases**:
- Amazon DynamoDB (original inspiration: Amazon Dynamo paper)
- Apache Cassandra
- Riak

Pros: No single point of failure, no leader election, tolerates node failures gracefully, writes succeed as long as W nodes are available.
Cons: Conflict resolution complexity (version vectors, LWW), quorum reads are more expensive than simple reads, harder to implement strong consistency.

---

### Production Example: IAM Platform

**Credentials and users** (Leader-Follower, PostgreSQL):
Writes go to the primary. Reads for auth go to primary (strong consistency). Reads for admin UI can go to read replicas (eventual is fine for listing users). Standard setup.

**Cross-region token caching** (Multi-Leader):
JWT signing keys are replicated multi-leader across regions. Each region can issue tokens locally. Signing key rotation is a write conflict risk — handled by application-level conflict resolution: a new key version number is assigned globally by a consensus service before rotation. All regions adopt the new key atomically.

**Session store** (Leaderless, Redis Cluster):
Redis Cluster uses a form of sharding with leader-per-shard (hybrid). But DynamoDB for sessions at scale: W=2, R=2, N=3. Fast writes, consistent reads at the cost of 2-node round trips.

---

### Common Failure Modes

**1. Split-brain in leader-follower**: Two nodes both believe they are the leader. Both accept writes. The network partition heals, and now two divergent histories must be reconciled. Databases use fencing tokens and STONITH (Shoot The Other Node In The Head) to prevent this.

**2. Replication lag causing stale reads**: Read replica is 5 seconds behind. A user updates their password and is immediately prompted to re-login. Login reads the old password hash from the replica. "Invalid password." User is confused. Fix: sticky reads to primary after writes.

**3. Losing writes on failover**: Asynchronous leader-follower. Leader processes 100 writes. Crashes. 20 writes hadn't replicated to any follower. New leader is elected. Those 20 writes are lost silently. Applications that assumed writes were durable (they got 200 OK) are now inconsistent.

**4. Multi-leader conflict avalanche**: Two DCs write conflicting records for 5 minutes during a network partition. When the partition heals, 10,000 conflicts flood the system. If conflict resolution is CPU-intensive (application-level), the reconciliation itself causes a performance incident.

---

### Tradeoffs

| Model | Write throughput | Write availability | Consistency | Complexity |
|---|---|---|---|---|
| Leader-Follower | Limited to leader | Leader is SPOF (with failover) | Strong from leader, eventual from replica | Low |
| Multi-Leader | Higher (distributed) | High (each DC independent) | Eventual (conflicts possible) | High |
| Leaderless | High | Very high | Tunable via quorum | Medium-High |

---

### Staff-Level Thinking

The question is not "which replication model should I use?" The question is: "What is my data's consistency requirement, and what is the cost of data loss?"

For financial data: Leader-Follower with synchronous replication (at least semi-sync). Pay the latency cost; do not lose writes.

For session data: Leaderless or Leader-Follower with async replication. Losing a session is annoying but not catastrophic. Fast writes more important.

For multi-DC writes: Multi-Leader — but *only* for data where conflict resolution is well-defined and testable. Never for balances or inventory without explicit conflict-free data structures.

---

## Chapter 46: Partitioning and Sharding (Advanced)

*This chapter builds on Chapter 17 (Sharding in Databases) and Chapter 15 (Partitioning) with distributed systems context.*

### Interview Question

> "You're sharding a 10 TB payments database. Walk me through the sharding key selection, hotspot prevention, and cross-shard transaction handling."

---

### Why Should You Care?

Sharding is the strategy of last resort for database scaling. It is operationally complex, introduces distributed transaction overhead, and is difficult to undo. Engineers who understand *when not to shard* are more valuable than those who can describe how to shard. But when sharding is necessary, getting the shard key wrong is a disaster.

---

### Plain English Explanation

Sharding (horizontal partitioning) distributes rows of a table across multiple database instances (shards). Each shard is an independent database holding a subset of the data.

The shard key determines which shard holds which row. Every read and write that includes the shard key can be routed to exactly one shard (fast). Queries that don't include the shard key must be sent to all shards (scatter-gather, slow).

---

### Internal Mechanics

#### Shard Key Selection

The shard key is the most consequential architectural decision when sharding. It determines:
- How evenly load is distributed
- Which queries are fast vs slow
- Whether hotspots form
- How hard it is to add shards later

**Good shard keys**:
- High cardinality (many distinct values)
- Uniform distribution (values don't cluster)
- Monotonically increasing? — see warning below
- Aligned with query patterns (most queries include this key)

**Bad shard keys and their failure modes**:

```text
Shard key: user_country
  US: 50% of users → 50% of writes → Shard 1 is a hotspot
  India: 30% of users → Shard 2 is a hotspot
  All others: 20% → Shard 3 is underloaded

Shard key: created_at (time-based)
  All new records → most recent time shard → hotspot
  Old shards: cold, mostly reads, underloaded
  New shard: hot, 100% of writes

Shard key: user_id (UUID or random)
  Even distribution across shards ✓
  Range queries (users created in March) → scatter-gather ✗
```

**The time-based hotspot problem** is extremely common in payments and analytics. All new transactions go to "this month's shard." Fix: use a composite key `hash(user_id) || created_month` to distribute writes while allowing time-based range queries per user.

#### Consistent Hashing for Shards

As covered in Chapter 17, consistent hashing minimizes data movement when nodes are added or removed:

```text
Hash ring (0 to 2^64):
         0
   N4         N1
 N3               N2
   (ring with 4 shards)

Adding N5:
  Only the portion of N2's range covered by N5 migrates.
  All other shards unaffected.

Without consistent hashing (simple modulo):
  Adding one shard changes hash(x) % 4 → hash(x) % 5
  Most records need to move. Massive data migration.
```

#### Cross-Shard Transactions

The core problem: ACID transactions within a single shard work normally. Transactions spanning two shards require distributed transaction protocols.

**Two-Phase Commit (2PC)**:

```text
Transaction coordinator (TC) manages a cross-shard transaction:

Phase 1 (Prepare):
  TC → Shard 1: "Prepare: deduct $100 from account A"
  TC → Shard 2: "Prepare: credit $100 to account B"
  Shard 1 → TC: "Prepared" (writes to WAL, locks row)
  Shard 2 → TC: "Prepared" (writes to WAL, locks row)

Phase 2 (Commit):
  TC → Shard 1: "Commit"
  TC → Shard 2: "Commit"
  Shards apply changes, release locks.
```

2PC is **blocking**: if the coordinator crashes after Phase 1 but before Phase 2, both shards hold locks indefinitely waiting for instruction. This is the "in-doubt transaction" problem.

**Saga Pattern** (alternative for long-running distributed transactions — Chapter 56): Break the transaction into a sequence of local transactions, each with a compensating action. If step N fails, run compensating actions for steps 1 through N-1.

More appropriate than 2PC for payment flows because it doesn't hold locks across services.

#### Scatter-Gather Queries

A query that doesn't include the shard key must go to all shards:

```text
Query: "Find all payments > $10,000 in March 2024"
(no shard key in predicate)

   ┌───────────────────────────────┐
   │    Query Coordinator          │
   └─────┬─────────┬──────┬────────┘
         │         │      │
      Shard 1   Shard 2  Shard 3
         │         │      │
      results  results  results
         │         │      │
         └─────────┴──────┘
                   │
            Merge + sort
                   │
              Final result
```

Performance: O(num_shards × per_shard_query_cost + merge_overhead)

As shard count grows, scatter-gather becomes prohibitive. Move analytics queries to a separate read store (data warehouse, columnar DB) that has all data denormalized.

---

### Production Example: Payments Database

**Shard key selection for payments**:

Option A: `payment_id` (UUID v4)
- Pros: Even distribution, no hotspots
- Cons: Most payment queries are by `user_id` + `created_at` → scatter-gather for user statements

Option B: `user_id`
- Pros: User's payment history is local to one shard (fast for user-facing queries)
- Cons: Uneven if user distribution is skewed (high-value merchants vs. casual consumers)

Option C: `hash(user_id)` with 1024 virtual shards across 8 physical shards
- Pros: Even distribution, all user queries go to one shard
- Cons: Cannot do range queries on user_id

**Recommended**: Shard by `hash(user_id)` with a virtual shard layer. User-facing queries (transaction history, balance) are in one shard. Analytics are served from a replicated data warehouse, not from the OLTP shards.

---

### Common Failure Modes

**1. Choosing the wrong shard key**: The most common and most expensive mistake. Resharding (changing the shard key) requires a full data migration — weeks of work, high risk.

**2. Hotspot shard**: One shard receives disproportionate traffic. The system degrades as if no sharding existed. Fix: add virtual shards (logical shards > physical shards) so individual rows can be redistributed without changing the sharding scheme.

**3. Cross-shard join explosion**: Queries that join across two sharded tables (e.g., users and orders) require scatter-gather on both tables. Performance collapses. Fix: co-locate related data (shard users and orders by the same user_id hash so they land on the same shard).

**4. 2PC coordinator failure**: In-doubt transactions hold locks. The database waits. Other transactions queue. The system appears deadlocked. Fix: implement 2PC coordinator HA (Paxos/Raft consensus), or avoid 2PC via Saga.

---

### Staff-Level Thinking

Before sharding, a staff engineer exhausts every alternative:
1. Can we add more RAM and fit the hot dataset in cache? (Days of work)
2. Can we use read replicas for reads? (Days of work)
3. Can we archive old data to object storage? (Days of work)
4. Can we vertically scale the database? (Hours of work)
5. Can we use a purpose-built database for the hot query pattern? (Weeks of work)

Sharding is months of work and permanent operational complexity. The question is always: "Is the problem we're solving worth that cost?"

---

## Chapter 47: Consensus

### Interview Question

> "How does distributed consensus work? How does Raft ensure that a leader failure doesn't cause data loss or split-brain?"

---

### Why Should You Care?

Consensus is the foundation of reliable distributed systems. Every time a cluster must agree on who the leader is, whether a transaction committed, or what the current configuration is — it needs consensus. Raft powers etcd, which powers Kubernetes. Paxos powers Chubby, which powers Google's entire infrastructure. Understanding consensus means understanding why your distributed systems are reliable.

---

### Plain English Explanation

Consensus is the problem of getting multiple nodes to agree on a single value, even if some nodes fail or messages are delayed.

The classic example: three servers must agree on who is the leader. If they disagree (split-brain), two servers might simultaneously believe they are leader, make conflicting decisions, and corrupt data.

Consensus algorithms solve this. The most widely understood is **Raft**.

---

### Internal Mechanics

#### The Consensus Problem

Requirements for a correct consensus algorithm:
- **Agreement**: All non-faulty nodes agree on the same value
- **Validity**: The agreed value was proposed by some node (not invented)
- **Termination**: All non-faulty nodes eventually decide
- **Fault tolerance**: Can tolerate up to f node failures in a cluster of 2f+1 nodes

A cluster of 3 nodes tolerates 1 failure. 5 nodes tolerate 2 failures. This is the quorum majority requirement.

#### Raft

Raft was designed to be understandable, unlike Paxos. It decomposes consensus into three sub-problems:

**1. Leader Election**

All nodes start as followers. They wait for heartbeats from a leader. If no heartbeat arrives within an election timeout (randomized: 150–300ms), a follower becomes a candidate and requests votes.

```text
Initial state:
  Node A: Follower
  Node B: Follower
  Node C: Follower

Election timeout fires on Node A:
  Node A: Candidate (increments term to 1)
  Node A → Node B: "Vote for me, term 1"
  Node A → Node C: "Vote for me, term 1"

Node B, C: "I haven't voted this term, and A's log is as up-to-date as mine. Vote granted."

Node A: 2 votes (majority in 3-node cluster) → becomes Leader for term 1

Node A → Node B, C: Heartbeat (empty AppendEntries)
```

Key guarantees:
- A node only votes once per term
- A node only votes for a candidate whose log is as up-to-date as its own
- Randomized timeouts prevent simultaneous candidates (split votes)

**2. Log Replication**

The leader receives all writes. It appends the entry to its log, then replicates to followers.

```text
Client: write(x=5)
Leader: [Append entry to local log] → Log: [..., (term=1, index=5, x=5)]
Leader → Follower 1: AppendEntries(entry: x=5)
Leader → Follower 2: AppendEntries(entry: x=5)

Follower 1: appends to log, responds OK
Follower 2: appends to log, responds OK

Leader: majority responded (2 of 2 followers) → mark entry COMMITTED
Leader → Client: 200 OK (write durable)
Leader → Followers: "entry at index 5 is committed, apply it"
```

An entry is committed once the leader receives acknowledgment from a majority of the cluster. Even if the leader dies after this point, any new leader will have this entry (because it's in the majority's logs).

**3. Safety**

Raft's key safety property: *a leader always has all committed entries*.

Why? A candidate can only be elected if its log is at least as up-to-date as a majority of the cluster. Committed entries are in a majority of logs. Therefore, any elected leader has all committed entries. No committed entry is ever lost.

```text
5-node cluster, entries through index 10 are committed (on all nodes):

Entry 11 is only on Node A (leader) when A crashes.
  → Entry 11 was NOT committed (not yet on majority)
  → New leader may not have entry 11
  → Entry 11 is lost

Entry 10 is committed (on a majority):
  → New leader must have it (was elected because its log is up-to-date)
  → Entry 10 is safe
```

#### Paxos

Paxos is mathematically equivalent to Raft but significantly harder to understand. It comes in many variants (Single-Decree Paxos, Multi-Paxos, Fast Paxos).

The core insight is the same: propose values in rounds, accept if a majority agrees, and ensure that any future round respects previously accepted values.

Paxos is used in: Google's Chubby lock service, Apache Zookeeper (ZAB protocol, Paxos-inspired), Google Spanner's TrueTime-assisted Paxos.

#### Leader Election in Practice

etcd (used by Kubernetes) uses Raft for:
- Leader election among Kubernetes control plane nodes
- Storing cluster state (pod assignments, secrets, configmaps)
- Ensuring only one master acts on scheduling decisions

```text
Kubernetes Control Plane (3 nodes):
  Node A: etcd leader (term 5)
  Node B: etcd follower
  Node C: etcd follower

  → Node A crashes
  → Node B and C notice: no heartbeat for 300ms
  → Election timeout fires (randomized)
  → Node B becomes candidate, wins election (term 6)
  → Node B is new leader
  → Kubernetes scheduler on Node B resumes work
  
  Time for new leader: ~300-600ms (election timeout)
```

#### Split-Brain Prevention

Split-brain: two nodes simultaneously believe they are the leader.

Raft prevents this with **terms**:
- Every message carries a term number
- If a node receives a message with a higher term, it immediately reverts to follower
- A leader with a stale term number immediately steps down

```text
Network partition: A and B are separated from C, D, E.

A was leader (term 5).
C, D, E don't receive A's heartbeats.
C wins election: becomes leader (term 6).

A is still running, thinks it's leader (term 5).

Partition heals.
C sends message to A with term=6.
A: "Term 6 > my term 5 → I am no longer leader. Revert to follower."

No split-brain.
```

---

### Production Example: IAM Platform

**Leader election for primary PostgreSQL**:

Patroni uses a consensus store (etcd or Consul, which uses Raft internally) to elect the PostgreSQL primary. Only one node holds the "leader" key in etcd at a time. If the primary's etcd lease expires (it stopped renewing because it crashed), etcd removes the key after the TTL. Patroni on a replica detects the missing key and initiates a new election.

```text
PostgreSQL Primary (Node A):
  Every 10s: etcd.keepAlive("pg_leader", ttl=30s)

  Node A crashes.
  10s: etcd not renewed.
  30s: TTL expires, key deleted.
  31s: Patroni on Node B: "No leader key. Initiating election."
  Node B: writes "pg_leader" to etcd.
  Node B: promotes PostgreSQL to primary.
  Applications reconnect to new primary.

Total downtime: ~31 seconds (tunable via TTL)
```

This is consensus serving a critical real-world function: ensuring exactly one database node accepts writes.

---

### Common Failure Modes

**1. Election storm**: All nodes have similar election timeouts. Multiple candidates start simultaneously, split the vote, fail to elect a leader, try again. Use randomized timeouts (Raft's core insight) to prevent this.

**2. etcd quorum loss**: 3-node etcd cluster, 2 nodes die. etcd cannot reach quorum — it stops accepting writes. Kubernetes control plane becomes read-only. Pods stop being scheduled, services fail to heal. This is a real production scenario that every Kubernetes operator must plan for.

**3. Slow follower blocking progress**: A stale follower causes quorum to require it. If it's responding but very slowly, the leader's write latency inflates. Fix: ensure replicas have sufficient I/O; use 5-node clusters so 2 followers can be slow without blocking.

**4. Log divergence on leader change**: An uncommitted entry from the old leader exists on a follower. The new leader sends conflicting entries. The follower's log must be overwritten (Raft truncates conflicting uncommitted entries). This is correct behavior — uncommitted entries are not durable — but applications must be aware that their write acknowledgment was "lost."

---

### Tradeoffs

| Algorithm | Understandability | Performance | Variants in production |
|---|---|---|---|
| Raft | High | Good | etcd, CockroachDB, TiKV, Consul |
| Paxos | Low | Good | Chubby, ZooKeeper (ZAB), Spanner |
| ZAB (ZooKeeper) | Medium | Good | Apache ZooKeeper |

---

### Staff-Level Thinking

Engineers reach for etcd or ZooKeeper without thinking about what they're actually asking those systems to do: run distributed consensus. Consensus is not free — it requires a quorum write for every operation, which means at least two network round trips. For high-throughput operations, consensus is the bottleneck.

Use consensus for: leader election, distributed locks, cluster configuration, critical metadata that must be consistent.

Do not use consensus for: high-frequency application data, hot path user requests. Use a database optimized for your read/write pattern; use consensus only for the coordination layer above it.

---

*End of Part 5: Distributed Systems*
# Part 6: Reliability

> "The question is not whether your system will fail. It will. The question is whether it fails safely, recovers quickly, and learns."

---

## Chapter 48: Availability, SLA, SLO, and Error Budgets

### Interview Question

> "Your service has an SLA of 99.9% uptime. You've had three incidents this quarter. How do you determine if you're within budget? What changes do you make?"

---

### Why Should You Care?

Reliability is not intuitive. "99.9% uptime" sounds excellent until you realize it means 8.7 hours of downtime per year. Engineers who can't speak the language of SLAs, SLOs, and error budgets cannot have informed conversations with product, legal, or business teams about reliability investments. And they can't objectively measure whether their reliability improvements actually worked.

---

### Plain English Explanation

**SLA (Service Level Agreement)**: A contract between a service provider and a customer specifying the expected level of service. If the SLA is violated, there are consequences (credits, penalties).

**SLO (Service Level Objective)**: An internal target — the reliability goal your team commits to maintaining. Usually stricter than the SLA (you want to stay ahead of SLA violations).

**SLI (Service Level Indicator)**: The actual measured metric. What you measure to determine if your SLO is being met.

**Error Budget**: The amount of unreliability you're allowed to have within the SLO. If your SLO is 99.9% availability, your error budget is 0.1% of time — 43.8 minutes per month.

---

### Internal Mechanics

#### Calculating Availability

```text
Availability = Successful Requests / Total Requests × 100%

OR for time-based:
Availability = (Total Time - Downtime) / Total Time × 100%
```

Common SLO targets and their downtime allowances:

| SLO | Downtime / year | Downtime / month | Downtime / week |
|---|---|---|---|
| 90% ("two nines") | 36.5 days | 73 hours | 16.8 hours |
| 99% ("two nines") | 3.65 days | 7.3 hours | 1.68 hours |
| 99.9% ("three nines") | 8.76 hours | 43.8 minutes | 10.1 minutes |
| 99.95% | 4.38 hours | 21.9 minutes | 5.04 minutes |
| 99.99% ("four nines") | 52.6 minutes | 4.38 minutes | 1.01 minutes |
| 99.999% ("five nines") | 5.26 minutes | 26.3 seconds | 6.05 seconds |

The leap from 99.9% to 99.99% reduces error budget from 43.8 minutes/month to 4.38 minutes/month — a 10x reduction in allowed downtime, typically requiring 10x more investment in reliability.

#### Error Budget

```text
Monthly error budget for SLO of 99.9%:
  = 0.1% × 30 days × 24 hours × 60 minutes
  = 43.2 minutes

Incident log this month:
  - Jan 5: 12 minutes downtime
  - Jan 14: 8 minutes downtime  
  - Jan 22: 30 minutes downtime (!)
  
Total: 50 minutes → Over budget by 6.8 minutes

Error budget remaining: -6.8 minutes (budget exhausted)
```

#### The Error Budget as a Decision Tool

The error budget is not just a measurement — it is a **decision mechanism**.

Google's SRE model: when the error budget is exhausted, the development team must stop shipping new features and focus exclusively on reliability improvements until the budget is restored.

This creates a self-regulating system:
- Teams that are too conservative ship slowly but have lots of budget to spend on innovation
- Teams that ship too fast exhaust their budget and are forced to slow down and fix reliability
- The budget quantifies the tradeoff objectively

```text
Error Budget State Machine:

Budget > 50%: Ship freely, experiment, take risks
Budget 10-50%: Review changes carefully, increase test coverage
Budget < 10%: Freeze feature deployments, focus on reliability
Budget exhausted: No new features until budget restored
```

#### Defining Good SLIs

An SLI must measure what users actually experience. Poor SLI choices are a common failure mode.

**Bad SLI**: Server CPU utilization < 80%
Why it's bad: CPU can be at 20% while the service is completely broken (database down, all requests returning 500).

**Good SLI**: Proportion of requests that complete successfully (non-5xx) within 500ms
Why it's good: Directly measures what users experience.

Common SLI types:
- **Availability**: Proportion of requests returning non-error responses
- **Latency**: Proportion of requests completing within a threshold (p99 < 500ms)
- **Throughput**: Requests processed per second (when relevant)
- **Correctness**: Proportion of responses that are correct (for data systems)
- **Freshness**: How stale is the data (for data pipelines)

#### Multi-Tier SLOs

Large systems have internal SLOs that aggregate into external SLAs:

```text
External SLA: 99.9% availability

Internal SLOs (must all be met to achieve external SLA):
  - API Gateway: 99.99% availability
  - Auth Service: 99.99% availability  
  - Payment Service: 99.99% availability
  - Database: 99.99% availability

Why 99.99% internal for 99.9% external?
  A request touches all 4 components:
  P(success) = P(GW) × P(Auth) × P(Payment) × P(DB)
             = 0.9999 × 0.9999 × 0.9999 × 0.9999
             = 0.9996 (99.96%)
  
  Still well above 99.9%. Safety margin built in.
```

This is the **reliability arithmetic** of distributed systems: each component's unreliability multiplies. With 10 components at 99.9%, overall availability is 99.9%^10 = 99%.

---

### Production Example: IAM Platform

**SLOs for an IAM platform at a bank**:

| Operation | SLO | Rationale |
|---|---|---|
| Token introspection | 99.99%, p99 < 10ms | Called on every API request; failure = all services down |
| Login (authenticate) | 99.9%, p99 < 500ms | Users tolerate minor login delay |
| Permission check | 99.99%, p99 < 5ms | Inline on every authorization; must be fast |
| Admin UI | 99.5% | Not critical path; admins can retry |
| Audit log reads | 99%, p99 < 2s | Compliance queries, not real-time |

Error budget for token introspection (monthly):
- 99.99% = 0.01% downtime = 4.38 minutes/month
- That means any deployment that causes even 5 minutes of degraded token validation exhausts the monthly budget
- This forces careful canary deployments, feature flags, and rollback plans

---

### Common Failure Modes

**1. SLA-SLO gap too small**: Your SLO is 99.9% and your SLA is also 99.9%. When an incident takes you to 99.85%, you've already violated the SLA. Always set your SLO more strictly than your SLA.

**2. Wrong SLIs**: Measuring server health instead of user-facing success rate. Server says "healthy," users experience failures. SLO shows green, engineers are paged by users.

**3. Error budget as a compliance exercise**: Error budget exists on paper but the team doesn't act differently based on it. No mechanism to freeze features when budget is exhausted. Error budget is theater.

**4. SLO for wrong percentile**: SLO is "p50 latency < 100ms." p50 is the median. 50% of requests can exceed 100ms and the SLO is still met. Users at p99 (the worst 1%) experience 5 seconds. SLOs should target p99 or p99.9.

---

### Staff-Level Thinking

The error budget transforms reliability from a gut feeling into a data-driven conversation.

When a product manager asks "why can't we ship more features?", the answer becomes: "Our error budget is at 8%. One more incident this month and we're in violation of our SLA. We are not shipping new features until we've addressed the incident backlog."

When engineering asks "why should we spend a sprint on reliability?" The answer becomes: "We have 4 minutes of error budget left this month. At our current incident rate, we'll exhaust it in 6 days. Burning budget now means engineers won't be able to deploy at all in Q2."

Numbers make the conversation objective.

---

## Chapter 49: Fault Tolerance Patterns

### Interview Question

> "Your payment service calls three downstream services. Any of them can be slow or fail. How do you ensure your service remains responsive and doesn't cascade-fail?"

---

### Why Should You Care?

Cascading failures are how incidents propagate across distributed systems. A single slow service can cause timeout threads to accumulate across all callers, exhausting connection pools, causing latency spikes, eventually taking down the entire system — even though only one component was the root cause.

Fault tolerance patterns are the circuit breakers of distributed systems: they contain failures before they propagate.

---

### Plain English Explanation

When Service A calls Service B, and B is slow or failing, A needs to decide:
- How long to wait before giving up (timeout)
- Whether to retry, and how often (retry)
- Whether to stop calling B entirely if it keeps failing (circuit breaker)
- How to limit the blast radius of B's failure (bulkhead)
- What to return when B is unavailable (fallback)

These patterns work together. Most production services need all of them.

---

### Internal Mechanics

#### Timeout

The simplest pattern: don't wait forever.

```text
Without timeout:
  Thread calls Service B → B is slow → Thread waits → Thread waits → Thread waits
  100 requests → 100 threads blocked → Thread pool exhausted → Service A stops responding

With timeout:
  Thread calls Service B → B is slow → After 500ms, timeout
  Thread returns error → Error handled → Thread freed
```

**Timeout selection**: Too short → false failures (legitimate slow requests fail). Too long → cascading failure risk.

Guidelines:
- Set timeout based on the **p99.9 of normal response time** × 2
- For payment processing: 3-5 seconds (rare but legitimate slow cases exist)
- For read-only lookups: 200-500ms
- For health checks: 1-2 seconds

**Connection timeout vs read timeout**:
- Connection timeout: how long to wait to establish the TCP connection
- Read timeout: how long to wait for a response after connection is established

Both must be configured. A common bug: read timeout is set but connection timeout is not — if the downstream is unreachable, TCP SYN retries can take minutes before the OS gives up.

#### Retry

After a failure, try again — but intelligently.

```text
Naive retry (DANGEROUS):
  Request → failure → immediately retry → failure → immediately retry → ...
  × 1000 concurrent users = 3000 requests hammering a failing service
  The service is now under 3x load, making recovery slower.

Smart retry with exponential backoff + jitter:
  1st retry: wait 100ms + random(0-50ms)
  2nd retry: wait 200ms + random(0-100ms)
  3rd retry: wait 400ms + random(0-200ms)
  Max retries: 3-5
  
  Backoff reduces load during recovery.
  Jitter prevents thundering herd (1000 clients retrying at identical times).
```

**What to retry**:
- 503 Service Unavailable: safe to retry
- 429 Too Many Requests: safe to retry (after waiting)
- 500 Internal Server Error: depends — if the request was idempotent
- 408 Request Timeout: safe to retry
- 400 Bad Request: **never retry** — the request will always fail
- Payment debit: **only retry if the operation is idempotent** (use idempotency keys)

#### Circuit Breaker

Inspired by electrical circuit breakers. When a downstream service is failing, stop calling it temporarily and return a failure immediately — without even trying.

```text
States:
  CLOSED (normal): requests pass through
    → If failure rate > threshold (e.g., 50% in 10s window) → OPEN

  OPEN (tripped): requests immediately fail without calling downstream
    → After a timeout (e.g., 30s) → HALF-OPEN

  HALF-OPEN (probing): one request allowed through
    → Success → CLOSED
    → Failure → OPEN

┌────────┐                    ┌──────────┐
│ CLOSED │ -- failure rate → │   OPEN   │
│(normal)│    exceeds         │(fail fast)│
└────────┘    threshold       └────┬─────┘
     ▲                             │
     │    success                  │ timeout
     │                             ▼
     └────────────────────── HALF-OPEN ──┐
                              (test)      │ failure
                                         └────────────→ OPEN
```

**Why it helps**:
1. Fast failure: callers get an immediate error instead of waiting for timeout
2. Recovery time: the failing service has a quiet period to recover
3. Prevents cascading: threads aren't blocked waiting for a failing service

**Circuit breaker configuration**:
- Error threshold: what percentage of requests must fail to trip the breaker (e.g., 50%)
- Window size: over what time window is the failure rate calculated (e.g., last 10 seconds)
- Recovery timeout: how long to wait before trying HALF-OPEN (e.g., 30 seconds)
- Minimum requests: don't trip on a small sample (e.g., require at least 10 requests)

#### Bulkhead

Isolate resources for different callers/operations so that one failure can't exhaust shared resources.

Named after the watertight compartments in a ship's hull — if one compartment floods, the others remain sealed.

```text
Without bulkhead:
  Shared thread pool: 100 threads
  
  Service B (slow): consumes 80 threads (all waiting on B)
  Service C (healthy): wants threads → 20 remaining
  Service C performance degrades → users see slowness
  Eventually all 100 threads blocked on B → Service C fails too

With bulkhead (separate thread pools):
  Service B pool: 30 threads (capped)
  Service C pool: 30 threads (capped)
  Service D pool: 40 threads (capped)
  
  Service B goes slow → uses its 30 threads → bounded blast radius
  Service C unaffected → users of C still served normally
```

**Bulkhead implementations**:
- Thread pool per downstream dependency (Hystrix/Resilience4j)
- Connection pool per downstream
- Rate limiter per API consumer (tenant bulkheading)
- Kubernetes CPU/memory limits per pod

#### Fallback

When a downstream is unavailable, return something useful instead of an error.

```text
Payment fraud check service is down.
Options:
  1. Fail the payment entirely (user sees error)
  2. Allow the payment with manual review flag (safe fallback)
  3. Apply stricter local fraud rules from cache (degraded-but-functional fallback)
  4. Return "fraud check unavailable" and log for offline review

For a payment service: option 2 is usually right.
For a real-time trading platform: option 1 might be right (cannot risk fraud).
```

**Fallback types**:
- **Stale cache**: Serve the last known value
- **Default value**: Return a sensible default ("allow request with review")
- **Fail open**: Allow the operation to proceed (security risk, use carefully)
- **Fail closed**: Deny the operation (safe default for security-sensitive paths)
- **Alternative service**: Fall back to a backup system

---

### Production Example: IAM Platform

A login request chain:

```text
User Login Request
       │
       ▼
┌─────────────────────────────────────────────────┐
│  Auth Service                                   │
│                                                 │
│  1. Validate credentials    ──── Circuit Breaker─────→  DB
│     Timeout: 500ms                                       
│     Retry: 2x (idempotent, read-only)                    
│                                                 │
│  2. Check MFA required      ──── Circuit Breaker─────→  MFA Service
│     Timeout: 1s                                          
│     Fallback: require MFA (fail closed)                  
│                                                 │
│  3. Audit log the event     ──── Fire and forget─────→  Audit Service
│     Timeout: 200ms (async)                               
│     Fallback: write to local queue, retry later          
│                                                 │
│  4. Issue token             ──── Local (no network)     
│     Always succeeds if steps 1-3 pass                    
│                                                 │
└─────────────────────────────────────────────────┘
```

Key decisions:
- MFA service failure → fail closed (require MFA) not fail open. Security before usability.
- Audit service failure → async, non-blocking. A failed audit log does not fail a login.
- DB circuit breaker trips → login returns 503, users see "service unavailable." Acceptable.

---

### Common Failure Modes

**1. Retry amplification**: A service under load begins failing. All callers retry. The service now receives 3x its original traffic. It falls further. This is a retry storm. Fix: exponential backoff with jitter, retry budgets.

**2. Circuit breaker not configured**: Engineers add Resilience4j but leave the default configuration (or don't configure it at all). Circuit never opens. Threads pile up on the failing dependency. Fix: explicit configuration for every downstream.

**3. Fallback that's worse than the error**: Fallback returns stale, incorrect data that causes downstream logic errors. Sometimes "return an error" is a better user experience than "return wrong data."

**4. Cascading timeout from wrong setting**: Service A has a 30s timeout for Service B. Service B has a 25s timeout for Service C. A user request can hang for up to 30s. In that time, 30 threads per service are blocked. Chain reactions. Fix: timeouts must decrease as they propagate inward (outer service timeout > inner service timeout).

**5. Bulkhead not isolating correctly**: Thread pools are separate but share the same database connection pool. A slow downstream causes the service to hold connections longer, exhausting the shared pool. Bulkhead isolation must be end-to-end.

---

### Tradeoffs

| Pattern | Protects against | Cost | Risk |
|---|---|---|---|
| Timeout | Thread accumulation | False failures if too short | Setting correctly requires profiling |
| Retry | Transient failures | Amplification if not careful | Non-idempotent retries cause duplicate actions |
| Circuit Breaker | Cascading failure | Complexity | Trips incorrectly on brief spikes |
| Bulkhead | Resource starvation | Thread/memory overhead | Under-provisioned pools still saturate |
| Fallback | Service unavailability | Stale/incorrect data | User sees degraded experience |

---

### Staff-Level Thinking

Every downstream call should have a **conversation** about: "What happens to our service if this downstream is down? Slow? Returning garbage?"

This conversation should result in explicit answers, not implicit hope. The answers are implemented as timeouts, circuit breakers, and fallbacks. Then tested with chaos engineering (kill the downstream service in a staging environment and verify the system degrades gracefully, not catastrophically).

---

## Chapter 50: Disaster Recovery

### Interview Question

> "Your primary data center goes offline. Walk me through your disaster recovery plan — RTO, RPO, and what happens to in-flight transactions."

---

### Why Should You Care?

Disasters happen. Data center fires (OVHcloud 2021), region-wide AWS outages (us-east-1 2021), infrastructure misconfiguration. Engineers who haven't thought about disaster recovery discover their backup strategy was either untested, incomplete, or nonexistent — during the disaster.

---

### Plain English Explanation

**RPO (Recovery Point Objective)**: How much data can we afford to lose? Measured in time. "We can lose up to 1 hour of data" = RPO of 1 hour.

**RTO (Recovery Time Objective)**: How long can we be down? "We need to be back up within 4 hours" = RTO of 4 hours.

These are business decisions, not engineering decisions. Engineering builds what the business specifies. But engineering must be honest about what achieving a given RPO/RTO costs.

---

### Internal Mechanics

#### RPO-RTO Matrix

```text
RPO/RTO Spectrum:

     High Cost                              Low Cost
     High Complexity                        High Complexity
     |                                              |
     ▼                                              ▼
┌───────────┬───────────┬───────────┬───────────────┐
│ Active-   │ Warm      │  Pilot    │   Backup/     │
│ Active    │ Standby   │  Light    │   Restore     │
├───────────┼───────────┼───────────┼───────────────┤
│ RPO: 0    │ RPO: secs │ RPO: mins │ RPO: hours    │
│ RTO: 0    │ RTO: mins │ RTO: hrs  │ RTO: days     │
├───────────┼───────────┼───────────┼───────────────┤
│ Cost: $$$$│ Cost: $$$  │ Cost: $$  │ Cost: $       │
└───────────┴───────────┴───────────┴───────────────┘
```

**Active-Active**: Both data centers handle live traffic simultaneously. No failover needed — traffic is rerouted immediately if one DC fails. RPO=0, RTO=0 (or near-zero for rerouting). Cost: 2x infrastructure, extreme complexity in data consistency.

**Warm Standby**: Secondary site has running infrastructure and a recent replica of the data. On failure, promote the secondary: update DNS, promote the database. RTO: minutes. RPO: seconds to minutes (depends on replication lag).

**Pilot Light**: Secondary site has minimal infrastructure running (databases in sync, but servers not provisioned). On failure, provision servers from AMIs/containers, restore from backup. RTO: hours. RPO: minutes.

**Backup/Restore**: Periodic backups (hourly, daily) stored in durable storage (S3). On failure, provision new infrastructure from scratch and restore the backup. RTO: hours to days. RPO: time since last backup.

#### Backup Strategy

**3-2-1 Rule**:
- 3 copies of data
- 2 different media types (disk + tape, or SSD + S3)
- 1 offsite copy

For cloud environments:
- Primary: live database (Availability Zone 1)
- Replica: streaming replica (AZ 2)
- Backup: automated daily snapshots to S3 (different region)

**Backup types**:
- **Full backup**: Complete copy of all data. Slow to create, simple to restore.
- **Incremental backup**: Only changes since the last backup. Fast to create, complex to restore (must apply all incrementals in order).
- **Differential backup**: Changes since the last *full* backup. Medium create time, simple restore (last full + last differential).

**WAL archiving (PostgreSQL)**: Archive Write-Ahead Log segments to S3 continuously. Allows point-in-time recovery (PITR) — restore to any moment in time. RPO = 0 (no data loss since last WAL segment, typically seconds).

```text
PostgreSQL WAL Archiving:
  Primary DB → WAL segment → archive_command → S3 (every 16MB or 5 min)

  On disaster:
  New instance ← base backup (S3) + WAL segments
  Replay WAL segments up to target time (PITR)
  RPO: 5 minutes (last WAL segment)
```

#### Testing DR Plans

A disaster recovery plan that has never been tested is not a plan. It is a hypothesis.

**DR testing approaches**:
- **Tabletop exercise**: Walk through the runbook on paper. Find gaps in documentation.
- **Partial restore test**: Restore a backup to a test environment. Verify data integrity.
- **Failover drill**: Actually promote the secondary, reroute traffic. Verify RTO.
- **Game Day**: Inject real failures in production (Netflix Chaos Monkey, AWS GameDay) to test real-world behavior.

Minimum: quarterly backup restore tests. At least annual full failover drill.

---

### Production Example: IAM Platform

**Scenario**: Primary data center (Mumbai) experiences total outage. DR plan:

**Tier 1: Authentication service** (RTO: 5 minutes, RPO: 30 seconds)
- Active-Active deployment across Mumbai and Chennai
- User authentication load balances between DCs in normal operation
- If Mumbai goes down: DNS failover via Route 53 Health Checks (30s failover)
- Session data in Redis with cross-DC replication (async, ~1s lag)
- Impact: Users mid-session may need to re-authenticate (sessions on Mumbai Redis lost)

**Tier 2: IAM database** (RTO: 15 minutes, RPO: 5 minutes)
- Warm standby: Chennai replica with async replication (~5 min lag)
- On failover: DBA promotes Chennai replica to primary, updates connection strings
- Some writes from last 5 minutes may be lost (the async replication lag)
- For a bank: this requires a business decision — those 5 minutes of IAM changes (password updates, role changes) must be replayed or re-applied manually

**Tier 3: Audit logs** (RTO: 1 hour, RPO: 0)
- WAL archiving to S3 (Glacier for cost savings)
- Logs are append-only; can be replayed to any point
- RPO=0: no audit events are lost; they may be delayed in appearing but never discarded

---

### Common Failure Modes

**1. Untested backups**: The backup exists but no one has tested restoration. During the disaster, the backup turns out to be corrupted, incomplete, or requires software that no longer exists. Fix: quarterly restoration tests are non-negotiable.

**2. RTO reality mismatch**: The stated RTO is 1 hour but no one has actually run a failover drill. In the real incident, it takes 6 hours because the runbook is outdated, DNS TTLs are high, and manual steps were forgotten. Fix: test the full runbook, not just individual steps.

**3. Forgetting in-flight transactions**: Failover happens. New primary comes up. But 100 payment transactions were in-flight at the moment of failure. Some were committed on the old primary before it died. Some were mid-flight. Without idempotency keys and reconciliation, the system is in an inconsistent state.

**4. Data loss from async replication**: Team says "RPO is 5 minutes" based on replication lag. But under write load, the replica falls further behind. Actual lag at failure time: 20 minutes. The stated RPO was a best-case, not a worst-case. Fix: measure maximum replication lag under load; that is the actual RPO.

**5. Cross-region data gravity**: The DR site is in another region, but all traffic is routed to the primary region. During failover, users in the primary region now talk to the DR site cross-region (high latency). Fix: use anycast or latency-based DNS routing so users connect to the nearest healthy site.

---

### Staff-Level Thinking

RPO and RTO are not technical targets — they are business decisions with cost implications. A staff engineer's job is to present clear options:

> "Zero data loss (RPO=0) requires synchronous replication across regions. Writes will be 60ms slower. The alternative is 5-minute RPO with async replication and fast writes. Which does the business require?"

The business must answer, not engineering. Engineering then builds accordingly.

---

## Chapter 51: Idempotency

### Interview Question

> "A user submits a payment. The network times out. Their app retries. How do you prevent charging them twice?"

---

### Why Should You Care?

Idempotency is the most important concept in payment systems, order processing, and any system where operations have real-world consequences. Without it, retries — which are essential for reliability — become dangerous. With it, you can retry freely without fear of duplicate effects.

---

### Plain English Explanation

An operation is **idempotent** if performing it multiple times has the same effect as performing it once.

- DELETE /users/123 — idempotent. Delete it twice? User is still deleted.
- GET /users/123 — idempotent. Read it 100 times? Still just reading.
- POST /payments (debit $100) — **NOT idempotent**. Post it twice → charged $200.

The internet is unreliable. Requests time out. Clients retry. Load balancers retry. This is fundamental. Any operation that must be retried safely must be idempotent.

---

### Internal Mechanics

#### Idempotency Keys

The standard pattern: the client generates a unique key for each logical operation. The server uses this key to deduplicate.

```text
Client                                  Server
  │                                        │
  │  POST /payments                        │
  │  Idempotency-Key: uuid-1234            │
  │  Body: { amount: 100, to: "acct-5" }  │
  │──────────────────────────────────────→│
  │                               Process payment
  │                               Store result with key uuid-1234
  │  200 OK { payment_id: pay-789 }        │
  │←──────────────────────────────────────│
  │                                        │
  │  [timeout — did that work?]            │
  │                                        │
  │  POST /payments (RETRY)                │
  │  Idempotency-Key: uuid-1234 (SAME KEY) │
  │──────────────────────────────────────→│
  │                     Key uuid-1234 found in cache
  │                     Return cached response (no second debit)
  │  200 OK { payment_id: pay-789 }        │
  │←──────────────────────────────────────│
```

The client **must generate the key** (a UUID before sending the request), not the server. If the server generates the key, a timeout before the response means the client doesn't know the key and can't deduplicate.

**Server-side implementation**:

```text
┌────────────────────────────────────────────────────────┐
│  Request handler: POST /payments                       │
│                                                        │
│  1. Extract Idempotency-Key from header                │
│  2. Check idempotency store:                           │
│     - MISS: proceed with payment processing            │
│     - HIT with COMPLETE status: return cached response │
│     - HIT with IN_PROGRESS status: wait or return 409  │
│  3. Mark key as IN_PROGRESS in idempotency store       │
│  4. Execute payment                                    │
│  5. Mark key as COMPLETE, store response               │
│  6. Return response                                    │
└────────────────────────────────────────────────────────┘

Idempotency Store Schema:
  idempotency_key (PK): uuid-1234
  status: IN_PROGRESS | COMPLETE | FAILED
  response_body: { payment_id: pay-789, ... }
  created_at: 2024-01-15T10:00:00Z
  expires_at: 2024-01-16T10:00:00Z  (24hr TTL)
```

**The race condition**: Two retries arrive simultaneously, both miss the idempotency cache.

```text
Retry 1: miss → start processing payment
Retry 2: miss → start processing payment  (DUPLICATE!)

Both are deducting from the same account.
```

Fix: use a distributed lock or database-level uniqueness constraint on the idempotency key. Only one can proceed; the other waits and then returns the cached result.

```sql
-- PostgreSQL: atomic check-and-insert
INSERT INTO idempotency_keys (key, status, created_at)
VALUES ('uuid-1234', 'IN_PROGRESS', NOW())
ON CONFLICT (key) DO NOTHING
RETURNING key;

-- If returns nothing: key already exists → return cached response
-- If returns the key: first request, proceed with payment
```

#### Natural Idempotency

Some operations are naturally idempotent without special handling:

- **PUT** (not POST): PUT /users/123 sets the user to exactly this state. Sending the same PUT twice is a no-op.
- **DELETE**: Deleting a resource twice leaves it deleted.
- **Upsert**: `INSERT ... ON CONFLICT UPDATE` — first run inserts, subsequent runs update to the same value.
- **Set operations**: Setting a value is idempotent. Incrementing a counter is not.

Design APIs to use PUT for updates instead of POST + PATCH where possible. The client sets the final state; the server applies it.

#### Idempotency in Message Queues

Consumers must handle duplicate message delivery. At-least-once delivery (Kafka, SQS) means every message consumer must be idempotent.

```text
Message: { event_type: "payment_processed", payment_id: "pay-789", amount: 100 }

Consumer (account service): apply credit of $100 for payment pay-789

Without idempotency:
  Message delivered twice → $200 credited

With idempotency:
  1st delivery: credit $100, record "pay-789 processed"
  2nd delivery: check "pay-789 processed" → already done, skip
```

Implementation: maintain a processed-events table or a deduplication window in Redis.

```sql
-- Mark payment as processed (idempotent insert)
INSERT INTO processed_payments (payment_id, processed_at)
VALUES ('pay-789', NOW())
ON CONFLICT (payment_id) DO NOTHING;

-- If conflict: payment already processed → skip
-- If inserted: first time → apply credit
```

---

### Production Example: Payment System

A payment initiation flow for an IAM-controlled payment API:

```text
Mobile App                    API Gateway                 Payment Service           DB
    │                              │                            │                    │
    │  POST /payments              │                            │                    │
    │  Idempotency-Key: abc-123    │                            │                    │
    │─────────────────────────────→│                            │                    │
    │                              │  auth check (token valid)  │                    │
    │                              │  forward request           │                    │
    │                              │───────────────────────────→│                    │
    │                              │                            │ check key abc-123   │
    │                              │                            │────────────────────→│
    │                              │                            │ MISS               │
    │                              │                            │←────────────────────│
    │                              │                            │ lock key abc-123    │
    │                              │                            │ process debit       │
    │                              │                            │ store result        │
    │                              │                            │────────────────────→│
    │                              │                            │ 200 OK              │
    │  200 { payment_id: p-456 }   │ 200 { payment_id: p-456 } │                    │
    │←─────────────────────────────│←──────────────────────────│                    │
    │                              │                            │                    │
    │  [network hiccup, retry]     │                            │                    │
    │  POST /payments              │                            │                    │
    │  Idempotency-Key: abc-123    │                            │                    │
    │─────────────────────────────→│                            │                    │
    │                              │───────────────────────────→│                    │
    │                              │                            │ check key abc-123   │
    │                              │                            │────────────────────→│
    │                              │                            │ HIT: p-456         │
    │                              │                            │←────────────────────│
    │  200 { payment_id: p-456 }   │ 200 { payment_id: p-456 } │ return cached      │
    │←─────────────────────────────│←──────────────────────────│ (NO DOUBLE CHARGE) │
```

---

### Common Failure Modes

**1. Server-generated idempotency keys**: Server returns the key as part of the response. Client times out before receiving the response. Client doesn't know the key. Client generates a new request with a new key. Double charge. Fix: always client-generated keys.

**2. Non-atomic key check**: Check key → miss → process → insert key (three separate steps, not atomic). Two simultaneous requests both miss. Fix: atomic insert-and-check in the database with a unique constraint.

**3. Short idempotency window**: Key expires after 5 minutes. A legitimate retry at 6 minutes (user navigates away and comes back) generates a new request, a new key is required, but the first payment already processed. Now the user is charged again. Stripe uses 24-hour idempotency windows. Design for the real retry time distribution.

**4. Idempotency without lock**: Key exists, status is IN_PROGRESS. Second request arrives, sees IN_PROGRESS, proceeds anyway. Duplicate processing. Fix: status must be respected; IN_PROGRESS should return 409 Conflict or block and wait.

**5. Not propagating idempotency keys downstream**: Payment service is idempotent, but when it calls the bank's core banking API, it generates a new UUID each time. Retry to payment service → new call to core banking → double debit at the bank level. Fix: derive child idempotency keys deterministically from the parent key: `child_key = hash(parent_idempotency_key + operation_name)`.

---

### Tradeoffs

| Approach | Safety | Complexity | Storage cost |
|---|---|---|---|
| Idempotency key (DB) | High | Medium | Low |
| Idempotency key (Redis) | High (with TTL) | Low | Low |
| Natural idempotency (PUT/upsert) | High | Low | None |
| Exactly-once semantics (Kafka) | High | Very high | Medium |
| No idempotency (hope) | None | None | None |

---

### Staff-Level Thinking

Idempotency is not just a payment concern. It applies to:
- Email sending (user should not receive 10 welcome emails because of a retry storm)
- Webhook delivery (recipient should process the event once even if retried)
- Infrastructure provisioning (apply a Terraform plan twice, get the same infrastructure)
- Distributed transactions (each step of a Saga must be idempotent for rollback/retry)

A staff engineer designs idempotency in from the start. Retrofitting idempotency into a running payment system is a multi-month, high-risk project.

The acid test for every write operation: "If this is executed exactly twice, what happens?" If the answer is "something bad," idempotency is required.

---

*End of Part 6: Reliability*
# Part 7: Architecture Patterns

> "Architecture is the set of decisions that are hard to change. Everything else is implementation."

---

## Chapter 52: Monolith vs Microservices

### Interview Question

> "When would you choose a monolith over microservices? What are the hidden costs of microservices that teams underestimate?"

---

### Why Should You Care?

Microservices became the default answer to "how should I architect my system?" somewhere around 2014–2017. Many teams adopted them not because they solved a real problem, but because they were fashionable. The result: enormous operational complexity, distributed system failure modes, and slower delivery — for systems that would have been better served by a well-structured monolith.

A staff engineer knows when to use microservices and when to resist them.

---

### Plain English Explanation

A **monolith** is a single deployable unit containing all of the application's functionality. All the code runs in one process.

**Microservices** decompose the application into independently deployable services, each responsible for a bounded domain.

Neither is universally better. Each has a domain where it excels.

---

### Internal Mechanics

#### Monolith Characteristics

```text
┌──────────────────────────────────────────────┐
│              Monolithic Application          │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  User    │  │ Payment  │  │  Auth    │  │
│  │  Module  │  │  Module  │  │  Module  │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│                                              │
│  In-process calls (nanoseconds)              │
│  Shared database                             │
│  Single deployment unit                      │
└──────────────────────────────────────────────┘
                      │
              ┌───────┴───────┐
              │  Database     │
              └───────────────┘
```

**Advantages**:
- **Simple to develop**: No network boundaries, no service discovery, no distributed tracing setup
- **Fast in-process calls**: Function calls, not HTTP round trips
- **Easy to test**: One test suite, one deployment, one environment
- **Simple transactions**: ACID across the whole system, no distributed transactions
- **Easier to debug**: One log file (or aggregated), one stack trace
- **Operational simplicity**: One process to deploy, monitor, and scale

**Disadvantages**:
- **Deployment coupling**: Changing one module requires deploying the whole application
- **Scaling coupling**: If one module is CPU-heavy, you must scale the entire app
- **Technology lock-in**: All modules must use the same language/framework
- **Long build times**: As the codebase grows, build and test cycles slow
- **Team scaling challenges**: Many engineers working on one codebase → merge conflicts, coordination overhead

**The modular monolith**: A monolith with strong internal module boundaries — clear APIs between modules, no direct database table sharing, enforced interfaces. Gets most of the operational simplicity of a monolith while maintaining the code organization benefits of microservices. Often the right intermediate architecture.

#### Microservices Characteristics

```text
┌──────────┐    ┌──────────┐    ┌──────────┐
│   User   │    │ Payment  │    │  Auth    │
│ Service  │    │ Service  │    │ Service  │
│          │    │          │    │          │
│ DB: Postgres│ │ DB: Postgres│ │ DB: Postgres│
└────┬─────┘    └────┬─────┘    └─────┬────┘
     │               │                │
     └───────────────┴────────────────┘
              HTTP / gRPC / Events
```

**Advantages**:
- **Independent deployability**: Ship the payment service without touching auth
- **Independent scalability**: Scale payment service to 100 nodes, leave user service at 3
- **Technology diversity**: Payment service in Java, analytics in Python, ML in Python
- **Team autonomy**: Each service has an owning team; Conway's Law alignment
- **Fault isolation**: Payment service crash doesn't take down user service

**Disadvantages**:
- **Network latency**: Every cross-service call adds 1-50ms
- **Distributed transactions**: No shared ACID. Use Sagas or 2PC (both complex)
- **Operational overhead**: N services × (service discovery + health checks + circuit breakers + distributed tracing + per-service dashboards)
- **Testing complexity**: Integration tests across services require service mocks or running all services locally
- **Distributed debugging**: Stack trace spans multiple services; correlation IDs required
- **Chatty interfaces**: Poor service boundaries → many fine-grained calls between services (the microservices equivalent of N+1 queries)
- **Consistency challenges**: Eventual consistency between services is the norm

#### The Hidden Costs

Teams that migrate to microservices often underestimate:

**1. Service ownership**: Who owns this service? Who is paged at 3am? Who has the expertise? Microservices without clear ownership become orphaned services — running in production but unmaintained.

**2. Local development complexity**: Running 20 services locally to develop one feature. Docker Compose helps but adds overhead. Kubernetes locally (minikube, k3d) is complex.

**3. API versioning**: When Service A calls Service B, B must maintain backward compatibility. Breaking changes require coordinated deployment or API versioning. In a monolith: just change the function signature.

**4. Data consistency across services**: A refund in the payment service must update the order status in the order service. How? Synchronous call (coupling), event (eventual consistency), saga (complex). In a monolith: one database transaction.

**5. Distributed tracing infrastructure**: Without distributed tracing (Jaeger, Zipkin), debugging a request that touches 6 services from a user's complaint is hours of log-grepping.

#### When to Choose What

```text
Choose MONOLITH when:
  - Small team (<15-20 engineers)
  - Early-stage product (domain not yet stable)
  - Strong consistency requirements across domains
  - Low operational overhead capacity
  - Codebase is small and manageable
  - Rate of change is relatively uniform across the system

Choose MICROSERVICES when:
  - Team is large (multiple teams; each team owns a service)
  - Different components have radically different scaling needs
  - Different components need different technology choices
  - Deployment independence is more valuable than operational simplicity
  - System is mature; domain boundaries are well-understood and stable
  - Strong DevOps capability (CI/CD, service mesh, distributed tracing in place)

The Progression: most successful systems start as monoliths and decompose to microservices as they grow.
Start monolith → identify the seams where decomposition adds value → extract services iteratively.
Don't start microservices from scratch unless you have a very good reason.
```

---

### Production Example: IAM Platform

An IAM platform is a natural microservices candidate — eventually. But it has a critical constraint: **cross-domain consistency**.

A "revoke all sessions" operation must:
1. Invalidate all active sessions (session store)
2. Revoke all tokens (token store)
3. Write an audit event (audit store)

In a microservices architecture, this is a distributed transaction (Saga). Each step can fail. Compensating transactions are complex.

In a monolith: one database transaction. Atomic, consistent, simple.

The IAM platform at many banks is a modular monolith: all IAM logic in one deployable, but with clear module APIs (session module, token module, auth module, admin module). The only microservice extracted is the **token introspection service** — it is called on every API request (500k req/sec), must scale independently, and is read-only (no consistency concerns). That's the right extraction: high independence, clear boundary, scaling need.

---

### Staff-Level Thinking

The question is not "monolith or microservices?" The question is: **"What is the dominant pain in this system, and does the chosen architecture address that pain?"**

- Pain: deployment coupling, one team blocking another → microservices
- Pain: operational complexity, insufficient DevOps capacity → monolith
- Pain: scaling one hot feature → extract that feature, keep rest as monolith
- Pain: strong consistency requirements → monolith or carefully bounded microservices with Saga

Architecture serves the organization as much as the technical requirements. A microservices architecture for a 5-person team is an organizational mismatch, regardless of the technical merits.

---

## Chapter 53: Microservices Design Principles

### Interview Question

> "What makes a good microservice boundary? How do you avoid distributed monolith anti-patterns?"

---

### Why Should You Care?

Most failed microservices migrations don't fail because of technology — they fail because of poor service boundaries. The result is a "distributed monolith": all the operational complexity of microservices, with none of the independence benefits. Services are tightly coupled, must be deployed together, and share databases.

---

### Internal Mechanics

#### Domain-Driven Design (DDD) and Bounded Contexts

The correct way to find service boundaries is through **Bounded Contexts** from Domain-Driven Design.

A bounded context is a portion of the system where a specific domain model applies and is consistent. The same word may mean different things in different bounded contexts.

```text
"Account" in different contexts:

Banking:
  Account (noun): a financial record; has balance, transactions, owner
  Account service: manages balances, debits, credits

IAM:
  Account (noun): a user's identity record; has credentials, roles, status
  Auth service: manages login, sessions, tokens

These are different bounded contexts with different "Account" meanings.
A single Account service trying to serve both would be confused and conflated.
Good microservice = one bounded context.
```

#### The Distributed Monolith Anti-Pattern

A distributed monolith looks like microservices but behaves like a monolith:

```text
DISTRIBUTED MONOLITH (anti-pattern):
  Service A calls Service B calls Service C calls Service D
  All synchronously, all in the critical path.
  
  To deploy Service A, you must also deploy B, C, D in coordination.
  A failure in D brings down A.
  
  You have: the operational complexity of 4 deployable units
         + the tight coupling of a monolith
         + network latency at every step
         
  You don't have: independent deployability, fault isolation

REAL MICROSERVICES:
  Services can be deployed independently.
  A failure in one service degrades, not destroys, others.
  Each service is the authoritative owner of its data.
```

#### Service Boundary Heuristics

**1. Single Responsibility**: A service should do one thing and do it well. If you can't name what a service does in one short phrase, it's probably doing too much.

**2. Data Ownership**: Each service exclusively owns its data. No other service queries its database directly.

```text
WRONG:
  Order Service queries Payment DB directly:
    SELECT * FROM payments WHERE order_id = ?

RIGHT:
  Order Service calls Payment Service API:
    GET /payments?order_id=xxx
```

**3. Minimal coupling, maximal cohesion**: High coupling = services need each other to function. Low coupling = services operate independently. Cohesion = things that change together belong together.

**4. The 2-pizza rule (Amazon)**: A service team should be small enough to be fed by two pizzas. If a service requires a large team, it's too big.

**5. Align with Conway's Law**: Systems tend to mirror the communication structure of the organization that builds them. Design your microservices to match your team structure — not the other way around.

#### API Design for Microservices

**Synchronous (REST/gRPC)**: Use when you need an immediate response.
```text
Payment Service → Fraud Service: "Is this transaction fraudulent?"
Fraud Service → Payment Service: "No, proceed" (must answer before payment proceeds)
```

**Asynchronous (events)**: Use when you don't need an immediate response.
```text
Payment Service emits: PaymentProcessed { payment_id, amount, user_id }
Notification Service consumes: sends email receipt
Analytics Service consumes: updates dashboards
Neither requires an immediate response from Payment Service.
```

The key principle: **prefer async/event-driven for cross-service communication** where the producer doesn't need the consumer's response. This reduces coupling and improves fault isolation.

---

### Production Example: IAM Platform Decomposition

```text
Bounded Contexts in an IAM Platform:

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Identity BC          Credential BC        Session BC      │
│  ─────────────        ─────────────        ──────────       │
│  Who is the user?     How do they prove    Are they         │
│  User profiles        their identity?      currently        │
│  Account status       Passwords, MFA,      logged in?       │
│  Identity linking     FIDO2 keys           Active tokens    │
│                                                             │
│  Authorization BC     Audit BC             Admin BC         │
│  ─────────────────    ────────────         ────────         │
│  What can they do?    What happened?       Configuration    │
│  Roles, permissions   Compliance logs      Management       │
│  RBAC/ABAC            Forensics            UI + APIs        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

Each bounded context has clear ownership, its own data, and well-defined APIs to other contexts. The audit context is a pure consumer — it subscribes to events from all other contexts and never writes back.

---

### Staff-Level Thinking

The test of a good microservice boundary is: **"Can this service be deployed without deploying any other service?"** If the answer is consistently yes, the boundaries are right. If the answer is often "we also need to deploy service X," the services are too tightly coupled.

---

## Chapter 54: Event-Driven Architecture (Advanced)

*This chapter builds on Chapter 32 (Event-Driven Architecture) with advanced patterns.*

### Interview Question

> "How do you design an event-driven system that maintains consistency across multiple services, handles out-of-order events, and survives broker restarts?"

---

### Why Should You Care?

Event-driven architecture enables loosely coupled systems that scale well. But it introduces new failure modes: out-of-order events, duplicate events, event schema evolution, and consumer lag. Engineers who haven't thought through these failure modes build fragile event-driven systems that seem reliable until production.

---

### Internal Mechanics

#### Event Types

**Domain Events**: Something that happened in the domain.
```json
{ "type": "PaymentProcessed", "payment_id": "p-123", "amount": 100, "timestamp": "..." }
```

**Integration Events**: A domain event published to other services.
Domain events stay internal to a service; integration events cross service boundaries.

**Commands**: A request to do something (not an event — an event is something that happened).
```json
{ "type": "ProcessPayment", "amount": 100, ... }  // This is a command, not an event
```

#### The Outbox Pattern

The hardest problem in event-driven systems: how do you publish an event atomically with the database transaction that caused it?

```text
NAIVE (WRONG):
  1. INSERT payment record to DB
  2. Publish "PaymentProcessed" event to Kafka
  
  Failure between 1 and 2: payment saved, event never published.
  Consumers never know the payment happened.

OUTBOX PATTERN (CORRECT):
  1. BEGIN transaction
  2.   INSERT payment record to payments table
  3.   INSERT event to outbox table (same transaction)
  4. COMMIT transaction
  
  5. [Background] Outbox processor reads unpublished events
  6.              Publishes to Kafka
  7.              Marks as published
  
  If step 4 fails: nothing published (consistent)
  If step 6 fails: retry until published (at-least-once delivery)
  Consumer handles duplicates via idempotency.
```

```sql
-- Outbox table
CREATE TABLE outbox_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  aggregate_type VARCHAR(100) NOT NULL,  -- 'Payment'
  aggregate_id UUID NOT NULL,             -- payment_id
  event_type VARCHAR(100) NOT NULL,       -- 'PaymentProcessed'
  payload JSONB NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  published_at TIMESTAMPTZ,               -- null = not yet published
  published BOOLEAN DEFAULT FALSE
);

-- In the payment transaction:
INSERT INTO payments (id, amount, user_id, ...) VALUES (...);
INSERT INTO outbox_events (aggregate_type, aggregate_id, event_type, payload)
  VALUES ('Payment', :payment_id, 'PaymentProcessed', :event_json);
-- Both in the same transaction → atomic
```

The outbox processor can be implemented with:
- **Polling**: SELECT ... WHERE published = FALSE ORDER BY created_at, publish, UPDATE
- **CDC (Change Data Capture)**: Debezium watches PostgreSQL WAL, publishes changes to Kafka directly. No polling overhead.

#### Event Ordering and Idempotency

Events from a Kafka partition are ordered. Events across partitions are not.

```text
Partition 0: [UserCreated, PaymentAdded, AccountClosed]  (ordered)
Partition 1: [OrderCreated, OrderShipped]                (ordered)

But: no ordering guarantee between partition 0 and partition 1.
A consumer might see OrderShipped before UserCreated if they're in different partitions.
```

Solutions:
- Use the same partition key for events that must be ordered (e.g., `user_id` as Kafka partition key → all user events go to the same partition)
- Design consumers to be tolerant of out-of-order events (idempotent state machines)
- Use an inbox pattern (consumers persist events before processing, re-order locally)

#### Event Schema Evolution

Events are a public contract between services. Schema changes must be backward compatible.

```text
BREAKING changes (avoid):
  - Remove a field consumers depend on
  - Change a field's type (string → integer)
  - Rename a field

NON-BREAKING changes (safe):
  - Add a new optional field (consumers that don't know it ignore it)
  - Add a new event type
  - Deprecate a field (still present, but consumers should stop using it)
```

Use a schema registry (Confluent Schema Registry with Avro, or JSON Schema) to enforce compatibility rules at publish time. A schema registry prevents publishing a schema that breaks existing consumers.

#### Consumer Lag

```text
Producer throughput: 100,000 events/sec
Consumer throughput: 80,000 events/sec
Consumer lag growth: 20,000 events/sec

After 1 hour: 72,000,000 unprocessed events (lag = 72M)
At current rate: 1 hour to process 1 hour of events → perpetual lag
```

Consumer lag is the symptom. Root causes:
- Consumer is too slow (heavy processing per event)
- Consumer is blocked on a slow downstream
- Not enough consumer instances (scale out, add partitions)
- GC pauses causing consumer to miss poll interval (rebalance)

Monitor consumer lag as a primary SLO for event-driven systems. Alert when lag grows unboundedly.

---

### Production Example: IAM Platform

IAM events flow using the Outbox pattern:

```text
User Service (source of truth for identity)
  → writes to: users table + outbox_events table (same transaction)
  
Outbox Processor (Debezium or polling job)
  → reads: outbox_events WHERE published = FALSE
  → publishes: Kafka topic "iam.user.events"
  
Consumers:
  - Notification Service: consumes UserCreated → sends welcome email
  - Audit Service: consumes all IAM events → writes compliance log
  - Access Provisioning: consumes UserCreated, RoleAssigned → provisions access
  
All consumers are idempotent (deduplicate by event_id).
Partition key = user_id (all events for one user are ordered).
```

---

### Staff-Level Thinking

Event-driven architecture shifts consistency from synchronous to asynchronous. Before adopting it, answer: **"Is eventual consistency acceptable here?"** For welcome emails: yes. For account balance updates: requires very careful design with compensating events.

The outbox pattern is non-negotiable for any event-driven system where consistency matters. Teams that skip it discover dual-write failures in production.

---

## Chapter 55: CQRS

### Interview Question

> "What is CQRS? When does it make sense? What are the failure modes engineers don't talk about?"

---

### Why Should You Care?

CQRS (Command Query Responsibility Segregation) is one of those patterns that sounds complex but solves a real problem that every large-scale system hits: the data model that's optimal for writing data is rarely optimal for reading it. Understanding CQRS — and its costs — is essential for designing data-intensive systems.

---

### Plain English Explanation

CQRS separates the data model into two:
- **Command side**: handles writes (creates, updates, deletes). Optimized for consistency and correctness.
- **Query side**: handles reads. Optimized for the specific query patterns of the UI/API.

The write model normalizes data correctly (3NF). The read model pre-materializes the data in the shape the UI needs — denormalized, indexed, possibly in a different database.

---

### Internal Mechanics

#### The Problem CQRS Solves

```text
WITHOUT CQRS (shared model):
  UI needs: "Show me all transactions for user 123, with merchant name, 
            category, balance after each transaction, sorted by date"
  
  Query: 5-table join, aggregation, window functions
  Performance: 200ms (slow for a mobile app)
  
  Write model: normalized tables (users, transactions, merchants, categories, balances)
  Write performance: good
  
  Read performance: poor (complex joins on write-optimized schema)

WITH CQRS:
  Write side: normalized schema (fast writes, consistent)
  
  Read side: pre-materialized "user_transaction_feed" table
    { user_id, transaction_id, merchant_name, category, balance_after, date }
    (denormalized, exactly what the UI needs)
    
  Read query: SELECT * FROM user_transaction_feed WHERE user_id = 123 ORDER BY date
  Performance: <5ms (single table scan, indexed)
```

#### CQRS Architecture

```text
                         ┌─────────────────────┐
Client writes ──────────→│  Command Handler     │
  POST /transactions      │  (validates, writes) │
                         └──────────┬──────────┘
                                    │ writes
                                    ▼
                         ┌─────────────────────┐
                         │   Write Database     │
                         │   (normalized)       │
                         └──────────┬──────────┘
                                    │ events / CDC
                                    ▼
                         ┌─────────────────────┐
                         │   Event Processor    │
                         │   (updates read model)│
                         └──────────┬──────────┘
                                    │ writes
                                    ▼
                         ┌─────────────────────┐
                         │   Read Database      │
                         │   (denormalized,     │
                         │    optimized views)  │
                         └──────────┬──────────┘
                                    │ reads
                                    ▼
Client reads ───────────→ GET /users/123/transactions
```

#### Synchronization Between Write and Read Models

The event processor keeps the read model in sync with the write model:
- Change Data Capture (CDC) via Debezium: watch the write DB's WAL, publish to Kafka, consume and update the read DB
- Application events: the write service explicitly publishes events when state changes
- Polling: periodically query the write DB and update the read DB (simplest but adds latency)

**The lag**: The read model is eventually consistent with the write model. After a write, there is a window (milliseconds to seconds) where the read model reflects the old state.

For most UIs, this is acceptable. For "you just made a payment, now view your balance" flows, it is not — these require read-your-writes consistency.

#### CQRS and Event Sourcing

CQRS is often paired with Event Sourcing (Chapter 32 advanced). In this combination:
- Write model: append-only event log (every state change is an event)
- Read model: projections built from the event log

Event sourcing is a powerful audit trail but adds significant complexity. CQRS can be used without event sourcing (and often should be, to avoid unnecessary complexity).

---

### Production Example: IAM Audit Reports

An IAM platform's audit report queries ("show all admin actions in the last 30 days") are expensive on the write-optimized schema (the audit_events table has 500M rows, multiple joins required).

CQRS solution:
- **Write side**: normalized `audit_events` table (fast inserts)
- **Read side**: pre-materialized `admin_action_summaries` table in a columnar store (e.g., BigQuery, Redshift)
- **Event processor**: CDC from PostgreSQL → Kafka → BigQuery loader

Admin reports: query the read side (BigQuery). Sub-second response on 500M row dataset.
Audit event writes: fast inserts to PostgreSQL write side.

---

### Common Failure Modes

**1. CQRS everywhere**: Apply CQRS to every domain, even simple ones. Now you have double the database complexity, consistency lag everywhere, and your team struggles with the added overhead. CQRS is for specific pain points, not a default.

**2. Ignoring the consistency window**: The read model is 2 seconds behind. User makes a payment, immediately checks their balance. Sees old balance. Files a support ticket. Fix: read-your-writes guarantee for time-sensitive operations (read from write model if very recent).

**3. Read model drift**: The event processor has a bug. The read model diverges from the write model silently. Queries return wrong data. Fix: periodic reconciliation jobs that compare write and read models, alerting on discrepancies.

**4. CQRS without CQRS-ready team**: CQRS increases complexity. A team that doesn't deeply understand it will be slower with it than without it. Consider whether the team has the bandwidth to manage the added operational overhead.

---

### Staff-Level Thinking

CQRS is a solution to **query model impedance mismatch** — the gap between the shape of your data and the shape of your queries. Before applying CQRS, verify the mismatch is real and costly.

Start with a better query (proper indexes, materialized views in PostgreSQL). CQRS is a bigger gun, and bigger guns have bigger recoil.

---

## Chapter 56: Saga Pattern

### Interview Question

> "You have a microservices architecture. A user's checkout process touches the order service, inventory service, and payment service. How do you handle failures while maintaining consistency?"

---

### Why Should You Care?

In a microservices architecture, distributed transactions are unavoidable — and 2PC (Two-Phase Commit) is often impractical (blocking, slow, requires all services to support it). The Saga pattern is the practical alternative for long-running distributed transactions in microservices.

---

### Plain English Explanation

A Saga is a sequence of local transactions. Each step completes a local transaction and publishes an event (or message) that triggers the next step. If a step fails, compensating transactions undo the preceding steps.

```text
Normal flow:
  Step 1: Create order (Order Service)
  Step 2: Reserve inventory (Inventory Service)
  Step 3: Charge payment (Payment Service)
  Step 4: Confirm order (Order Service)
  ✓ All succeed → checkout complete

Failure at Step 3 (payment failed):
  Step 3 fails: "Insufficient funds"
  Compensate Step 2: Release inventory reservation
  Compensate Step 1: Cancel order
  User sees: "Payment failed"
```

---

### Internal Mechanics

#### Choreography-Based Saga

Each service publishes events; other services listen and react. No central coordinator.

```text
Order Service            Inventory Service       Payment Service
     │                        │                       │
     │  Create order           │                       │
     │  Publish OrderCreated   │                       │
     │─────────────────────────│──────────────────────→│
     │                         │ On OrderCreated:       │
     │                    Reserve inventory             │
     │                    Publish InventoryReserved     │
     │                         │──────────────────────→│
     │                                                  │
     │                                         On InventoryReserved:
     │                                         Charge payment
     │                                         On success: Publish PaymentProcessed
     │                                         On failure: Publish PaymentFailed
     │                         │←──────────────────────│
     │←────────────────────────│                       │
     │                         │                       │
     │ On InventoryReserved     │                       │
     │  + PaymentProcessed:    │                       │
     │  Confirm order          │                       │
```

Pros: No central coordinator, loosely coupled.
Cons: Hard to track the overall state of a saga, difficult to debug, business logic scattered across services.

#### Orchestration-Based Saga

A central saga orchestrator tells each service what to do and tracks the state.

```text
                    ┌─────────────────────┐
                    │   Saga Orchestrator  │
                    │   (state machine)    │
                    └──────────┬──────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
          ▼                    ▼                    ▼
   Order Service        Inventory Service     Payment Service
   
Orchestrator steps:
  1. Command: CreateOrder → Order Service
  2. On OrderCreated event: Command: ReserveInventory → Inventory Service
  3. On InventoryReserved event: Command: ChargePayment → Payment Service
  4. On PaymentProcessed event: Command: ConfirmOrder → Order Service
  5. Saga Complete
  
  On PaymentFailed:
  6. Command: ReleaseInventory → Inventory Service (compensating)
  7. Command: CancelOrder → Order Service (compensating)
  8. Saga Failed
```

Pros: Clear state machine, easy to debug (one place tracks state), easier to add steps.
Cons: Orchestrator is a central point of coordination (must be robust); risk of logic coupling in the orchestrator.

#### Compensating Transactions

Compensating transactions undo the effect of a successful step:

| Step | Action | Compensating Action |
|---|---|---|
| Create order | `INSERT INTO orders (status='PENDING')` | `UPDATE orders SET status='CANCELLED'` |
| Reserve inventory | `UPDATE inventory SET reserved=reserved+1` | `UPDATE inventory SET reserved=reserved-1` |
| Charge payment | `Debit account` | `Credit account (refund)` |

**Key property**: Compensating transactions must be idempotent. They may be executed multiple times on retry.

**What cannot be compensated**: Some actions cannot be undone. "Send an email" cannot be un-sent. "Print a label" cannot be un-printed. For these, design the saga to perform the irreversible action last — after all other steps have succeeded.

#### Saga State Machine

```text
PENDING_ORDER
     │ CreateOrder success
     ▼
PENDING_INVENTORY
     │ ReserveInventory success
     ▼
PENDING_PAYMENT
     │ ChargePayment success
     ▼
COMPLETED
     
Failure paths:
PENDING_INVENTORY → (CreateOrder failed) → CANCELLED
PENDING_PAYMENT → (ReserveInventory failed) → COMPENSATING → CANCELLED
PENDING_PAYMENT → (ChargePayment failed) → COMPENSATING → CANCELLED
COMPENSATING → (compensation failed) → MANUAL_INTERVENTION_REQUIRED
```

The saga's state must be durably persisted. If the orchestrator crashes, it must be able to resume the saga from its last known state.

---

### Production Example: IAM Platform

User onboarding saga in an IAM platform:

```text
New User Onboarding Saga:
  Step 1: Create identity record (Identity Service)
  Step 2: Set initial password hash (Credential Service)
  Step 3: Assign default roles (Authorization Service)
  Step 4: Create audit record (Audit Service)
  Step 5: Send welcome email (Notification Service)
  
Compensations:
  Step 1 fail: nothing to compensate
  Step 2 fail: delete identity record
  Step 3 fail: delete identity record, delete credential
  Step 4 fail: delete identity record, delete credential, remove roles
  Step 5 fail: no compensation (send duplicate welcome email is acceptable; email is not critical path)
  
Step 5 (email) is last because it's irreversible. Even if it fails, the account is fully created.
```

---

### Common Failure Modes

**1. Saga in an inconsistent intermediate state**: The orchestrator crashes between steps 2 and 3. The inventory is reserved but payment not yet charged. The orchestrator restarts, reads its persisted state, and resumes from step 3. This works only if the state is durably persisted before commanding each step.

**2. Compensation failure**: Compensating transaction fails. Now the system is in a stuck state (e.g., inventory is reserved but order is cancelled). Fix: implement retry with backoff for compensations, and alert for manual intervention when compensation repeatedly fails.

**3. Semantic deadlock**: Saga A reserves inventory for user X. Saga B reserves the same inventory for user Y. Saga B completes first. Saga A tries to compensate (release inventory) but Saga B already used it. Inventory count goes negative. Fix: careful design of compensation semantics; use database constraints to prevent impossible states.

**4. Missing idempotency in steps**: A saga step is retried and the service processes it twice (creates two orders). Fix: all saga step commands must be idempotent (idempotency key per saga step).

---

### Tradeoffs

| Approach | Consistency | Complexity | Locks held |
|---|---|---|---|
| 2PC | Strong (ACID) | Very high | Long-lived across services |
| Saga (choreography) | Eventual | Medium | Short (per local transaction) |
| Saga (orchestration) | Eventual | High | Short (per local transaction) |
| No coordination | None | Low | None |

The Saga trades ACID consistency for availability and independence. It is the right choice for most microservices distributed transactions. The cost is eventual consistency and compensation logic complexity.

---

### Staff-Level Thinking

A staff engineer designs the compensation path with the same care as the happy path. "What does rollback look like?" is a first-class design question. The answers reveal hidden coupling between services, irreversible operations, and temporal dependencies that the happy path obscures.

If the compensation logic is extraordinarily complex, the service boundaries may be wrong. A transaction that is painful to distribute is a hint that the data should live in the same service.

---

*End of Part 7: Architecture Patterns*
# Part 8: Performance Engineering

> "Premature optimization is the root of all evil — but so is naïve ignorance of where your bottleneck is."

---

## Chapter 57: Latency

### Interview Question

> "Your API p99 latency is 800ms. Your SLO is 500ms. Walk me through how you diagnose and fix this."

---

### Why Should You Care?

Latency is the dimension users experience most directly. A 2-second page load loses 50% of mobile users. A 100ms degradation in API latency causes observable drop in conversion rates. Understanding latency — where it comes from, how to measure it, and how to eliminate it — is one of the most high-value engineering skills.

---

### Plain English Explanation

Latency is the time from when a request is sent to when the response is received. It is experienced as "slowness."

The critical insight: **latency is a distribution, not a single number**. The average (p50) tells you about the typical request. The p99 tells you about the worst 1 in 100 requests. The p999 tells you about the worst 1 in 1000. For high-traffic systems, even p999 affects thousands of users per second.

---

### Internal Mechanics

#### Latency Percentiles

```text
Distribution of API response times:
  p50:  45ms  (half of requests are faster)
  p75:  89ms
  p90: 180ms
  p95: 350ms
  p99: 820ms  ← SLO target
  p999: 4200ms

Average (mean): 90ms  ← THIS IS MISLEADING

"Average latency is 90ms" sounds fine.
"p99 is 820ms" means 1% of users wait nearly a second — at 100k req/sec, that's 1000 users/sec experiencing 820ms.
```

**Why averages lie**: a few extremely slow requests (outliers) barely affect the average but are experienced by real users. Always measure and SLO on percentiles, not averages.

#### Sources of Latency

```text
Anatomy of a request's latency:

  Total = Network (client→server) 
        + Queueing (in load balancer, thread pool)
        + Processing (CPU work: auth, business logic, serialization)
        + Database queries (N queries × per-query latency)
        + External service calls (payment gateway, fraud service)
        + Network (server→client)

Typical breakdown for a web API:
  Network: 5-20ms (within same region)
  Queueing: 0-5ms (under normal load); can be huge under load
  CPU processing: 1-10ms
  DB queries: 10-200ms (dominant)
  External calls: 50-500ms (most variable)
```

The database is almost always the dominant latency source. External service calls are the most variable.

#### Diagnosing Latency

**Step 1: Identify where time is being spent.**

Add instrumentation (spans) at each layer:

```text
Total request: 800ms
  ├── Auth validation:      8ms
  ├── Database query 1:    15ms (SELECT user)
  ├── Business logic:       5ms
  ├── Database query 2:   620ms  ← HOTSPOT
  ├── Response serialization: 3ms
  └── Network:             12ms

Database query 2 accounts for 77% of latency. Investigate it.
```

Without distributed tracing, you're guessing.

**Step 2: Diagnose the slow query.**

```sql
EXPLAIN ANALYZE
SELECT p.*, u.name, m.merchant_name 
FROM payments p
JOIN users u ON p.user_id = u.id
JOIN merchants m ON p.merchant_id = m.id
WHERE p.created_at > NOW() - INTERVAL '30 days'
ORDER BY p.created_at DESC;

-- Output:
-- Seq Scan on payments (cost=0.00..45231.00 rows=1234567 width=200)
--   Filter: (created_at > (now() - '30 days'::interval))
--   Rows Removed by Filter: 10234567
--
-- Planning time: 2ms
-- Execution time: 620ms
```

Sequential scan on 10 million rows. Missing index on `payments.created_at`.

```sql
CREATE INDEX idx_payments_created_at ON payments(created_at DESC);
-- After index creation:
-- Index Scan using idx_payments_created_at on payments
-- Execution time: 4ms
```

800ms → 192ms. p99 target met.

**Step 3: N+1 queries.**

A common pattern that silently kills latency:

```java
// N+1 query problem:
List<Order> orders = orderRepo.findByUserId(userId);  // 1 query
for (Order order : orders) {
    Merchant merchant = merchantRepo.findById(order.merchantId);  // N queries
    // ...
}
// Result: 1 + N queries for N orders. If user has 50 orders: 51 DB round trips.
```

Fix: join the query (fetch everything at once) or batch the secondary queries.

```java
// Batch query: 2 total queries regardless of order count
List<Order> orders = orderRepo.findByUserId(userId);           // 1 query
List<UUID> merchantIds = orders.stream().map(o -> o.merchantId).toList();
Map<UUID, Merchant> merchants = merchantRepo.findAllById(merchantIds)
    .stream().collect(toMap(m -> m.id, identity()));           // 1 query
```

#### The Critical Path

The critical path is the longest sequential chain in a request. To reduce overall latency, reduce critical path latency — parallelizing off-critical-path work doesn't help.

```text
Sequential (slow):
  Auth check (50ms) → DB query A (100ms) → DB query B (150ms) → Response
  Total: 300ms

Optimized (parallel off-critical-path):
  Auth check (50ms) runs during which DB query A (100ms) is also running
  No — auth is sequential with everything else.
  
Better: can A and B run in parallel?
  Auth check (50ms) → [DB query A (100ms) ‖ DB query B (150ms)] → Response
  Total: 50 + 150 = 200ms  (B is critical path; A completes first, is not bottleneck)
```

Use CompletableFuture, async/await, or parallel streams to run independent operations concurrently.

#### JVM Latency Concerns (Java/Spring Boot)

For Java-based services (Spring Boot, Kotlin):

**GC pauses**: Stop-The-World GC pauses cause latency spikes at p99/p999. A 200ms GC pause → your p999 is at least 200ms.

```text
GC tuning for low latency:
  - Use G1GC (JDK 11+) or ZGC (JDK 15+) for low-pause GC
  - ZGC: sub-1ms pause times, suitable for latency-sensitive services
  - Monitor GC: -Xlog:gc*
  - Heap sizing: enough to avoid frequent GC, not so much that full GC is catastrophic
  
JVM flags for low latency:
  -XX:+UseZGC                    # ZGC (near-zero pause times)
  -Xmx4g -Xms4g                 # pre-allocate heap (avoid resize pauses)
  -XX:+AlwaysPreTouch            # pre-touch heap at startup (no page fault delays)
```

**Thread contention**: Synchronized blocks, database connection pool exhaustion, and lock contention cause threads to queue. p99 spikes during load.

**Connection pool sizing**:
```text
Hikari CP (Spring Boot default):
  Too small: threads wait for connections → queueing latency
  Too large: overwhelms DB → DB latency

Rule of thumb: pool_size = (core_count × 2) + effective_spindle_count
For a 4-core API server + PostgreSQL SSD:
  (4 × 2) + 1 = 9 connections

Monitor: hikari.pool.WaitTime (should be near 0)
```

---

### Production Example: IAM Token Introspection

Token introspection latency optimization:

```text
Before optimization:
  p50: 8ms
  p99: 450ms  ← SLO violation (target: 10ms)
  
Diagnosis:
  - Added distributed tracing (Micrometer + Jaeger)
  - Trace showed: 380ms spent in Redis lookup with intermittent timeouts
  
Root cause:
  - Redis client had no connection pool: each request opened a new connection
  - TCP connection setup: ~30ms
  - Occasional connection refused (max connections reached): timeout → retry → 380ms
  
Fix:
  - Configure Lettuce connection pool (spring.redis.lettuce.pool.max-active=20)
  - Add circuit breaker on Redis (fallback: verify locally if JWK cached)
  
After optimization:
  p50: 2ms
  p99: 12ms (SLO met)
  
Lesson: connection pooling for Redis is as critical as for PostgreSQL.
```

---

### Staff-Level Thinking

Latency SLO violations are prioritized by impact per request × request frequency. A 100ms regression on a 10,000 req/sec endpoint is more urgent than a 2-second regression on a 10 req/day admin report.

Build a latency budget for each critical path: "We have 500ms. Auth gets 50ms. DB gets 200ms. External gets 150ms. Processing gets 50ms. Serialization gets 20ms. Network gets 30ms." Every team member knows their slice of the budget.

---

## Chapter 58: Throughput

### Interview Question

> "What's the difference between latency and throughput? How do you increase throughput without degrading latency?"

---

### Why Should You Care?

Engineers often confuse latency and throughput, or assume that optimizing one improves the other. They don't always — and sometimes they trade off against each other. Understanding throughput is essential for capacity planning and high-volume system design.

---

### Plain English Explanation

**Latency**: How long does one request take? (Time per request)

**Throughput**: How many requests can the system process per second? (Requests per unit time)

```text
Analogy: highway traffic
  Latency = how long does it take to drive from A to B?
  Throughput = how many cars can pass through per hour?
  
  A narrow road with no traffic: low latency, low throughput
  A 10-lane highway at rush hour: higher throughput, but high latency (congestion)
```

They are related (Little's Law) but not the same.

---

### Internal Mechanics

#### Little's Law

```
L = λW

Where:
  L = average number of requests in the system (queue + processing)
  λ = throughput (requests per second)
  W = average latency (seconds per request)
```

Example: If your service has 100 concurrent requests in flight, and average latency is 100ms:
- λ = L / W = 100 / 0.1 = 1000 requests/sec throughput

If you double latency (200ms) with same concurrency: throughput halves (500 req/sec).

This is why latency and throughput are linked: to maintain throughput as latency increases, you must increase concurrency (add more threads/connections).

#### Throughput Bottlenecks

**CPU-bound**: Computational work dominates. More CPU cores → more throughput. Examples: encryption, hashing, serialization.

**I/O-bound**: Waiting for disk/network dominates. Async I/O or more concurrent threads helps. Examples: database queries, HTTP calls to external services.

**Memory-bound**: Memory bandwidth limits processing. Cache efficiency matters.

**Connection-bound**: The number of database or external connections limits concurrency. Connection pools set the ceiling.

```text
Throughput ceiling analysis:
  Scenario: API server, 10ms average DB query, 4 core machine
  
  Thread-per-request model (Spring MVC):
    - JVM: ~1000 threads max before overhead
    - Each thread: one in-flight DB query at a time
    - Throughput ceiling: 1000 threads / 0.010s = 100,000 req/sec (theoretical)
    - But: thread context switching overhead → practical limit ~20-30k req/sec
  
  Async reactive model (Spring WebFlux):
    - Small thread pool (= CPU cores = 4 threads)
    - Each thread handles many concurrent requests via non-blocking I/O
    - No blocking wait → threads never idle
    - Throughput ceiling: much higher, bounded by CPU and DB connection pool
```

#### Batching for Throughput

Batching trades latency for throughput: instead of processing one item at a time, process many together.

```text
Without batching (high latency per item, low throughput):
  For each item:
    INSERT INTO audit_events (event_id, ...) VALUES (...)
  100 rows = 100 INSERT round trips = 100 × 5ms = 500ms

With batching (lower latency per item, high throughput):
  INSERT INTO audit_events (event_id, ...) VALUES (x1), (x2), ..., (x100)
  100 rows = 1 INSERT round trip = 5ms

Throughput improvement: 100x
Individual insert latency: goes from 5ms to up to 50ms (wait for batch to fill)
```

Kafka producers use batching by default: events accumulate in a local buffer before being sent in bulk.

#### Horizontal Scaling for Throughput

The cleanest way to increase throughput: more stateless nodes behind a load balancer.

```text
1 node:  5,000 req/sec
2 nodes: 10,000 req/sec
10 nodes: 50,000 req/sec (assuming the DB can keep up)
```

This works until the database becomes the bottleneck. Then: read replicas, caching, sharding.

#### Async Processing for Throughput

Offload slow work from the critical path:

```text
Synchronous (slow):
  User action → validate → process → write to DB → write audit log → send email → respond
  Total: 500ms (email sending is 200ms)

Async (fast critical path):
  User action → validate → process → write to DB → publish event → respond (50ms)
                                                  ↓
                                         Background worker → write audit log + send email
  
  Critical path: 50ms. Audit and email happen asynchronously.
  Tradeoff: audit log is delayed (eventual consistency), email might be delayed.
  For most operations: this is acceptable.
```

---

### Production Example: IAM Audit Log Throughput

The IAM platform generates audit events at peak: 500,000 events/sec.

A naive synchronous write to PostgreSQL would saturate the DB immediately (PostgreSQL can handle ~10,000-50,000 writes/sec on good hardware, depending on row size and indexes).

Architecture for 500k/sec:
1. **API nodes**: Fire-and-forget publish audit event to Kafka (non-blocking, microseconds)
2. **Kafka**: Buffers the events (1M/sec capacity per partition)
3. **Consumer group (Kafka → PostgreSQL)**: Batch consumers write 1000 events per batch INSERT
4. **PostgreSQL**: Receives ~500 batch writes/sec instead of 500k individual writes/sec

Throughput: 500,000/sec handled. DB write rate: 500 batches/sec × 1000 rows = manageable.
Latency: Audit event appears in DB within 1-2 seconds of the action (acceptable for compliance use case).

---

### Staff-Level Thinking

Throughput and latency are not enemies — but they require different optimization strategies. Throughput: more parallelism, batching, async. Latency: fewer serial steps, faster I/O, better data structures.

The common trap: optimizing latency at the expense of throughput (or vice versa) without first understanding which constraint matters for the use case.

---

## Chapter 59: Capacity Planning

### Interview Question

> "You're launching a new payment feature. How do you determine how many servers you need to handle the expected load? What metrics do you monitor?"

---

### Why Should You Care?

Capacity planning is the bridge between business projections and infrastructure. Under-provision and your launch causes an incident. Over-provision and you waste significant cost. Engineers who can reason about capacity from first principles are rare and valuable.

---

### Internal Mechanics

#### The Capacity Planning Process

**Step 1: Understand the workload.**

```text
Questions to answer:
  - What is the expected peak QPS at launch?
  - What is the request mix? (reads vs writes, by endpoint)
  - What is the expected data volume? (storage growth per day/month)
  - What is the expected traffic pattern? (business hours? spikes? gradual ramp?)
  - Are there external dependencies with their own capacity limits?
```

**Step 2: Benchmark your service.**

Load test in a production-like environment:

```text
Tool: k6, JMeter, wrk, Gatling

Benchmark script: ramp from 10 req/sec to 10,000 req/sec over 10 minutes.
Monitor:
  - Throughput (req/sec)
  - Latency (p50, p95, p99)
  - CPU utilization
  - Memory utilization
  - DB connection wait time
  - Error rate

Find: at what QPS does latency start to degrade? That's your capacity ceiling per node.
```

```text
Example benchmark results for 1 API node (4 CPU, 8 GB RAM):
  0-2000 req/sec: p99 < 50ms, CPU < 30%  ← comfortable
  2000-4000 req/sec: p99 rising to 150ms, CPU 60-70%  ← moderate load
  4000-5000 req/sec: p99 > 500ms, CPU 90%, DB connections saturating  ← near limit
  5000+ req/sec: errors, connection timeouts  ← over limit

Practical capacity: 3000 req/sec per node (leave headroom below limit)
```

**Step 3: Calculate nodes needed.**

```text
Expected peak QPS: 12,000 req/sec
Capacity per node: 3,000 req/sec
Nodes needed for traffic: 12,000 / 3,000 = 4 nodes

Add redundancy (lose one node without degrading):
  N+1 redundancy: 4 + 1 = 5 nodes

Add headroom for unexpected spikes (e.g., 20% burst):
  12,000 × 1.20 = 14,400 req/sec → 14,400 / 3,000 = 5 nodes + 1 = 6 nodes

Final: 6 nodes
```

**Step 4: Database capacity.**

```text
Writes: 1,000 write req/sec → PostgreSQL primary must handle 1,000 writes/sec
  - Benchmark: PostgreSQL on r6g.4xlarge handles ~15,000-20,000 simple writes/sec
  - 1,000 writes/sec: comfortably within capacity

Reads: 11,000 read req/sec → Can't hit primary (concurrent reads would slow writes)
  - Use read replicas + cache
  - Cache hit rate: 80% (for idempotent lookups)
  - Cache handles: 8,800 req/sec
  - Read replicas handle: 2,200 req/sec each (need 2 replicas for safety)
  
Storage: 100 bytes per event × 1,000 events/sec × 86,400 sec/day
       = 8.64 GB/day
       = 259 GB/month
  - Use partitioning, archive after 90 days to S3
```

**Step 5: Monitor and adjust.**

Capacity planning is a prediction. After launch, monitor actuals:

```text
Key metrics to monitor post-launch:
  - CPU saturation: alert if > 70% sustained
  - Memory: alert if > 80%
  - DB connection pool utilization: alert if > 80% in use
  - Latency p99: alert if approaching SLO
  - Error rate: alert if > 0.1%
  - Queue depth (Kafka lag): alert if growing
  
Auto-scaling:
  - Scale out (add nodes) when CPU > 60% for 3 minutes
  - Scale in (remove nodes) when CPU < 20% for 15 minutes
  - Set minimum nodes = (peak_load / node_capacity) × 1.5 for safety
```

#### Storage Capacity Planning

```text
Transaction data:
  - 500 bytes/record × 100,000 transactions/day = 50 MB/day
  - Compressed (PostgreSQL TOAST): ~25 MB/day
  - 30-day hot storage: 750 MB
  - 1-year archive: 9 GB (trivial)

Audit logs:
  - 200 bytes/event × 500,000 events/day = 100 MB/day
  - 90-day retention: 9 GB  
  - 7-year retention (compliance): 255 GB  ← start thinking about archival
```

---

### Production Example: IAM Platform Launch

New IAM platform launch for a bank:
- Current load: 100,000 users, 500 authentications/sec peak
- Target load after migration: 5,000,000 users, 25,000 authentications/sec peak

Benchmark on candidate hardware (c7g.4xlarge, 16 vCPU, 32 GB):
- Comfortable throughput: 8,000 auth/sec
- SLO: p99 < 200ms

Nodes needed: 25,000 / 8,000 = 3.1 → 4 nodes + 2 for redundancy/headroom = **6 nodes**

Database:
- Authentication requires a DB read (credential lookup)
- 25,000 reads/sec → use connection pooling (PgBouncer) + 3 read replicas
- Write load (new sessions, audit): 1,000 writes/sec → 1 primary is fine

Token introspection (separate, much higher load):
- 500,000 introspections/sec → Redis cluster (not DB)
- Redis cluster: 3 primary shards × 2 replicas = 6 nodes
- Redis throughput: 1M ops/sec per node → 3M ops/sec total → 6x headroom

---

### Staff-Level Thinking

Capacity planning conversations are opportunities to ask the right questions of the business: "Is this traffic estimate based on real data, or is it a guess?" Planned load rarely matches real load at launch. Always build in surge capacity (2x), and verify that auto-scaling is tested before the launch.

The real failure mode in capacity planning is not under-provisioning — it's not having tested the auto-scaling triggers before a big launch.

---

## Chapter 60: Backpressure

### Interview Question

> "Your payment service is receiving traffic faster than your downstream payment processor can handle. How do you prevent your service from overwhelming it — or crashing itself?"

---

### Why Should You Care?

Backpressure is how healthy systems protect themselves from being overwhelmed. Without it, a traffic spike propagates all the way to your database or external services, causing cascading failures. With it, the system degrades gracefully: it slows down rather than crashes, preserving availability for what capacity it has.

---

### Plain English Explanation

Backpressure is the mechanism by which a downstream consumer signals to an upstream producer that it should slow down.

```text
Without backpressure:
  Producer: sends 10,000 events/sec
  Consumer: can process 1,000 events/sec
  
  After 10 seconds: 90,000 events queued
  After 100 seconds: 900,000 events queued → memory exhausted → crash

With backpressure:
  Consumer: "I can only handle 1,000/sec. Please slow down."
  Producer: adjusts to 1,000/sec
  
  Queue: stays small, system stable
```

---

### Internal Mechanics

#### Rate Limiting

The producer limits its own output rate. Types:

**Token bucket**: A bucket holds N tokens. Each request consumes one token. Tokens are added at rate R/sec. If the bucket is empty, requests are rejected or queued.

```text
Bucket capacity: 100 tokens
Refill rate: 50 tokens/sec

Second 1: 100 tokens, 100 requests arrive → all served, bucket empty
Second 2: 50 tokens refilled, 50 requests arrive → all served
Second 3: 50 tokens, 100 requests arrive → 50 served, 50 rejected (or queued)
```

**Leaky bucket**: Requests queue, and are processed at a fixed rate. Excess requests overflow (are rejected). Smooths bursty traffic.

**Fixed window counter**: Count requests in a time window. If count exceeds limit, reject.

**Sliding window**: More accurate rate limiting over a rolling time window.

#### Queue-Based Backpressure

Insert a queue between producer and consumer. The queue absorbs bursts:

```text
Producer → Queue → Consumer
           (buffer)

Producer can produce faster than consumer; queue absorbs the difference.
Queue has a maximum depth (bounded queue).
When queue is full: apply backpressure to producer (block or reject).
```

The queue must be bounded. An unbounded queue defers the problem: eventually memory is exhausted.

In Kafka: consumer lag is the queue depth. If lag grows unboundedly, add consumer instances or optimize the consumer.

#### Reactive Streams Backpressure

In reactive programming (Spring WebFlux, RxJava, Project Reactor):

```java
// Consumer requests N items at a time (backpressure via demand)
source.onBackpressureBuffer(1000)  // buffer 1000 items
      .limitRate(100)              // request 100 from upstream at a time
      .flatMap(event -> processEvent(event), 8)  // max 8 concurrent
      .subscribe();
```

The consumer controls how fast the producer sends events. This is pull-based backpressure.

#### Shed Load (Drop Requests)

When the system is overloaded, it is better to reject some requests fast than to accept all and fail all slowly:

```text
Normal:  request → processed → 200 OK (100ms)
Overload: request → reject → 503 Service Unavailable (5ms) ← better than 10s timeout

Load shedding criteria:
  - Request queue depth > threshold
  - CPU > 90%
  - Active requests > max_concurrency
  
Load shedding response:
  - 503 with Retry-After header
  - Clients should back off and retry
```

#### The Coordination Problem with External Services

When your service calls an external payment processor:

```text
Your Service → Payment Processor (rate limit: 1,000/sec)
  
You receive 5,000 req/sec
You send 5,000 req/sec to processor → processor returns 429 Too Many Requests

Without backpressure:
  Your threads pile up retrying → thread pool exhausted → your service crashes

With backpressure:
  1. Detect 429 from processor
  2. Circuit breaker: stop sending requests for 30s
  3. Queue requests locally (bounded, e.g., 10,000 max)
  4. After 30s: retry at a controlled rate
  5. If queue fills: reject new requests with 429 to *your* callers
  
Your service degrades gracefully rather than crashing.
```

---

### Production Example: IAM Token Issuance Under Load

During a flash sale at a bank, login rate spikes to 5x normal:

```text
Normal load: 1,000 logins/sec
Flash sale: 5,000 logins/sec

Token issuance calls HSM (Hardware Security Module) for signing.
HSM capacity: 2,000 signing operations/sec

Without backpressure:
  5,000 login requests → 5,000 HSM calls → 2,000 handled, 3,000 queued
  Queue grows → memory pressure → JVM GC thrash → all logins slow → timeout
  → 5,000 login failures per second

With backpressure:
  1. Rate limiter on login: admit 2,000/sec (HSM capacity)
  2. Remaining 3,000/sec: 429 with Retry-After: 2
  3. Mobile app retries after 2 seconds
  4. Load spreads over time instead of instant pile-up
  5. 2,000 logins/sec complete successfully; 3,000/sec are delayed but served

User experience: brief delay during flash sale. Not a crash.
```

---

### Staff-Level Thinking

Backpressure is how systems maintain **graceful degradation** under overload. A system without backpressure has a sharp cliff: below capacity, everything works; at capacity, everything crashes. A system with backpressure has a gentle slope: at capacity, some requests are rejected but the system remains stable and serves what it can.

Design every integration with an external service (or every queue consumer) with an explicit answer to: "What happens when they can't keep up with us?" The answer is a backpressure mechanism. The silence is a future incident.

---

*End of Part 8: Performance Engineering*
# Part 9: Security

> "Security is not a feature you add. It is a property that must be designed in from the beginning and verified continuously."

---

## Chapter 61: Authentication — Sessions, JWT, OAuth2, OIDC, SAML

### Interview Question

> "Walk me through the full login flow for a user accessing a banking app — from credential entry to receiving an API response — covering all authentication tokens involved."

---

### Why Should You Care?

Authentication is the foundation of every secure system. Engineers who understand it only at the "paste the JWT library" level introduce vulnerabilities in production. The difference between understanding OAuth2 flow and copy-pasting code is the difference between a secure system and a security incident.

---

### Plain English Explanation

Authentication answers: **Who are you?**

Different authentication mechanisms exist for different use cases:
- **Sessions**: Traditional server-side state. The server remembers who you are.
- **JWT**: Client-side state. The token carries claims; the server verifies the signature.
- **OAuth2**: Delegated authorization. "Allow this app to access your data."
- **OIDC**: Identity layer on top of OAuth2. "Who is this user?"
- **SAML**: Enterprise SSO. XML-based federation.

---

### Internal Mechanics

#### Session-Based Authentication

```text
Login:
  1. User: POST /login { username, password }
  2. Server: validates credentials against DB
  3. Server: generates session_id (cryptographically random, 128+ bits)
  4. Server: stores session { session_id → { user_id, roles, created_at } } in Redis/DB
  5. Server: Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict
  6. Browser: stores cookie, sends on every subsequent request

Subsequent request:
  1. Browser: GET /dashboard  (with Cookie: session=abc123)
  2. Server: reads session_id from cookie
  3. Server: looks up session in Redis (fast, ~1ms)
  4. Server: session found → user is authenticated; attach user context to request
  5. Server: serve the dashboard

Logout:
  1. Server: DELETE session from Redis
  2. Server: Clear-Cookie header
  3. Browser: cookie deleted
  → Token is invalidated immediately (server controls state)
```

**Advantages**: Immediate revocation (delete session, user is logged out instantly). Server controls state completely.

**Disadvantages**: Requires server-side session storage (Redis). Horizontal scaling requires shared session store. Not suitable for API-to-API authentication.

#### JWT (JSON Web Token)

```text
Structure: header.payload.signature (base64 encoded, dot separated)

Header:  { "alg": "RS256", "typ": "JWT" }
Payload: { "sub": "user-123", "roles": ["PAYMENT_USER"], "iat": 1700000000, "exp": 1700003600 }
Signature: RSA256(base64(header) + "." + base64(payload), private_key)

Verification (stateless):
  1. Server receives JWT
  2. Server verifies signature with public key (no DB lookup needed)
  3. Server checks exp claim (not expired)
  4. Server trusts claims in payload
```

**Stateless**: The server needs no storage — the token is self-contained and self-verifying.

**The revocation problem**: A JWT is valid until it expires. If a user's role changes or they're disabled, the old JWT still works until expiry. Solution:
- Short expiry (15 minutes) + refresh tokens
- Token revocation list in Redis (partially defeats the statelessness benefit)
- Use short-lived access tokens and revoke refresh tokens on logout

**JWT attacks**:
- **alg:none attack**: Attacker sends JWT with `"alg":"none"` and no signature. Some libraries accepted this. Fix: always explicitly specify the allowed algorithm; never allow "none."
- **Algorithm confusion attack**: RSA private key used to sign, public key used to verify. Attacker takes the public key (it's public), sets `"alg":"HS256"` (HMAC), signs with the public key. Fix: always validate that the algorithm matches expectations.
- **JWT secret brute force**: Weak HMAC secrets can be brute-forced offline. Fix: use RS256 (asymmetric) or a strong HS256 secret (256+ bits of entropy).

#### OAuth2

OAuth2 is a **delegation protocol**: "Allow this third-party app to access my data on my behalf."

```text
Authorization Code Flow (most secure, for web apps):

User → Browser → App (client)
                    │ redirect to Authorization Server
                    ↓
         Authorization Server (e.g., Keycloak, Okta, ForgeRock)
                    │ user logs in, grants consent
                    ↓
         Browser → App (callback with authorization_code)
                    │ exchange code for tokens (server-to-server)
                    ↓
         Authorization Server → App: {
           access_token: "...",      (short-lived, e.g., 15 min)
           refresh_token: "...",     (long-lived, e.g., 30 days)
           id_token: "...",          (OIDC: who is the user)
           expires_in: 900
         }
                    │ API call with access_token
                    ↓
                Resource Server (your API)
                    │ introspect/validate token
                    ↓
                    response to App
```

**Key OAuth2 concepts**:
- **Client**: The application requesting access
- **Resource Owner**: The user who owns the data
- **Authorization Server**: Issues tokens after authentication (Keycloak, Okta, ForgeRock)
- **Resource Server**: The API protecting resources with access tokens
- **Scopes**: Fine-grained permissions (`read:payments`, `write:profile`)
- **PKCE** (Proof Key for Code Exchange): Required for public clients (mobile, SPA) to prevent authorization code interception attacks

#### OIDC (OpenID Connect)

OIDC is an identity layer on top of OAuth2. OAuth2 says "access this resource." OIDC says "and here's who the user is."

```text
OAuth2 only:
  Client gets access_token → can call APIs → but who is the user?

OIDC adds:
  Client gets access_token + id_token (JWT)
  id_token contains: {
    "sub": "user-123",          // user identifier
    "name": "Lalchand",         // user's name
    "email": "l@bank.com",      // email
    "phone": "+91...",
    "aud": "my-banking-app",    // intended audience
    "iss": "https://idp.bank.com", // issuer
    "exp": 1700003600           // expiry
  }
```

Standard flows:
- **Authorization Code Flow**: Browser-based apps
- **Client Credentials Flow**: Machine-to-machine (no user involved); service A calls service B
- **Device Flow**: TV, IoT devices where you can't type a password

#### SAML (Security Assertion Markup Language)

Enterprise SSO protocol, XML-based. Used predominantly in corporate environments for employee SSO.

```text
SAML Flow:
  1. User accesses Service Provider (SP): https://salesforce.com
  2. SP redirects to Identity Provider (IdP): https://sso.bank.com
     (with SAML AuthnRequest, base64 encoded XML)
  3. User logs in at IdP
  4. IdP sends SAMLResponse (XML with user attributes, roles) to SP via browser POST
  5. SP validates SAMLResponse signature
  6. SP grants access based on assertions

Key concepts:
  - IdP (Identity Provider): knows who you are (e.g., Active Directory, ForgeRock)
  - SP (Service Provider): the app you're accessing (Salesforce, Workday)
  - Assertion: XML statement about the user (name, roles, groups)
  - Metadata exchange: SP and IdP exchange XML metadata before federation
```

**SAML vs OIDC**:
- SAML: XML, enterprise-focused, mature, complex tooling
- OIDC: JSON/JWT, modern web/mobile apps, simpler, developer-friendly
- Trend: new integrations use OIDC; legacy enterprise apps still use SAML

---

### Production Example: Banking App Full Auth Flow

```text
User opens banking app (OIDC flow with ForgeRock as IdP):

1. App launches → no valid access token → initiate login
2. App generates PKCE code_verifier (random), code_challenge = SHA256(code_verifier)
3. App → Browser → ForgeRock:
   GET /authorize?
     response_type=code&
     client_id=banking-mobile&
     redirect_uri=bankapp://callback&
     scope=openid profile payments:read&
     code_challenge=abc&code_challenge_method=S256

4. ForgeRock: shows login page
5. User: enters credentials, MFA (TOTP or push notification)
6. ForgeRock: validates, issues authorization_code
7. Browser → App callback: bankapp://callback?code=xyz

8. App → ForgeRock (POST /token):
   code=xyz&
   code_verifier=original_verifier&  (proves app sent the original challenge)
   client_id=banking-mobile&
   grant_type=authorization_code

9. ForgeRock → App:
   { "access_token": "...", "id_token": "...", "refresh_token": "...", "expires_in": 900 }

10. App: parse id_token (who am I?), store access_token + refresh_token securely
11. App → Banking API: GET /accounts  (Authorization: Bearer <access_token>)
12. API Gateway: validate JWT signature (ForgeRock public key), check exp, check scope
13. API: serve accounts

15 minutes later:
14. access_token expired → App → ForgeRock: POST /token with refresh_token
15. ForgeRock: new access_token issued
```

---

### Staff-Level Thinking

Authentication protocol selection:
- User-facing web/mobile: OIDC (Authorization Code + PKCE)
- Service-to-service: OAuth2 Client Credentials
- Enterprise SSO with legacy IdPs: SAML (you'll encounter this in every bank)
- Internal API auth: JWT (short-lived, signed by the platform's JWKS)

The hardest part of authentication is not the protocol — it's token lifecycle management: refresh token rotation, revocation on logout, JWK rotation, and handling clock skew in token expiry validation.

---

## Chapter 62: Authorization — RBAC, ABAC, ReBAC

### Interview Question

> "Explain RBAC vs ABAC vs ReBAC. How would you design a fine-grained authorization system for a payment platform?"

---

### Why Should You Care?

Authorization is the answer to "what are you allowed to do?" It is where most access control bugs live. Too permissive: security incidents. Too restrictive: users can't do their jobs. Getting authorization right — and proving it at scale — requires understanding the models deeply.

---

### Plain English Explanation

**RBAC (Role-Based Access Control)**: Permissions are assigned to roles. Users are assigned to roles.

**ABAC (Attribute-Based Access Control)**: Access decisions based on attributes of the user, resource, and environment.

**ReBAC (Relationship-Based Access Control)**: Access based on the relationship graph between users and resources.

---

### Internal Mechanics

#### RBAC

```text
Roles: PAYMENT_VIEWER, PAYMENT_APPROVER, ADMIN

Role → Permissions:
  PAYMENT_VIEWER:   [read:payments, read:transactions]
  PAYMENT_APPROVER: [read:payments, read:transactions, approve:payments]
  ADMIN:            [* (all permissions)]

User → Roles:
  user-123: PAYMENT_VIEWER
  user-456: PAYMENT_APPROVER

Decision: Can user-123 approve a payment?
  → user-123 has PAYMENT_VIEWER role
  → PAYMENT_VIEWER has no approve:payments permission
  → DENY
```

**Advantages**: Simple, easy to understand and audit, well-understood by compliance teams.

**Disadvantages**: Role explosion. As the system grows, you need hundreds of roles for fine-grained scenarios. "Salesperson who can approve payments under $100k from customers in their territory" requires either a dedicated role (leading to thousands of roles) or ABAC.

#### ABAC

```text
Policy (in Rego/OPA):
  allow if {
    input.user.role == "PAYMENT_APPROVER"
    input.payment.amount <= 100000
    input.user.territory == input.payment.customer.territory
    input.time.hour >= 9
    input.time.hour <= 17  -- business hours only
  }

Decision: Can user-456 (PAYMENT_APPROVER, territory=WEST) approve payment-789 
          ($50,000, customer in EAST)?
  user.role check: ✓
  amount check: ✓
  territory check: user.territory (WEST) ≠ payment.territory (EAST) → DENY
```

**Advantages**: Extremely expressive, can encode any business rule.
**Disadvantages**: Complex to implement correctly, hard to reason about ("why was I denied?"), performance (policy evaluation on every request).

**OPA (Open Policy Agent)**: A policy engine that evaluates ABAC/RBAC rules written in Rego. Can be deployed as a sidecar or central policy service. Returns `allow/deny` decisions in milliseconds.

#### ReBAC (Relationship-Based Access Control)

```text
Relationships (graph):
  user:alice owner-of document:budget-2024
  user:bob editor-of document:budget-2024
  group:finance member document:budget-2024  (group has access)
  user:carol member-of group:finance

Decision: Can carol view document:budget-2024?
  carol → member-of → finance → member → budget-2024
  There exists a path from carol to budget-2024 through the group relationship.
  → ALLOW (viewer)
```

ReBAC models (Zanzibar, OpenFGA, SpiceDB) are essentially graph databases for authorization. They handle:
- Hierarchical resources (can access parent → can access child)
- Group membership transitivity
- Resource sharing (share with user X who shares with user Y)

**Google Zanzibar**: Google's global authorization system. Checks like "does user X have role Y on object Z?" are answered in <10ms at millions of QPS. All of Google Drive's sharing model runs on Zanzibar.

**OpenFGA** (Okta's open source Zanzibar): Production-ready, used in many enterprise IAM platforms.

---

### Production Example: Payment Authorization Model

```text
IAM Platform authorization model for payments:

RBAC for coarse access:
  PAYMENT_USER:    can initiate payments
  PAYMENT_REVIEWER: can view all payments
  PAYMENT_APPROVER: can approve payments (with limits)
  FRAUD_ANALYST:   can view and flag suspicious payments
  ADMIN:           full access

ABAC policies for fine-grained rules (OPA):
  # High-value payment: requires two-person approval
  require_dual_approval {
    input.payment.amount > 1000000  # > $1M
  }
  
  # Approver cannot approve their own payment
  deny {
    input.user.id == input.payment.initiator_id
    input.action == "approve"
  }
  
  # Time-based: no payments after business hours except pre-scheduled
  deny {
    input.action == "initiate"
    input.time.hour > 17
    not input.payment.scheduled
  }

OPA deployed as sidecar to payment service:
  POST /v1/data/payments/allow
  Input: { user, payment, action, time, context }
  Response: { result: true/false }
  Latency: <2ms (local process, in-memory policy)
```

---

### Staff-Level Thinking

Most systems need layered authorization:
1. RBAC for coarse-grained access (can this user even see this section?)
2. ABAC for policy-level rules (amount limits, time restrictions, segregation of duties)
3. ReBAC for resource ownership and sharing

The anti-pattern: encoding authorization logic in application code (`if (user.role == "ADMIN") { ... }` scattered everywhere). This is unmaintainable and unauditable. Centralize authorization in a policy engine (OPA) and keep application code clean.

---

## Chapter 63: Secrets Management and Encryption

### Interview Question

> "How do you manage secrets (database passwords, API keys, signing keys) in a production environment? What happens when a secret is compromised?"

---

### Why Should You Care?

Secrets in code are one of the most common and preventable security vulnerabilities. A GitHub scan of public repositories finds live API keys and database passwords daily. In 2023, a single leaked AWS key caused a $100M+ incident. Secrets management is not optional.

---

### Plain English Explanation

A secret is any credential that grants access: passwords, API keys, TLS private keys, signing keys, database connection strings, OAuth2 client secrets.

Secrets must never be:
- Hardcoded in source code
- Committed to version control
- Stored in environment variables on developer machines unencrypted
- Logged

Secrets must be:
- Encrypted at rest
- Rotated regularly
- Audited for access
- Revocable

---

### Internal Mechanics

#### Secrets Storage Options

**HashiCorp Vault**:
- Centralized secrets store
- Dynamic secrets: Vault generates a short-lived DB password for each service instance
- Encryption-as-a-service: encrypt/decrypt data without exposing keys
- Audit log: every secret access is logged
- Policies: fine-grained access control (service X can only read `secret/payments/db-password`)

```text
Application startup:
  1. App → Vault: "I am payment-service, here is my Kubernetes service account token"
  2. Vault: validates token with Kubernetes API
  3. Vault: token is valid, payment-service is authorized to read secret/payments/*
  4. Vault → App: { "db_password": "dynamic-password-abc123", "expires_in": 3600 }
  5. App: connects to DB with dynamic password
  6. After 3600s: Vault invalidates the password, issues a new one
  
  If the app instance is compromised and the password leaked:
  It expires in <1 hour automatically (minimal blast radius)
```

**AWS Secrets Manager / GCP Secret Manager**:
- Cloud-native secrets stores
- Automatic rotation for supported databases
- IAM-based access control
- CloudTrail audit logs

**Kubernetes Secrets**:
- Base64 encoded, NOT encrypted by default
- Enable encryption at rest with etcd encryption
- Use External Secrets Operator to sync from Vault/AWS Secrets Manager into Kubernetes Secrets

#### Encryption

**Symmetric encryption (AES-256-GCM)**:
- One key for encryption and decryption
- Fast, suitable for bulk data
- Problem: key distribution (how does the recipient get the key securely?)

**Asymmetric encryption (RSA-2048, RSA-4096, ECDSA P-256)**:
- Public key encrypts, private key decrypts
- Public key can be shared freely
- Suitable for key exchange, signing
- Slower than symmetric; used to encrypt the symmetric key (hybrid encryption)

**Envelope encryption**:
- Generate a random Data Encryption Key (DEK) for each piece of data
- Encrypt the data with DEK (symmetric AES-256)
- Encrypt the DEK with a master Key Encryption Key (KEK) — stored in KMS (AWS KMS, GCP KMS, HashiCorp Vault)
- Store encrypted DEK alongside encrypted data

```text
Envelope encryption flow:
  Encrypt:
    DEK = generate_random_key()
    ciphertext = AES_GCM_encrypt(plaintext, DEK)
    encrypted_DEK = KMS.encrypt(DEK, master_key_id)
    store(ciphertext, encrypted_DEK)
  
  Decrypt:
    DEK = KMS.decrypt(encrypted_DEK)  // only if caller has KMS permission
    plaintext = AES_GCM_decrypt(ciphertext, DEK)
```

Benefits: Rotating the master key only requires re-encrypting DEKs, not all data. Compromised DEK affects only one record.

#### Key Rotation

All cryptographic keys should be rotated on a schedule:
- JWT signing keys: rotate every 90 days (JWKS endpoint serves multiple valid keys during rotation)
- Database encryption keys: annually
- TLS certificates: before expiry (automate with cert-manager / Let's Encrypt)
- API keys: rotate immediately upon suspected compromise; otherwise every 90 days

**Zero-downtime key rotation for JWTs**:
```text
Current key: kid=key-2024-01, used for signing new tokens
Old key: kid=key-2023-10, still in JWKS for verifying old tokens
              
JWKS endpoint returns both keys:
  { keys: [key-2024-01, key-2023-10] }
  
JWT verification: try all keys in JWKS until one validates the signature
  → Old tokens (signed with key-2023-10) still valid
  → New tokens (signed with key-2024-01) valid
  → After all old tokens expire (15-60 minutes): remove key-2023-10 from JWKS
```

#### Secret Compromise Response

When a secret is compromised:

```text
Immediate actions (within minutes):
  1. Revoke the compromised credential immediately
     - DB: ALTER USER payment_svc_user PASSWORD 'new-password-here'
     - API key: revoke in provider dashboard
     - JWT signing key: remove from JWKS (if long-lived tokens exist: trigger forced re-auth)
  
  2. Rotate to a new credential
     - Generate new credential
     - Update all consumers via Vault/Secrets Manager
     - Verify applications are using new credential
  
  3. Audit
     - Who had access to this secret? (Vault audit log)
     - Was the compromised secret used maliciously? (access logs, timeframe of exposure)
     - What data was accessible with this credential?

Short-term actions (within days):
  4. Root cause: how was the secret exposed? (git commit, log file, environment variable leak)
  5. Remediate the root cause
  6. Review all secrets in the same category for similar exposure
  
Longer-term:
  7. Improve secret scanning in CI/CD (git-secrets, truffleHog, Semgrep)
  8. Reduce secret lifetime (dynamic secrets, shorter TTLs)
```

---

### Production Example: IAM Platform Secrets

```text
IAM Platform secrets inventory:

Secret                          | Storage          | Rotation
─────────────────────────────── | ──────────────── | ─────────────
JWT signing keys (RSA-2048)     | HSM              | 90 days
PostgreSQL password             | Vault (dynamic)  | 1 hour (dynamic)
Redis auth password             | Vault            | 30 days
OPA policy signing key          | Vault            | 90 days
ForgeRock admin API key         | AWS Secrets Mgr  | 90 days
TLS certificates                | cert-manager     | Auto (before expiry)
OAuth2 client secrets           | Vault            | 90 days

Kubernetes pods access secrets via:
  - External Secrets Operator → Vault → Kubernetes Secret
  - Secret is projected as env var or mounted file
  - Pod never has Vault credentials; uses Kubernetes service account (IRSA/workload identity)
```

---

### Staff-Level Thinking

The single most impactful secrets hygiene practice: **never put secrets in source code or environment variables**. Use a secrets manager from day one. It costs an afternoon to set up; the alternative can cost months of incident response.

The second most impactful: **dynamic, short-lived secrets**. A password that expires in 1 hour is vastly more secure than a password that never expires, even if both are stored equally securely.

---

*End of Part 9: Security*
# Part 10: Observability

> "You cannot improve what you cannot observe. You cannot debug what you cannot trace."

---

## Chapter 64: Logging

### Interview Question

> "A user reports a failed payment that happened 2 hours ago. You have no alert for it. How do you investigate? What should your logging system have captured?"

---

### Why Should You Care?

Logs are your time machine. They are the primary investigation tool for incidents that have already happened. Engineers who don't think carefully about logging strategy discover this during their first serious incident, when they desperately need a specific piece of information and it's not there.

---

### Plain English Explanation

A log is a timestamped record of something that happened in your system. Logs are written to answer future questions you haven't thought of yet — which means the strategy is to capture *context*, not just events.

---

### Internal Mechanics

#### Structured Logging

Unstructured logs (the traditional approach) are human-readable but machine-unqueryable:

```
2024-01-15 10:00:01 INFO  User login successful: user123
2024-01-15 10:00:02 ERROR Payment failed for user123: insufficient funds
```

Structured logs are machine-parseable JSON, enabling queries:

```json
{"timestamp":"2024-01-15T10:00:01Z","level":"INFO","service":"auth","event":"login_success","user_id":"user123","ip":"192.168.1.1","duration_ms":45}
{"timestamp":"2024-01-15T10:00:02Z","level":"ERROR","service":"payment","event":"payment_failed","user_id":"user123","payment_id":"pay-456","reason":"insufficient_funds","amount":100}
```

With structured logs: `SELECT * WHERE user_id='user123' AND level='ERROR'` in seconds.

With unstructured logs: grep, regex, prayer.

#### Log Levels

| Level | Use case | Example |
|---|---|---|
| DEBUG | Detailed diagnostic info (dev/staging) | "Entering method processPayment with args..." |
| INFO | Normal operational events | "Payment processed", "User logged in" |
| WARN | Unexpected but non-critical | "Retry attempt 2 of 3", "Rate limit approaching" |
| ERROR | Something failed; requires investigation | "Payment failed", "DB connection timeout" |
| FATAL/CRITICAL | System cannot continue | "DB unreachable, shutting down" |

Production: INFO and above. DEBUG disabled (too verbose, expensive).

Configurable at runtime without restart — invaluable for investigating incidents without a deployment.

#### What to Log

**Always log**:
- Request received (with request ID, user ID, IP, endpoint)
- Request completed (with status code, duration, payment_id for payment events)
- External calls made (to which service, with what parameters, response status, duration)
- Errors (with full stack trace, context that led to the error)
- Security events (login success/failure, permission denied, token issued/revoked)
- State transitions (payment_initiated → payment_processing → payment_complete)

**Never log**:
- Passwords, PINs, CVV numbers
- Full credit card numbers (PCI-DSS: log only last 4 digits)
- JWT tokens or session IDs (they grant access)
- AADHAAR/SSN or full PAN (Indian regulations)
- Encryption keys
- Full request bodies if they might contain secrets

#### Correlation IDs / Request IDs

The single most important logging practice for distributed systems:

```text
Client → API Gateway:
  Gateway generates: X-Request-ID: req-7f3a9b2c

Gateway → Auth Service (passes X-Request-ID header)
Auth Service → logs: { "request_id": "req-7f3a9b2c", "event": "token_validated" }

Gateway → Payment Service (passes X-Request-ID header)
Payment Service → logs: { "request_id": "req-7f3a9b2c", "event": "payment_initiated" }

Payment Service → DB:
  DB logs: { "request_id": "req-7f3a9b2c", "query_duration_ms": 45 }

QUERY: SELECT * WHERE request_id = 'req-7f3a9b2c'
→ Returns ALL log lines across ALL services for this single user request
→ Complete picture of the request lifecycle in seconds
```

Without correlation IDs: impossible to reconstruct a single request's journey across services.

#### Log Aggregation Architecture

```text
Service A → [stdout/file] → Fluentd/Filebeat → Elasticsearch/OpenSearch
Service B → [stdout/file] → Fluentd/Filebeat →   (Kibana for UI queries)
Service C → [stdout/file] → Fluentd/Filebeat →
                                    ↑
                             structured JSON
                             indexed by:
                               - timestamp
                               - service
                               - request_id
                               - user_id
                               - level
```

Common stacks:
- **ELK**: Elasticsearch + Logstash + Kibana
- **OpenSearch** (AWS): open source Elasticsearch fork
- **Grafana Loki**: log aggregation optimized for Kubernetes, pairs with Grafana
- **Splunk**: enterprise log management (expensive, powerful)
- **CloudWatch Logs**: if on AWS and don't want to manage the stack

#### Log Retention Policy

```text
Hot storage (fast query):   7-30 days in Elasticsearch
Warm storage (slower query): 30-90 days in compressed S3 (queryable via Athena)
Cold storage (archive):      1-7 years in S3 Glacier

For PCI-DSS: 1 year minimum, 3 months immediately accessible
For RBI: 5+ year retention for certain event types
```

---

### Production Example: Payment Failure Investigation

User complaint: "I tried to pay ₹5,000 to merchant X at 10:15 AM but it failed."

Investigation with structured logs:

```sql
-- Step 1: Find all requests from this user around that time
SELECT * FROM logs 
WHERE user_id = 'user-123' 
  AND timestamp BETWEEN '10:10' AND '10:20'
  AND event IN ('payment_initiated', 'payment_failed')

-- Returns:
{ "request_id": "req-abc123", "event": "payment_initiated", "amount": 5000, 
  "merchant_id": "merch-456", "timestamp": "10:14:32" }
{ "request_id": "req-abc123", "event": "payment_failed", 
  "reason": "fraud_check_declined", "timestamp": "10:14:33" }

-- Step 2: Get full request trace
SELECT * FROM logs WHERE request_id = 'req-abc123' ORDER BY timestamp

-- Returns complete call chain:
10:14:32.001 | api-gateway    | request_received       | user-123
10:14:32.005 | auth-service   | token_validated        | user-123
10:14:32.010 | payment-service| payment_initiated      | pay-789
10:14:32.012 | fraud-service  | fraud_check_started    | pay-789
10:14:32.105 | fraud-service  | fraud_check_declined   | reason=velocity_limit_exceeded
10:14:32.110 | payment-service| payment_failed         | fraud_check_declined
10:14:32.115 | api-gateway    | response_sent          | 402 Payment Required

-- Finding: fraud check declined due to velocity limit (too many payments in 24h)
-- Resolution: customer can call to temporarily lift limit, or it resets automatically at midnight
```

Total investigation time: 2 minutes. Without structured logs and correlation IDs: 2 hours minimum.

---

### Staff-Level Thinking

Logs are a contract. When you write a log line, you're committing to its presence for the retention period. Adding a new log field is easy; removing one breaks queries that dashboards and alert rules depend on.

Before shipping a feature, ask: "What questions will I need to answer when this breaks?" Write those log lines. The investment pays off during your first production incident.

---

## Chapter 65: Metrics and the Four Golden Signals

### Interview Question

> "What metrics would you set up for a payment service? How do you decide which ones to alert on?"

---

### Why Should You Care?

Logs tell you what happened. Metrics tell you what's happening right now and whether the trend is alarming. The Four Golden Signals (from Google SRE) are the minimum viable metric set for any service. Engineers who monitor meaningfully are paged appropriately; engineers who don't are paged too late or constantly.

---

### Plain English Explanation

Metrics are aggregated numerical measurements: rates, gauges, histograms. Unlike logs (individual events), metrics summarize behavior over time and enable trend analysis, anomaly detection, and SLO tracking.

Google's SRE book introduced the Four Golden Signals as the minimum set any service must instrument: **Latency, Traffic, Errors, Saturation**.

---

### Internal Mechanics

#### The Four Golden Signals

**1. Latency — How long do requests take?**

```text
Metric: http_request_duration_seconds (histogram)
Labels: service, endpoint, status_code

Alerts:
  - p99 latency > SLO threshold for 5 minutes → WARNING
  - p99 latency > 2× SLO threshold for 2 minutes → CRITICAL
  
Dashboard: p50/p95/p99 latency per endpoint, compared to SLO target
```

Critical insight: distinguish latency of successful requests vs error responses. A spike in errors (which are fast) can make your average latency look good even when your service is broken.

**2. Traffic — How much demand is hitting the system?**

```text
Metric: http_requests_total (counter) → rate(http_requests_total[5m])
Labels: service, endpoint, method

Use:
  - Understand the current load (is this traffic normal for this time of day?)
  - Detect traffic spikes (flash sale, DDoS)
  - Correlate with latency/error changes (is the latency spike because traffic increased?)
  
Alerts:
  - Traffic > 3× p99 historical for this hour → potential DDoS or viral event
  - Traffic drops to 0 → deployment failure, DNS issue
```

**3. Errors — What fraction of requests are failing?**

```text
Metric: rate(http_requests_total{status_code=~"5.."}[5m]) / rate(http_requests_total[5m])
= error rate as a fraction

Types of errors:
  - Explicit errors: 4xx (client), 5xx (server)
  - Implicit errors: 200 OK but response contains error payload
                     (always return explicit HTTP status codes!)

Alerts:
  - error_rate > 0.01 (1%) for 5 minutes → WARNING
  - error_rate > 0.05 (5%) for 2 minutes → CRITICAL (page on-call)
```

**4. Saturation — How full is the system?**

```text
Saturation = how close is the service to capacity?

Key saturation metrics:
  - CPU utilization: > 80% sustained → near limit
  - Memory: > 85% → risk of OOM
  - DB connection pool: active/max > 80% → connections becoming a bottleneck
  - Thread pool: queue depth > 0 → requests waiting for threads
  - Queue depth (Kafka lag): growing → consumer can't keep up
  - Disk: > 85% full → risk of write failures

Alert on approach to saturation, not at saturation.
If you alert when CPU hits 100%, you're already degraded.
Alert at 80% to give time to react.
```

#### RED Method (for Services)

Complementary to the Four Golden Signals, focused on services:
- **R**ate: requests per second
- **E**rrors: failed requests per second
- **D**uration: time for each request (latency distribution)

#### USE Method (for Resources)

For infrastructure resources (CPU, memory, disk, network):
- **U**tilization: % of time resource is busy
- **S**aturation: amount of work waiting (queue depth)
- **E**rrors: error events (disk read errors, TCP retransmits)

#### Metric Types

**Counter**: Only goes up. Total requests, total errors. Use `rate()` to get per-second rate.
```
http_requests_total{endpoint="/payments", status="200"} = 1234567
rate(http_requests_total[5m]) = requests/sec over last 5 minutes
```

**Gauge**: Can go up or down. Current value. CPU utilization, memory, active connections.
```
db_connections_active = 47
jvm_memory_used_bytes = 3758096384
```

**Histogram**: Distribution of values. Latency, request sizes.
```
http_request_duration_seconds_bucket{le="0.1"} = 9800  (9800 requests under 100ms)
http_request_duration_seconds_bucket{le="0.5"} = 9990
http_request_duration_seconds_bucket{le="+Inf"} = 10000
→ p99 ≈ 0.5s  (10000 × 0.99 = 9900 requests were under 0.5s, close to 9990)
```

#### Alerting Philosophy

**Alert on symptoms, not causes.**

Bad alert: "CPU > 90%"
Better alert: "p99 latency > 500ms" (the symptom the user experiences)
Even better: "error_rate > 1% for 5 minutes"

CPU at 90% may or may not affect users. High latency definitely does.

**Alert on what matters to users (SLO-based alerting)**:
```text
SLO: 99.9% of requests complete successfully within 500ms

Alert: 
  IF error_rate × 100 > 0.1% (burning through error budget at 1x rate)
  AND duration > 30 minutes
  → page on-call (budget will be exhausted in a reasonable timeframe)

  IF error_rate × 100 > 1% (burning through budget at 10x rate)
  AND duration > 5 minutes
  → page on-call immediately (budget will be exhausted quickly)
```

This is multi-burn-rate alerting — Google's SRE Workbook approach to SLO-based alerting.

---

### Production Example: IAM Platform Metrics Dashboard

```text
Grafana dashboard for IAM platform:

Panel 1: Traffic
  rate(http_requests_total[1m]) by (endpoint)
  → Shows: login: 950/s, token_introspect: 48000/s, admin: 12/s

Panel 2: Error Rate
  rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])
  → SLO line at 0.1%, current: 0.002% ✓

Panel 3: Latency (p99)
  histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
  by endpoint
  → token_introspect p99: 8ms ✓ (SLO: 10ms)
  → login p99: 180ms ✓ (SLO: 500ms)

Panel 4: Saturation
  db_pool_connections_active / db_pool_connections_max  → 0.43 (43%)
  jvm_memory_used_bytes / jvm_memory_max_bytes          → 0.61 (61%)
  redis_connections_active                              → 18/50 (36%)

Panel 5: Error Budget Burn Rate
  error_budget_remaining_percent: 84% (with 8 days left in month)
  burn_rate: 0.3× (burning slowly, will end month at ~78%)
```

---

### Staff-Level Thinking

Metrics are only valuable if they drive action. For every metric you expose, answer: "What alert would cause a human to respond to this? What would they do?"

A dashboard with 200 panels and no alerts is decoration. A dashboard with 5 panels and 3 alert rules that reliably page on the right issues is operational excellence.

---

## Chapter 66: Distributed Tracing

### Interview Question

> "A request is taking 2 seconds. It goes through 6 microservices. How do you find which service is responsible for the slowdown?"

---

### Why Should You Care?

In a distributed system, a slow request doesn't have a single stack trace — it has a tree of spans across multiple services. Without distributed tracing, debugging cross-service latency is like detective work with missing evidence. With tracing, it's a 30-second investigation.

---

### Plain English Explanation

A trace represents the journey of one request through your entire system. It is composed of **spans** — individual units of work — each with a start time, end time, and metadata.

Traces answer: "Which component in this multi-service request caused the slowdown?"

---

### Internal Mechanics

#### Trace Structure

```text
Trace: req-abc123 (total: 1950ms)
  │
  ├─ [0ms-15ms]     API Gateway: incoming request
  │
  ├─ [15ms-45ms]    Auth Service: validate token
  │                   ├─ Redis: JWK lookup [2ms]
  │                   └─ JWT signature verification [5ms]
  │
  ├─ [45ms-1900ms]  Payment Service: process payment  ← 1855ms HERE
  │                   ├─ Fraud Check: [400ms]
  │                   │    └─ ML Model inference: [380ms] ← ML slow
  │                   ├─ DB: read account: [12ms]
  │                   ├─ DB: write payment: [8ms]
  │                   └─ Core Banking API call: [1400ms] ← external slow
  │
  └─ [1900ms-1950ms] API Gateway: send response
```

At a glance: two problems — ML model inference is slow (380ms, should be <50ms), and Core Banking API call is very slow (1400ms, expected <200ms). Everything else is fast.

Without tracing: you'd add logs to each service one by one, redeploy, reproduce the issue. Hours of work.

#### OpenTelemetry

The standard for distributed tracing (and metrics and logs) across languages and frameworks.

```java
// Spring Boot + OpenTelemetry (auto-instrumented)
// Most HTTP clients and DB drivers are automatically instrumented.

// Manual span for custom code:
@Service
public class PaymentService {
  private final Tracer tracer;
  
  public Payment processPayment(PaymentRequest request) {
    Span span = tracer.spanBuilder("processPayment")
      .setAttribute("payment.amount", request.getAmount())
      .setAttribute("payment.currency", request.getCurrency())
      .startSpan();
    
    try (Scope scope = span.makeCurrent()) {
      // business logic here
      // child spans are automatically nested under this span
      return doProcessPayment(request);
    } catch (Exception e) {
      span.recordException(e);
      span.setStatus(StatusCode.ERROR);
      throw e;
    } finally {
      span.end();
    }
  }
}
```

#### Trace Propagation

For traces to work across services, the trace context must be passed with every call:

```text
Service A calls Service B:
  HTTP header: traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
                              │  │                                  │               │
                              │  trace_id (128-bit)                 span_id         flags

Service B receives the header:
  B extracts trace_id and parent_span_id
  B creates a new span (child of A's span) with the same trace_id
  B's span is nested under A's span in the trace tree
```

OpenTelemetry auto-instruments most HTTP clients (RestTemplate, WebClient, OkHttp) and message consumers (Kafka) to propagate trace context automatically.

#### Sampling

At high traffic (100k req/sec), recording every trace is expensive. Sampling strategies:

**Head-based sampling**: Decide at the start of the trace whether to record it. Random 1% sampling = 1000 traces/sec (cheap, but may miss rare error cases).

**Tail-based sampling**: Buffer all trace data at a collector, decide *after* the trace completes whether to record. Record all error traces, all slow traces (>1s), and 1% of normal traces. More expensive but much more valuable — you never miss errors.

```text
Tail-based sampling policy:
  - ALWAYS sample: error traces, traces > 2s
  - SAMPLE 10%:    traces 500ms-2s
  - SAMPLE 1%:     traces < 500ms
  
At 100k req/sec with 0.1% error rate:
  - Errors: 100 traces/sec (all sampled)
  - Slow (0.5-2s): estimated 500/sec → sample 50/sec
  - Normal: 99,400/sec → sample 994/sec
  → Total: ~1144 traces/sec → manageable storage
```

#### Tracing Stack

- **Instrumentation**: OpenTelemetry SDK (auto + manual)
- **Collector**: OpenTelemetry Collector (aggregates from all services, forwards to backend)
- **Backend/UI**: Jaeger (open source), Tempo (Grafana), Zipkin (simple), AWS X-Ray (managed)

---

### Production Example: IAM Token Introspection Trace

A token introspection request that's slower than usual (p99 spike):

```text
Trace: introspect-req-9f2a (total: 450ms, normal: 8ms)

API Gateway              [0-2ms]    Parse request
Auth Service             [2-450ms]  validate_token
  ├─ Redis: get JWKS     [2-280ms]  ← 278ms!? (should be <1ms)
  │    (connection pool exhausted, waited 275ms for a connection)
  ├─ JWT signature verify [280-283ms]
  └─ Check revocation list [283-450ms] ← 167ms (Redis also slow here)

Finding: Redis connection pool exhausted. All 20 connections in use.
  Why? A batch job started at the same time, making 20 concurrent Redis queries for a report.
  Fix: Separate Redis client for batch jobs vs real-time API; bump introspection pool to 50.
```

The trace pinpointed the root cause in under a minute. Without tracing: "Redis is slow" would have required profiling, guessing, and multiple deployment cycles.

---

### Staff-Level Thinking

Distributed tracing is the third pillar of observability alongside logs and metrics. Its unique value: **latency attribution across service boundaries**. No other tool answers "which service is responsible for this slow request" as directly.

The investment: instrument all services with OpenTelemetry (mostly auto-instrumented), run a collector, deploy Jaeger or Grafana Tempo. The payback is in the first production incident where you use a trace to find the root cause in 30 seconds instead of 2 hours.

---

*End of Part 10: Observability*
# Part 11: Staff Engineer Topics

> "A senior engineer solves problems. A staff engineer changes the nature of the problems the team needs to solve."

---

## Chapter 67: Data Modeling

### Interview Question

> "You're designing the data model for a payment system. Walk me through your process — from understanding requirements to final schema."

---

### Why Should You Care?

The data model is the foundation of every system. Wrong choices at this stage are among the most expensive to fix — changing a primary key type, adding a sharding dimension, or moving from normalized to denormalized all require migrations on live production tables with millions or billions of rows. Staff engineers think through data models with the rigor they deserve.

---

### The Data Modeling Process

**Step 1: Understand the access patterns first.**

Data models should be designed around how data is queried, not how the domain looks. The wrong starting point is "let me map the domain objects to tables." The right starting point is "what queries does the system need to run?"

```text
Access patterns for a payment system:
  1. Create a payment (write, one at a time)
  2. Get payment by ID (point lookup)
  3. List user's payment history, sorted by date, paginated (range scan by user + date)
  4. Get all payments for a merchant in a date range (range scan by merchant + date)
  5. Find payments above $X in the last 30 days (analytics scan — different read model)
  6. Check if a user has any pending payments (existence check by user + status)
```

Each access pattern may suggest different indexes, partition keys, or even different tables/read models.

**Step 2: Normalize for correctness; denormalize for performance.**

Start with 3NF (normalized). Each fact lives in one place. Referential integrity is enforced.

```sql
-- Normalized schema
CREATE TABLE users (
  id UUID PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE accounts (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES users(id),
  balance NUMERIC(18,2) NOT NULL DEFAULT 0,
  currency CHAR(3) NOT NULL,
  status VARCHAR(20) NOT NULL
);

CREATE TABLE payments (
  id UUID PRIMARY KEY,
  from_account_id UUID NOT NULL REFERENCES accounts(id),
  to_account_id UUID NOT NULL REFERENCES accounts(id),
  amount NUMERIC(18,2) NOT NULL,
  currency CHAR(3) NOT NULL,
  status VARCHAR(20) NOT NULL,  -- PENDING, COMPLETED, FAILED
  idempotency_key UUID UNIQUE,  -- prevents duplicate payments
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Then add indexes for access patterns:
```sql
-- Access pattern 3: list user's payments
CREATE INDEX idx_payments_from_account_created 
  ON payments(from_account_id, created_at DESC);

-- Access pattern 4: merchant payments in date range
CREATE INDEX idx_payments_to_account_created 
  ON payments(to_account_id, created_at DESC);

-- Access pattern 6: pending payments by user  
CREATE INDEX idx_payments_status_account 
  ON payments(from_account_id, status) 
  WHERE status = 'PENDING';  -- partial index (only indexes pending)
```

**Step 3: Model balance correctly.**

```text
WRONG: store balance as a mutable column, update on each transaction
  Pros: simple to query current balance
  Cons: race conditions (concurrent updates), no history, can't reconcile

RIGHT: event sourcing for balance
  Each debit/credit is an immutable event
  Balance = SUM of all debits and credits for an account
  
  Pros: full audit trail, reconcilable, no lost updates, accurate
  Cons: expensive to compute balance (requires summing all events)
  
  Practical approach: event sourcing + balance snapshot
    Store immutable ledger entries (every transaction)
    Cache the running balance in accounts.balance (updated transactionally)
    If balance cache corrupts: recompute from ledger entries
```

**Step 4: Think about time.**

All payment records should carry two timestamps:
- `created_at`: when the event happened in the real world (business time)
- `inserted_at`/`recorded_at`: when it was written to the database (processing time)

These differ when systems are delayed (batch processing, offline devices, retries). Querying by business time gives correct results for reporting; querying by processing time gives correct results for data pipeline processing.

**Step 5: Think about growth.**

```text
Payments table: 100,000 payments/day × 365 days × 5 years = 182,500,000 rows

At 500 bytes/row: 91 GB uncompressed (manageable)
With indexes: ~200 GB

Strategy:
  - Partition by created_at (monthly partitions)
  - Archive partitions older than 2 years to cold storage
  - Keep recent 2 years in hot PostgreSQL

No sharding needed for this volume.
```

---

### Production Example: IAM Data Model

```sql
-- Core IAM schema fragments

CREATE TABLE identities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  external_id VARCHAR(255),      -- maps to external system ID
  identity_type VARCHAR(20) NOT NULL,  -- USER, SERVICE_ACCOUNT
  status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE credentials (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  identity_id UUID NOT NULL REFERENCES identities(id),
  credential_type VARCHAR(20) NOT NULL,  -- PASSWORD, TOTP, FIDO2
  credential_hash TEXT,                  -- hashed (bcrypt/argon2)
  credential_metadata JSONB,             -- public key for FIDO2, etc.
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at TIMESTAMPTZ,
  last_used_at TIMESTAMPTZ
);
-- Note: credentials are in a separate table from identities.
-- A credential change does not require updating the identity record.
-- Multiple credential types per identity (password + TOTP).

CREATE TABLE roles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) UNIQUE NOT NULL,
  description TEXT,
  is_system_role BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE TABLE identity_roles (
  identity_id UUID NOT NULL REFERENCES identities(id),
  role_id UUID NOT NULL REFERENCES roles(id),
  assigned_by UUID REFERENCES identities(id),
  assigned_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at TIMESTAMPTZ,
  PRIMARY KEY (identity_id, role_id)
);
-- Append-only: never DELETE role assignments; set expires_at instead (audit trail)

CREATE TABLE audit_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_type VARCHAR(100) NOT NULL,
  actor_id UUID REFERENCES identities(id),
  target_id UUID,
  target_type VARCHAR(50),
  payload JSONB,
  ip_address INET,
  request_id UUID,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);
-- Partitioned because it grows fastest; archive old partitions
```

---

### Staff-Level Thinking

Data models embody business rules. The `identity_roles` table's "never delete, set expires_at" rule is a business requirement (audit trail) encoded in the schema design. The idempotency_key UNIQUE constraint in payments is a safety requirement encoded at the database level — not just application logic.

Every schema decision should have a written rationale. The next engineer who reads it shouldn't have to guess why a column exists, what a NULL value means, or why there's a partial index on this particular condition.

---
# Chapter 67 (Expanded): Data Modeling — Deep Dive

> "Many architecture decisions are really data modeling decisions in disguise. Get the model wrong and no amount of clever infrastructure saves you."

---

## Interview Question

> "Walk me through how you'd model the data for an IAM platform — covering identity, credentials, roles, sessions, and audit events. What are the bounded contexts? How do reads and writes differ?"

---

## Why This Chapter Exists

Chapter 67 introduced the basics of data modeling. This expansion goes deeper into the disciplines that separate pragmatic data modeling from academic schema design: Domain-Driven Design, aggregates, bounded contexts, event modeling, relational vs NoSQL tradeoffs, and the explicit separation of read models from write models.

In practice, the question "should we use Kafka or REST here?" is often actually the question "where does the ownership of this data live, and how must it flow?" Architecture is downstream of data modeling.

---

## Domain-Driven Design (DDD) Fundamentals

DDD is a discipline for aligning the code model with the business domain. It provides vocabulary and patterns for finding the natural boundaries of a system.

### Ubiquitous Language

The most underrated DDD concept. Every word that appears in the model, code, and conversations should mean the same thing to engineers and domain experts.

```text
WITHOUT ubiquitous language:
  Engineer says: "I'll create a User record and add a Role association."
  Domain expert says: "We need to provision access for an Identity with appropriate Entitlements."
  
  They're talking about the same thing but using different words.
  The code reflects the engineer's vocabulary. Domain expert can't verify correctness.
  
WITH ubiquitous language:
  Both sides agree: "An Identity is provisioned with one or more Entitlement Grants."
  The code matches:
  
  identity = Identity.provision(external_id:, identity_type:)
  identity.grant_entitlement(entitlement:, granted_by:, expires_at:)
```

When you hear a domain expert describe a concept that doesn't have a name in your code, that is a modeling gap. Create the concept.

### Entities vs Value Objects

**Entity**: Has identity that persists across time. Two entities are equal if they have the same ID, even if all their attributes differ.

```java
// Entity: a User — identified by ID
public class Identity {
    private final UUID id;         // the identity
    private String displayName;    // changes over time
    private IdentityStatus status; // changes over time
    
    // Same identity even if name changes
    @Override
    public boolean equals(Object o) {
        return o instanceof Identity && ((Identity)o).id.equals(this.id);
    }
}
```

**Value Object**: Has no identity. Two value objects are equal if all their attributes are equal. Immutable.

```java
// Value Object: a Money amount — no identity
public record Money(BigDecimal amount, Currency currency) {
    // Two Money(100, USD) instances are equal and interchangeable.
    // If the amount changes, you create a new Money, not mutate existing.
}

// Value Object: an address
public record Address(String street, String city, String pincode, String country) {}
```

**Why it matters for modeling**: Entities need IDs and identity tracking. Value objects should be immutable — if you find yourself updating a value object, you've made it an entity by accident.

In IAM:
- `Identity` → Entity (has a persistent ID; name and status change but it's the same identity)
- `Credential` → Entity (has ID; the hash changes on password reset)
- `Role` → Entity
- `Permission` (read:payments) → Value Object (no meaningful ID; `read:payments` == `read:payments`)
- `Money` amount on a payment → Value Object
- `IPAddress` on an audit event → Value Object

### Aggregates

An Aggregate is a cluster of entities and value objects that must be treated as a unit for consistency purposes. Every aggregate has an **Aggregate Root** — the single entry point through which all operations on the aggregate must flow.

**Rules**:
1. Operations on objects inside an aggregate must go through the aggregate root
2. External objects can only hold references to the aggregate root (by ID), not to internal entities
3. Each transaction should span at most one aggregate (the "aggregate per transaction" rule)

```text
IAM Aggregate: Identity

Aggregate Root: Identity
  └── Credentials (entity, owned by Identity)
        ├── PasswordCredential
        └── TOTPCredential
  └── EntitlementGrants (entity, owned by Identity)
        ├── Grant { role_id, granted_by, granted_at, expires_at }
        └── Grant { ... }
  └── ContactInfo (value object)

CORRECT operation:
  identity = identityRepo.findById(id)  // load aggregate root
  identity.changePassword(newHash)      // operate through the root
  identityRepo.save(identity)           // save the whole aggregate

WRONG operation:
  credential = credentialRepo.findById(credentialId)  // bypassing root
  credential.setHash(newHash)                          // mutating internal entity directly
  credentialRepo.save(credential)                      // saving internal entity directly
  // → Breaks consistency rules, business invariants can be violated
```

**Why aggregates matter for database design**: Each aggregate typically maps to one or a few closely related tables. The aggregate root's ID is the consistency boundary. Transactions within an aggregate are local (fast, ACID). Transactions spanning aggregates require eventual consistency or sagas.

In a payment system:
- `Order` aggregate contains `LineItems` (you never mutate a line item without going through the Order)
- `Account` aggregate contains `Balance` and `AccountStatus` (Balance is never updated except through Account operations)

### Bounded Contexts

A bounded context is a region of the domain where a specific model applies consistently. The same word can mean different things in different bounded contexts — and that's intentional and correct.

```text
"Account" in the Banking domain:

Identity Context:
  Account = user's identity record (username, credentials, status, roles)
  Primary concern: authentication, authorisation

Ledger Context:
  Account = financial record (balance, currency, account number, owner)
  Primary concern: money movement, balance accuracy

Notification Context:
  Account = contact preferences (email, phone, opt-ins)
  Primary concern: how to reach the user

These three "accounts" are different models for different concerns.
Trying to merge them into one mega-Account table = a muddy, unmaintainable model.

Context Map:
  Identity Context ──customer-supplier──→ Ledger Context
  (Identity provides identity; Ledger consumes it for ownership)
  
  Identity Context ──customer-supplier──→ Notification Context
  (Identity provides the actor; Notification provides the channel)
```

**Context Map relationships**:
- **Shared Kernel**: Two contexts share a common model (dangerous; creates tight coupling)
- **Customer-Supplier**: Upstream provides data; downstream consumes
- **Conformist**: Downstream conforms to upstream's model (pragmatic when upstream can't change)
- **Anti-Corruption Layer (ACL)**: Translation layer between contexts so internal model stays clean
- **Published Language**: A well-documented, stable API between contexts

In an IAM platform: the Identity context publishes a `UserProvisioned` integration event. Every other context (notification, ledger, provisioning) subscribes and translates into their own model via an ACL. The Identity context knows nothing about the downstream consumers.

---

## Relational Modeling Patterns

### The Access-Pattern-First Method

```text
Step 1: List every query the application needs (not the domain objects):
  Q1: Find user by email (login)
  Q2: Get user's current roles (authorisation)
  Q3: List all users with a given role (admin UI)
  Q4: Get user's login history, last 30 days, paginated
  Q5: Find all sessions for a user (session management)
  Q6: Audit events by actor + time range (compliance)
  Q7: All failed logins in the last hour (security monitoring)

Step 2: Map each query to its index:
  Q1: UNIQUE INDEX on identities(email)
  Q2: INDEX on identity_roles(identity_id)
  Q3: INDEX on identity_roles(role_id)
  Q4: INDEX on login_events(identity_id, created_at DESC)
  Q5: INDEX on sessions(identity_id, status) WHERE status = 'ACTIVE'
  Q6: INDEX on audit_events(actor_id, created_at)
  Q7: INDEX on login_events(created_at) WHERE success = FALSE
      (partial index: only indexes failed logins → small, fast)

Step 3: Schema falls out naturally from the queries.
```

### Temporal Modeling

Many domain objects have histories that matter:

```sql
-- WRONG: losing history by overwriting
UPDATE identities SET status = 'SUSPENDED' WHERE id = ?;
-- Previous status is gone. When was it suspended? By whom? We don't know.

-- RIGHT: event log for immutable history
CREATE TABLE identity_status_events (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  identity_id  UUID NOT NULL REFERENCES identities(id),
  old_status   VARCHAR(20),
  new_status   VARCHAR(20) NOT NULL,
  changed_by   UUID REFERENCES identities(id),
  reason       TEXT,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Current status = latest event's new_status (or use a materialized column)
-- Full history = SELECT * FROM identity_status_events WHERE identity_id = ? ORDER BY created_at

-- Optimisation: keep current_status denormalized on the identities table
-- (updated transactionally alongside the event insert)
ALTER TABLE identities ADD COLUMN current_status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE';
```

This is the "append-only" principle: never delete or overwrite facts. Add new facts. Query the latest.

### Soft Deletes vs Hard Deletes

```sql
-- Hard delete: record is gone. Referential integrity breaks if other tables reference it.
DELETE FROM roles WHERE id = ?;
-- If any identity has this role, the foreign key prevents deletion (or cascades incorrectly).

-- Soft delete: mark as deleted, preserve referential integrity and audit trail.
ALTER TABLE roles ADD COLUMN deleted_at TIMESTAMPTZ;
ALTER TABLE roles ADD COLUMN deleted_by UUID REFERENCES identities(id);

-- "Delete" a role:
UPDATE roles SET deleted_at = NOW(), deleted_by = :actor_id WHERE id = :role_id;

-- Queries for active roles:
SELECT * FROM roles WHERE deleted_at IS NULL;

-- Pitfall: unique constraint on name must include deleted_at or allow re-creation:
CREATE UNIQUE INDEX idx_roles_name_active ON roles(name) WHERE deleted_at IS NULL;
```

---

## NoSQL Modeling Patterns

### The Fundamental Shift

Relational: model for flexibility (normalize, join at query time).
NoSQL: model for your queries (denormalize, pre-compute what you need).

```text
Relational query (flexible, compute-heavy):
  SELECT u.name, r.name as role_name
  FROM users u
  JOIN user_roles ur ON u.id = ur.user_id
  JOIN roles r ON ur.role_id = r.id
  WHERE u.id = ?

DynamoDB equivalent (pre-computed, fast):
  Table: IAMData
  PK: USER#<user_id>
  SK: ROLE#<role_id>
  Attributes: { role_name, granted_at, granted_by }

  Query: PK = USER#123 AND SK begins_with ROLE#
  → Returns all roles for user 123 in a single index lookup, no join.
```

### Single-Table Design (DynamoDB Pattern)

In DynamoDB (and similar key-value/wide-column stores), all entity types live in one table. The partition key and sort key encode the entity type and relationships.

```text
PK              | SK                   | Attributes
─────────────── | ──────────────────── | ─────────────────────────
USER#abc-123    | PROFILE              | { name, email, created_at }
USER#abc-123    | ROLE#admin           | { granted_at, granted_by }
USER#abc-123    | ROLE#payment_user    | { granted_at, granted_by }
USER#abc-123    | SESSION#sess-456     | { created_at, expires_at, ip }
ROLE#admin      | METADATA             | { description, is_system_role }
ROLE#admin      | PERMISSION#read:all  | { granted_at }

Access patterns:
  Get user profile:           PK=USER#abc-123, SK=PROFILE
  Get user's roles:           PK=USER#abc-123, SK begins_with ROLE#
  Get user's sessions:        PK=USER#abc-123, SK begins_with SESSION#
  Get all permissions of role: PK=ROLE#admin, SK begins_with PERMISSION#
  
  All O(1) or O(log n) — no scans, no joins.
```

**GSI (Global Secondary Index)** for reverse lookups:

```text
GSI1PK (inverted): ROLE#admin → returns all USERs with this role
  GSI1PK         | GSI1SK        | PK
  ─────────────── | ───────────── | ──────────────
  ROLE#admin      | USER#abc-123  | USER#abc-123
  ROLE#admin      | USER#def-456  | USER#def-456

Query: all users with admin role → GSI1 query, PK=ROLE#admin
```

### Cassandra Modeling (Wide-Column)

In Cassandra, the data model is the query. Design the table for the query, not the domain.

```sql
-- Query: "Get all payments for a user, sorted by date descending, paginated"
CREATE TABLE payments_by_user (
  user_id     UUID,
  created_at  TIMESTAMP,
  payment_id  UUID,
  amount      DECIMAL,
  status      TEXT,
  merchant_id UUID,
  PRIMARY KEY (user_id, created_at, payment_id)
) WITH CLUSTERING ORDER BY (created_at DESC);

-- This table is ENTIRELY designed around this one query.
-- The same payment data may exist in multiple tables for different access patterns.
-- This is normal and expected in Cassandra.

-- Query: "Get all payments for a merchant, sorted by date"
CREATE TABLE payments_by_merchant (
  merchant_id UUID,
  created_at  TIMESTAMP,
  payment_id  UUID,
  amount      DECIMAL,
  user_id     UUID,
  PRIMARY KEY (merchant_id, created_at, payment_id)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

Cassandra anti-patterns:
- `ALLOW FILTERING` → full partition scan, avoid at all costs
- Too wide a partition → unbounded row growth causes hotspots
- Too many secondary indexes → every write updates many indexes; use sparingly

---

## Event Modeling

Event Modeling is a technique for designing systems around the events that happen, not the objects that exist.

### The Event Storm

Before writing any schema, do an event storm: write every domain event on sticky notes (one per note), place them on a timeline.

```text
IAM Platform Event Storm:

Timeline left → right:
  [IdentityProvisioned] → [CredentialSet] → [LoginAttempted] → [LoginSucceeded]
  → [SessionCreated] → [RoleGranted] → [PermissionChecked] → [PasswordChanged]
  → [LoginAttempted] → [LoginFailed] → [AccountLocked] → [SessionInvalidated]
  → [AccountUnlocked] → [RoleRevoked] → [IdentityDeprovisionedA]

Commands (what triggered each event):
  ProvisionIdentity → [IdentityProvisioned]
  SetCredential     → [CredentialSet]
  Login             → [LoginAttempted] → either [LoginSucceeded] or [LoginFailed]

Read models (what the UI needs to display):
  "Current user's roles" → materialized from RoleGranted + RoleRevoked events
  "Active sessions"      → materialized from SessionCreated + SessionInvalidated events
  "Recent login history" → materialized from LoginSucceeded + LoginFailed events
```

The event storm reveals:
- What facts the system must record
- What read models are needed (and how they're derived from events)
- Where commands can fail and what the failure events are
- The natural aggregate boundaries (events that must be consistent together)

### Write Model vs Read Model

The write model is optimized for correctness and capturing intent. The read model is optimized for query performance.

```text
WRITE MODEL (normalized, command-oriented):

Table: identity_role_grants
  id, identity_id, role_id, granted_by, granted_at, expires_at, revoked_at, revoked_by

Every grant and revocation is an immutable row.
Source of truth for "who has what role and why."
Audit trail built in.

READ MODEL (denormalized, query-oriented):

Table: identity_current_roles (materialized)
  identity_id, role_id, role_name, granted_at, expires_at

Only contains ACTIVE grants (no revoked rows).
Joined with role name for fast display.
Rebuilt from the write model whenever the grants table changes.

Query: "Show me user X's current roles"
  → Read model: O(log n) index scan, no joins, returns immediately
  → Write model: would require filtering revoked_at IS NULL + joining roles table
```

The read model is eventually consistent with the write model (updated asynchronously). This is CQRS in practice.

---

## Modeling for the IAM Platform

Putting it all together: a complete IAM platform data model using DDD principles.

```text
BOUNDED CONTEXTS:

1. Identity Context
   Aggregate: Identity
   Root: Identity { id, external_id, type, status }
   Members: Credentials (PasswordCredential, TOTPCredential, FIDO2Key)
   Invariants: An Active identity must have at least one credential.

2. Authorisation Context
   Aggregate: EntitlementGrant
   Root: EntitlementGrant { id, identity_id, role_id, granted_by, expires_at }
   (Note: Grant is its own aggregate because grants have their own lifecycle.
    Identity doesn't own grants; the Authorisation context does.)
   
3. Session Context
   Aggregate: Session
   Root: Session { id, identity_id, created_at, expires_at, status }
   (Sessions have independent lifecycle from Identity)

4. Audit Context
   No aggregates — purely append-only event log. No invariants to enforce.
   Consumes integration events from all other contexts.

INTEGRATION EVENTS (flow between contexts):
  Identity Context emits:
    IdentityProvisioned { identity_id, external_id, type, timestamp }
    IdentityStatusChanged { identity_id, new_status, changed_by, timestamp }
    CredentialChanged { identity_id, credential_type, timestamp }
  
  Authorisation Context emits:
    RoleGranted { identity_id, role_id, role_name, granted_by, expires_at }
    RoleRevoked { identity_id, role_id, revoked_by, timestamp }
  
  Session Context emits:
    SessionCreated { session_id, identity_id, ip, created_at }
    SessionInvalidated { session_id, identity_id, reason, timestamp }
```

The Audit Context subscribes to all of the above. It never calls back — it's a pure consumer. This prevents the audit log from becoming a dependency of the other contexts (which would mean a slow audit write could fail a login).

---

## Common Data Modeling Mistakes

**1. Modeling the domain, not the queries**: Building a perfectly normalized schema that requires 8-table joins for the most common query. The normalization is beautiful; the performance is catastrophic.

**2. God table**: One table with 80 columns, a `type` discriminator column, and many nullable columns that only apply to certain types. Every query hits every row even though most columns are irrelevant. Fix: table-per-type inheritance or JSONB for flexible attributes on a narrow base table.

**3. Implicit state through nullable columns**: `completed_at TIMESTAMPTZ NULL` means "null = not complete, non-null = complete." Two months later, a bug sets `completed_at` to a wrong timestamp. Now completed_at is not null, so the system thinks it completed. Explicit status columns with constraints are safer.

**4. Crossing aggregate boundaries in transactions**: `BEGIN; UPDATE orders ...; UPDATE inventory ...; COMMIT;` — orders and inventory are separate aggregates in different bounded contexts. This transaction couples them. In a distributed system, it becomes a 2PC nightmare. Fix: use sagas; design aggregates to be independently consistent.

**5. Missing the write model**: Storing only the current state, not how it got there. The current role assignments tell you what's true now. The grant/revoke history tells you what happened, by whom, and when. Compliance needs the history. Always separate current state from history.

---

## Staff-Level Thinking on Data Modeling

Every significant performance problem, consistency bug, and operational nightmare in a mature system traces back to a data modeling decision made early. The questions to ask upfront:

- **What are the aggregates?** Where are the consistency boundaries?
- **What events happened?** Model history as events, not overwritten state.
- **Who owns what data?** Which context is the source of truth for each concept?
- **What queries must be fast?** Design the model around the critical read paths.
- **What changes independently?** Things that change together belong together; things that change independently should be separate.
- **What must be consistent together?** If two things must always agree, they should be in the same aggregate and the same transaction.

Data modeling is not a one-time activity. Revisit it as the domain evolves. The cost of a wrong model grows with every row in the database.

---

*End of Chapter 67 (Expanded)*


## Chapter 68: Tradeoff Analysis

### Interview Question

> "How do you decide between two architecturally different approaches? Walk me through your decision-making framework."

---

### Why Should You Care?

Every significant technical decision involves tradeoffs. The engineer who can articulate what is being traded — not just what they recommend — is far more valuable than the engineer who only knows how to implement one approach. Staff engineers make tradeoff analysis the explicit, documented center of their architectural decision-making.

---

### The Tradeoff Analysis Framework

**Step 1: Clarify the real constraints.**

Not all constraints are real. Before analyzing tradeoffs, challenge each constraint:
- Is this a genuine requirement or a historical assumption?
- What is the cost of relaxing this constraint?
- Who owns this constraint? Can they change it?

**Step 2: Enumerate the axes of comparison.**

Common axes:
- Consistency: can the system return stale or incorrect data?
- Availability: what happens during failures?
- Latency: what is the user-facing response time?
- Throughput: how many requests can the system handle?
- Cost: infrastructure, operational, development
- Complexity: how hard is it to build, test, debug, and maintain?
- Operability: how hard is it to deploy, monitor, and recover from failures?
- Team fit: does the team have expertise in this?
- Time to market: how long to implement?

**Step 3: Score each option against each axis.**

Not every axis matters equally. Weight them by the actual constraints:

```text
Decision: Token validation — in-DB lookup vs JWT vs Redis cache

Constraints (weighted):
  Latency: HIGH (p99 < 10ms required)
  Consistency: MEDIUM (can tolerate 30s stale for revocations)
  Operational simplicity: MEDIUM
  Cost: LOW (we have budget)

                | DB Lookup | JWT (stateless) | Redis Cache
----------------|-----------|-----------------|-------------
Latency         |    LOW    |   HIGH ✓        |  HIGH ✓
Consistency     |   HIGH ✓  |   LOW (expiry)  |  MEDIUM ✓
Operational     |  MEDIUM   |   HIGH ✓        |  MEDIUM
Cost            |   MEDIUM  |   LOW ✓         |  MEDIUM
Revocation      |   HIGH ✓  |   LOW ✗         |  MEDIUM ✓

Winner: Redis cache (balances latency + consistency + revocability)
JWT considered but consistency (revocation) disqualified it for this use case.
DB lookup disqualified by latency.
```

**Step 4: Make the decision, document the reasoning.**

An Architecture Decision Record (ADR) captures:
- Context: what forced this decision?
- Decision: what was chosen?
- Consequences: what are the known tradeoffs?
- Alternatives considered: what else was evaluated and why it was rejected?

ADRs are irreplaceable 18 months later when someone asks "why did we do this?"

---

### Sample ADR

```markdown
# ADR 0042: Token Validation Strategy

## Status: Accepted (2024-01-15)

## Context
Token introspection is called for every API request. Target: p99 < 10ms.
Revocation must take effect within 60 seconds (security requirement).

## Decision
Use Redis cache with 30-second TTL for token validation.
JWT stateless validation is insufficient (revocation > 60s).
Direct DB lookup fails latency SLO (p99 ~200ms under load).

## Consequences
- Positive: p99 < 5ms achieved. Revocation effective within 30s.
- Negative: Redis becomes a critical dependency. Redis outage = auth outage.
  Mitigation: Redis Cluster with 3 primary shards + replica. Circuit breaker
  falls back to DB on Redis failure (latency SLO violated but system functional).
- Negative: Added operational complexity (Redis cluster to maintain).

## Alternatives Considered
- Pure JWT: rejected (revocation window > 30 minutes with 15-min expiry).
- DB with read replica: rejected (p99 ~80ms, fails SLO even with replica).
- Hybrid (JWT + revocation list): evaluated, adds complexity equivalent to Redis
  with worse latency characteristics. Redis preferred.
```

---

### Staff-Level Thinking

The best tradeoff analysis acknowledges uncertainty: "We chose X based on the assumption that Y. If Y turns out to be false, we would revisit this decision." Stated assumptions are revisable. Unstated assumptions are landmines.

---

## Chapter 69: Multi-Region Design

### Interview Question

> "How would you design the IAM platform to serve users across 5 geographic regions with sub-100ms login latency everywhere?"

---

### Why Should You Care?

Multi-region architecture is where consistency, availability, latency, and cost collide at their hardest. It is the design challenge that separates staff engineers from senior engineers — not because of exotic technology, but because of the precision required in reasoning about each system component's consistency requirements across geographic distance.

---

### The Multi-Region Framework

#### Why Multi-Region?

- **Latency**: Users in São Paulo should not talk to servers in Mumbai. 200ms cross-region latency vs 5ms local.
- **Compliance**: Data residency requirements. Indian users' PII must stay in India (RBI regulations). GDPR in EU.
- **Availability**: A region-level failure (AWS us-east-1 outage) should not take down the global service.

#### What is Replicated Where

The key question for every piece of data: what is the latency tolerance, consistency requirement, and data residency constraint?

```text
Data Classification for IAM:

GLOBAL (same dataset in all regions):
  - Role definitions (change infrequently, read from every region)
  - Tenant configuration
  - JWKS (public keys for JWT verification)
  - Permission policies
  
  Consistency: multi-leader or leader + async replicas
  Conflict resolution: last-write-wins (infrequent changes, low conflict risk)

REGIONAL (different dataset per region):
  - User credentials (PII, must stay in region)
  - Sessions (latency-sensitive, must be in same region as user)
  - Audit logs (compliance, must stay in region)
  
  Strategy: each region has its own primary DB; no cross-region writes

GLOBAL BUT CRITICAL (consistent across regions):
  - Active session count (for rate limiting login attempts)
  - Distributed locks (prevent duplicate processing)
  
  Strategy: strong consistency with quorum (pays latency cost)
```

#### Login Flow in Multi-Region

```text
User in Singapore → Login request

DNS resolves to nearest region (Singapore DC):
  Singapore → validate credentials (Singapore DB) → issue token signed with Singapore key
  
BUT: Singapore key must be trusted in all regions.
  Solution: JWKS endpoint (served from all regions) includes Singapore public key.
  Any region can verify a token issued by any other region.
  
  JWKS is a read-heavy, infrequently-changing global dataset:
  Replicate to all regions with async replication (minutes stale is fine for JWKS).
  Cache at API tier with 60-minute TTL.
```

#### Cross-Region Consistency for Critical Operations

**Scenario**: User account is suspended in India. They immediately try to login from Singapore.

Without cross-region consistency: Singapore region hasn't received the suspension update yet (async replication lag). Login succeeds.

Solution options:
1. **Synchronous cross-region write**: Mark account suspended in all regions before acknowledging. Latency: 150-200ms for a Mumbai→Singapore round trip. Acceptable for infrequent admin actions.
2. **Token short expiry + revocation list**: Account is suspended → add to cross-region revocation list (multi-leader, ~5s propagation). Next token introspection (within 30s) picks up the suspension. User gets at most 30 more seconds.
3. **Accept the consistency window**: For account suspension, a 60-second consistency window is acceptable. Login in Singapore succeeds; the next token refresh fails.

The right answer depends on the risk tolerance of the suspension action. Fraud suspension: synchronous. Administrative suspension: eventual consistency window acceptable.

---

### Staff-Level Thinking

Multi-region architecture is not "run the same thing in every region." It is a deliberate classification of every data type into a consistency tier, then choosing the right replication and consistency mechanism for each tier.

The question to answer for each piece of data: "Can this data exist in a local region without knowing about changes happening in other regions? For how long? What is the risk if it's stale?"

---

## Chapter 70: Failure Mode Analysis

### Interview Question

> "Walk me through every way your payment system can fail. How do you reason about and mitigate failure modes?"

---

### Why Should You Care?

Systems fail in ways their designers didn't expect. Failure mode analysis is the practice of *expecting the unexpected* — systematically enumerating how components can fail before they do, and designing mitigations. The engineers who do this produce systems that degrade gracefully rather than catastrophically.

---

### The Failure Mode Framework

**Technique: Failure Mode and Effects Analysis (FMEA)**

For each component:
1. What can fail?
2. How will it fail (what does the failure look like)?
3. What is the impact to the user?
4. What is the likelihood?
5. What is the detectability (how quickly will we know)?
6. Risk = Severity × Likelihood / Detectability
7. Mitigation?

```text
Component: Redis Session Cache
┌─────────────────┬─────────────────┬──────────┬────────────┬──────────────────────────┐
│ Failure Mode    │ How it manifests│ Severity │ Likelihood │ Mitigation               │
├─────────────────┼─────────────────┼──────────┼────────────┼──────────────────────────┤
│ Node crash      │ Cache miss spike│ Medium   │ Medium     │ Redis Cluster (replica)  │
│ Network partition│ Timeout errors │ High     │ Low        │ Circuit breaker → DB     │
│ Memory overflow │ Eviction storm  │ High     │ Low        │ maxmemory-policy, alerts │
│ Corrupt data    │ Wrong user data │ Critical │ Very Low   │ TTL-based natural expiry │
│ Slow response   │ Latency spike   │ High     │   Low      │ Timeout + circuit breaker│
└─────────────────┴─────────────────┴──────────┴────────────┴──────────────────────────┘
```

#### Failure Taxonomy

**Hardware failures**: Disk failure, NIC failure, power loss. Mitigated by: RAID, redundant power, multiple AZs.

**Software failures**: Bugs, memory leaks, deadlocks. Mitigated by: testing, gradual rollouts, automated restarts (Kubernetes).

**Dependency failures**: Database down, external API slow. Mitigated by: circuit breakers, timeouts, fallbacks.

**Configuration failures**: Wrong environment variable, missing feature flag. Mitigated by: configuration validation on startup, canary deployments.

**Operational failures**: Human error (wrong command, wrong environment). Mitigated by: RBAC on production access, infrastructure-as-code, automated runbooks.

**Cascading failures**: One failure causes another. Mitigated by: bulkheads, circuit breakers, rate limiting.

**Thundering herd**: Many clients retry simultaneously. Mitigated by: jitter, exponential backoff.

**Partial failures**: One of three replicas fails. Mitigated by: quorum reads/writes, health checks.

#### Chaos Engineering

Chaos engineering is proactively injecting failures to test mitigation:
- Kill a pod randomly (Chaos Monkey)
- Slow a network interface (add 200ms latency to outgoing packets)
- Fill a disk
- Terminate a database primary and verify failover time

Netflix pioneered this. The goal: discover failure modes before users do.

```text
Chaos experiment plan for IAM:
  1. Kill one Redis node → verify session reads continue from replica (latency impact?)
  2. Delay PostgreSQL responses by 500ms → verify circuit breaker trips, fallback activates
  3. Flood the login endpoint at 10x capacity → verify rate limiting, graceful degradation
  4. Terminate ForgeRock pod → verify token introspection fails gracefully, service continues
  5. Partition the network between app tier and DB tier for 30 seconds → verify reconnection
```

Run chaos experiments in staging, then carefully in production during low-traffic periods.

---

### Staff-Level Thinking

The value of failure mode analysis is not the document — it's the *thinking*. Walking through failure modes changes how engineers design mitigations. It turns "I hope Redis doesn't go down" into "when Redis goes down, here's exactly what happens and here's the circuit breaker that prevents cascading failure."

---

## Chapter 71: Operational Excellence

### Interview Question

> "Your team just had a major production incident. How do you run the postmortem? What systemic changes do you make?"

---

### Why Should You Care?

Production incidents are inevitable. How a team responds — during and after — determines whether the same incident happens again. Operational excellence is not about being perfect; it is about learning fast and improving continuously.

---

### The Incident Management Framework

#### During an Incident

**Roles**:
- **Incident Commander (IC)**: Coordinates the response. Makes final decisions. Communicates status. Does NOT do technical debugging — that's others' job.
- **Technical Leads**: Debug and implement fixes.
- **Communications Lead**: Updates stakeholders, writes status page updates.
- **Scribe**: Documents the timeline in real-time.

**Process**:
1. Declare the incident (severity level, assign IC)
2. Create incident channel (Slack: #incident-2024-01-15)
3. IC: "What do we know? What is impacted? What's the current status?"
4. Communicate to stakeholders immediately (even if "we're investigating")
5. Work the problem: diagnose → mitigate → verify → resolve
6. All-clear: communicate resolution, update status page

#### Postmortem

**Purpose**: Learn why the incident happened and prevent recurrence. Not to assign blame.

**Blameless culture**: Humans operate in systems. If a human made a mistake, the system allowed them to make it. The fix is the system, not the human.

**Postmortem structure**:

```markdown
# Incident Postmortem: Payment Service Outage (2024-01-15)

## Impact
  - 23 minutes of payment failures (10:14 AM - 10:37 AM)
  - ~15,000 payment failures
  - Estimated impact: ₹12 crore in failed transactions

## Timeline
  10:05 AM: Deployment of payment-service v2.3.1 started
  10:14 AM: Error rate spike detected (alert: 5%)
  10:15 AM: On-call paged
  10:18 AM: Incident declared, IC assigned
  10:23 AM: Root cause identified (see below)
  10:31 AM: Rollback initiated
  10:37 AM: Error rate returned to baseline
  10:45 AM: All-clear declared

## Root Cause
  payment-service v2.3.1 introduced a new database query in the payment processing path
  without an index. Under production load, the query performed a sequential scan on the 
  100M-row payments table, taking 8-12 seconds per query.
  
  The query was tested in staging with a 50,000-row table and completed in 5ms.
  Production's 100M-row table exposed the O(n) scan.

## Contributing Factors
  1. No performance testing with production-scale data in staging
  2. No query plan review in code review process
  3. Deployment proceeded without canary (full rollout)

## Action Items
  ┌─────────────────────────────────────────────────┬──────────┬──────────┐
  │ Action                                          │  Owner   │  Due     │
  ├─────────────────────────────────────────────────┼──────────┼──────────┤
  │ Add production-scale data to performance tests  │ Team A   │ Jan 22   │
  │ Add EXPLAIN ANALYZE to code review checklist    │ Tech Lead│ Jan 19   │
  │ Implement mandatory canary for payment service  │ Platform │ Jan 29   │
  │ Add "slow query" alert (>100ms) in staging      │ Team A   │ Jan 22   │
  └─────────────────────────────────────────────────┴──────────┴──────────┘

## Lessons Learned
  The staging-production data volume gap is a systemic risk.
  This was the third incident involving a query that worked in staging but not production.
  Action item: production shadow testing infrastructure (priority).
```

#### Measuring Operational Health

- **MTTR** (Mean Time to Recover): How long incidents last on average. Target: < 30 minutes.
- **MTTD** (Mean Time to Detect): How long before incidents are detected. Target: < 5 minutes (automated alerts).
- **MTBF** (Mean Time Between Failures): How often incidents occur. Target: increasing.
- **Change Failure Rate**: Percentage of deployments that cause incidents. Target: < 5%.

---

### Staff-Level Thinking

A staff engineer's role in operational excellence is systemic: not just fixing individual incidents, but identifying patterns across incidents and driving the platform changes that prevent categories of failure. Three incidents caused by the same root cause (staging/production data gap) is a systemic risk that deserves a dedicated project, not just individual action items.

---

## Chapter 72: Cost Optimization

### Interview Question

> "Your AWS bill is $500k/month and growing 20% per month. The CTO asks for a 30% reduction plan. Where do you start?"

---

### Why Should You Care?

Cloud costs are engineering costs. An inefficient system architecture is not just technically suboptimal — it has a monthly financial cost that grows with scale. Staff engineers are expected to understand the cost implications of their designs.

---

### The Cost Optimization Framework

**Step 1: Understand where the money goes.**

```text
AWS cost breakdown (typical):
  EC2 instances:      40%
  RDS databases:      25%
  Data transfer:      15%
  S3 storage:         10%
  Other (Lambda, etc): 10%
```

Use AWS Cost Explorer or cloud cost observability tools (CloudHealth, Infracost) to get line-item visibility by service, team, and environment.

**Step 2: Identify the high-leverage opportunities.**

The Pareto principle applies: 20% of resources typically account for 80% of cost. Find the expensive clusters first.

**Right-sizing**: Most engineers over-provision "to be safe." A database instance running at 15% CPU average was probably provisioned for a hypothetical peak that never happened.

```text
Right-sizing analysis:
  Current: db.r6g.4xlarge ($4,000/month), avg CPU: 12%, avg RAM: 35%
  Right-size: db.r6g.2xlarge ($2,000/month) — still leaves 2x headroom
  Savings: $2,000/month (50% reduction on this one instance)
```

**Savings Plans / Reserved Instances**: Pay upfront for 1-3 year commitment, get 30-60% discount. For stable, long-running workloads (production databases, steady-state applications), this is almost always worth it.

**Spot Instances**: Up to 90% cheaper than on-demand. Can be terminated with 2-minute warning. Suitable for: batch jobs, development environments, stateless services with graceful shutdown.

**Storage tiering**: S3 Intelligent-Tiering automatically moves infrequently accessed data to cheaper storage classes. Audit logs from 2 years ago don't need to be in S3 Standard ($0.025/GB/month) — move to Glacier ($0.004/GB/month).

**Data transfer costs**: Egress from cloud to internet is expensive. Egress between AZs within the same region is cheaper. Egress between regions is expensive. Architecture that moves large data cross-region incurs significant cost.

**Step 3: Eliminate waste.**

```text
Common waste:
  - Development/staging environments running 24/7 (shut down nights/weekends: 70% reduction)
  - Unused Elastic IPs (charged even when not attached)
  - Unattached EBS volumes (previous instance terminated, volume left)
  - Old AMI snapshots (accumulate over years)
  - Nat Gateway for every AZ (consolidate where possible)
  - Over-provisioned Kafka clusters (reduce retention, rightsize brokers)
```

**Step 4: Architecture-level changes (longer-term).**

- Move batch/analytics workloads off expensive RDS to S3 + Athena
- Replace always-on services with event-driven Lambda for low-traffic endpoints
- Implement efficient caching to reduce database read volume
- Use CDN for static assets (dramatically reduces EC2 egress)

---

### Staff-Level Thinking

Cost optimization should be a regular practice, not a crisis response. Establish a cost review cadence (monthly) and ownership (which team owns which cost). Hold teams accountable for their cost efficiency as a metric, alongside their SLO.

The most cost-efficient code is code that doesn't need to run. Caching, batching, and algorithmic efficiency are free cost reductions.

---

## Chapter 73: Incident Response

*(Building on Chapter 71 with technical detail)*

### The Technical Incident Response Toolkit

During an active incident, having these skills ready matters:

**Diagnosing database saturation**:
```sql
-- Who is holding locks?
SELECT pid, query, state, wait_event_type, wait_event, query_start
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;

-- Active locks
SELECT * FROM pg_locks WHERE NOT granted;

-- Kill a blocking query (last resort)
SELECT pg_terminate_backend(pid) WHERE pid = <blocking_pid>;
```

**Diagnosing memory pressure** (JVM):
```bash
# Heap dump for analysis
jmap -dump:live,format=b,file=heap.hprof <pid>

# GC statistics
jstat -gcutil <pid> 1000  # every 1 second

# Thread dump (for deadlock detection)
kill -3 <pid>  # prints thread dump to stdout
```

**Diagnosing network issues**:
```bash
# Check connectivity
curl -v --connect-timeout 5 http://service:8080/health

# Check DNS resolution time
time nslookup payment-service.svc.cluster.local

# TCP connections by state
ss -s

# Which process is using which port
ss -tlnp
```

**Rollback decision framework**:
```text
Rollback if:
  - Error rate > 5% AND not improving after 5 minutes
  - Data corruption detected
  - Security vulnerability in deployment
  - No clear path to fix forward within 15 minutes

Fix forward if:
  - Root cause identified and fix is ready
  - Error rate is moderate (1-3%) and improving
  - Rollback would cause its own issues (schema migration not reversible)
  
When in doubt: rollback. Time spent investigating > time to rollback + investigate.
```

---

## Chapter 74: Architecture Reviews

### Interview Question

> "How do you run an architecture review? What makes one effective vs a rubber stamp?"

---

### Why Should You Care?

Architecture reviews are the organizational mechanism for catching expensive mistakes before they're built. Done poorly, they slow teams down with bureaucratic overhead. Done well, they prevent incidents, security vulnerabilities, and rework that would cost 10x more to fix post-launch.

---

### Effective Architecture Review Framework

**Who attends**:
- The team proposing the design (engineers + tech lead)
- Staff/Principal engineers whose systems interact with or depend on this
- Security engineer (for any data or auth changes)
- Platform/infra engineer (for deployment/operational concerns)
- NOT: managers, executives, people who will rubber-stamp without reading the design doc

**What is reviewed**:

A good design doc answers:
1. What problem are we solving? (Context)
2. What are the requirements? (Functional + non-functional: SLOs, data volumes)
3. What did we consider and reject? (Alternatives)
4. What did we choose and why? (Decision + tradeoffs)
5. What does it look like? (Architecture diagram, data model, API design)
6. How does it fail? (Failure modes + mitigations)
7. How do we verify it works? (Testing, rollout plan, monitoring)

**The review process**:
- Design doc distributed 48 hours before the meeting
- Reviewers read it and add comments in advance
- Meeting: not a presentation — discussion of unresolved issues and tradeoffs
- Outcome: approved / approved with conditions / revise and re-review

**Questions that surface real issues**:
- "What happens if [service X] is down during this flow?"
- "How do we roll this back if we discover a bug in production?"
- "Where does the data live, and who can access it?"
- "What does this look like at 10x the expected load?"
- "How will we know if this is working correctly in production?"
- "What did you consider and reject? Why?"

**The rubber stamp anti-pattern**: Nobody read the design doc before the meeting. The meeting is a presentation. Everyone says "looks good." The team leaves thinking they have approval. Three months later, a fundamental issue is discovered in production.

The fix: make reading the design doc a prerequisite for attending the meeting. Reviewers who haven't read it cannot participate.

---

### Staff-Level Thinking

An architecture review is not a gate that slows teams down. It is a forcing function for *thinking carefully*, which the team should be doing anyway. The best architecture reviews happen when the team proposing the design has already done rigorous self-review — the meeting simply confirms that the thinking is sound and catches blind spots.

A staff engineer's role: ask the question the team didn't think to ask. Not to be obstructionist, but to surface the hidden assumption that will cause problems later.

---

*End of Part 11: Staff Engineer Topics*
# Final Section: System Design Interview Framework

> "The goal is not to produce the perfect design in 45 minutes. The goal is to demonstrate how a senior engineer thinks."

---

## The 10-Step Repeatable Framework

System design interviews test *how you think*, not whether you know the one correct answer. Every system design problem can be approached with the same structured framework. Practice it until it becomes automatic.

---

### Step 0: Breathe, Then Structure (2 minutes)

Before saying anything else: pause and structure your time.

```text
"I'd like to take a couple of minutes to understand the requirements before designing 
anything. Then I'll walk through the design systematically — starting with requirements, 
then high-level architecture, then drilling into the parts that matter most for this 
system. Please interrupt me at any point."
```

This signals:
- You're disciplined (won't jump to solutions immediately)
- You're collaborative (invite interruption — interviewers want dialogue)
- You're structured (they can follow your reasoning)

**Time budget for a 45-minute interview**:
```text
Requirements clarification:  5-7 minutes
High-level design:           10 minutes
Deep dives (2-3 areas):      20 minutes
Wrap-up and questions:        5 minutes
```

---

### Step 1: Clarify Functional Requirements (3 minutes)

Ask what the system must *do*. Only the core features — you don't have time for edge cases.

**Template questions**:
- "Who are the users, and what is the primary action they perform?"
- "What is the core user journey? Walk me through the happy path."
- "Which features are in scope for this design? Which are out of scope?"
- "Is this read-heavy, write-heavy, or balanced?"

**Example — Design Twitter**:
```text
You ask:
  - Core features: tweet, follow, home timeline?
  - Is DMs, search, trending in scope?
  - Do we care about media (images/video) or text-only?
  - Who are the users — individual accounts or also brands with 100M followers?

Interviewer: "Focus on tweet, follow, and home timeline. Text only. Assume celebrity 
accounts with up to 10M followers."
```

Write the functional requirements on the board/paper. Refer back to them throughout.

---

### Step 2: Clarify Non-Functional Requirements (2 minutes)

Ask about scale, latency, availability, and constraints.

**Template questions**:
- "What is the expected scale? DAU, requests/sec?"
- "What are the latency requirements? Is there an SLO?"
- "What is the availability requirement? How much downtime is acceptable?"
- "Are there any regulatory constraints? (data residency, PCI-DSS, GDPR)"
- "Is consistency critical or is eventual consistency acceptable?"

**Example — Design Twitter (continued)**:
```text
You ask:
  - "What's the scale? How many DAU?"
  - "What's the acceptable latency for home timeline load?"
  - "Is eventual consistency acceptable on the timeline?"

Interviewer: "500M DAU, 100k tweets/sec write, 1M timeline reads/sec.
             Timeline latency: p99 < 500ms. Eventual consistency is fine — 
             a 30-second delay in seeing a new tweet is acceptable."
```

These numbers drive every subsequent architectural decision. Write them down.

---

### Step 3: Capacity Estimation (3 minutes)

Back-of-envelope numbers. Not a precise calculation — an order of magnitude sense-check.

```text
Twitter example:
  Writes: 100,000 tweets/sec
    → 100k × 280 chars × 2 bytes = ~56 MB/sec raw text
    → Storage: 56 MB/sec × 86,400 sec/day = ~4.8 TB/day
    
  Reads: 1,000,000 timeline reads/sec
    → Read/Write ratio: 10:1 (read-heavy — cache is important)
    
  Storage for 10 years:
    → 4.8 TB/day × 365 days × 10 years = ~17.5 PB
    → Must use distributed storage, aggressive compression, archival tiers
    
  Fanout:
    → Average user: 200 followers
    → 100k tweets/sec × 200 followers = 20B timeline writes/sec (push model is expensive)
    → Celebrity (10M followers): 1 tweet → 10M timeline updates instantly?
    → This is the key design challenge — pull vs push fanout
```

Capacity estimation reveals the core technical challenge. In Twitter's case: celebrity fanout. This tells you where to spend your design effort.

---

### Step 4: High-Level Architecture (5 minutes)

Draw the major components. Don't optimize yet — just get the skeleton right.

```text
Client → Load Balancer → API Gateway → Services → Data Stores

Be explicit about:
  - Which components are stateless vs stateful
  - Where data lives (which database, which cache)
  - Which connections are synchronous vs asynchronous
  - Where the read path differs from the write path
```

**For Twitter**:
```text
                     ┌─────────────────────┐
Clients ──────────→  │   API Gateway       │  (auth, rate limiting, routing)
                     └──────────┬──────────┘
                                │
             ┌──────────────────┼──────────────────┐
             ▼                  ▼                  ▼
      Tweet Service      User Service       Timeline Service
      (write path)       (follow graph)     (read path)
             │                  │                  │
             ▼                  ▼                  ▼
      Tweets DB           Social Graph DB    Timeline Cache
      (append-only,        (followers,       (Redis: pre-built
       immutable)          followees)        timelines)
             │
             ▼
      Fan-out Service
      (Kafka: async
       timeline updates)
```

State your assumptions explicitly as you draw:
- "I'm assuming tweet content is immutable (no edits) — simplifies the design."
- "I'm separating read and write paths because the read load is 10x write load."
- "The timeline cache holds pre-built timelines for each user."

---

### Step 5: Data Model (3 minutes)

Define the core schemas. Focus on the primary entities and their relationships.

```text
Twitter data model:

tweets:
  id (snowflake ID — time-ordered, distributed unique)
  user_id
  content (280 chars max)
  created_at
  
users:
  id
  username
  follower_count (denormalized for fast display)
  following_count (denormalized)
  
follows:
  follower_id
  followee_id
  created_at
  INDEX: (follower_id, created_at)  — "who does user X follow?"
  INDEX: (followee_id, created_at)  — "who follows user X?"

timeline (in Redis, not DB):
  Key: timeline:{user_id}
  Value: Sorted Set of tweet_ids (score = timestamp)
  TTL: 7 days (inactive users' timelines expire, rebuilt on demand)
```

**Key IDs**: Snowflake IDs (Twitter's choice) encode timestamp — you can sort by ID to get chronological order. UUID v7 achieves the same in modern systems.

---

### Step 6: API Design (2 minutes)

Define the critical endpoints. REST or gRPC. Be specific about request/response shapes.

```text
POST /tweets
  Request: { content: "Hello world", media_ids: [] }
  Response: { tweet_id: "1234567890", created_at: "..." }
  Auth: Bearer token required
  Rate limit: 300 tweets/hour per user

GET /timeline
  Request: ?limit=20&cursor=<last_tweet_id>  (cursor-based pagination)
  Response: { tweets: [...], next_cursor: "..." }
  Auth: Bearer token required
  
POST /follows
  Request: { followee_id: "456" }
  Response: 204 No Content
  Auth: Bearer token required
```

Note cursor-based pagination (not offset) for timelines — offsets are expensive on large datasets and produce inconsistent results as new data arrives.

---

### Step 7: Deep Dive — The Core Technical Challenge (10 minutes)

Every system design problem has one or two hard problems. Spend most of your time here. For Twitter, it is **fanout on write vs fanout on read**.

```text
FANOUT ON WRITE (push model):
  When user X (200 followers) tweets:
    → Write the tweet to Tweets DB
    → For each follower: insert tweet_id into their Timeline (Redis sorted set)
    → Timeline is pre-built; reads are instant
  
  Problem: Celebrity with 10M followers tweets:
    → 10M Redis writes synchronously? Takes too long.
    → Solution: async fanout (Kafka fan-out service), but with ~30s lag for celebrities
  
  Trade-off: Fast reads, slow writes for high-follower accounts. Acceptable given 
  the 30s eventual consistency tolerance.

FANOUT ON READ (pull model):
  When user Y loads their timeline:
    → Fetch list of people Y follows
    → For each: query their recent tweets
    → Merge and sort
  
  Problem: If Y follows 1,000 people → 1,000 DB queries per timeline load
  → Too slow, too much DB load
  
  Solution: Use fanout on read ONLY for celebrity accounts (selective fanout).
  Regular users: push model.
  Celebrities: pull model (fetched and merged on read, cached per user session).

HYBRID (Twitter's actual approach):
  - Non-celebrity accounts: fanout on write (push to followers' timelines)
  - Celebrity accounts (>10k followers threshold): fanout on read
  - Timeline construction: merge pre-built timeline (non-celebrities) 
    + recent celebrity tweets (pull)
  
  This is a beautiful example of SCC lens thinking:
    State: timeline is the hot state (must be fresh, fast to read)
    Coordination: how writes propagate (async Kafka fan-out)
    Concentration: celebrity accounts are the hot key problem; hybrid model mitigates
```

Interviewers specifically want to see this kind of tradeoff reasoning. It's what separates staff engineer candidates from senior engineer candidates.

---

### Step 8: Database and Storage Choices (3 minutes)

For each data store, explain *why* you chose it, not just *what* it is.

```text
Tweets DB: Cassandra
  Why: Write-heavy (100k/sec), immutable records, time-ordered queries by user.
  Partition key: user_id
  Clustering key: tweet_id (time-ordered snowflake)
  Read pattern: SELECT * WHERE user_id = X ORDER BY tweet_id DESC LIMIT 20
  (Cassandra is optimized for this exact pattern)
  
Social Graph DB: PostgreSQL or specialized graph DB (Neo4j for complex queries)
  Why: Follower/following relationships need JOIN semantics.
  At Twitter's scale: custom in-memory graph store (FlockDB), but for this design PostgreSQL.
  
Timeline Store: Redis (Sorted Sets)
  Why: O(log N) range reads, fast write, perfect for time-ordered tweet_id lists.
  Per-user timeline: sorted set with score = timestamp.
  Read: ZREVRANGE timeline:user_id 0 19 (latest 20 tweets)
```

---

### Step 9: Reliability and Scalability (3 minutes)

Address the failure modes and scaling strategy.

```text
Single points of failure:
  - API Gateway: multiple instances behind LB ✓
  - Tweet Service: stateless, horizontally scalable ✓
  - Redis: Redis Cluster (sharded by user_id hash) ✓
  - Cassandra: inherently distributed ✓
  - Fan-out Service: Kafka consumer group; adds consumers as write rate grows ✓

Scaling bottlenecks:
  - Timeline reads: 1M/sec → Redis Cluster (3 primary shards × 333k req/sec each)
    Redis single node: 1M ops/sec → cluster with 2-3 shards is comfortable
  - Write path: 100k tweets/sec → Cassandra handles this easily (designed for write-heavy)
  - Fan-out: 100k tweets × 200 avg followers = 20M Redis writes/sec
    → Async via Kafka, consumer group scales to match throughput
    → Celebrity writes bypass fan-out (pulled on read)

Caching strategy:
  - Pre-built timelines: Redis (primary cache)
  - User profile data: Redis with 10-minute TTL
  - Hot tweets (trending): CDN + Redis
```

---

### Step 10: Monitoring and Operational Concerns (2 minutes)

Show you think about production, not just design.

```text
Key metrics to monitor:
  - Tweet write p99 latency (SLO: < 200ms)
  - Timeline read p99 latency (SLO: < 500ms)
  - Fan-out Kafka lag (growing lag = fan-out can't keep up)
  - Redis memory utilization (< 80% before considering scaling)
  - Error rate per endpoint (SLO: < 0.1%)

Operational concerns:
  - Timeline cache warm-up: on new deployment, popular timelines need rebuilding
  - Snowflake ID generation: if ID generator goes down, tweet writes fail
    → Run multiple ID generator instances; clock sync is critical
  - Celebrity threshold: detect when users cross the follower threshold
    → Background job recalculates and switches between push/pull model
```

---

## SCC Lens Applied to System Design

Apply the SCC framework (State / Coordination / Concentration) to any system design to find the hard problems quickly.

**State**: What data must persist? Who owns it? How is it accessed?
- In Twitter: tweets, user profiles, follower graph, timelines
- The timeline is the hottest state — must be pre-built and cached

**Coordination**: How do components agree on state changes? How do writes propagate?
- In Twitter: tweet write → fan-out service → timeline cache updates
- The coordination challenge is the fan-out at scale (Kafka + async)

**Concentration**: Are there hot spots? What happens under uneven load?
- In Twitter: celebrity accounts concentrate writes. 10M followers = 10M fan-out writes
- The hybrid model (push for normal, pull for celebrities) is the concentration mitigation

Every hard problem in a distributed system maps to one of these three categories.

---

## Interview Anti-Patterns to Avoid

**Jumping to solutions**: "I'd use Kafka and Redis and Cassandra." Without discussing requirements first, this sounds like technology-dropping.

**Lack of tradeoffs**: "We'll use Redis for the timeline cache." — But why Redis? Why not Memcached? What are you giving up?

**Over-designing**: Designing a system for 10B users when the requirement is 1M. Shows you can't distinguish what matters.

**Under-designing**: Designing a monolith with a single database for a 500M DAU system. Shows you're not thinking at scale.

**Not asking clarifying questions**: Making assumptions without stating them. The interviewer may correct a wrong assumption early, saving you 20 minutes of wrong-direction design.

**Solving a different problem**: Spending 20 minutes on authentication when the interesting problem is the fan-out. Show you can identify the core challenge and dive into it.

**Silence**: Thinking without narrating. Think out loud. Interviewers want to hear your reasoning, not just your conclusion.

---

## Practice Problem Library

Apply this framework to these classic problems. The IAM platform case studies build on the foundations in this handbook.

| Problem | Core Challenge | Key Concepts |
|---|---|---|
| URL Shortener | Write-through caching, redirect latency | Hashing, caching, consistent hashing |
| WhatsApp | Real-time message delivery, offline sync | WebSockets, message queues, fan-out |
| Payment System | Idempotency, consistency, fraud detection | ACID, Saga, idempotency keys |
| IAM Platform | Token validation at scale, revocation | JWT, OAuth2, Redis, OIDC |
| Netflix | Video streaming, CDN, recommendations | CDN, blob storage, collaborative filtering |
| Uber | Real-time location, matching, surge | Geospatial indexes, pub/sub, pricing |
| Rate Limiter | Distributed counting, burst tolerance | Token bucket, Redis, sliding window |
| Web Crawler | Distributed crawling, deduplication | BFS, Bloom filters, queue |
| Search Engine | Indexing, ranking, query performance | Inverted index, tf-idf, sharding |
| Distributed Cache | Consistent hashing, eviction, replication | Consistent hashing, LRU, Redis Cluster |

---

## The Staff Engineer Interview Mindset

Senior engineer interviews test: "Can you design a system?"
Staff engineer interviews test: "Can you reason about *why* a system is designed a certain way, articulate tradeoffs clearly, and adapt the design to changing constraints?"

The difference in practice:
- Senior: designs the happy path well
- Staff: identifies what can go wrong, proposes mitigations, discusses what to monitor

Questions that signal staff-level thinking:
- "I'd choose [approach] because [reason], but this comes at the cost of [tradeoff]. Given your consistency requirement, [approach] is worth that cost."
- "The critical path here is [X]. Everything else can be async."
- "Under normal load, [A] is fine. Under 10x load, [A] breaks because [reason]. At that point we'd need [B]."
- "The hot spot in this design is [X]. Here's how we'd mitigate it."

Practice narrating your tradeoff reasoning out loud. The reasoning is the interview.

---

*End of Final Section: System Design Interview Framework*

---

# Appendix: Quick Reference

## CAP / PACELC Cheat Sheet
- Network partitions are unavoidable → CP or AP is your choice
- CP: returns errors during partition (ZooKeeper, etcd, RDBMS sync replication)
- AP: serves stale data during partition (Cassandra, DynamoDB, CouchDB)
- PACELC: even without partition, choose EC (low latency) vs EC (consistency)

## Consistency Models Quick Reference
- Linearizable: behaves as single copy, highest latency
- Sequential: total order, high latency
- Causal: cause-effect preserved, medium latency
- Eventual: will converge, lowest latency

## Back-of-Envelope Constants
| Metric | Value |
|---|---|
| SSD sequential read | 500 MB/s |
| SSD random read | 100 µs |
| HDD sequential read | 100 MB/s |
| Network same DC | 0.5 ms |
| Network cross-region | 50-150 ms |
| Redis GET (in-process) | ~0.1 ms |
| Redis GET (network) | ~1 ms |
| PostgreSQL simple query | 1-5 ms |
| PostgreSQL complex query | 10-200 ms |
| HTTP connection setup | 5-15 ms |
| DNS lookup | 2-10 ms |

## Database Selection Quick Guide
| Use case | Database |
|---|---|
| Relational, ACID, complex queries | PostgreSQL |
| Relational, simple queries, high write | MySQL |
| Key-value, sub-ms latency, caching | Redis |
| High-write, time-series, wide column | Cassandra |
| Document store, flexible schema | MongoDB |
| Graph relationships | Neo4j / Amazon Neptune |
| Distributed SQL, global | CockroachDB / Spanner |
| Analytics, columnar | BigQuery / Redshift / ClickHouse |
| Search with ranking | Elasticsearch |
| Message streaming | Kafka |
| Managed queue | SQS / RabbitMQ |

## SLO / Downtime Reference
| SLO | Monthly downtime |
|---|---|
| 99% | 7.3 hours |
| 99.9% | 43.8 minutes |
| 99.95% | 21.9 minutes |
| 99.99% | 4.38 minutes |
| 99.999% | 26.3 seconds |

## Common Idempotency Patterns
- Idempotency key: UUID generated by client, stored by server, deduplicates retries
- Natural idempotency: PUT (set state), DELETE (already absent is fine), upsert
- Message deduplication: `processed_events` table, unique constraint on `message_id`
- Kafka: `enable.idempotence=true` for producer, unique key + upsert for consumer

---

*Staff Engineer System Design Fundamentals Handbook*
*— End of Document —*

---

---

# Part 12: Architecture Building Blocks

> "These patterns appear in every production system at scale. Recognising them by name, knowing their internals, and articulating their failure modes is a baseline staff engineer skill."

---

## Chapter 80: Architecture Building Blocks

### Pattern 1: API Gateway

#### What It Is

A single, managed entry point for all external client traffic. It sits at the perimeter of your system, before your internal services, and owns cross-cutting concerns that would otherwise be duplicated in every service.

```text
Internet                     Internal Network
   │                               │
   ▼                               │
┌──────────────────────────────┐   │
│         API Gateway          │───┤
│                              │   │
│  JWT validation              ├───► Payment Service
│  Rate limiting               │   │
│  Path-based routing          ├───► User Service
│  SSL/TLS termination         │   │
│  Request ID injection        ├───► Auth Service
│  Response caching            │   │
│  Circuit breaking            ├───► Notification Service
│  Request transformation      │
│  Observability injection     │
└──────────────────────────────┘
```

#### What It Does

**Authentication once**: Validate JWT tokens at the gateway. Services receive the verified identity in a trusted header (`X-Authenticated-User-Id`, `X-User-Roles`). No service reimplements token validation.

**Rate limiting**: Per API-key, per IP, per user, per endpoint. Token bucket or sliding window. Returns `429 Too Many Requests` with `Retry-After` header.

**Routing**: `/payments/*` → payment-service. `/users/*` → user-service. The topology of your services is invisible to clients.

**SSL termination**: TLS ends at the gateway. Internal traffic is plain HTTP within a trusted VPC, or mTLS between services via a service mesh.

**Request ID injection**: Every inbound request gets `X-Request-ID: <uuid>` if absent. Propagated to all downstream services. Foundation for distributed tracing.

**Circuit breaking**: If a downstream service fails, the gateway can trip a circuit and return a fast `503` instead of waiting for timeouts to cascade.

#### Implementations

| Tool | Type | Best for |
|---|---|---|
| Kong | OSS + managed, plugin-based | Feature-rich, Lua extensibility |
| AWS API Gateway | Fully managed | AWS-native, Lambda integration |
| Envoy (standalone) | Proxy | L7 power, service mesh integration |
| Nginx | Reverse proxy | High performance, simple routing |
| Traefik | Dynamic discovery | Kubernetes-native, auto-routing |

#### Failure Modes

**Single point of failure**: All traffic flows through one component. Fix: multiple gateway instances behind a cloud load balancer; health checks with fast failover.

**Security bypass**: An internal service accidentally exposed directly (Kubernetes NodePort, misconfigured firewall). Security enforced only at the gateway is bypassed. Fix: defence-in-depth — services also validate tokens independently.

**Payload transformation brittleness**: Gateway transforms request/response bodies to match legacy services. Service API changes require synchronised gateway config changes. Fix: minimise transformation logic; use it for protocol bridging only.

**Latency addition**: Auth validation, rate-limit counters, and logging all add overhead. A gateway that calls Redis synchronously for every request adds 1–5ms per hop. Profile and cache aggressively.

---

### Pattern 2: BFF (Backend for Frontend)

#### What It Is

A dedicated backend service tailored to the needs of a specific client type. Instead of one generic API trying to serve mobile, web, and partner integrations simultaneously, each client has an API shaped precisely for it.

```text
WITHOUT BFF                          WITH BFF

Mobile ──────────────────┐          Mobile ──→ Mobile BFF ──────────┐
Web ─────────────────────┼──→ API   Web ────→ Web BFF ──────────────┼──→ Microservices
Partners ────────────────┘          Partners → Partner API (v1/v2) ──┘

API: must serve all three           Each BFF: serves exactly one client
     → over-fetches for mobile          → right data, right shape
     → under-fetches for web            → right auth flow
     → unstable contracts for partners → stable versioned contract
```

#### What It Does

**Aggregation**: Mobile BFF calls Identity + Payment + Notification services and returns one response. One network round-trip from the client.

**Response shaping**: Web BFF returns rich objects. Mobile BFF strips to essential fields (battery, bandwidth).

**Client-specific auth flows**: Mobile BFF handles OAuth2 PKCE. Web BFF handles session cookies. Partner API handles client-credentials JWT.

**Versioning**: Partner API is versioned (`/v1/`, `/v2/`). Internal services evolve freely; the BFF provides a stable contract.

#### Failure Modes

**BFF sprawl**: 6 client types → 6 BFFs → business logic duplicated in 6 places. A pricing rule must be updated in 6 codebases. Fix: BFFs aggregate and transform only. Business logic lives in domain services.

**Logic leakage**: "It's easier to put this rule in the BFF." Three months later, the BFF owns 40% of the business logic. Fix: explicit BFF contract — it may call services, compose responses, and translate protocols. It may not make business decisions.

**Auth duplication**: JWT validation logic in every BFF. A security fix requires 6 deployments. Fix: delegate auth to the API gateway upstream; BFFs receive a verified identity header.

---

### Pattern 3: Service Mesh

#### What It Is

A dedicated infrastructure layer that handles all service-to-service communication. Implemented as a sidecar proxy (Envoy) injected into every pod, controlled by a centralised control plane (Istio, Linkerd, Consul Connect).

```text
WITHOUT service mesh:                WITH service mesh (Istio + Envoy):

Service A ──HTTP──→ Service B        Service A ──HTTP──→ [Envoy] ──mTLS──→ [Envoy] ──HTTP──→ Service B
                                                   ↑                  ↑
                                             (A's sidecar)      (B's sidecar)
                                             
Service A knows nothing about:
  - mTLS (Envoy handles cert rotation)
  - Circuit breaking (Envoy trips automatically)
  - Retries (Envoy retries on 503)
  - Distributed tracing (Envoy injects B3 headers)
  - Traffic splitting (canary deployments via Istio VirtualService)
  - Metrics (Envoy exposes Prometheus scrape endpoint)
```

#### Control Plane vs Data Plane

```text
Control Plane (Istio Pilot/istiod):
  - Distributes routing rules to all Envoy proxies
  - Issues mTLS certificates (SPIFFE/SPIRE)
  - Manages traffic policies (retries, timeouts, circuit breakers)
  - Aggregates telemetry

Data Plane (Envoy sidecars):
  - Intercepts all inbound/outbound traffic
  - Enforces policies from control plane
  - Reports metrics and traces
  - Handles actual packet forwarding
```

#### What a Service Mesh Provides

| Capability | Without mesh | With mesh |
|---|---|---|
| mTLS between services | Each service manages certs | Automatic, rotated every 24h |
| Circuit breaking | Code in each service | Config in VirtualService YAML |
| Retries + timeouts | Code in each service | Config in DestinationRule YAML |
| Traffic splitting (canary) | Manual + code | `weight: 90/10` in VirtualService |
| Distributed tracing | Library in each service | Envoy injects headers automatically |
| Metrics | Micrometer/Prometheus in each service | Envoy exposes per-service metrics |

#### Failure Modes

**Control plane outage**: Istiod goes down. Envoy proxies continue with last-known config (fail-open by default). New pods may not get certs. Fix: HA control plane (3 replicas); monitor control plane health.

**Sidecar resource tax**: Each Envoy sidecar uses 50–100 MB RAM. 500 pods = 25–50 GB RAM for sidecars alone. Fix: tune sidecar resource requests; exclude low-traffic pods from mesh if appropriate.

**Startup race**: Application container starts before Envoy is ready. First requests fail. Fix: `holdApplicationUntilProxyStarts: true` in Istio config, or application retry on startup.

**mTLS confusion with health checks**: Kubernetes liveness probes don't have mTLS certs. If strict mTLS is enforced, health checks fail. Fix: use `PeerAuthentication` in PERMISSIVE mode for health check ports, STRICT for all other traffic.

---

### Pattern 4: Sidecar

#### What It Is

A secondary container co-located with the primary application container in the same pod (Kubernetes) or VM. They share network namespace and can share volumes. The sidecar extends the application without modifying it.

```text
┌─────────────────────────────────────────────────┐
│  Kubernetes Pod                                 │
│                                                 │
│  ┌──────────────────┐   ┌─────────────────────┐ │
│  │  App Container   │   │  Sidecar Container  │ │
│  │  (IAM service)   │   │  (Envoy proxy)      │ │
│  │                  │   │                     │ │
│  │  Business logic  │   │  mTLS enforcement   │ │
│  │  Port 8080       │   │  Circuit breaking   │ │
│  │  Knows nothing   │   │  Retry logic        │ │
│  │  about infra     │   │  Trace propagation  │ │
│  └──────────────────┘   │  Metrics scraping   │ │
│                         └─────────────────────┘ │
│                                                 │
│  Shared: network (localhost), volumes           │
└─────────────────────────────────────────────────┘
```

#### Common Sidecar Roles

**Service mesh proxy (Envoy)**: Intercepts all traffic. Implements mTLS, circuit breaking, load balancing, distributed tracing — transparently.

**Log shipper (Fluentd/Fluent Bit)**: App writes logs to stdout or a shared volume. Sidecar tails and ships to Elasticsearch/Loki. App is unaware of the logging backend.

**Secret injector (Vault Agent)**: Authenticates to HashiCorp Vault, fetches secrets, writes to a shared tmpfs volume. App reads secrets from files. No Vault credentials in the app.

**Config syncer**: Polls a config server (Consul, Spring Cloud Config). Writes updated config to a shared volume. App watches the file and reloads hot configuration.

**Metrics exporter**: For apps that can't natively expose Prometheus metrics (legacy JARs, binaries), a sidecar scrapes the app's proprietary metrics endpoint and translates to Prometheus format.

#### Failure Modes

**Init container ordering**: App starts before secret sidecar has written credentials. App crashes on startup (missing DB password). Fix: use Kubernetes init containers to run the secret fetch before the app container starts.

**Volume mount race**: Both containers share a tmpfs volume. Secret sidecar hasn't written secrets yet when app reads them. Fix: app startup should wait and retry reading secrets.

**Sidecar crash loop**: Envoy crashes (misconfigured cert). All traffic to the pod is blocked. The pod appears "Running" but requests fail. Fix: monitor sidecar readiness separately from app readiness; sidecar restarts should page.

---

### Pattern 5: Outbox Pattern

#### What It Is

A technique for atomically writing to a database and publishing a message/event, using the database itself as the reliable intermediate store.

#### The Problem It Solves

```text
NAIVE DUAL WRITE (broken):
  BEGIN transaction
    INSERT INTO payments (...)
  COMMIT
  
  publish("PaymentCreated", payload)  ← if this fails, event is lost forever
                                        payment exists but nobody knows

OUTBOX PATTERN (correct):
  BEGIN transaction
    INSERT INTO payments (...)
    INSERT INTO outbox_events (type="PaymentCreated", payload=...)  ← same transaction
  COMMIT
  
  Background process:
    SELECT * FROM outbox_events WHERE published = FALSE
    → publish to Kafka
    UPDATE outbox_events SET published = TRUE, published_at = NOW()
```

The database transaction is the consistency boundary. Either both the payment and the outbox event are written, or neither is.

#### Implementation Details

```sql
CREATE TABLE outbox_events (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  aggregate_type VARCHAR(100) NOT NULL,   -- 'Payment'
  aggregate_id   UUID NOT NULL,
  event_type     VARCHAR(100) NOT NULL,   -- 'PaymentCreated'
  payload        JSONB NOT NULL,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  published_at   TIMESTAMPTZ,
  published      BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE INDEX idx_outbox_unpublished ON outbox_events(created_at)
  WHERE published = FALSE;
```

**Outbox processor options**:

- **Polling**: `SELECT ... WHERE published = FALSE LIMIT 100`, publish, update. Simple. Adds 100ms–5s latency depending on poll interval.
- **Debezium CDC**: Watches PostgreSQL WAL. Publishes to Kafka as WAL changes arrive. Near-real-time (~100ms). No polling overhead. Requires Kafka and Debezium infrastructure.

#### Failure Modes

**Outbox table growth**: Kafka is down for 2 hours. 2 hours × write rate of events accumulate unpublished. When Kafka recovers, the outbox processor must drain the backlog. Monitor outbox depth; alert if > N minutes of backlog.

**At-least-once delivery**: If the processor crashes after publishing but before marking as published, the event is published again on restart. Downstream consumers MUST be idempotent (deduplicate by `event_id`).

**Long outbox polling interval**: Poll every 10 seconds → events delivered with up to 10-second lag. For time-sensitive events (fraud alerts, permission revocations), reduce the interval or use CDC.

---

### Pattern 6: CDC (Change Data Capture)

#### What It Is

CDC captures every row-level insert, update, and delete from a database's transaction log and streams those changes as events — without requiring any application code changes.

```text
PostgreSQL WAL (Write-Ahead Log)
  │  Every write is recorded here before being applied
  │
  ▼
Debezium Connector
  │  Reads WAL via logical replication slot
  │  Converts row changes to structured events
  │
  ▼
Kafka Topic: db.public.identities
  │
  ├──→ Search indexer (Elasticsearch sync)
  ├──→ Cache invalidator (Redis key deletion)
  ├──→ Data warehouse (analytics pipeline)
  ├──→ Audit service (compliance log)
  └──→ Notification service (user changed → send email)
```

#### Change Event Structure (Debezium)

```json
{
  "op": "u",
  "before": { "id": "user-123", "status": "ACTIVE",    "updated_at": "2024-01-15T10:00:00Z" },
  "after":  { "id": "user-123", "status": "SUSPENDED", "updated_at": "2024-01-15T10:01:00Z" },
  "source": {
    "table": "identities",
    "db": "iam_db",
    "ts_ms": 1705312860000,
    "lsn": 12345678
  }
}
```

Operations: `c` (create/insert), `u` (update), `d` (delete), `r` (read — initial snapshot).

#### Failure Modes

**Replication slot lag causes disk exhaustion**: The Debezium consumer is down. PostgreSQL cannot clean WAL files referenced by the replication slot. Disk fills. Database goes read-only. Fix: monitor replication slot lag (`pg_replication_slots`); set `max_slot_wal_keep_size`; alert aggressively.

```sql
-- Monitor this:
SELECT slot_name, active, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn))
  AS lag_size
FROM pg_replication_slots;
```

**Schema change breaks consumer**: Adding a `NOT NULL` column without a default changes the event structure. Consumers parsing the old schema break. Fix: CDC schemas are public contracts. Use a schema registry (Confluent Schema Registry) with backward-compatibility enforcement.

**Initial snapshot blocking**: Debezium takes an initial snapshot of existing table data before streaming changes. On a 500M-row table, this snapshot can run for hours and may lock tables (depending on snapshot mode). Fix: use `snapshot.mode=schema_only` for large tables if you only care about future changes; use `exported` mode for consistent snapshot without locks.

---

### Pattern 7: Leader Election

#### What It Is

A coordination mechanism ensuring exactly one node in a cluster performs a given role or operation at a time — even as nodes fail, restart, and join.

#### Why It's Needed

```text
Scenario: 5 replicas of a payment reconciliation job.
Without leader election: all 5 run simultaneously.
  → 5 instances send the same payment to the bank
  → 5 duplicate charges
  → Financial disaster

With leader election: exactly 1 instance runs at a time.
  → If it dies: a new leader elected in < 30 seconds
  → Reconciliation resumes
```

#### How It Works (etcd/Kubernetes Pattern)

```text
All candidates attempt to acquire a lock in etcd:
  PUT /locks/reconciliation-leader <my-node-id> NX PX 30000
  (NX = only if key doesn't exist; PX 30000 = expires in 30s)

Node A wins (key didn't exist):
  Node A: is leader. Runs reconciliation. Renews lease every 10s.
  
Nodes B, C, D, E: lost. Watch the key for deletion.

Node A crashes at T=15s. Last renewal at T=10s.
  At T=40s: lease expires (30s TTL). Key deleted.
  
  Nodes B–E: detect key deletion. Race to acquire.
  Node C wins. Node C is new leader.
  Reconciliation resumes within ~30 seconds of failure.
```

**Fencing token**: Monotonically increasing epoch number from etcd. Leader passes its epoch to every write. Storage layer rejects writes with a stale epoch (from a deposed leader that doesn't know it's deposed).

```text
Node A is leader, epoch=5. Makes writes to DB: "epoch=5, debit $100"
Network partition: A isolated, C elected as new leader, epoch=6.
A's partition heals: A still thinks it's leader (epoch=5).
A tries to write: "epoch=5, debit $100"
DB: "epoch=5 < current epoch=6. Rejected."
A: realises it's not leader. Steps down.
```

#### Implementations

| Mechanism | How | TTL |
|---|---|---|
| etcd lease | `PUT key NX PX ttl` | Configurable |
| Redis `SET NX PX` | Atomic set-if-not-exists | Configurable |
| PostgreSQL advisory lock | `pg_try_advisory_lock(key)` | Session-scoped |
| Kubernetes leader election | `leaderelection` package | Configurable |
| ZooKeeper ephemeral node | Ephemeral znode creation | Session-scoped |

#### Failure Modes

**Lease too long → long unavailability**: TTL=5min. Leader dies. Next leader not elected for 5 minutes. Fix: short TTL (15–30s) with frequent renewal (every TTL/3).

**Lease too short → false failovers**: TTL=1s. A GC pause of 2 seconds looks like a crash. New leader elected. Old leader wakes up from GC and tries to act as leader. Fix: TTL must be longer than the worst-case GC pause (use G1GC or ZGC; monitor GC pause p99).

**Split-brain during network partition**: Two network segments each elect a leader. Both act simultaneously. Fix: require quorum (majority of nodes) to win election. With 5 nodes, a partition of 2 vs 3: only the 3-node side can elect a leader.

---

### Pattern 8: Service Discovery

#### What It Is

The mechanism by which services find each other in a dynamic environment where IP addresses and ports change constantly (containers, autoscaling, rolling deployments).

#### Two Models

**Server-side (Kubernetes DNS)**:
```text
Service A: HTTP GET http://payment-service:8080/payments
           ↓
           CoreDNS resolves "payment-service" → ClusterIP (virtual IP)
           ↓
           kube-proxy routes to one of the healthy payment-service pods
           ↓
           Pod receives request
```

The client knows only the service name. Everything else is infrastructure.

**Client-side (Consul/Eureka)**:
```text
Service A at startup:
  → register with Consul: "I am payment-service, at 10.0.1.45:8080"
  
Service B making a call:
  → query Consul: "Where are payment-service instances?"
  → Consul returns: [10.0.1.45:8080, 10.0.1.67:8080, 10.0.1.89:8080]
  → Service B picks one (client-side load balancing: round-robin, least-conn)
  → Service B calls 10.0.1.67:8080 directly (no intermediate proxy)
```

Client-side gives more control (load balancing algorithm, circuit breaking) but adds library complexity to every service.

#### Health Registration

Services register with health check endpoints. Discovery only returns healthy instances:

```text
Consul health check:
  GET http://10.0.1.45:8080/health every 10s
  HTTP 200: instance is healthy → included in discovery
  Timeout or non-200: instance is unhealthy → removed from discovery
  After 3 failures: deregistered
```

#### Failure Modes

**JVM DNS caching**: Java caches DNS responses for the JVM lifetime by default (`networkaddress.cache.ttl = -1`). A restarted service gets a new IP. JVM callers keep hitting the old IP. Fix: set `networkaddress.cache.ttl=5` in `$JAVA_HOME/jre/lib/security/java.security`, or use a service mesh (which bypasses JVM DNS for outbound calls).

**Health check lag**: Service crashes. Health check fires every 10s. For up to 10s, requests still route to the dead instance. Fix: aggressive health checks (5s interval, 1s timeout, 1 failure = unhealthy). Balance against false-positive deregistrations under GC pressure.

**Discovery data staleness (client-side)**: Client caches the list of instances for 30s. A new instance is added. Client doesn't see it for 30s. Fix: short cache TTL (5–10s) or use a reactive push-based registry (Consul watches, Kubernetes informers).

---

### Pattern 9: Circuit Breaker

#### What It Is

A proxy that wraps calls to a downstream service. It tracks failures and, when failure rate exceeds a threshold, "trips" — returning fast failures without calling the downstream at all. This protects the caller from cascading failure and gives the downstream time to recover.

#### State Machine

```text
         ┌─────────────────────────────┐
         │                             │
         ▼                             │
      CLOSED ──── failure rate       OPEN
    (normal)      exceeds             │
         ▲        threshold           │ timeout (e.g. 30s)
         │                            ▼
         └─────────────────────── HALF-OPEN
            success                (one probe request)
                                        │
                                        └── failure → OPEN
```

**CLOSED**: All requests pass through. Circuit tracks success/failure rate over a rolling window.

**OPEN**: All requests fail immediately (no network call made). Fast `503` returned to caller. Downstream gets quiet period to recover.

**HALF-OPEN**: After the recovery timeout, one request is allowed through. If it succeeds, circuit closes. If it fails, circuit re-opens.

#### Configuration (Resilience4j)

```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .slidingWindowType(SlidingWindowType.COUNT_BASED)
    .slidingWindowSize(20)              // evaluate last 20 calls
    .failureRateThreshold(50)           // trip if >50% fail
    .waitDurationInOpenState(Duration.ofSeconds(30))  // stay open 30s
    .permittedNumberOfCallsInHalfOpenState(3)         // 3 probe calls
    .minimumNumberOfCalls(5)            // don't trip on tiny sample
    .build();

CircuitBreaker cb = CircuitBreaker.of("fraud-service", config);

// Wrap the call:
Supplier<FraudResult> decorated = CircuitBreaker.decorateSupplier(
    cb, () -> fraudService.check(payment));

try {
    FraudResult result = decorated.get();
} catch (CallNotPermittedException e) {
    // Circuit is OPEN: return fallback immediately
    return FraudResult.defaultAllow();
}
```

#### Failure Modes

**Threshold too sensitive**: `slidingWindowSize=5, failureRateThreshold=20%` → circuit trips after 1 failure in 5. A single slow upstream GC pause trips the circuit. All downstream calls fail for 30s. Fix: minimum call count requirement; larger window.

**No fallback after trip**: Circuit opens. Service throws `CallNotPermittedException`. There's no fallback handler. User sees a 500 error. Fix: always define a fallback — stale cache, default response, or a meaningful error message — for every circuit-broken call.

**Metric collection errors**: Timeout exceptions and circuit-open exceptions are both counted as failures. Actual downstream failures get conflated with circuit-breaker-induced failures in metrics. Fix: tag metrics by exception type; distinguish `CallNotPermittedException` (circuit open) from actual downstream failures.

---

### Pattern 10: Bulkhead

#### What It Is

Resource isolation that limits the blast radius of a failing dependency. Named after the watertight compartments in a ship hull: one flooded compartment doesn't sink the ship.

#### Thread Pool Bulkhead

```text
WITHOUT bulkhead (shared thread pool: 200 threads):

  Payment Service (slow, takes 5s): acquires 150 threads, all blocked
  User Service (fast, 10ms): needs threads → only 50 available
  User Service performance degrades → timeouts cascade
  Both services effectively down, even though only Payment is slow.

WITH bulkhead (separate thread pools):

  Payment Service pool: 50 threads (capped)
  User Service pool:    50 threads (capped)
  Auth Service pool:    30 threads (capped)
  Default pool:         70 threads (other calls)

  Payment Service slow: uses its 50 threads. BLOCKED.
  User Service: unaffected, has its own 50 threads.
  Blast radius: contained to Payment Service's pool.
```

#### Semaphore Bulkhead (Reactive / Non-Blocking)

For reactive systems (WebFlux) where thread-per-request doesn't apply, use semaphores to limit concurrent calls:

```java
BulkheadConfig config = BulkheadConfig.custom()
    .maxConcurrentCalls(25)         // max 25 concurrent calls to fraud service
    .maxWaitDuration(Duration.ofMillis(100))  // wait up to 100ms for a slot
    .build();

Bulkhead bulkhead = Bulkhead.of("fraud-service", config);

Supplier<FraudResult> decorated = Bulkhead.decorateSupplier(
    bulkhead, () -> fraudService.check(payment));
```

#### Connection Pool Bulkhead

Separate database connection pools per concern:

```text
Payment writes:        pool max=20 connections to payments DB
Analytics reads:       pool max=10 connections to reporting replica
Background jobs:       pool max=5 connections to jobs DB
Admin operations:      pool max=3 connections to admin DB

A runaway analytics query doesn't consume connections needed for payment writes.
```

#### Failure Modes

**Undersized pools**: Bulkhead pool for Payment Service is 10 threads. At 500 concurrent payment requests, 490 are rejected immediately. Payment service appears broken. Fix: size bulkhead pools based on actual concurrency requirements + headroom.

**Bulkheads without monitoring**: Pool is full, requests are being rejected. No alert fires. Engineers discover during user complaint investigation. Fix: monitor `bulkhead.available_concurrent_calls` and `bulkhead.rejected_calls`; alert when rejection rate > 0.

**Shared downstream not isolated**: Thread pools are separate per-caller, but all callers share the same database connection pool. A slow caller fills the connection pool. Other callers are indirectly affected. Fix: bulkhead isolation must extend to shared resources (DB connections, Redis connections).

---

*End of Chapter 80: Architecture Building Blocks*

---

# Chapter 81: Failure Pattern Catalog

> "Staff engineers think about failure first. Senior engineers think about the happy path. The difference shows at 3 AM during an incident."

---

## Failure 1: Hot Keys

### What Causes It

A single cache key or database row receives a disproportionate share of traffic. All requests for that key concentrate on one node, exhausting its capacity.

```text
Redis Cluster (3 shards):
  Shard 1: key "top_products"  → 180,000 req/sec  ← SATURATED
  Shard 2: distributed keys    →  10,000 req/sec
  Shard 3: distributed keys    →   8,000 req/sec

Why: "top_products" is read on every page load.
     Its hash lands on shard 1.
     Shard 1 CPU: 100%. GET latency: 80ms (should be <1ms).
```

Common triggers:
- Celebrity accounts (social media: one user's profile queried by millions)
- Trending products (Black Friday sale item)
- Auth tokens for shared service accounts
- Single configuration key queried by every service instance

### How to Detect It

```bash
# Redis: find hot keys
redis-cli --hotkeys   # requires maxmemory-policy = allkeys-lfu or volatile-lfu

# Redis: manual keyspace analysis
redis-cli monitor | head -10000 | awk '{print $4}' | sort | uniq -c | sort -rn | head -20

# Prometheus:
redis_keyspace_hits_total by (key)  # if key-level metrics enabled
```

Application-level: log the cache key for every cache operation; aggregate by key in Kibana/Loki; identify outliers.

### How to Fix It

**Local in-process cache (L1 cache)**:
```java
// Caffeine in-process cache in front of Redis
LoadingCache<String, ProductList> l1 = Caffeine.newBuilder()
    .maximumSize(100)
    .expireAfterWrite(5, TimeUnit.SECONDS)  // very short TTL for hot keys
    .build(key -> redis.get(key));          // L2 miss → fetch from Redis

// Result: Redis traffic for "top_products" drops from 50k/sec to 10/sec
//         (one request per JVM instance per 5s, not one per request)
```

**Key duplication (replication)**:
```text
Instead of one key "top_products":
  Shard the key: "top_products:shard:0" through "top_products:shard:9"
  On write: write all 10 shards
  On read: pick random shard (round-robin or random)
  
  Traffic: 180,000 req/sec across 10 shards = 18,000/shard
  Each shard can now handle it.
```

**Read-through with jitter**: Add a small random delay before reading the key. Under burst load, the reads spread across slightly different moments, reducing simultaneous thundering.

---

## Failure 2: Hot Partitions

### What Causes It

A Kafka partition or database shard receives a disproportionate share of traffic because the partition key is poorly chosen — concentrating events from one high-volume entity into one partition.

```text
Kafka topic: payment-events
Partition key: merchant_id

Partition 0: merchant "BigRetailer" (2M events/day)  → 120,000 events/sec at peak
Partition 1: 500 small merchants                      →   3,000 events/sec
Partition 2: 300 small merchants                      →   2,500 events/sec

Partition 0's single consumer: overwhelmed, processing lag grows.
BigRetailer's payments: 45-second delay.
Small merchants: <1 second.
```

### How to Detect It

```bash
# Kafka: consumer group lag per partition
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group payment-processor

# Output:
# PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# 0          1234567         9999999         8765432  ← massive lag
# 1          456789          458000          1211
# 2          234567          235000          433

# Prometheus:
kafka_consumergroup_lag{partition="0"}  # one partition >> others
```

### How to Fix It

**Add entropy to partition key**:
```java
// Instead of: key = merchantId
// Use: key = merchantId + "_" + (System.currentTimeMillis() % 10)
// Spreads one merchant across 10 partitions

String partitionKey = merchantId + "_" + (epochSecond % 10);
producer.send(new ProducerRecord<>(topic, partitionKey, payload));

// Consumer: aggregate across the 10 sub-partitions for this merchant
// Ordering within a merchant is lost (within a sub-partition it's preserved)
// For most analytics/audit use cases: acceptable
```

**Increase partition count**: More partitions = more parallelism. But: changing partition count reorders existing keys (hash(key) % N changes). Plan for this at topic creation.

**Separate hot entities into dedicated topics**:
```text
topic: payment-events-bigretailer    (dedicated consumer group, more consumers)
topic: payment-events-standard       (regular consumer group)

Routing at producer: if merchant is in "VIP" list → VIP topic
```

---

## Failure 3: Cache Stampede

### What Causes It

A popular cached item expires. Many concurrent requests simultaneously find a cache miss. All race to recompute the value from the database simultaneously.

```text
Timeline:
  T=0: "top_products" key expires (TTL reached)
  T=0.001: 1000 concurrent requests all see cache miss
  T=0.001: 1000 requests all query the DB for top_products
  T=0.001: DB: 1000 concurrent identical queries → CPU 100%
  T=0.500: DB starts returning slow/timeouts
  T=1.0: Top-products page shows errors for all users
  T=1.5: First DB query completes, cache is repopulated
  T=1.5: All subsequent requests served from cache
  
  1500ms of failure, self-inflicted, from one TTL expiry.
```

### How to Detect It

```text
Prometheus:
  spike in: db_query_duration_seconds{query="get_top_products"}
  spike in: cache_misses_total{key="top_products"}
  
Both spikes appear simultaneously at regular intervals (every TTL seconds).
Pattern: clock-like periodic database load spike.
```

### How to Fix It

**Mutex (single-filler) lock**:
```java
String value = cache.get("top_products");
if (value == null) {
    String lockKey = "lock:top_products";
    boolean acquired = redis.set(lockKey, "1", "NX", "PX", "5000"); // 5s lock
    if (acquired) {
        try {
            value = db.query("SELECT ...");
            cache.set("top_products", value, 60);  // repopulate
        } finally {
            redis.del(lockKey);
        }
    } else {
        // Wait for winner to populate cache
        Thread.sleep(50);
        value = cache.get("top_products");  // retry
        if (value == null) value = db.query("SELECT ...");  // fallback
    }
}
```

**Stale-while-revalidate**:
```java
// Cache stores value + "soft TTL" + "hard TTL"
// Soft TTL: return stale and trigger background refresh
// Hard TTL: value must be refreshed (never return after this)

CacheEntry entry = cache.get("top_products");
if (entry == null || entry.isPastHardTtl()) {
    entry = db.query(...);
    cache.set("top_products", entry, hardTtl=120, softTtl=60);
} else if (entry.isPastSoftTtl()) {
    // Return stale immediately, refresh in background
    executor.submit(() -> refreshCache("top_products"));
}
return entry.value;  // fast, even if slightly stale
```

**TTL jitter**: `TTL = base_ttl + random(0, base_ttl * 0.2)`. Spreads expiries so multiple concurrent misses on the same key can't all hit at T=0.

---

## Failure 4: Retry Storm

### What Causes It

A downstream service becomes overloaded or unhealthy. Callers retry aggressively. The additional retry traffic increases load on the struggling service, making it worse, causing more retries. Positive feedback loop.

```text
Normal:    API → Fraud Service: 1000 req/sec. Fraud handles it.

Incident:  Fraud Service DB has a slow query. Response time: 5s (was 50ms).

Without retry control:
  1000 original req/sec are now each waiting 5s.
  Each times out, retries 3 times (naive retry).
  
  Fraud Service now receives: 1000 × 3 retries = 3000 req/sec.
  Fraud DB: 3× normal query load → even slower → 10s responses.
  
  Callers retry more: 1000 × 3 × 3 = 9000 req/sec to Fraud.
  Fraud collapses. All payments fail.
  
Root cause: one slow DB query. Amplified to a total outage by retries.
```

### How to Detect It

```text
Metrics:
  - Spike in outbound_request_total{target="fraud-service", retry="true"}
  - fraud-service: inbound_request_total spikes 3-10× above normal
  - fraud-service: CPU and DB connections spike simultaneously
  - Pattern: fraud-service's metrics spike AFTER the initial incident
    (the retry amplification arrives slightly delayed)
```

### How to Fix It

**Exponential backoff with full jitter**:
```java
int attempt = 0;
while (attempt < MAX_RETRIES) {
    try {
        return fraudService.check(payment);
    } catch (ServiceUnavailableException e) {
        attempt++;
        if (attempt >= MAX_RETRIES) throw e;
        
        long baseDelay = 100L * (1L << attempt);  // 200, 400, 800ms
        long jitter = ThreadLocalRandom.current().nextLong(0, baseDelay);
        Thread.sleep(baseDelay + jitter);  // full jitter: random(0, base*2^n)
    }
}
```

**Retry budget**: Total retries allowed per time window — not per request.
```java
// Token bucket: 100 retry tokens per second (refilled at 100/sec)
// Each retry consumes one token.
// When budget exhausted: fail fast, no retry.
// Prevents: 10,000 concurrent requests × 3 retries = 30,000 retries/sec
```

**Circuit breaker + retry composition**: Retry up to N times, then open circuit if failure rate threshold exceeded. Retries stop entirely. Recovery window for downstream.

---

## Failure 5: Split Brain

### What Causes It

A network partition causes two nodes to simultaneously believe they are the sole leader (or primary database). Both accept writes. When the partition heals, two divergent histories must be reconciled — often losing data.

```text
3-node PostgreSQL (Patroni) cluster. Node A = primary.

Network partition at T=0:
  Segment 1: Node A (isolated)
  Segment 2: Nodes B and C

Node A: "I can't reach B and C, but I'm still primary. Accepting writes."
Nodes B+C: "A is unreachable for 30s. Electing new primary."
Node B elected: "I'm primary. Accepting writes."

T=0 to T=5min:
  Some clients (reached A): writes to A's timeline
  Other clients (reached B): writes to B's timeline
  
  Account balance: $1000 at T=0
  A timeline: debit $200 → $800
  B timeline: debit $300 → $700
  
  Reality: user has been debited $500. Balance should be $500.
  A says $800. B says $700. Neither is correct.

Partition heals at T=5min:
  Patroni must choose one winner. Loser's writes discarded.
  If B wins: A's $200 debit is lost. User credited $200 they shouldn't have.
  Financial data corrupted.
```

### How to Detect It

```text
Patroni: 
  patronictl -c /etc/patroni.yml list
  → Two nodes show as "Leader". That is split-brain.

Metrics:
  Multiple nodes reporting write_queries_total > 0 simultaneously
  Replication lag metric showing no replication happening (partitioned)
  
Alerts:
  Alert if two or more DB nodes simultaneously report primary status.
  Alert if replication lag > 60s (potential partition forming).
```

### How to Fix It

**STONITH (Shoot The Other Node In The Head)**: When a new primary is elected, the old primary is forcibly powered off or fenced before the new one starts accepting writes. Even if the old primary doesn't know it's deposed, it's fenced.

**Quorum writes**: Write is only committed when acknowledged by a majority (N/2 + 1) of nodes. A partitioned node with no quorum cannot commit. Even if it accepts the write locally, it can't commit it to the quorum log.

**Fencing tokens**: Every write includes a monotonically increasing epoch from the consensus system. Storage rejects writes with a stale epoch. A deposed primary's writes are rejected by the storage layer, even if it doesn't know it's deposed.

**Minimise the partition window**: Short Patroni TTL (15–30s), fast health checks, automated fencing. The shorter the partition duration, the less data diverges.

---

## Failure 6: Poison Messages

### What Causes It

A message in a queue is malformed, triggers a bug in the consumer, or violates a business invariant. The consumer fails every time it processes the message. The message is redelivered. The consumer fails again. Infinite loop. All messages after this one are blocked.

```text
Kafka partition 0: [msg1, msg2, POISON_MSG, msg4, msg5, ...]

Consumer:
  Process msg1 → success, commit offset 1
  Process msg2 → success, commit offset 2
  Process POISON_MSG → NullPointerException → crash
  
  Pod restarts (Kubernetes).
  Consumer re-reads from last committed offset (2).
  Process POISON_MSG → NullPointerException → crash
  
  Repeat every 30 seconds (pod restart interval).
  msg4, msg5, ... never processed.
  Consumer group lag for partition 0 grows unboundedly.
  Alert fires at T+10min. Engineers paged.
```

### How to Detect It

```text
# Kafka consumer lag growing on one partition, not others:
kafka-consumer-groups.sh --describe --group payment-processor
# PARTITION  LAG
# 0          0
# 1          0
# 2          847293   ← growing, others stable

# Kubernetes: pod crash-looping
kubectl get pods -n payments
# payment-consumer-abc123  CrashLoopBackOff  15  (45m)

# Logs: same exception on every restart, referencing the same offset
```

### How to Fix It

**Dead Letter Queue (DLQ)**:
```java
@KafkaListener(topics = "payment-events")
public void consume(ConsumerRecord<String, String> record) {
    try {
        paymentService.process(record.value());
    } catch (Exception e) {
        int retryCount = getRetryCount(record);
        if (retryCount >= MAX_RETRIES) {
            // Give up: send to DLQ
            dlqProducer.send(new ProducerRecord<>("payment-events-dlq", 
                record.key(), record.value()));
            // Commit offset: don't reprocess
        } else {
            incrementRetryCount(record);
            throw e;  // Retry
        }
    }
}
```

**Defensive consumer**: Never let an individual message crash the consumer. All exceptions caught, logged, and handled:
```java
try {
    process(record);
} catch (Exception e) {
    log.error("Failed to process record key={} offset={}", 
        record.key(), record.offset(), e);
    sendToDlq(record);
    // DO NOT rethrow — do not crash the consumer
}
```

**Schema validation at publish time**: Validate event schema before publishing (JSON Schema, Avro with schema registry). Prevents malformed events entering the queue.

**DLQ monitoring**: Alert when DLQ has any messages. A message in the DLQ is always a problem requiring human attention.

---

## Failure 7: Thundering Herd

### What Causes It

A large number of processes simultaneously wake up and compete for the same resource. Each process is behaving correctly; the failure is emergent.

```text
Scenario A: Cache TTL expiry
  500 API nodes. All started together (batch deployment).
  Each caches "config" key with TTL=60min.
  At T=60min: all 500 caches expire simultaneously.
  500 nodes simultaneously query DB for "config".
  DB: 500 × 20 connections = overwhelmed.

Scenario B: Cron job
  50 worker nodes. All run a reconciliation job at 00:00.
  All 50 simultaneously acquire DB connections, run heavy queries.
  DB saturated for 2 minutes at midnight.

Scenario C: Cache invalidation fan-out
  A popular product price changes.
  CDN edge nodes receive cache invalidation.
  1000 edge nodes simultaneously request the new price from origin.
  Origin: 1000 simultaneous requests for the same resource.
```

### How to Detect It

```text
Pattern: periodic, sharp load spikes at predictable intervals.
  - DB CPU spikes every 60 minutes (cache TTL)
  - DB spikes at :00 of every hour (cron alignment)
  - Origin spikes correlating with CDN invalidation events

Prometheus:
  db_connections_active: sharp sawtooth pattern
  cache_misses_total: spike coinciding with DB spike
```

### How to Fix It

**TTL jitter**: `TTL = base + random(0, base * 0.2)`. Spreads expiries over 20% of the base TTL window.

**Staggered cron scheduling**: Instead of all workers at `00 * * * *`, use:
```
Node 0:  00 * * * *   (00 seconds)
Node 1:  12 * * * *   (12 seconds)
Node 2:  24 * * * *   (24 seconds)
...
```
Or: use a job queue (Kafka, SQS) with one producer publishing the job, workers compete to pull — natural load distribution.

**Request coalescing at origin**: When 1000 CDN nodes request the same URL simultaneously, origin serves one computation and broadcasts to all waiting requesters. Implemented in Nginx (`proxy_cache_lock on`).

**Pre-warming**: Before nodes go live after a deployment, pre-populate their caches from a snapshot source. They never start cold.

---

## Failure 8: Deadlock

### What Causes It

Two or more transactions each hold a lock the other needs. Neither can proceed. Both wait indefinitely.

```text
Transaction T1:                  Transaction T2:
  LOCK account A (exclusive)       LOCK account B (exclusive)
  ... (processing) ...             ... (processing) ...
  Try to LOCK account B            Try to LOCK account A
  → WAITING (B held by T2)         → WAITING (A held by T1)
  
  T1 waits for T2.
  T2 waits for T1.
  Neither makes progress.
  Both hold their locks.
  
PostgreSQL detects after deadlock_timeout (default 1s):
  ERROR: deadlock detected
  DETAIL: Process 1234 waits for ShareLock on transaction 5678;
          blocked by process 5678.
          Process 5678 waits for ShareLock on transaction 1234;
          blocked by process 1234.
  HINT: See server log for query details.
```

### How to Detect It

```sql
-- PostgreSQL: active deadlocks
SELECT pid, query, state, wait_event_type, wait_event
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';

-- PostgreSQL logs: search for "deadlock detected"
grep "deadlock detected" /var/log/postgresql/postgresql.log

-- Application metrics:
deadlock_errors_total counter (from exception type in application logs)
```

### How to Fix It

**Consistent lock ordering**: Always acquire locks in the same order across all transactions. If T1 and T2 both always lock A before B, deadlocks are impossible.

```java
// WRONG: T1 locks A then B; T2 locks B then A → deadlock possible
// RIGHT: always lock by account ID ascending
List<UUID> accountIds = Arrays.asList(accountA, accountB);
Collections.sort(accountIds);  // canonical order
for (UUID id : accountIds) {
    db.execute("SELECT * FROM accounts WHERE id = ? FOR UPDATE", id);
}
```

**Reduce lock scope**: Hold locks for the minimum time. Don't do external calls (HTTP, file I/O) while holding a DB lock.

**Optimistic locking**: Instead of locking rows, add a `version` column. Read the row, check the version, update only if version unchanged. Retry if version mismatch. No locks held = no deadlocks.

```sql
UPDATE accounts
SET balance = balance - 100, version = version + 1
WHERE id = ? AND version = ?;
-- 0 rows updated → version changed, retry
```

**`SELECT FOR UPDATE SKIP LOCKED`**: Competing consumers pattern. Skip locked rows rather than waiting.

```sql
SELECT * FROM payment_queue
WHERE status = 'PENDING'
LIMIT 1
FOR UPDATE SKIP LOCKED;
-- Gets a row no other consumer is processing; never blocks
```

---

## Failure 9: Cascading Failure

### What Causes It

A failure in one component causes dependent components to fail, propagating through the system until a large portion of the platform is down — even though only one component was the original cause. Each component's failure is individually "correct" behaviour.

```text
Initial condition: Fraud DB has a slow query (IO spike)

Step 1: Fraud Service response time degrades: 50ms → 5s
Step 2: Payment Service calls Fraud Service; threads block waiting (5s timeout)
Step 3: Payment Service thread pool fills with blocked threads
Step 4: New payment requests queue; queue fills; requests are rejected
Step 5: Load balancer routes to other Payment Service pods
Step 6: Other Payment pods also call Fraud Service → they also fill up
Step 7: All Payment pods saturated → 100% of payments failing
Step 8: Mobile app retries → retry storm → more load on all services
Step 9: Retry storm reaches User Service (payments rejected → app retries user auth)
Step 10: User Service thread pool also fills → logins fail

Root cause: One slow DB query on Fraud Service.
Final state: Entire platform (payments + logins) down.
Time elapsed: 90 seconds.
Each component behaved "correctly" at each step.
```

### How to Detect It

```text
Pattern: Multiple unrelated services degrading simultaneously.
         Alert storm across many services at once.
         Incident timeline shows failure propagating outward from one service.
         
Metrics:
  error_rate_total across multiple services spikes simultaneously
  thread_pool_active_threads → all pools simultaneously saturating
  
Key indicator: the first service to show degradation is the root cause.
               All others are victims.
```

### How to Fix It

The mitigations form a defence-in-depth stack:

**Timeouts** (first line): Fraud Service call times out in 500ms, not 5s. Threads freed promptly. Blocking window is 10× smaller.

**Circuit breaker** (second line): After 50% of Fraud Service calls fail, circuit opens. Calls fail instantly. No threads consumed waiting. Recovery period for Fraud Service.

**Bulkheads** (third line): Fraud Service calls use a dedicated thread pool (30 threads). Even if all 30 are blocked, User Service and other operations have their own pools.

**Load shedding** (fourth line): When Payment Service queue depth exceeds threshold, new requests are rejected with `503` immediately — before they queue, before they consume threads.

**Fallback** (fifth line): Fraud Service unavailable → apply conservative local rules ("allow payments under $100 from known devices; reject others"). Degraded mode, not failed mode.

Together, these prevent a single component failure from propagating.

---

*End of Chapter 81: Failure Pattern Catalog*

---

# Chapter 82: Technology Selection Matrix

> "Most candidates know what technologies exist. Staff engineers know when to choose them — and when not to."

---

## The Selection Discipline

Technology selection is a tradeoff decision, not a knowledge quiz. For every choice, answer:
1. What specific requirement drives this selection?
2. What does this technology give up?
3. What is the simpler alternative, and why is it insufficient?

The following matrices encode these answers.

---

## Storage Selection

| Requirement | Technology | Why | What you give up |
|---|---|---|---|
| ACID transactions, relational data, strong consistency | **PostgreSQL** | Battle-tested, rich feature set, excellent tooling | Horizontal write scaling |
| High write throughput (>100k writes/sec), eventual consistency acceptable | **Cassandra** | Designed for write-heavy distributed workloads | Joins, transactions, strong consistency |
| Sub-millisecond reads, caching, ephemeral state | **Redis** | In-memory, nanosecond latency | Durability (configurable), not a primary store |
| Full-text search, relevance ranking, facets | **Elasticsearch / OpenSearch** | Inverted index, near-real-time search | Transactional consistency; not a primary store |
| Flexible schema, document-oriented, embedded objects | **MongoDB** | Schema flexibility, rich query language | Strong consistency by default, join support |
| Time-series data (metrics, IoT, telemetry) | **TimescaleDB** or **InfluxDB** | Columnar compression for time-series, time-based queries | General-purpose query flexibility |
| Graph relationships, traversal queries | **Neo4j** | Native graph storage, Cypher query language | Not for OLTP volumes, operational overhead |
| Object storage, files, backups, data lake | **S3 / GCS / Azure Blob** | Infinite scale, low cost, durability | Not queryable directly (use Athena/BigQuery) |
| Columnar analytics, large aggregations | **ClickHouse** | Extremely fast OLAP queries, columnar storage | Not for transactional writes |
| Multi-region ACID transactions | **CockroachDB / Spanner** | Distributed SQL with strong consistency | Higher latency (quorum across regions), cost |
| General OLAP, managed, petabyte scale | **BigQuery / Redshift / Snowflake** | Serverless, fast analytics, no ops | Cost at scale, not for OLTP |

**Default rule**: Start with PostgreSQL. Move to a specialised store only when you hit a demonstrated limitation. The cost of early specialisation (dual writes, sync, ops complexity) almost always exceeds the cost of running PostgreSQL a bit longer.

---

## Caching Selection

| Requirement | Technology | Notes |
|---|---|---|
| Distributed cache, sessions, rate limiting | **Redis** | Default cache choice. Cluster for HA. |
| In-process, low-latency, same JVM | **Caffeine (Java)** | L1 cache; use before Redis for hot keys |
| Static assets, geographic distribution | **CDN (CloudFront, Fastly)** | Edge caching; mandatory for global apps |
| Pre-computed aggregations, relational | **PostgreSQL Materialized Views** | Refreshed on schedule or on-trigger |
| Write-through cache (always consistent) | **Redis + write-through pattern** | Writes go to DB + cache atomically |
| Eventually consistent read model | **Redis + CDC/outbox** | Updated asynchronously from DB changes |

---

## Messaging Selection

| Requirement | Technology | Why | What you give up |
|---|---|---|---|
| Event streaming, replay, multiple consumer groups | **Kafka** | Durable log, replay, high throughput, fan-out | Operational complexity, not for RPC |
| Task queues, work distribution, simple async | **RabbitMQ / SQS** | Simple, reliable, push-based delivery | Event replay, log semantics |
| Cloud-native streaming (no Kafka ops) | **AWS Kinesis / GCP Pub/Sub** | Managed, auto-scaling | Less control, vendor lock-in |
| In-process events (same JVM) | **Spring ApplicationEvent / Guava EventBus** | Zero overhead, no infra | Not durable, not distributed |
| Low-volume async within PostgreSQL | **LISTEN/NOTIFY** | No additional infra | Not durable after notify, limited throughput |

**The Kafka vs RabbitMQ question in one sentence**: Do consumers need to replay past events? If yes: Kafka. If no: RabbitMQ is simpler and cheaper to operate.

---

## Communication Protocol Selection

| Requirement | Protocol | Notes |
|---|---|---|
| Public API, broad compatibility | **REST/HTTP** | Universal client support; cacheable |
| Service-to-service, strong typing, streaming | **gRPC** | Binary efficiency, code generation, bidirectional streaming |
| Server pushing updates to browser | **SSE** | Simpler than WebSockets for unidirectional |
| Bidirectional real-time, browser client | **WebSockets** | Full duplex; chat, collaboration, gaming |
| Peer-to-peer audio/video | **WebRTC** | Browser P2P, STUN/TURN, encrypted |
| Flexible client-driven queries | **GraphQL** | Multiple client types with different data needs |
| Internal async processing | **Kafka / SQS** | Not a protocol — an architecture choice |

---

## Consistency Model Selection

| Requirement | Model | Implementation |
|---|---|---|
| Financial transactions (no data loss, no duplicates) | **Strong consistency** | PostgreSQL + synchronous replication, quorum reads |
| Social features (likes, follower counts, feed) | **Eventual consistency** | Cassandra, Redis with async replication |
| User sees their own writes immediately | **Read-your-writes** | Sticky reads to primary, or session-consistent routing |
| User never sees data going backwards | **Monotonic reads** | Route all reads for a session to same replica |
| Distributed lock, leader election | **Linearisable** | etcd, ZooKeeper, Redis Redlock |
| Multi-region with conflicting writes | **CRDT or LWW** | CRDTs for counters/sets; LWW for last-value semantics |

---

## Architecture Pattern Selection

| Situation | Pattern | Notes |
|---|---|---|
| Small team (<15), evolving domain | **Modular Monolith** | Clear internal module APIs; single deploy |
| Multiple teams, stable domain boundaries | **Microservices** | One team per service; Conway's Law alignment |
| Multiple client types with different data needs | **BFF** | One backend per client type |
| Migrating legacy system incrementally | **Strangler Fig** | Route traffic progressively to new service |
| Read-heavy, query model ≠ write model | **CQRS** | Separate write and read databases |
| Audit trail required, full history | **Event Sourcing** | Append-only event log; derived state |
| Distributed transactions across services | **Saga** | Choreography or orchestration; compensations |
| Loose coupling, async business workflows | **Event-Driven** | Domain events via Kafka; decoupled consumers |
| Need mTLS, circuit breaking, observability without code changes | **Service Mesh** | Istio/Linkerd + Envoy sidecars |

---

## Database Decision Tree

```text
START: What is the primary access pattern?

├── Structured data, complex relationships, transactions?
│   ├── Single region, team knows SQL?
│   │   └── PostgreSQL (default choice)
│   ├── Need to scale writes globally?
│   │   └── CockroachDB / Spanner (pay latency cost)
│   └── Massive write throughput, can relax consistency?
│       └── Cassandra (design queries first)
│
├── Key-value, <1ms reads, ephemeral or cache?
│   └── Redis
│       ├── Persistence needed? → Enable AOF/RDB
│       └── High availability? → Redis Cluster or Sentinel
│
├── Full-text search, relevance, facets?
│   └── Elasticsearch — alongside a primary DB, not instead of one
│
├── Document store, flexible/nested schema?
│   └── MongoDB (watch: transactions are expensive, consistency is eventual by default)
│
├── Time-series (metrics, IoT, events by time)?
│   └── TimescaleDB (PostgreSQL extension) or InfluxDB
│
├── Graph traversal (social network, permissions graph)?
│   └── Neo4j or Amazon Neptune
│
├── Analytics, large aggregations, columnar?
│   ├── Self-hosted, high QPS analytics?
│   │   └── ClickHouse
│   └── Managed, petabyte-scale, batch?
│       └── BigQuery / Redshift / Snowflake
│
└── Object storage (files, images, backups)?
    └── S3 (AWS) / GCS (GCP) / Azure Blob
```

---

## Consistency Decision Tree

```text
Does this operation involve money, security, or inventory?
│
├── YES → Strong consistency required
│   │
│   ├── Single region?
│   │   └── PostgreSQL with synchronous commit
│   │
│   └── Multi-region?
│       ├── Low write volume? → CockroachDB / Spanner
│       └── High write volume? → CP system per region + cross-region saga
│
└── NO → Evaluate staleness tolerance
    │
    ├── Can users see their own writes immediately?
    │   └── YES → Read-your-writes consistency
    │            (route user reads to primary after their write)
    │
    ├── Can users see data appear to go backwards? (e.g., 150 likes → 140 likes)
    │   └── NO → Monotonic reads
    │            (route session reads to same replica)
    │
    └── Neither → Eventual consistency acceptable
                  Cassandra, DynamoDB, Redis replication
```

---

## Real-Time Communication Decision Tree

```text
Does the client need to SEND data to the server in real-time?
│
├── YES (bidirectional) → WebSockets
│   (chat, collaborative editing, multiplayer gaming)
│
└── NO (server → client only)
    │
    ├── Browser client?
    │   ├── Native browser EventSource API sufficient? → SSE
    │   └── Need binary data or more control? → WebSockets
    │
    ├── Update frequency low (>1 second interval)?
    │   └── Long Polling (simpler, works behind all proxies)
    │
    └── Peer-to-peer audio/video?
        └── WebRTC (P2P, STUN/TURN, browser-native)
```

---

*End of Chapter 82: Technology Selection Matrix*

---

# Chapter 83: Capacity Estimation — From Numbers to Architecture

> "The goal of capacity estimation is not the exact number. It is identifying which dimension will break first, and what architectural implication that has."

---

## The Five-Step Method

1. **Identify the scale inputs**: DAU, actions per user, data size per action
2. **Calculate QPS**: requests per second (average and peak)
3. **Calculate storage**: per day, per year, with replication and overhead
4. **Calculate bandwidth**: inbound and outbound bytes per second
5. **Identify the binding constraint**: which resource (CPU, storage, bandwidth, connections) hits its limit first

The binding constraint determines the architecture. Storage-bound → archival strategy. Bandwidth-bound → CDN. Connection-bound → connection pooler. CPU-bound → horizontal scaling.

---

## Constants to Memorise

### Time
```text
1 minute   =    60 seconds
1 hour     =  3,600 seconds
1 day      = 86,400 seconds  ← memorise this
1 month    =  2,592,000 seconds (~2.6M)
1 year     = 31,536,000 seconds (~31.5M)
```

### Storage
```text
1 KB  = 10³ bytes   = 1,000
1 MB  = 10⁶ bytes   = 1,000,000
1 GB  = 10⁹ bytes   = 1,000,000,000
1 TB  = 10¹²bytes
1 PB  = 10¹⁵ bytes

Typical sizes:
  UUID (string):         36 bytes
  UUID (binary):         16 bytes
  Typical DB row:       100–500 bytes
  JSON API response:     1–10 KB
  Profile photo:        50–200 KB (compressed)
  1 min SD video:        ~5 MB
  1 min HD video:       ~50 MB
  1 hour HD video:       ~3 GB
```

### Latency Reference
```text
In-process cache (Caffeine):    ~50 ns
Redis GET (local network):      ~0.5 ms
PostgreSQL simple query:        ~1–5 ms
PostgreSQL with 1 join:         ~5–20 ms
Cross-AZ network round-trip:    ~1 ms
Cross-region round-trip:        50–200 ms
SSD random read:                ~0.1 ms
HDD random read:                ~10 ms
```

---

## QPS Estimation

### Formula
```text
Average QPS = DAU × actions_per_user_per_day ÷ 86,400
Peak QPS    = Average QPS × peak_multiplier

Peak multiplier:
  Uniform traffic (B2B, batch):          1.5×
  Business-hours traffic (enterprise):   3×
  Consumer with spikes (social, retail):  5–10×
```

### Worked Examples

**Example 1: Instagram-scale photo feed**
```text
DAU: 500M
Actions/user/day: feed scroll (10 views), upload (0.1 photos), like (5 actions) = 15.1 actions
Average QPS: 500,000,000 × 15.1 ÷ 86,400 = 87,384 QPS ≈ 87K QPS
Peak QPS (5×): 437K QPS

→ Implication: 437K QPS requires a large stateless API tier (100+ nodes at 4K QPS each),
  Redis cluster for sessions/feed cache, CDN for photo delivery.
```

**Example 2: IAM token introspection**
```text
DAU: 20M active users
API calls/user/day: 50 (mobile app checking in)
Each API call = 1 token introspection

Average introspections: 20,000,000 × 50 ÷ 86,400 = 11,574 QPS ≈ 12K QPS
Peak (3×): 36K QPS

Redis single node: ~1M ops/sec capacity
→ 36K QPS: single Redis node with headroom. No cluster needed yet.
→ At 200M DAU: 360K QPS → Redis Cluster (3 shards).
```

**Example 3: Payments**
```text
Daily transactions: 5M
Average transaction = 3 API calls (initiate, process, confirm)
Write QPS: 5,000,000 × 3 ÷ 86,400 = 174 QPS writes (low)
Read QPS (balance checks, history): 10× writes = 1,740 QPS
Peak (5×): 870 write QPS, 8,700 read QPS

→ Implication: PostgreSQL single primary easily handles 870 writes/sec.
  Read replicas for 8,700 read QPS (or Redis cache for balance lookups).
  No sharding needed at this scale.
```

---

## Storage Estimation

### Formula
```text
Daily writes (rows): write_QPS × 86,400
Row size: sum of column sizes + overhead (indexes ~20%, TOAST ~variable)
Daily raw storage: daily_writes × row_size
Annual storage: daily × 365
With replication (3×): × 3
With compression (~0.5×): × 0.5
Net annual: daily × 365 × 3 × 0.5
```

### Worked Examples

**Example 1: Audit events (IAM)**
```text
Write rate: 1,000 events/sec
Daily events: 1,000 × 86,400 = 86.4M events/day

Row size:
  id (UUID):            16 bytes
  event_type (varchar): 30 bytes
  actor_id (UUID):      16 bytes
  target_id (UUID):     16 bytes
  payload (JSONB):     200 bytes avg
  ip_address:           16 bytes
  created_at:            8 bytes
  Total:               ~302 bytes ≈ 300 bytes/row

Daily raw: 86.4M × 300 = 25.9 GB/day
Compressed (50%): 13 GB/day
With 3× replication: 39 GB/day

Annual: 39 × 365 = 14.2 TB/year (with replication)

Retention: 7 years (regulatory)
Total: 14.2 TB × 7 = 99.4 TB ≈ 100 TB

→ Implication: Cannot keep 7 years in PostgreSQL OLTP tables.
  Strategy: PostgreSQL (last 90 days, hot), S3 Parquet (90 days – 7 years, cold).
  Query compliance data via Athena (SQL on S3).
```

**Example 2: User profiles**
```text
Total users: 100M
Profile row size: 500 bytes (id, name, email, address, metadata JSONB)

Total storage: 100M × 500 = 50 GB
With 3× replication: 150 GB
With indexes (30% overhead): 195 GB

→ Implication: Single PostgreSQL instance (r7g.4xlarge has 128 GB RAM).
  Entire dataset fits in memory → no disk reads for hot profiles.
  No sharding needed.

Cache sizing:
  20% of profiles = 80% of reads (Pareto)
  20% of 100M = 20M profiles
  20M × 500 bytes = 10 GB hot dataset
  Redis memory overhead (~2.5×): 25 GB
  → One Redis node (r7g.xlarge = 32 GB RAM) handles the hot 20%.
```

**Example 3: Video storage (YouTube-scale)**
```text
Uploads: 500 hours of video/minute (YouTube figure)
1 hour HD video ≈ 1.5 GB (compressed H.264, 720p)
Upload rate: 500 × 1.5 GB/min = 750 GB/min = 12.5 GB/sec

Daily storage added: 12.5 GB/sec × 86,400 = 1.08 PB/day

→ Implication: S3 or distributed object storage mandatory.
  Transcoding pipeline needed (1 upload → 5 resolution variants).
  CDN is not optional — serving 1.08 PB/day of new content from origin is impossible.
```

---

## Bandwidth Estimation

### Formula
```text
Inbound bandwidth  = write_QPS × avg_request_size
Outbound bandwidth = read_QPS  × avg_response_size

Key thresholds:
  1 Gbps NIC:     125 MB/sec
  10 Gbps NIC:  1,250 MB/sec
  25 Gbps NIC:  3,125 MB/sec
```

### Worked Example

**Social feed API**
```text
Read QPS: 100,000/sec
Average response: 20 posts × 500 bytes/post = 10 KB

Outbound bandwidth: 100,000 × 10,000 bytes = 1 GB/sec

→ Implication:
  Single 1Gbps NIC: saturated. Need 10Gbps or multiple servers.
  AWS data transfer: $0.09/GB × 1 GB/sec × 86,400 sec/day = $7,776/day = $233K/month.
  
  CDN with 95% cache hit rate:
    Origin handles: 1 GB/sec × 5% = 50 MB/sec ← manageable
    CDN egress: cheaper ($0.01–0.04/GB vs $0.09)
    Cost: dramatically reduced

→ CDN is a financial necessity, not a performance optimisation.
```

---

## Cache Sizing

### Formula
```text
Cache size needed = hot_dataset_size × safety_multiplier

Hot dataset: the % of data that accounts for 80% of reads.
For most systems: 20% of data = 80% of reads (Pareto).

Safety multiplier: account for Redis overhead (~2–3× raw data size).
```

### Worked Example

**IAM permission cache**
```text
Total user-role assignments: 5M records
Average serialised size: 200 bytes/record
Total dataset: 5M × 200 = 1 GB (tiny)

→ Cache entire dataset: 1 GB × 3 (Redis overhead) = 3 GB
  One small Redis instance (r7g.medium = 8 GB) caches everything.
  Permission check: O(1) Redis lookup, no DB query.
  Cache invalidation: on role grant/revoke, delete user's permission key.

At 50M user-role assignments:
  Dataset: 50M × 200 = 10 GB
  Redis: 30 GB → r7g.xlarge (32 GB) or Redis Cluster (3 × 16 GB).
```

---

## Kafka Sizing

### Formula
```text
Partitions = max(
  target_throughput_MB/sec ÷ producer_throughput_per_partition,  [typically 10 MB/sec]
  target_throughput_MB/sec ÷ consumer_throughput_per_partition   [typically 50 MB/sec]
)

Add 50% headroom. Round up to a power of 2.

Consumers = partitions (max 1 consumer per partition in a group)

Brokers:
  Total partition replicas = partitions × replication_factor
  Partition replicas per broker ≤ 4,000 (rule of thumb)
  Brokers = ceil(total_replicas / 4000) + 1 (for safety)
```

### Worked Example

**Payment events (IAM audit)**
```text
Target: 50,000 events/sec
Event size: 300 bytes
Throughput: 50,000 × 300 = 15 MB/sec

Partitions (producer-bound at 10 MB/sec per partition):
  15 MB/sec ÷ 10 MB/sec = 1.5 → round up → 2
  With 50% headroom: 3 → round to power of 2 → 4 partitions

Consumers: max 4 concurrent consumers per consumer group
Consumer throughput check: 4 × 50 MB/sec = 200 MB/sec >> 15 MB/sec ✓

Brokers:
  Partition replicas = 4 × 3 (replication factor) = 12 replicas
  3 brokers: 4 replicas each → well within the 4,000 limit

Storage per broker:
  15 MB/sec × 86,400 sec/day × 7 day retention ÷ 3 brokers = 302 GB/broker
  → 3 brokers with 500 GB SSD each.

Result: 3-broker Kafka cluster, 4 partitions, replication factor 3.
```

---

## DB Connection Pool Sizing

### Formula (Little's Law)
```text
Concurrent DB connections = QPS × average_query_duration_seconds

If payment service has:
  Write QPS: 500/sec
  Average write duration: 10ms = 0.010s
  Concurrent connections: 500 × 0.010 = 5 connections

For read QPS 5,000/sec, avg 5ms:
  5,000 × 0.005 = 25 connections

Total: ~30 connections for this service.

With 10 replicas of the service: 30 × 10 = 300 connections to PostgreSQL.
PostgreSQL sweet spot: 100–300 connections.
→ PgBouncer needed if >300.

PgBouncer:
  App nodes: 10 × 30 app connections = 300 connections to PgBouncer
  PgBouncer → PostgreSQL: 50 server connections
  PostgreSQL handles 50 concurrent queries → well within capacity.
```

---

## Full Worked Example: IAM Platform at Scale

```text
Input:
  50M active users
  5,000 logins/sec peak
  500,000 token introspections/sec peak
  1,000 admin operations/sec

Step 1 — QPS
  Login:           5,000 req/sec (given)
  Introspection: 500,000 req/sec (given)
  Admin:           1,000 req/sec (given)
  Audit events:  506,000 events/sec (sum of above operations)

Step 2 — Storage (annual)
  User records:  50M × 500 bytes = 25 GB (trivial)
  Audit events:  506,000/sec × 300 bytes = 152 MB/sec
                 × 86,400 = 13.1 TB/day
                 × 365 = 4.8 PB/year
                 → Cannot keep in PostgreSQL. S3 + Parquet for compliance.
                 → PostgreSQL: last 90 days only = 13.1 TB × 90 = 1.2 TB

Step 3 — Bandwidth
  Login responses: 5,000 × 2KB = 10 MB/sec (trivial)
  Introspection:   500,000 × 500 bytes = 250 MB/sec outbound
  → Needs 10Gbps NICs or load-balanced across nodes

Step 4 — Binding constraint
  For login: DB is binding (credential lookup).
    500ms per login × 5,000 concurrent = 2,500 concurrent DB queries
    → PgBouncer + 3 PostgreSQL read replicas for credential reads

  For introspection: Redis is binding (token lookup).
    500,000/sec → Redis Cluster (3 shards × ~167K ops/sec each, 1M ops/sec capacity)
    
  For audit: Storage is binding.
    4.8 PB/year → S3, not PostgreSQL

Step 5 — Architecture
  Stateless API tier:      10 nodes (login) + 20 nodes (introspection) + 3 (admin)
  Token cache:             Redis Cluster, 3 primary shards + replicas
  Credential store:        PostgreSQL, 1 primary + 3 read replicas + PgBouncer
  Audit pipeline:          Kafka (4 partitions) → Debezium → S3 Parquet
  Hot audit queries:       PostgreSQL (last 90 days partition)
  Cold compliance queries: Athena on S3 Parquet

Monthly infra estimate:
  API nodes (33 × c7g.xlarge):         $5,000
  PostgreSQL (r7g.4xlarge × 4):        $8,000
  Redis Cluster (r7g.large × 6):       $1,500
  Kafka (MSK, 3 brokers):              $  800
  S3 (4.8 PB/year ≈ 400 GB/month new): $ 10/month growing
  Total:                               ~$15,300/month
```

---

*End of Chapter 83: Capacity Estimation — From Numbers to Architecture*

---

# Part 13: Data Modeling in Depth

> "Most architecture failures are data modeling failures. Fix the model; the architecture simplifies."

---

## Chapter 84: Domain Modeling and Ubiquitous Language

### What Domain Modeling Is

Domain modeling is the discipline of creating a software representation of a business domain that the code, the database, and the conversations all share. It is the translation layer between "what the business does" and "what the code does."

When the translation is good, engineers and domain experts speak the same language. When it is bad, the code says `UserAccount` where the business says `Customer Identity`, and the engineer can never be sure they're solving the right problem.

### Ubiquitous Language

Every concept in the domain must have a single, precise name agreed upon by engineers and domain experts. That name appears:
- In class and interface names
- In method names
- In database column and table names
- In API endpoint paths and request/response field names
- In Kafka topic names and event payload field names
- In team conversations, tickets, and documentation

```text
MISALIGNED (engineering-domain language split):

Domain expert says: "An Identity must be provisioned before it can be granted an Entitlement."
Engineer hears:     "A User must be created before a Role can be assigned."

Code:               createUser(), assignRole(), UserRoleJunction table
DB column:          user_id, role_id

6 months later:
  Domain expert: "When an Identity is deprovisioned, all Entitlements must be revoked."
  Engineer:      "When a User is deleted? Or status=INACTIVE? What about UserRoleJunction?"
  
  The language gap has become a bug risk.

ALIGNED (ubiquitous language):

Code:               provisionIdentity(), grantEntitlement(), EntitlementGrant table
DB column:          identity_id, entitlement_id, granted_at, revoked_at

Engineer + domain expert: both say "provision," "entitlement," "revoke."
Code is self-documenting. Less ambiguity. Fewer bugs from misunderstanding.
```

### Building the Ubiquitous Language

**Step 1: Event storming**. Write every domain event on a sticky note. Put them on a timeline. Events are named in past tense: `IdentityProvisioned`, `EntitlementGranted`, `SessionExpired`, `PaymentProcessed`.

**Step 2: Identify the commands**. What triggers each event? `ProvisionIdentity` → `IdentityProvisioned`. `GrantEntitlement` → `EntitlementGranted`.

**Step 3: Name the actors**. Who sends each command? `Administrator` sends `ProvisionIdentity`. `System` sends `SessionExpired`.

**Step 4: Extract the nouns**. What objects do commands operate on? `Identity`, `Entitlement`, `Session`, `Payment`. These become your entities.

**Step 5: Write a glossary**. Literal document: term → definition → synonyms that are NOT used in code.

```markdown
## Glossary

**Identity**: A principal that can authenticate and be authorised.
  Do not use: User, Account, Principal, Subject (except in OIDC context where "sub" is standard)

**Entitlement**: A named capability that an Identity may hold.
  Do not use: Permission, Role, Right, Privilege (Role is a grouping of Entitlements)

**Provisioning**: The act of creating an Identity and preparing it for use.
  Do not use: Registration, Creation, Onboarding
  
**Deprovisioning**: The act of permanently revoking an Identity.
  Do not use: Deletion, Deactivation (Suspension is temporary; Deprovisioning is permanent)
```

---

## Chapter 85: Aggregates and Consistency Boundaries

### The Problem Aggregates Solve

In a system with many interconnected entities, it is easy to write code that modifies multiple entities inconsistently:

```java
// No aggregate boundary: anything can modify anything
identity.setStatus("ACTIVE");
credential.setHash(newHash);
roles.add(new RoleGrant(identityId, roleId));
auditLog.append(new AuditEvent(...));
// Are these 4 writes atomic? What invariants are enforced? Who checks?
```

Aggregates define a boundary within which consistency is guaranteed. Operations enter through the root. The root enforces invariants.

### Defining an Aggregate

An **aggregate** is a cluster of entities and value objects with:
- One **aggregate root** (the entry point)
- A **consistency boundary** (all members are consistent after any operation)
- A set of **invariants** (rules that must always hold within the boundary)

```text
IAM Aggregate: Identity

  Aggregate Root: Identity
    - enforces: "A SUSPENDED identity cannot be granted new entitlements"
    - enforces: "An ACTIVE identity must have at least one valid credential"
    - enforces: "Entitlement grants cannot be created without a valid grantor identity"
  
  Members:
    - Credentials (list): password, TOTP, FIDO2 keys
    - EntitlementGrants (list): role assignments with grants/revocations
    - ContactInfo (value object): email, phone
  
  Operations (all through the root):
    identity.suspend(reason, actorId)
    identity.grantEntitlement(entitlementId, grantedBy, expiresAt)
    identity.revokeEntitlement(entitlementId, revokedBy)
    identity.changePassword(newHash, changedBy)
```

### Implementation Pattern

```java
public class Identity {  // Aggregate Root
    private UUID id;
    private IdentityStatus status;
    private List<Credential> credentials;        // owned by this aggregate
    private List<EntitlementGrant> grants;       // owned by this aggregate
    
    // Invariant enforcement in the root:
    public void grantEntitlement(UUID entitlementId, UUID grantedBy, Instant expiresAt) {
        if (this.status == IdentityStatus.SUSPENDED) {
            throw new DomainException("Cannot grant entitlement to suspended identity");
        }
        if (grants.stream().anyMatch(g -> g.entitlementId().equals(entitlementId) && g.isActive())) {
            throw new DomainException("Entitlement already granted");
        }
        grants.add(new EntitlementGrant(entitlementId, grantedBy, Instant.now(), expiresAt));
        registerEvent(new EntitlementGranted(this.id, entitlementId, grantedBy));
    }
    
    public void suspend(String reason, UUID actorId) {
        if (this.status == IdentityStatus.DEPROVISIONED) {
            throw new DomainException("Cannot suspend a deprovisioned identity");
        }
        this.status = IdentityStatus.SUSPENDED;
        // Implicit: grants remain but cannot be used (auth checks identity status)
        registerEvent(new IdentitySuspended(this.id, reason, actorId));
    }
}
```

### The "One Aggregate Per Transaction" Rule

Each transaction should span at most one aggregate. If a use case seems to require updating two aggregates atomically, there are three options:

1. **Merge them**: Are they actually the same aggregate? (Only if they share invariants that must hold atomically.)
2. **Use eventual consistency**: Update one, emit an event, the other aggregate updates asynchronously. Accept temporary inconsistency.
3. **Use a Saga**: Explicit multi-step transaction with compensating actions.

```text
Use case: "Revoking a role from an identity must also invalidate all their active sessions."

Option A (same transaction): Identity aggregate updates grants + sessions.
  → Sessions are not part of the Identity aggregate; they have independent lifecycle.
  → This would make Identity too large and sessions tightly coupled to identity.

Option B (eventual consistency):
  T1: Identity.revokeEntitlement() → emits EntitlementRevoked event
  Background: SessionService subscribes to EntitlementRevoked
               → invalidates sessions for that identity (within ~1 second)
  
  During the ~1 second window: sessions are technically valid but the grant is revoked.
  For most IAM use cases: acceptable (tokens have short TTL anyway).
  
Option C (Saga): explicit, with compensation if session invalidation fails.

→ Best answer: Option B (eventual consistency). Simpler, more scalable, acceptable risk window.
  Document the consistency window as a known design decision.
```

---

## Chapter 86: Bounded Contexts in Practice

### What a Bounded Context Is

A bounded context is a region of the system where a specific domain model applies with internal consistency. The same word may mean different things in different bounded contexts. Each context has its own:
- Ubiquitous language
- Data model
- Team ownership
- Database (ideally)

```text
"Account" in a banking system:

Identity Context:
  Account = a record of who someone is (username, credentials, status)
  Team: IAM team
  DB: iam_db
  
Ledger Context:
  Account = a financial record (balance, account number, currency, owner)
  Team: Payments team
  DB: payments_db
  
Notification Context:
  Account = a delivery record (email, phone, notification preferences)
  Team: Notifications team
  DB: notifications_db

All three exist simultaneously. None is "the" Account.
Merging them into one God Account table would be a mistake.
```

### Context Mapping

When bounded contexts interact, they need integration patterns:

**Shared Kernel**: Two contexts share a portion of the model. Changes require agreement from both teams. Use sparingly — it creates coupling.

**Customer/Supplier**: Upstream context provides data; downstream context consumes it. The downstream team can request changes from the upstream team.

**Anti-Corruption Layer (ACL)**: The downstream context translates the upstream model into its own model at the boundary. The downstream's internal model stays clean.

```text
Identity Context → (Integration Event: IdentityProvisioned) → Ledger Context

Ledger Context has an ACL:
  class IdentityProvisionedHandler:
    def handle(event: IdentityProvisioned):
      # Translate from Identity model to Ledger model
      account = Account(
        owner_identity_id = event.identity_id,  # reference only; not embedding
        account_number = generate_account_number(),
        status = AccountStatus.PENDING_KYC,
        currency = "INR"
      )
      account_repo.save(account)
      
The Ledger Context doesn't "know about" Identities beyond their ID.
It translates at the boundary and works in its own model inside.
```

**Published Language**: A well-documented, versioned event schema that all contexts consume. The upstream context owns and versions it. Downstream contexts conform or translate via ACL.

### Context Map for IAM Platform

```text
┌─────────────────────────────────────────────────────────────────┐
│                    IAM Platform Contexts                        │
│                                                                 │
│  ┌───────────────┐    Published Language    ┌────────────────┐  │
│  │   Identity    │──── IdentityProvisioned ─►│ Authorisation  │  │
│  │   Context     │──── IdentitySuspended ──►│  Context       │  │
│  │               │                          │                │  │
│  │  identity_db  │                          │ authorisation_db│  │
│  └───────┬───────┘                          └────────┬───────┘  │
│          │                                           │          │
│          │ Published Language                        │          │
│          ▼                                           ▼          │
│  ┌───────────────┐                          ┌────────────────┐  │
│  │    Session    │                          │     Audit      │  │
│  │    Context    │                          │    Context     │  │
│  │               │                          │                │  │
│  │  session_db   │                          │   audit_db     │  │
│  └───────────────┘                          │  (append-only) │  │
│                                             └────────────────┘  │
└─────────────────────────────────────────────────────────────────┘

Audit Context: pure consumer. Subscribes to all integration events.
               Never emits. Never blocks any other context.
```

---

## Chapter 87: Event Modeling

### What Event Modeling Is

Event Modeling is a technique for designing systems by focusing on what *happens* (events) rather than what *exists* (entities). It creates a shared, visual understanding of system behaviour before writing any code or schema.

The output: a timeline of events, the commands that cause them, and the read models built from them.

### The Four Building Blocks

**Events** (orange sticky notes): Things that happened. Past tense. Immutable facts.
```text
IdentityProvisioned
CredentialSet
LoginAttempted
LoginSucceeded
LoginFailed
RoleGranted
SessionCreated
PasswordChanged
AccountLocked
RoleRevoked
IdentityDeprovisioned
```

**Commands** (blue sticky notes): Intentions to change state. Sent by users or systems. Can fail.
```text
ProvisionIdentity → IdentityProvisioned
SetCredential     → CredentialSet
Login             → LoginSucceeded | LoginFailed
GrantRole         → RoleGranted
RevokeRole        → RoleRevoked
```

**Read Models** (green sticky notes): Views materialised from events for user interfaces.
```text
"Active Sessions View": built from SessionCreated + SessionExpired
"User Role Summary":    built from RoleGranted + RoleRevoked
"Login History":        built from LoginSucceeded + LoginFailed
```

**Policies** (purple sticky notes): Automated reactions to events.
```text
When LoginFailed (5th time in 1 hour) → trigger LockAccount command
When SessionExpired → update "Active Sessions View"
When RoleRevoked → trigger InvalidateSessions command
```

### Event Modeling vs Data Modeling

```text
ENTITY-FIRST APPROACH (traditional):
  Draw entities (User, Role, Session, AuditEvent)
  Define relationships (User has many Roles, User has many Sessions)
  Derive tables
  
  Problem: doesn't capture how state changes over time.
  "User has status=SUSPENDED" doesn't tell you why, when, or by whom.

EVENT-FIRST APPROACH (event modeling):
  Draw the timeline of events
  Derive the commands that cause them
  Derive the read models needed to display current state
  Schema falls out naturally
  
  Result: write model captures history.
           read model is optimised for display.
           Commands are explicit and named.
```

### From Events to Schema

```sql
-- Events (write model: immutable history)
CREATE TABLE identity_events (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  identity_id  UUID NOT NULL,
  event_type   VARCHAR(100) NOT NULL,  -- 'IdentityProvisioned', 'IdentitySuspended', ...
  payload      JSONB NOT NULL,
  actor_id     UUID,                   -- who caused this event
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Read model: current state (derived from events, kept in sync via trigger or CDC)
CREATE TABLE identity_current_state (
  identity_id   UUID PRIMARY KEY,
  display_name  TEXT NOT NULL,
  status        VARCHAR(20) NOT NULL,
  last_login_at TIMESTAMPTZ,
  active_roles  JSONB,                 -- denormalised for fast display
  updated_at    TIMESTAMPTZ NOT NULL
);
```

The event table is the source of truth. The current state table is a cache — it can be rebuilt from the events at any time.

---

## Chapter 88: Relational Modeling Deep Dive

### Access-Pattern-First Design

Never start with the domain entities. Start with the queries. Let the queries drive the schema.

```text
DOMAIN-FIRST (wrong approach):
  "I have Users. Users have Roles. Roles have Permissions."
  → Users table, Roles table, Permissions table, UserRoles junction, RolePermissions junction
  → Query: "Does this user have permission X?" requires 3 joins.
  → At 500k checks/sec: catastrophically slow.

ACCESS-PATTERN-FIRST (right approach):
  "The critical query is: given a user_id and permission_name, is this allowed?"
  
  Option A: Materialise flattened permissions
    CREATE TABLE user_effective_permissions (
      user_id    UUID,
      permission VARCHAR(100),
      PRIMARY KEY (user_id, permission)
    );
    Query: SELECT 1 FROM user_effective_permissions WHERE user_id=? AND permission=?
    → O(1) lookup. No joins.
    Trade-off: must update when roles change (maintain via trigger or CDC).
  
  Option B: Cache in Redis
    Key: "permissions:{user_id}" → SET of permission strings
    → O(1) Redis lookup. No DB query.
    Trade-off: TTL-based staleness or explicit invalidation on role change.
```

### Modelling Time

Two timestamps every record should have:

```sql
-- business_time: when the event happened in the real world
-- system_time:   when it was recorded in the database

CREATE TABLE payments (
  id             UUID PRIMARY KEY,
  amount         NUMERIC(18,2) NOT NULL,
  payment_time   TIMESTAMPTZ NOT NULL,  -- when the payment actually occurred
  recorded_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),  -- when we recorded it
  ...
);
```

`payment_time` is used for financial reporting ("all payments in March"). `recorded_at` is used for data pipeline processing ("records added since last run"). They differ when payments arrive out-of-order (offline mobile, delayed batch imports).

### Modelling State Changes as Events

```sql
-- WRONG: mutable status column loses history
UPDATE identities SET status = 'SUSPENDED', updated_at = NOW() WHERE id = ?;
-- Was it ever ACTIVE? Who suspended it? When exactly?

-- RIGHT: append-only status history
CREATE TABLE identity_status_history (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  identity_id UUID NOT NULL REFERENCES identities(id),
  old_status  VARCHAR(20),
  new_status  VARCHAR(20) NOT NULL,
  changed_by  UUID REFERENCES identities(id),
  reason      TEXT,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Keep current_status on identities for fast reads (denormalised)
-- Update both atomically in a transaction:
BEGIN;
  INSERT INTO identity_status_history (identity_id, old_status, new_status, changed_by, reason)
    SELECT id, status, 'SUSPENDED', :actor_id, :reason FROM identities WHERE id = :id;
  UPDATE identities SET status = 'SUSPENDED', updated_at = NOW() WHERE id = :id;
COMMIT;
```

### Referential Integrity as Documentation

```sql
-- FK constraints communicate intent AND enforce it:
ALTER TABLE identity_roles
  ADD CONSTRAINT fk_identity FOREIGN KEY (identity_id) REFERENCES identities(id)
  ON DELETE RESTRICT;  -- explicit: don't delete identities that have role assignments
  -- ON DELETE CASCADE would silently remove role assignments — dangerous for audit

-- This constraint documents: role assignments are tied to identity lifecycle.
-- A developer reading the schema immediately understands: identities can't be
-- hard-deleted while role assignments exist.
```

### Schema Evolution Without Downtime

```sql
-- Adding a column safely (PostgreSQL 11+):
ALTER TABLE identities ADD COLUMN phone_number VARCHAR(20);
-- Default NULL: no table rewrite, no lock. Existing rows get NULL.

-- Adding a NOT NULL column requires care:
-- Step 1: Add nullable with default
ALTER TABLE identities ADD COLUMN mfa_required BOOLEAN DEFAULT FALSE;
-- Step 2: Backfill (background, no lock)
UPDATE identities SET mfa_required = FALSE WHERE mfa_required IS NULL;
-- Step 3: Add NOT NULL constraint (acquires brief lock to verify no NULLs)
ALTER TABLE identities ALTER COLUMN mfa_required SET NOT NULL;

-- Renaming a column without downtime:
-- Step 1: Add new column
ALTER TABLE identities ADD COLUMN display_name VARCHAR(255);
-- Step 2: Backfill
UPDATE identities SET display_name = name;
-- Step 3: Update application to write to both columns
-- Step 4: Remove old column after all code uses new name
ALTER TABLE identities DROP COLUMN name;
```

---

## Chapter 89: NoSQL Modeling Patterns

### The Fundamental Rule

In relational modeling: normalise for correctness, denormalise for performance. In NoSQL: **design the table for the query**. The query comes first. The table is built to serve it.

### DynamoDB Single-Table Design

All entities in one table. The partition key (PK) and sort key (SK) encode the entity type and relationship.

```text
IAM Platform: Single-Table DynamoDB

PK                  | SK                      | Attributes
─────────────────── | ─────────────────────── | ──────────────────────────────────
IDENTITY#abc-123    | PROFILE                 | { displayName, status, createdAt }
IDENTITY#abc-123    | CREDENTIAL#PASSWORD     | { hash, lastChangedAt }
IDENTITY#abc-123    | CREDENTIAL#TOTP         | { secret, enrolledAt }
IDENTITY#abc-123    | GRANT#ROLE#admin        | { grantedAt, grantedBy, expiresAt }
IDENTITY#abc-123    | GRANT#ROLE#payment_user | { grantedAt, grantedBy }
IDENTITY#abc-123    | SESSION#sess-456        | { createdAt, expiresAt, ipAddress }
ROLE#admin          | METADATA                | { description, isSystemRole }
ROLE#admin          | PERMISSION#read:all     | { addedAt }

Access patterns:
  Get identity profile:    PK=IDENTITY#abc-123, SK=PROFILE → O(1)
  Get identity credentials: PK=IDENTITY#abc-123, SK begins_with CREDENTIAL# → O(log n)
  Get identity grants:     PK=IDENTITY#abc-123, SK begins_with GRANT# → O(log n)
  Get identity sessions:   PK=IDENTITY#abc-123, SK begins_with SESSION# → O(log n)
  Get role permissions:    PK=ROLE#admin, SK begins_with PERMISSION# → O(log n)

All O(log n) key lookups. No joins. No scans.
```

**GSI for reverse lookups** (who has this role?):

```text
GSI: ROLE_MEMBERS
  GSI_PK = SK field value starting with "GRANT#ROLE#"
  GSI_SK = PK field value

Row: PK=IDENTITY#abc-123, SK=GRANT#ROLE#admin
GSI index: GSI_PK=GRANT#ROLE#admin, GSI_SK=IDENTITY#abc-123

Query: all identities with role "admin"
  GSI query: GSI_PK = GRANT#ROLE#admin
  Returns: [IDENTITY#abc-123, IDENTITY#def-456, ...]
```

### Cassandra: Query-First Design

Every Cassandra table is designed for exactly one query pattern. Multiple tables for the same data is normal and expected.

```sql
-- Query 1: "Get all payments for a user, recent first, paginated"
CREATE TABLE payments_by_user (
  user_id      UUID,
  created_at   TIMESTAMP,
  payment_id   UUID,
  amount       DECIMAL,
  status       TEXT,
  merchant_id  UUID,
  PRIMARY KEY (user_id, created_at, payment_id)
) WITH CLUSTERING ORDER BY (created_at DESC, payment_id ASC);

-- Query 2: "Get all payments for a merchant, recent first"
CREATE TABLE payments_by_merchant (
  merchant_id  UUID,
  created_at   TIMESTAMP,
  payment_id   UUID,
  amount       DECIMAL,
  user_id      UUID,
  PRIMARY KEY (merchant_id, created_at, payment_id)
) WITH CLUSTERING ORDER BY (created_at DESC);

-- Same payment data stored twice, optimised for different queries.
-- This is the Cassandra way. Disk is cheap. Scans are not.
```

**Cassandra anti-patterns**:

`ALLOW FILTERING`: Forces a full partition scan. Equivalent to a table scan in SQL. Never use in production.

Unbounded partitions: A partition key with a high-volume entity (one merchant, millions of rows) creates a hot partition and an unbounded row size. Add a bucket: `PRIMARY KEY ((merchant_id, bucket), created_at)` where `bucket = YEAR_MONTH`.

---

## Chapter 90: CQRS and Read/Write Model Separation

### The Core Insight

The model that is optimal for writing data is rarely optimal for reading it. Normalization serves writes. Denormalization serves reads. Rather than compromising one for the other, maintain two models explicitly.

```text
Write model (normalised, command-oriented):

  identities:          { id, display_name, status }
  credentials:         { id, identity_id, type, hash }
  identity_roles:      { identity_id, role_id, granted_at, revoked_at }
  roles:               { id, name, description }

  Optimised for: integrity, consistency, atomic updates.
  Query pattern: one table at a time, by primary key.

Read model (denormalised, query-oriented):

  identity_dashboard_view:
    { identity_id, display_name, status, active_roles: JSON[], 
      last_login_at, active_session_count }
  
  Optimised for: the admin dashboard query.
  Query pattern: SELECT * WHERE identity_id = ? → one row, all data needed.
  No joins. Sub-millisecond response.
```

### Keeping Models in Sync

```text
Write model update → triggers read model update:

Option A: Database trigger
  AFTER UPDATE ON identity_roles:
    UPDATE identity_dashboard_view
    SET active_roles = (SELECT json_agg(role_name) FROM ... WHERE identity_id = NEW.identity_id)
    WHERE identity_id = NEW.identity_id;
  Pro: immediate consistency.
  Con: trigger logic is hard to test; adds latency to writes.

Option B: Application-level dual write
  Service writes to identity_roles (write model)
  Service also updates identity_dashboard_view (read model)
  Pro: testable, explicit.
  Con: dual-write risk (one fails, models diverge).

Option C: CDC + async updater (preferred at scale)
  Debezium watches identity_roles table changes → Kafka
  Read model updater consumes Kafka → updates identity_dashboard_view
  Pro: decoupled, scalable, retryable.
  Con: eventual consistency (seconds of lag between write and read model).
  Use case: admin dashboard can tolerate 2-second lag. Acceptable.
```

### CQRS in the IAM Platform

```text
Commands (write path):
  ProvisionIdentity    → Identity Service → identity_db (write model)
  GrantRole            → Auth Service    → authorisation_db (write model)
  RevokeRole           → Auth Service    → authorisation_db (write model)
  
  Each command: one aggregate, one database, ACID transaction.

Queries (read path):
  GetIdentityDashboard → read from identity_read_db (read model)
  GetActiveRoles       → read from Redis (cached read model)
  GetLoginHistory      → read from audit_db (append-only, no joins needed)
  
  Each query: one read model, optimised index, no joins.

Sync:
  CDC (Debezium): identity_db → Kafka → identity_read_db updater
  Lag: ~1-3 seconds
  Acceptable: admin dashboard showing data 2 seconds old is fine.
```

---

## Chapter 91: Common Data Modeling Mistakes

### Mistake 1: The God Table

One table with 100+ columns, a `type` discriminator, and many nullable columns that only apply to certain types.

```sql
-- GOD TABLE (anti-pattern):
CREATE TABLE entities (
  id            UUID PRIMARY KEY,
  entity_type   VARCHAR(50),  -- 'USER', 'SERVICE_ACCOUNT', 'GROUP', 'ROLE'
  
  -- User-specific (null for others):
  email         VARCHAR(255),
  password_hash TEXT,
  last_login_at TIMESTAMPTZ,
  
  -- Service account-specific (null for others):
  api_key_hash  TEXT,
  service_name  VARCHAR(100),
  
  -- Group-specific (null for others):
  member_count  INTEGER,
  
  -- Role-specific (null for others):
  permission_set JSONB,
  
  created_at    TIMESTAMPTZ DEFAULT NOW()
);

-- Problems:
-- 1. Every query hits every row even for unrelated types
-- 2. No NOT NULL constraints possible (columns nullable for other types)
-- 3. "What does this column mean for type=GROUP?" → read the code
-- 4. Adding a type requires altering the table (touching all rows)
```

```sql
-- RIGHT APPROACH: separate tables per entity type
CREATE TABLE identities (
  id          UUID PRIMARY KEY,
  type        VARCHAR(20) NOT NULL,  -- 'USER' or 'SERVICE_ACCOUNT' (small enum)
  status      VARCHAR(20) NOT NULL,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE user_profiles (
  identity_id UUID PRIMARY KEY REFERENCES identities(id),
  email       VARCHAR(255) UNIQUE NOT NULL,
  display_name VARCHAR(100) NOT NULL,
  last_login_at TIMESTAMPTZ
);

CREATE TABLE service_account_profiles (
  identity_id UUID PRIMARY KEY REFERENCES identities(id),
  service_name VARCHAR(100) NOT NULL,
  description TEXT
);
-- Each table: focused, all columns NOT NULL where appropriate, clean queries.
```

### Mistake 2: Implicit State Through Nullable Timestamps

```sql
-- IMPLICIT STATE (error-prone):
ALTER TABLE payments ADD COLUMN completed_at TIMESTAMPTZ;
-- NULL = not completed. Non-null = completed. Implicit.

-- A bug accidentally sets completed_at = NOW() during an error path.
-- Payment is now "completed" even though it failed.
-- No constraint prevents it. No code catches it.

-- EXPLICIT STATE (safe):
CREATE TYPE payment_status AS ENUM ('PENDING', 'PROCESSING', 'COMPLETED', 'FAILED', 'REFUNDED');
ALTER TABLE payments ADD COLUMN status payment_status NOT NULL DEFAULT 'PENDING';

-- State machine is explicit. Only valid transitions allowed:
-- PENDING → PROCESSING → COMPLETED
-- PENDING → PROCESSING → FAILED
-- COMPLETED → REFUNDED
-- Enforce in application code; optionally enforce with CHECK constraints.
```

### Mistake 3: Crossing Aggregate Boundaries in Transactions

```sql
-- WRONG: crossing two aggregate boundaries in one transaction
BEGIN;
  UPDATE orders SET status = 'PAID' WHERE id = :order_id;
  UPDATE inventory SET reserved = reserved - 1 WHERE product_id = :product_id;
COMMIT;

-- Orders and inventory are separate aggregates in separate bounded contexts.
-- In a distributed system, they may be in separate databases.
-- This transaction creates tight coupling that prevents independent scaling.

-- RIGHT: use an event/saga
-- T1: Order Service: UPDATE orders SET status = 'PAID'; emit OrderPaid event
-- T2: Inventory Service: on OrderPaid event, UPDATE inventory SET reserved = reserved - 1
-- Eventual consistency. If inventory update fails: compensating event to revert order.
```

### Mistake 4: Overloading the Primary Key

```sql
-- OVERLOADED PK (anti-pattern):
CREATE TABLE users (
  email VARCHAR(255) PRIMARY KEY,  -- email as PK
  ...
);

-- Problems:
-- 1. Email changes → must update every FK reference → expensive migration
-- 2. Users who want to change email: complex UPDATE cascade
-- 3. Leaks PII into foreign key columns of other tables
-- 4. Clustering index on email → random inserts → index fragmentation

-- RIGHT: surrogate UUID PK + unique constraint on natural key
CREATE TABLE identities (
  id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  ...
);
-- UUID PK: stable forever, no PII leakage
-- UNIQUE on email: still enforces uniqueness
-- Email change: UPDATE identities SET email = ? WHERE id = ? → one row, no cascade
```

### Mistake 5: Storing Derived Data Without a Source of Truth

```sql
-- DANGEROUS PATTERN:
CREATE TABLE accounts (
  id      UUID PRIMARY KEY,
  balance NUMERIC(18,2) NOT NULL  -- updated on every transaction
);

-- What if a bug credits $100 twice?
-- What if a transaction fails after updating balance but before recording the movement?
-- The balance is the only record. There's no way to audit or reconcile.

-- RIGHT: immutable ledger + derived balance
CREATE TABLE ledger_entries (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id UUID NOT NULL REFERENCES accounts(id),
  amount     NUMERIC(18,2) NOT NULL,  -- positive=credit, negative=debit
  type       VARCHAR(50) NOT NULL,    -- 'PAYMENT', 'REFUND', 'FEE', 'ADJUSTMENT'
  reference  UUID,                   -- payment_id, refund_id, etc.
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Balance = sum of all ledger entries (materialized for performance):
CREATE TABLE account_balances (
  account_id UUID PRIMARY KEY REFERENCES accounts(id),
  balance    NUMERIC(18,2) NOT NULL,
  version    BIGINT NOT NULL DEFAULT 0  -- optimistic locking
);

-- Balance is a cache. Ledger is the truth.
-- If balance ever seems wrong: recompute from ledger entries.
-- Full audit trail: every cent is accounted for.
```

### Mistake 6: Forgetting the Read Model

```sql
-- WRITE-MODEL-ONLY DESIGN:
-- Admin needs: "Show all active users with their roles, last login, and session count"
SELECT u.id, u.name, u.status, u.last_login_at,
       COUNT(s.id) AS session_count,
       json_agg(r.name) AS roles
FROM identities u
LEFT JOIN sessions s ON s.identity_id = u.id AND s.status = 'ACTIVE'
LEFT JOIN identity_roles ir ON ir.identity_id = u.id AND ir.revoked_at IS NULL
LEFT JOIN roles r ON r.id = ir.role_id
WHERE u.status = 'ACTIVE'
GROUP BY u.id, u.name, u.status, u.last_login_at;

-- At 1M identities: this query takes 30 seconds.
-- Admin dashboard is unusable.

-- RIGHT: maintain a read model
CREATE MATERIALIZED VIEW active_identity_summary AS
SELECT
  u.id,
  u.display_name,
  u.status,
  u.last_login_at,
  COUNT(DISTINCT s.id) AS active_session_count,
  json_agg(DISTINCT r.name) FILTER (WHERE r.name IS NOT NULL) AS active_roles
FROM identities u
LEFT JOIN sessions s ON s.identity_id = u.id AND s.status = 'ACTIVE'
LEFT JOIN identity_roles ir ON ir.identity_id = u.id AND ir.revoked_at IS NULL
LEFT JOIN roles r ON r.id = ir.role_id
WHERE u.status = 'ACTIVE'
GROUP BY u.id, u.display_name, u.status, u.last_login_at;

CREATE UNIQUE INDEX ON active_identity_summary(id);

-- Refresh strategy:
REFRESH MATERIALIZED VIEW CONCURRENTLY active_identity_summary;
-- Schedule: every 60 seconds (pg_cron). Or CDC-triggered. Acceptable staleness.
-- Query time: <5ms.
```

---

### The Five Data Modeling Principles

1. **Access patterns drive schema.** Write down every query before designing any table. The indexes and denormalisations emerge from the queries.

2. **Separate write models from read models.** The model that's correct for capturing state changes is not the model that's fast for reading it. Maintain both explicitly.

3. **Events are more valuable than current state.** An event log tells you what happened, when, by whom, and why. Current state tells you only where you are now. For anything that matters (financial records, security events, compliance), store events.

4. **Name things in the domain language.** The schema is documentation. `identity_status_events` is more self-documenting than `user_audit_log`. The code, database, and conversations should use the same words.

5. **Every schema decision has a rationale.** The next engineer shouldn't have to guess why a column exists, what NULL means, or why there's a partial index. Write it down in the migration comment, the ADR, or the schema documentation.

---

*End of Part 13: Data Modeling in Depth*
# Chapter 75: System Design Building Blocks Cheat Sheet

> Quick-reference for technology selection, pattern application, and capacity estimation. Use this to quickly orient a design before drilling into depth.

---

## Storage

| Technology | Use When | Avoid When |
|---|---|---|
| PostgreSQL | ACID, transactions, joins, financial systems | Massive write scaling across regions |
| MySQL | General OLTP workloads | Complex analytics |
| Redis | Cache, sessions, rate limiting | Long-term storage |
| Cassandra | Massive writes, global scale | Complex joins |
| MongoDB | Flexible schema | Strong relational modeling |
| S3 | Object storage, backups, data lake | Low-latency queries |
| Elasticsearch | Search | Primary database |
| Neo4j | Relationship-heavy data | Simple CRUD |

---

## Communication

| Technology | Use When | Avoid When |
|---|---|---|
| REST | Public APIs | Ultra-low latency internal calls |
| gRPC | Service-to-service communication | Browser clients |
| Kafka | Event streaming | Request-response |
| RabbitMQ | Task queues | Massive event replay |
| WebSockets | Real-time communication | Simple CRUD APIs |
| SSE | One-way real-time updates | Bidirectional communication |

---

## Data Access Patterns

| Pattern | Use When |
|---|---|
| CRUD | Standard business apps |
| Event Sourcing | Audit trail required |
| CQRS | Read-heavy systems |
| Materialized Views | Expensive aggregations |
| Outbox Pattern | Reliable event publishing |
| CDC | Database changes must be streamed |

---

## Scaling

| Technique | Use When |
|---|---|
| Cache | Read bottleneck |
| Replication | Read scaling |
| Partitioning | Large tables |
| Sharding | Database size exceeds one node |
| CDN | Static content |
| Load Balancing | Multiple servers |
| Autoscaling | Variable traffic |

---

## Consistency

| Technique | Use When |
|---|---|
| Strong Consistency | Payments |
| Eventual Consistency | Social media |
| Quorum | Distributed databases |
| Read Your Writes | User profile updates |
| Monotonic Reads | User-facing systems |

---

## Reliability

| Pattern | Use When |
|---|---|
| Retry | Transient failures |
| Timeout | Prevent hanging calls |
| Circuit Breaker | Failing downstream dependency |
| Bulkhead | Prevent cascading failures |
| Fallback | Graceful degradation |
| DLQ | Poison messages |
| Idempotency | Payments |
| Saga | Distributed transactions |

---

## Security

| Technology | Use When |
|---|---|
| Session | Traditional web apps |
| JWT | Stateless APIs |
| OAuth2 | Delegated access |
| OIDC | Authentication |
| SAML | Enterprise SSO |
| RBAC | Role-based permissions |
| ABAC | Attribute-based permissions |
| ReBAC | Relationship-based permissions |

---

## Observability

| Tool | Use When |
|---|---|
| Logs | Debugging |
| Metrics | Monitoring |
| Tracing | Request flow analysis |
| Alerts | Operational response |
| Dashboards | System visibility |

---

## Database Selection Matrix

| Requirement | Best Choice |
|---|---|
| Financial transactions | PostgreSQL |
| User sessions | Redis |
| Search | Elasticsearch |
| Event streaming | Kafka |
| Time-series | TSDB (InfluxDB/TimescaleDB) |
| Graph relationships | Neo4j |
| Data lake | S3 |
| Massive write throughput | Cassandra |

---

## Load Balancing

| Strategy | Use When |
|---|---|
| Round Robin | Uniform traffic |
| Least Connections | Long-lived requests |
| IP Hash | Sticky sessions |
| Consistent Hashing | Cache clusters |

---

## Partitioning Strategy

| Strategy | Use When |
|---|---|
| Range | Time-based data |
| Hash | Uniform distribution |
| Directory | Custom routing |
| Geo-based | Multi-region systems |

---

## Authentication Decision Tree

```
User login only
    ↓
Session or JWT

Third-party access
    ↓
OAuth2

Authentication + Identity
    ↓
OIDC

Enterprise SSO
    ↓
SAML
```

---

## Messaging Decision Tree

```
Need request-response?
    ↓ REST / gRPC

Need async processing?
    ↓ RabbitMQ

Need event streaming?
    ↓ Kafka

Need realtime browser updates?
    ↓ WebSocket
```

---

## Architecture Pattern Selection

| Pattern | Use When |
|---|---|
| Monolith | Small teams |
| Microservices | Large domains |
| Event-Driven | Loose coupling |
| CQRS | Read-heavy systems |
| Event Sourcing | Auditability |
| Saga | Distributed transactions |
| BFF | Multiple client types |

---

## Multi-Region Design

| Pattern | Use When |
|---|---|
| Active-Passive | Simpler DR |
| Active-Active | Global systems |
| Geo Routing | Regional latency |
| Data Sovereignty | Regulatory requirements |

---

## Capacity Planning Quick Estimates

| Metric | Formula |
|---|---|
| Storage | `users × records × retention` |
| QPS | `DAU × actions/day ÷ 86,400` |
| Bandwidth | `QPS × payload_size` |
| Kafka Partitions | `throughput ÷ partition_throughput` |
| DB Connections | `QPS × latency_sec` |

---

*Staff Engineer System Design Fundamentals Handbook — Complete (75 Chapters)*
# Chapter 76: Common Architecture Patterns

> "These patterns appear so frequently in production systems that recognising them instantly — and knowing their failure modes — is a baseline staff engineer skill."

---

## Interview Question

> "You're reviewing a design that uses an API gateway, sidecars, and service discovery. Explain what each does, why it's there, and what breaks if it's removed."

---

## Pattern 1: API Gateway

### What It Is

A single entry point for all client requests. Sits at the edge of your system, before your internal services.

```text
Clients                    Internal Services
  │                               │
  │  HTTP/HTTPS                   │
  ▼                               │
┌─────────────────────────┐       │
│      API Gateway         │──────┤
│                          │      │
│  • Auth (JWT validate)   │ → Payment Service
│  • Rate limiting         │ → User Service
│  • Path-based routing    │ → Notification Service
│  • SSL termination       │
│  • Request logging       │
│  • Response caching      │
│  • Request transformation│
└─────────────────────────┘
```

### What It Does

**Authentication and authorisation**: Validate JWT tokens once at the gateway rather than in every service. Services receive the verified identity via a trusted header (`X-Authenticated-User-Id`).

**Rate limiting**: Global rate limiting per API key, per IP, per user. Prevents individual consumers from exhausting the system.

**Routing**: `/payments/*` → payment service; `/users/*` → user service. The client talks to one URL; the gateway handles service topology.

**SSL termination**: TLS ends at the gateway. Internal traffic is plain HTTP (within a trusted VPC) or mTLS.

**Request ID injection**: Every inbound request gets a `X-Request-ID` header. All downstream services log it. Correlates logs across services.

**Response caching**: GET requests with cacheable responses served from gateway cache.

### Common Implementations

- **Kong**: Open source, plugin-based, Lua scripting
- **AWS API Gateway**: Managed, integrated with Lambda, IAM
- **Nginx/Envoy**: Configured as edge proxy
- **Traefik**: Dynamic configuration via service labels

### Failure Modes

**Single point of failure**: If the gateway goes down, everything goes down. Fix: multiple gateway instances behind a cloud load balancer; health checks.

**Tight coupling via transformation**: Gateway transforms request payloads to match backend services. When a service changes its API, the gateway transform must also change. Coordinate carefully.

**Bypassed security**: An internal service is accidentally exposed without going through the gateway (misconfigured firewall, direct NodePort in Kubernetes). Security that only exists at the gateway is bypassed. Fix: defence in depth — services also validate tokens, not only the gateway.

**Latency added**: Every request goes through the gateway. A misconfigured gateway (large request body buffering, synchronous auth call on every request) adds significant latency. Profile gateway overhead.

---

## Pattern 2: BFF (Backend for Frontend)

### What It Is

A dedicated backend service for each type of frontend client. Instead of one generic API serving mobile, web, and partner integrations, each client has its own "backend for frontend."

```text
WITHOUT BFF (one API, many compromises):
  Mobile App ────────────────────────────┐
  Web Browser ───────────────────────────┼──→ Generic API → Microservices
  Partner Integration ────────────────────┘
  
  The API must accommodate all clients:
  Mobile wants minimal payloads (battery, bandwidth)
  Web wants rich data (no round trips)
  Partners want stable, versioned contracts
  One API fails to optimise for all three.

WITH BFF:
  Mobile App ─────→ Mobile BFF ──────────┐
  Web Browser ────→ Web BFF ─────────────┼──→ Microservices
  Partner ────────→ Partner API (BFF) ────┘
  
  Each BFF aggregates exactly what its client needs.
```

### What It Does

**Data aggregation**: Mobile BFF calls User Service + Payment Service + Notification Service and returns one response. The mobile app makes one call, not three.

**Response shaping**: Web BFF returns full objects. Mobile BFF strips to minimal fields.

**Client-specific auth**: Mobile BFF handles OAuth2 PKCE flow. Partner API handles client credentials. Web BFF handles session cookies.

**Versioning isolation**: Partner API is versioned (`/v1/`, `/v2/`). Internal services can change without breaking the partner contract (the BFF translates).

### IAM Platform Example

```text
IAM Mobile BFF:
  POST /login → calls Auth Service, returns { access_token, user_display_name }
  (Strips all internal metadata; mobile only needs the token and name)

IAM Admin Web BFF:
  GET /admin/identities → calls Identity Service + Role Service, returns paginated
  list with roles embedded in each identity object
  (Single request, rich payload, admin-optimised)

IAM Partner API (BFF):
  POST /v1/tokens/introspect → stable contract, versioned
  GET /v1/users/{id} → returns Partner-contractual fields only
  (Partners don't care about internal IAM fields; BFF projects the right view)
```

### Failure Modes

**BFF sprawl**: Every team builds their own BFF, each with subtle differences. The "single source of truth" service is now 7 BFFs each interpreting business rules slightly differently.

**Logic leakage**: Business logic (pricing, entitlement rules) creeps into the BFF because it's convenient. BFFs should aggregate and transform, not make business decisions.

**Duplication**: Auth logic duplicated across 4 BFFs. A security fix must be applied in 4 places. Fix: delegate auth to the API gateway or a shared auth middleware library.

---

## Pattern 3: Sidecar

### What It Is

A secondary container (in Kubernetes: a container in the same pod) that extends or enhances the primary application container without modifying it.

```text
Kubernetes Pod:
  ┌──────────────────────────────────────────┐
  │                                          │
  │  ┌────────────────┐  ┌────────────────┐  │
  │  │  App Container │  │   Sidecar      │  │
  │  │  (Payment Svc) │  │  (Envoy Proxy) │  │
  │  │                │  │                │  │
  │  │  Business logic│  │  mTLS          │  │
  │  │  Port 8080     │  │  Circuit break │  │
  │  │                │  │  Metrics       │  │
  │  └────────────────┘  │  Tracing       │  │
  │                       │  Auth          │  │
  │                       └────────────────┘  │
  │                                          │
  │  Shared network namespace (localhost)    │
  │  Shared volume mounts                    │
  └──────────────────────────────────────────┘
```

### Common Sidecar Uses

**Service mesh proxy (Envoy/Istio)**: Intercepts all inbound and outbound traffic. Implements mTLS, circuit breaking, retries, distributed tracing, and traffic management — without the application knowing. The app just makes plain HTTP calls; the sidecar handles everything else.

**Log shipping**: App writes logs to a local file; sidecar tails the file and ships to Elasticsearch/Loki. App doesn't know about the logging backend.

**Secret refresh**: App mounts a secret file; sidecar watches HashiCorp Vault and rewrites the file when the secret rotates. App re-reads the file; no restart required.

**Configuration synchronisation**: Sidecar polls a config server and writes updated config to a shared volume. App polls the file on a short interval.

### IAM Platform Example

Every IAM service pod has an Envoy sidecar injected by Istio:

```text
Client → [Envoy sidecar] → App container
                ↓
  Istio control plane: "This service needs mTLS. Enforce it."
  
  Envoy handles:
    - mTLS certificate rotation (15-minute cert TTL, automatically renewed)
    - Mutual authentication with upstream services
    - Circuit breaker for downstream calls
    - Prometheus metrics scraping (/stats/prometheus endpoint)
    - Distributed trace context propagation (injects B3/W3C headers)
    
  App container:
    - Unaware of mTLS (talks plain HTTP to localhost Envoy)
    - Unaware of circuit breaking (Envoy handles the 503s)
    - Unaware of tracing (headers injected/extracted by Envoy)
```

### Failure Modes

**Sidecar startup race**: App container starts before sidecar is ready. First few requests fail because the proxy isn't up. Fix: `initContainers` to wait for sidecar readiness, or app retries on startup.

**Resource contention**: Sidecar consumes CPU and memory in the same pod. Under load, the sidecar's overhead is non-trivial (Envoy uses 50-100MB RAM per pod). At 200 pods, that's 10-20 GB consumed by sidecars. Rightsize sidecar resource requests.

**Sidecar bug affecting all services**: If the Envoy sidecar has a bug, it affects every service simultaneously. Istio upgrades must be managed carefully (canary the control plane update).

**Observability gap**: The sidecar intercepts traffic, so its metrics may not match the app's metrics. "Sidecar sees 500 requests, app logs 490" → 10 requests rejected at the proxy layer before reaching the app. Understand which layer your metrics come from.

---

## Pattern 4: Strangler Fig

### What It Is

A migration pattern for incrementally replacing a legacy system with a new one — without a "big bang" rewrite.

Named after the strangler fig vine, which wraps itself around a host tree, growing over it until the host is fully replaced.

```text
Phase 1: Route new feature to new service, legacy handles rest
  Clients → Proxy/Gateway
                │
    ┌───────────┼────────────┐
    │           │            │
    ▼           │            ▼
  [/payments/new]     [Everything else]
  New Payment Svc       Legacy Monolith

Phase 2: Migrate more features to new service
  Clients → Proxy/Gateway
                │
    ┌───────────┼────────────┐
    │           │            │
    ▼           ▼            ▼
 [/payments]  [/users]  [remaining legacy routes]
  New Svc      New Svc     Legacy (shrinking)

Phase 3: Legacy fully replaced
  Clients → New Services (Legacy decommissioned)
```

### Implementation

**Step 1**: Put a routing proxy (nginx, API gateway, feature flag service) in front of the monolith. All traffic still goes to the monolith.

**Step 2**: Build the new service alongside, not instead of, the legacy code. Build the strangler, not a replacement.

**Step 3**: Route specific paths/features to the new service. Start with low-risk, non-critical features. Learn the pattern.

**Step 4**: Progressively move traffic. New service is the target; monolith handles fallback.

**Step 5**: Decommission legacy routes as new service covers them.

### Failure Modes

**Data synchronisation during migration**: Both systems write to the same database during the transition. Schema changes in the new service break the legacy. Solution: the new service gets its own data store; a sync layer bridges them during migration. Sync layer is removed at cutover.

**Dual-write inconsistency**: During migration, writes go to both old and new systems. A failure in one leaves them out of sync. Use the outbox pattern on one side as the source of truth.

**Never completing the migration**: Phase 1 goes smoothly. Phase 2 gets deprioritised. The strangler is "in progress" for 3 years. Two systems must be maintained. The strangler becomes a technical liability. Fix: commit to a completion timeline; the migration must have a deadline.

---

## Pattern 5: Outbox Pattern

*See Chapter 54 (Event-Driven Architecture) for deep coverage.*

### Quick Reference

**Problem**: Write to a database AND publish an event atomically. Network calls can't be part of a database transaction.

**Solution**: Write the event to an `outbox` table in the same transaction as the business data. A background process reads the outbox and publishes to the message broker. Mark published rows complete.

```text
Same transaction:
  INSERT INTO payments (...)
  INSERT INTO outbox_events (event_type='PaymentCreated', payload={...})
COMMIT

Background:
  SELECT * FROM outbox_events WHERE published = FALSE
  → publish to Kafka
  UPDATE outbox_events SET published = TRUE
```

**Key properties**: At-least-once delivery (events may be published more than once on retry). Consumers must be idempotent.

**Implementation options**: Polling (simple, adds latency), Debezium CDC (watches WAL, near-real-time, no polling overhead).

**Failure modes**: Outbox processor crashes between publish and mark-published → duplicate event on restart → consumer must handle it. Long outbox backlogs during Kafka downtime → outbox table grows → monitor outbox depth.

---

## Pattern 6: CDC (Change Data Capture)

### What It Is

CDC captures every insert, update, and delete from a database's transaction log and streams the changes as events — without modifying the application.

```text
PostgreSQL WAL
  │ (every write is recorded here)
  │
  ▼
Debezium (CDC tool)
  │ reads WAL via logical replication slot
  │ converts row changes to structured events
  │
  ▼
Kafka
  │
  ├──→ Elasticsearch (search index updated in near-realtime)
  ├──→ Data Warehouse (analytics updated without ETL jobs)
  ├──→ Cache Invalidation (Redis entries invalidated on DB write)
  └──→ Notification Service (user updated → send email)

Application writes to PostgreSQL normally.
CDC handles all downstream propagation automatically.
```

### Change Event Structure

```json
{
  "op": "u",
  "before": { "id": "user-123", "status": "ACTIVE" },
  "after":  { "id": "user-123", "status": "SUSPENDED" },
  "source": { "table": "identities", "ts_ms": 1700000000000 }
}
```

Operations: `c` (create), `u` (update), `d` (delete), `r` (read/snapshot).

### IAM Platform Use: Permission Cache Invalidation

```text
Admin revokes a role from a user (UPDATE identity_roles SET revoked_at = NOW())
  │
  ▼
Debezium detects the WAL change
  │
  ▼
Kafka: identity_role_changes topic
  │
  ▼
Cache Invalidation Consumer:
  DELETE redis key "permissions:{user_id}"
  
Next permission check: cache miss → loads fresh permissions from DB
User's revoked role is no longer effective.
Latency from DB write to cache invalidation: ~1-3 seconds.
```

### Failure Modes

**Replication slot lag**: If the Debezium consumer is down, the PostgreSQL replication slot accumulates WAL. WAL is not cleaned up until the slot consumer catches up. Disk fills up. Fix: monitor replication slot lag; set `max_slot_wal_keep_size` to limit WAL retention (slot is invalidated if this is exceeded — you get a gap in events).

**Schema changes break consumers**: Adding a NOT NULL column to a table changes the event structure. Consumers expecting the old structure break. Fix: treat database schemas as public contracts; use schema registry and backward-compatible changes only.

**Initial snapshot**: A new CDC consumer needs to bootstrap from existing data (not just future changes). Debezium's snapshot mode reads a consistent full snapshot of the table first, then switches to streaming. During snapshot, the table may be locked briefly (depending on the snapshot mode).

---

## Pattern 7: Leader Election

*See Chapter 47 (Consensus) for Raft/Paxos detail.*

### Quick Reference

**Problem**: In a cluster of services, exactly one node must perform a given operation at a time (database primary, scheduled job executor, partition owner).

**Solution**: Use a distributed consensus store to elect a leader. The leader holds a lock with a TTL. It must renew the TTL continuously. If it fails to renew (crash, network partition), the lock expires and a new leader is elected.

```text
3 nodes competing for leadership:

Node A: acquires lock "leader" in etcd with TTL=30s
  → Node A is leader. Processes jobs.
  → Every 10s: renews TTL (keepalive)

Node A crashes at T=15s. Last renewal was T=10s.
  → T=40s: TTL expires. Lock released.
  
Node B: acquires "leader" lock.
  → Node B is now leader.
  
Node A restarts at T=50s. Tries to renew old lock.
  → Lock no longer exists (or has a new term). Node A becomes follower.
```

**Implementation in practice**:
- Kubernetes leader election: `leaderelection` package in `client-go`
- Database-based: `INSERT INTO locks (name) ON CONFLICT DO NOTHING` + periodic heartbeat
- Redis: `SET leader node-id NX PX 30000` (SET if not exists, with TTL)
- etcd / ZooKeeper: dedicated lock primitives

**For IAM**: The session cleanup job (remove expired sessions) must run on exactly one node. If it runs on all nodes simultaneously, they interfere. Leader election ensures exactly one node runs it. If that node dies, another takes over within TTL seconds.

**Failure modes**: Fencing token (monotonically increasing epoch number from etcd) must be passed to any downstream storage write. A deposed leader that doesn't know it's deposed might try to write stale state — the storage must reject writes with old epoch numbers.

---

## Pattern 8: Service Discovery

### What It Is

In a dynamic environment (containers, autoscaling), services don't have fixed IP addresses. Service discovery allows services to find each other by name without hardcoded addresses.

```text
WITHOUT service discovery:
  Payment Service → hardcoded "http://10.0.1.45:8080/users"
  User Service instance restarts, gets new IP 10.0.1.67.
  Payment Service calls old IP → connection refused.
  Config must be manually updated and redeployed.

WITH service discovery (Kubernetes DNS):
  Payment Service → "http://user-service:8080/users"
  Kubernetes DNS resolves "user-service" to the current ClusterIP.
  ClusterIP load-balances to healthy pods.
  Pods can scale up/down, restart, move nodes — service name is stable.
```

### Two Models

**Client-side discovery**: The service queries the service registry itself, gets a list of healthy instances, and picks one (with load balancing).

```text
Payment Service → Consul/Eureka: "Where is user-service?"
               ← [10.0.1.45:8080, 10.0.1.67:8080, 10.0.1.89:8080]
Payment Service picks one (round-robin/random/least-connections)
Payment Service → 10.0.1.67:8080 directly
```

**Server-side discovery**: The service calls a stable name; a proxy or load balancer handles the resolution.

```text
Payment Service → "user-service:8080" (Kubernetes DNS)
Kubernetes DNS → ClusterIP (virtual IP)
ClusterIP (kube-proxy) → one of the healthy pods
```

In Kubernetes: server-side discovery is the default. `kube-dns` / `CoreDNS` resolves service names. `kube-proxy` load-balances to pod endpoints. Envoy sidecar (Istio) provides richer client-side discovery with circuit breaking.

### Failure Modes

**DNS caching**: A JVM application caches DNS resolution for the JVM lifetime by default (`networkaddress.cache.ttl=-1`). If a service moves (pod restarts, new IP), the cached IP is stale. Fix: set `networkaddress.cache.ttl=5` in JVM apps, or use a service mesh that doesn't rely on DNS.

**Health check lag**: A service instance crashes. The load balancer still routes to it for up to `health_check_interval` seconds. Requests fail. Fix: aggressive health checks (5-second interval, 2-second timeout, 1 failure = unhealthy).

**Service registry split-brain**: In a partitioned network, one half of the cluster sees different service registries. Fix: use a consensus-based registry (etcd, Consul with Raft) that handles partitions correctly.

---

*End of Chapter 76: Common Architecture Patterns*

---

# Chapter 77: System Design Failure Patterns

> "Staff engineers think about failure first. Senior engineers think about the happy path. The difference shows at 3 AM."

---

## Interview Question

> "Walk me through five ways a distributed system can fail in non-obvious ways — situations where the system appears to be working but is silently broken."

---

## Why This Chapter Exists

Every pattern in this handbook has an anti-pattern — a way the system can go wrong. Engineers who know the solutions but not the failure patterns build systems that work in tests and break in production. This chapter is the catalogue of failure modes that appear most frequently in real systems.

---

## Failure 1: Hot Partitions

### What It Is

A hot partition (or hot shard) is when a disproportionate amount of load lands on one shard, one Kafka partition, or one database row — exhausting its capacity while other shards sit idle.

### How It Happens

```text
Kafka Topic: payments
  Partition 0: 95,000 events/sec  ← saturated
  Partition 1: 3,000 events/sec
  Partition 2: 2,000 events/sec

Why? Partition key = merchant_id.
     One merchant (Flipkart, 10M transactions/day) hashes to partition 0.
     All their traffic lands on one partition.
     Partition 0's single consumer is overwhelmed.
     End-to-end latency for Flipkart payments: 45 seconds (backlogged).
     All other merchants: <1 second.
```

Database version: a `payments` table sharded by `user_id`. One shard contains all users in Mumbai (30% of users). That shard handles 30% of all writes while others handle ~17% each.

### Detection

```text
Kafka:
  kafka-consumer-groups.sh --describe → one partition lag >> others
  prometheus: kafka_consumergroup_lag by (partition) → one partition far ahead

Database:
  pg_stat_user_tables → one shard has 3x more seq_tup_read, heap_blks_read
  Dashboards: p99 latency for shard-1 >> other shards
```

### Mitigations

**Increase partition count**: More Kafka partitions = more consumers = more parallelism. But: repartitioning requires rebalancing; downstream consumers must handle potentially out-of-order events.

**Add entropy to partition key**: `hash(merchant_id + random_salt_from_set)` — spread one merchant across N partitions deliberately. Consumers merge the streams. Ordering within a merchant is lost (acceptable for most use cases).

**Virtual shards (Cassandra)**: Add a bucket suffix to the partition key: `merchant_id + '_' + (event_ts % 10)`. Distributes one merchant's data across 10 virtual shards. Queries must now aggregate across 10 shards.

**Adaptive routing**: Detect hot shards at runtime and re-route to less loaded shards. Requires a coordination layer.

---

## Failure 2: Thundering Herd

### What It Is

A large number of processes simultaneously wake up, attempt to perform the same action, and overwhelm a resource — even though the individual processes are behaving correctly.

### How It Happens

```text
Scenario: 500 API server instances, each caching user profiles with TTL=60 minutes.
All instances started at the same time (batch deployment).
At T=60min: all 500 caches expire simultaneously.
500 instances simultaneously try to load user profiles from the database.
500 × 20 connections = 10,000 DB queries in the same second.
Database CPU: 100%. Connections queued. Timeouts cascade.
Entire platform degrades for 30-60 seconds while caches rebuild.

Repeat every 60 minutes.
```

Other triggers: Cache restart (all keys gone), application restart (cold caches), scheduled job firing on all instances at once.

### Mitigations

**TTL jitter**: Instead of TTL=3600, use TTL=3600 + random(0, 600). Expiries spread over 10 minutes. No simultaneous expiry.

```python
ttl = base_ttl + random.randint(0, base_ttl * 0.15)
cache.set(key, value, ttl)
```

**Probabilistic early recomputation (XFetch)**: Recompute the cache value slightly before it expires, with probability increasing as expiry approaches. Avoids the sudden expiry cliff.

```python
def cached_fetch(key, fetch_fn, base_ttl):
    result = cache.get_with_ttl(key)  # returns (value, remaining_ttl)
    if result is None:
        value = fetch_fn()
        cache.set(key, value, base_ttl)
        return value
    value, remaining_ttl = result
    # Recompute early with probability proportional to how close we are to expiry
    if random() < (1 - remaining_ttl / base_ttl) ** 2:
        value = fetch_fn()
        cache.set(key, value, base_ttl)
    return value
```

**Mutex / single-filler lock**: When a cache miss occurs, acquire a distributed lock. Only one thread/process fills the cache. Others wait for the lock and then read the freshly-filled value.

**Pre-warming on deployment**: Before new instances start serving traffic, pre-warm their caches from a backup source (hot standby cache, snapshot). They never experience a cold start.

---

## Failure 3: Cache Stampede

*See Chapter 29 for deep coverage. Quick reference here:*

Cache stampede is a specific thundering herd where a popular cached item expires. All concurrent requests simultaneously find a cache miss and simultaneously query the database for the same value.

```text
10,000 req/sec hitting key "top_products"
TTL expires.
10,000 simultaneous cache misses → 10,000 DB queries for the same value.
```

Mitigations: mutex lock (only one fills the cache; others block/wait), stale-while-revalidate (serve stale while one request refreshes in background), XFetch (probabilistic early recomputation).

---

## Failure 4: Retry Storm

### What It Is

A downstream service becomes slow or unhealthy. Callers retry. Retries increase load on the already-struggling service. The additional load makes the service slower. More retries. Positive feedback loop: the service never recovers.

### How It Happens

```text
Payment Service → Bank Core API (external)
Bank Core API starts returning 503 (overloaded)

Without retry control:
  500 req/sec incoming to Payment Service
  Each req retries 3 times with 100ms delay
  Bank Core API now receives 500 × 3 = 1500 req/sec
  Bank Core API is even more overloaded → more 503s → more retries
  
  Bank Core API was handling 600 req/sec fine.
  Now receiving 1500 req/sec → complete failure.
  Payment Service appears to be "helping" by retrying.
  Actually making things worse.
```

### Mitigations

**Exponential backoff with jitter**: Reduces retry rate as failure continues. Jitter prevents synchronised retry waves.

**Retry budget**: Each service allocates a budget of total retries per time window. When the budget is exhausted, requests fail immediately (no more retries). Caps the total retry amplification.

**Circuit breaker**: After N failures in a window, stop retrying entirely. Give the downstream time to recover without additional load.

**Retry on 503 with Retry-After header**: The struggling service tells callers when to retry. Callers that respect the header create a coordinated recovery window.

**Total timeout budget**: Each request has an overall deadline (e.g., 5 seconds total). Retries must fit within the budget. A 3-retry × 1s retry schedule with a 2s budget = 2 retries maximum. Prevents long retry chains.

---

## Failure 5: Cascading Failure

### What It Is

A failure in one component causes failures in dependent components, which cascade through the system until the entire platform is down — even though only one service was the original failure.

### How It Happens

```text
Normal state:
  API Tier (100 threads) → DB (handles 100 queries/sec fine)
  
DB starts responding slowly (disk I/O spike):
  API threads wait for DB responses → threads accumulate (now 100 blocked threads)
  New requests arrive → no threads available → queue builds
  Queue fills → API tier starts rejecting requests
  
Load balancer: "API tier is slow/failing, sending traffic to other API instances"
  Other instances also call DB → their threads also fill up
  
DB is now receiving MORE queries (from all blocked threads that haven't timed out yet):
  DB: 500 concurrent queries → more I/O → slower → more threads block
  
Within 90 seconds: entire platform is down.
Root cause: one disk I/O spike on one DB instance.
```

### Why It's Insidious

Each component is behaving "correctly" — threads wait for responses (correct), load balancer routes away from slow instances (correct), retries happen (correct). The failure is *emergent* from individually correct behaviours.

### Mitigations

**Timeouts everywhere**: The DB query that's slow must time out in 500ms, not block a thread for 30 seconds. Timeouts are the first defence.

**Bulkheads**: Separate thread pools for separate downstream dependencies. DB slowness fills the DB thread pool, not the shared thread pool.

**Circuit breakers**: After the DB starts failing, the circuit opens. New requests fail fast (not blocking threads) while the DB recovers.

**Load shedding**: When the request queue depth exceeds a threshold, reject new requests immediately (503). Better to fail 20% of requests fast than fail 100% slowly.

**Graceful degradation design**: "If the DB is down, what can we return?" Can we serve stale cache? Can we return a degraded response? Planning degradation modes prevents a partial failure from becoming a total one.

---

## Failure 6: Split Brain

### What It Is

Two nodes simultaneously believe they are the leader (or primary) and act independently, creating two divergent histories of the system state.

### How It Happens

```text
3-node PostgreSQL cluster. Node A is primary.

Network partition: A is separated from B and C.

A: "I can't reach B and C, but I'm still the primary. I'll keep accepting writes."
B: "I can't reach A. It might be dead. After 30s timeout, I'll elect a new primary."
C: "Same as B. Votes for B."
B: becomes new primary. Accepts writes.

Now:
  A: accepting writes to account balances (some clients reach it)
  B: accepting writes to account balances (other clients reach it)
  
  Two divergent histories of account balances.
  When partition heals: which writes win? A's or B's?
  
  If last-write-wins: one set of writes is silently discarded.
  Financial data is corrupted.
```

### Mitigations

**STONITH (Shoot The Other Node In The Head)**: When a failover is initiated, the old primary is forcibly shut down before the new one is promoted. Even if the old primary is still partially reachable, it is stopped. Prevents both nodes from acting as primary simultaneously.

**Fencing tokens**: Every write to storage includes the epoch (term) from the consensus system. Storage rejects writes with an old epoch. A deposed primary's writes are rejected even if it doesn't know it's deposed.

```text
B becomes primary at epoch 6.
A (still thinks it's primary at epoch 5) tries to write.
Storage: "Write from epoch 5 rejected. Current epoch is 6."
A: "My epoch is stale. I must not be primary." → steps down.
```

**Quorum writes**: Writes are only committed when acknowledged by a majority. A partitioned primary with no quorum cannot commit writes. No divergence possible.

**Avoid async replication for financial data**: Synchronous replication ensures followers have all committed writes before a failover. No writes can be "ahead" on the old primary that the new primary doesn't have.

---

## Failure 7: Poison Messages

### What It Is

A message in a queue that causes the consumer to crash or error every time it processes it. The message is retried endlessly, blocking progress and consuming resources.

### How It Happens

```text
Kafka topic: payment-events
Consumer: PaymentProcessingConsumer

Message arrives: { "payment_id": null, "amount": -50, "currency": "INVALID" }
Consumer: tries to process → NullPointerException
Consumer: commits suicide (exception bubbles up, pod restarts)
Container orchestrator: restarts the pod
Pod restarts: reads the same message (offset not committed)
Pod: NullPointerException again → crash → restart → ...

This pod is now crash-looping indefinitely on one bad message.
All messages after this one are never processed.
Consumer lag grows. Alerts fire. Engineers paged.
```

### Mitigations

**Dead Letter Queue (DLQ)**: After N failed attempts, move the message to a DLQ (a separate topic or queue). Processing continues on other messages. A human (or automated process) inspects the DLQ, fixes the message, and re-queues.

```text
Consumer receives message.
If processing fails:
  retry_count++
  if retry_count >= max_retries (e.g., 5):
    send to dead-letter-queue
    commit offset (don't process again)
  else:
    exponential backoff and retry
```

**Defensive consumer code**: Catch and log all exceptions. Never let an individual message crash the consumer. Treat every message as potentially malformed.

**Schema validation at publish**: Validate event schema before publishing to the queue. A schema registry with enforce-mode prevents malformed messages from entering the queue in the first place.

**DLQ monitoring**: Alert when the DLQ depth is non-zero. A message in the DLQ is a problem that requires attention. Teams that don't monitor their DLQs discover weeks of accumulated poison messages during an audit.

---

## Failure 8: Coordinated Omission

### What It Is

A subtle but severe benchmark and monitoring flaw where the measurement system fails to count or time requests that were waiting to be sent — only measuring requests that were actually dispatched and responded to. The result: benchmarks show good latency even when the system is saturated, because the worst-case waiting time is never recorded.

This is Gill Tene's terminology, but it describes a real production monitoring failure mode too.

### How It Happens

```text
Load test: 1000 req/sec target.
System responds in 1ms normally.
At T=30s, system pauses for 10 seconds (GC, DB lock, whatever).

Naive measurement: only records requests that were sent and responded.
  - Requests 1-30,000: ~1ms each ✓
  - During the 10s pause: no new requests sent (test tool is blocked waiting for responses)
  - After pause: responses arrive, recorded as ~1ms each
  
  Reported p99: 1ms
  Reported max: 10ms
  
  WHAT ACTUALLY HAPPENED:
  All 10,000 requests that arrived during the 10s pause waited 10+ seconds.
  The test tool that was blocked did not record them as waiting.
  They appear as fast requests when they eventually complete.
  
  Real user experience p99: 10+ seconds.
  Reported p99: 1ms.
```

### In Production Monitoring

The same flaw appears in production:
- Metrics that only record completed requests, not queued/waiting ones
- Load balancer metrics that don't include time spent in queue
- "Time to first byte" metrics that start timing when the server receives the request, not when the client sent it

### Mitigations

**Measure from the client, not the server**: Client-side instrumentation captures the full round-trip including queueing time. Server-side misses the queue.

**Use Coordinated-Omission-Corrected benchmarking tools**: wrk2 and HDR Histogram are designed to correct for coordinated omission. They schedule requests at a fixed rate regardless of whether previous requests have completed.

**Track queue depth as a metric**: If the request queue depth is growing, something is wrong — even if completed-request latency looks fine.

**End-to-end latency SLO**: Measure and alert on p99 latency from the client's perspective (browser timing APIs, mobile instrumentation, API gateway timestamps) not only the server's processing time.

---

*End of Chapter 77: System Design Failure Patterns*

---

# Chapter 78: Technology Selection Framework

> "The right technology is the one that solves your specific problem with the least complexity you can get away with. Nothing more."

---

## Interview Question

> "You have a new feature to build. Walk me through how you select the right database, messaging system, and caching strategy."

---

## The Framework: Four Questions

Before selecting any technology, answer four questions in order:

1. **What access pattern must this support?** (Point lookup? Range scan? Full-text? Graph traversal?)
2. **What are the consistency requirements?** (ACID? Eventual? Strong per-key?)
3. **What is the scale?** (10k rows? 10B rows? 1k req/sec? 1M req/sec?)
4. **What does the team already know?** (The second-best technology your team knows well beats the best technology they don't.)

---

## Database Selection

```text
START: What type of data is this?
│
├── Structured, relationships matter, transactions required?
│   │
│   ├── Single-region, team knows SQL well?
│   │   └── PostgreSQL ← default choice for most systems
│   │
│   ├── Massive global write throughput (>100k writes/sec)?
│   │   └── Cassandra (accept: no joins, eventual consistency)
│   │
│   └── Multi-region with ACID transactions?
│       └── CockroachDB / Google Spanner (accept: higher latency, cost)
│
├── Key-value, sub-millisecond latency, caching?
│   └── Redis
│       ├── Need persistence? → Redis with AOF/RDB
│       └── Need distributed? → Redis Cluster
│
├── Document, flexible schema, embedded objects?
│   └── MongoDB (accept: eventual consistency by default, watch transactions cost)
│
├── Full-text search, relevance ranking?
│   └── Elasticsearch / OpenSearch
│       └── Never as primary store — always alongside a primary DB
│
├── Time-series (metrics, IoT, telemetry)?
│   └── TimescaleDB (PostgreSQL extension) or InfluxDB
│
├── Graph: relationships are first-class, traversal is the query?
│   └── Neo4j (accept: not for OLTP volume, higher operational overhead)
│
└── Analytics, large-scale aggregations, columnar?
    ├── Cloud data warehouse? → BigQuery / Redshift / Snowflake
    └── Self-hosted? → ClickHouse (exceptional for OLAP at high volume)
```

**Default rule**: Start with PostgreSQL. Migrate when you hit a specific, demonstrated limitation. The cost of premature database specialisation is high; the cost of running PostgreSQL a bit longer than ideal is low.

---

## Caching Layer Selection

```text
START: Why do you need a cache?
│
├── Reduce DB read load (read-heavy, same data accessed repeatedly)?
│   ├── Data fits in memory, single node?
│   │   └── Redis (Cache-aside pattern)
│   │
│   ├── High availability required?
│   │   └── Redis Sentinel or Redis Cluster
│   │
│   └── Latency must be sub-millisecond, same process?
│       └── In-process cache (Caffeine / Guava / HashMap with TTL)
│           (First level; Redis is second level)
│
├── Static content, geographic distribution?
│   └── CDN (CloudFront, Fastly, Cloudflare)
│       └── For API responses? → Edge caching with CDN + Cache-Control headers
│
└── Pre-computed query results, complex aggregations?
    ├── Materialized views in PostgreSQL (maintained by DB)
    └── Pre-built read models in Redis (maintained by application/CDC)
```

**Cache invalidation strategy selection**:
- TTL only: simple, eventually consistent, for data where staleness is acceptable
- Active invalidation (on write): for data where staleness is unacceptable (permissions, sessions)
- CDC-driven invalidation: for complex invalidation (when any of N tables change, invalidate these M keys)

---

## Messaging System Selection

```text
START: What is the communication pattern?
│
├── Synchronous request-response (caller needs the answer now)?
│   ├── Internal service-to-service?
│   │   └── gRPC (typed, efficient, streaming support)
│   └── Public API or browser client?
│       └── REST/HTTP (universal client support)
│
├── Asynchronous, fire-and-forget, tasks?
│   ├── Simple task queue, guaranteed delivery, order within queue?
│   │   └── RabbitMQ / AWS SQS
│   └── Need priorities, routing rules, dead-letter?
│       └── RabbitMQ (with exchange routing)
│
├── Event streaming (replay, fan-out to multiple consumers)?
│   ├── High throughput (>10k events/sec), replay required, multiple consumer groups?
│   │   └── Kafka (Apache or Confluent)
│   └── Managed, don't want to operate Kafka?
│       └── AWS Kinesis / Google Pub/Sub
│
└── Real-time, push to browser/mobile client?
    ├── Bidirectional (server + client both send)?
    │   └── WebSockets
    └── Server → client only (updates, notifications)?
        └── SSE (Server-Sent Events) — simpler than WebSockets, HTTP/2 native
```

**The Kafka vs RabbitMQ decision in one question**: Do consumers need to replay events? If yes: Kafka. If no: RabbitMQ is simpler to operate and reason about.

---

## Architecture Pattern Selection

```text
START: What is the team size and domain maturity?
│
├── Small team (<15 engineers), early stage, domain still evolving?
│   └── Modular Monolith
│       (Clear module boundaries, shared DB, single deployment)
│
├── Multiple teams, domain well-understood, independent deployability needed?
│   └── Microservices
│       (Each team owns 1-2 services, bounded context alignment)
│
├── Multiple client types (mobile, web, B2B partners) with different data needs?
│   └── Add BFF layer on top of microservices or monolith
│
└── Migrating from legacy system?
    └── Strangler Fig (route incrementally, never "big bang" rewrite)

For any choice: add API Gateway at the edge.
```

---

## The Staff Engineer's Technology Veto List

These are technologies that are frequently proposed and frequently wrong:

| "Let's use X" | Usually wrong because |
|---|---|
| Microservices for a 3-person team | Operational overhead kills velocity |
| NoSQL because "it scales" | Scales for what? Most systems don't need Cassandra scale |
| Kafka for a 100 req/sec system | Kafka is expensive to operate; SQS or Postgres LISTEN/NOTIFY suffices |
| Redis as primary database | Data loss risk (async persistence); Redis is a cache, not a DB |
| Event sourcing by default | Adds significant complexity; only worth it when audit trail is primary requirement |
| Elasticsearch as the only query layer | Must be kept in sync with primary DB; adds operational overhead; not a DB |
| GraphQL because "flexibility" | Adds complexity; REST with good design is usually sufficient |

**The right question is always**: "What specific problem does this solve that a simpler choice doesn't?"

---

*End of Chapter 78: Technology Selection Framework*

---

# Chapter 79: Capacity Estimation Cheat Sheet

> "Numbers in your head. Not because you'll remember them forever — because the right order of magnitude changes the architecture."

---

## Interview Question

> "How many servers do you need to handle 100M daily active users for a social feed app?"

---

## Why Capacity Estimation Matters

Capacity estimation in interviews is not about the exact number. It is about:
1. Demonstrating that you reason from first principles, not guesses
2. Arriving at the right order of magnitude (1 server vs 10 vs 1000)
3. Identifying which dimension (storage? bandwidth? CPU?) is the binding constraint
4. Finding the architectural implications (do we need sharding? CDN? caching?)

---

## The Constants You Must Know

### Time
```text
Seconds per minute:    60
Seconds per hour:      3,600
Seconds per day:       86,400   ← memorise this
Seconds per month:     2,592,000  (~2.6M)
Seconds per year:      31,536,000 (~31.5M)
```

### Storage
```text
Kilobyte (KB):   10^3 bytes  (1,000)
Megabyte (MB):   10^6 bytes  (1,000,000)
Gigabyte (GB):   10^9 bytes  (1,000,000,000)
Terabyte (TB):   10^12 bytes
Petabyte (PB):   10^15 bytes

Quick size references:
  ASCII char:         1 byte
  UUID:               16 bytes (binary), 36 bytes (string)
  Typical DB row:     100-500 bytes
  Typical JSON API:   1-10 KB
  Profile photo:      50-200 KB (after compression)
  HD video frame:     ~100 KB
  1 minute HD video:  ~50-100 MB (compressed)
  1 hour HD video:    3-6 GB
```

### Network / Throughput
```text
1 Gbps NIC:       125 MB/sec
10 Gbps NIC:      1.25 GB/sec
SSD sequential:   500 MB/sec read, 400 MB/sec write
SSD random read:  100 microseconds (~0.1ms)
Disk (HDD):       100 MB/sec sequential, 10ms random
Redis GET:        ~0.1ms in-process, ~1ms over network
PostgreSQL query: 1-5ms simple, 10-200ms complex
Cross-AZ latency: ~1ms
Cross-region:     50-200ms
```

---

## QPS Estimation

**From DAU to QPS:**

```text
Formula: QPS = DAU × actions_per_user_per_day ÷ 86,400

But: traffic is not uniform. Peak ≈ 2-3× average.
Peak QPS = average QPS × peak_multiplier

Example: Twitter
  DAU: 500M
  Actions/user/day: 5 (scroll feed, like, retweet, view profile, search)
  Average QPS: 500,000,000 × 5 ÷ 86,400 = 28,935 ≈ 29,000 QPS
  Peak QPS (3× multiplier): ~87,000 QPS

Example: IAM Token Introspection
  DAU: 10M users, each makes 20 API calls, each API call = 1 introspection
  Average introspection QPS: 10,000,000 × 20 ÷ 86,400 = 2,315 QPS
  Peak (2×): ~4,630 QPS
  
  → At this scale: single Redis node handles it (1M ops/sec capacity). No cluster needed.
  → At 50M DAU: 23,000 QPS → Redis Cluster with 3 shards (plenty of headroom).
```

---

## Storage Estimation

**Formula:**

```text
Total storage = rows × row_size × retention_period

Step 1: How many writes per day?
  writes/day = write_QPS × 86,400

Step 2: How large is each row?
  Estimate the columns and their sizes.

Step 3: Multiply by retention.

Step 4: Add overhead (indexes: 20-50% extra, replication: 2-3×, compression: 0.3-0.7×)
```

**Example: IAM audit events:**

```text
Write rate: 500,000 events/sec (peak), ~300,000/sec average
Daily writes: 300,000 × 86,400 = 25,920,000,000 ≈ 26B events/day

Row size estimate:
  event_id:       16 bytes  (UUID)
  event_type:     20 bytes  (varchar, e.g., "LOGIN_SUCCESS")
  actor_id:       16 bytes
  target_id:      16 bytes
  payload:        200 bytes (avg JSONB)
  ip_address:     16 bytes
  created_at:     8 bytes   (timestamptz)
  Total: ~292 bytes/row ≈ 300 bytes

Daily storage (raw): 26B × 300 = 7.8 TB/day
Compressed (0.5×):   3.9 TB/day
With 1 replica:      7.8 TB/day total

Retention: 7 years (regulatory)
  Hot (1 year): 7.8 TB × 365 ≈ 2.8 PB  → S3 Standard + columnar format (Parquet)
  Cold (6 years): → S3 Glacier (cheap, rarely accessed)
  
→ Architecture implication: cannot keep 7 years in PostgreSQL.
  Use PostgreSQL for last 90 days (hot queries), archive to S3 Parquet for compliance.
  Query compliance data via Athena (S3 SQL queries).
```

---

## Bandwidth Estimation

```text
Formula: bandwidth = QPS × avg_response_size

Example: Social feed API
  Read QPS: 100,000/sec
  Average response (20 posts with text): 50 KB
  
  Bandwidth out: 100,000 × 50,000 bytes = 5 GB/sec
  
  → This is enormous. Implications:
    1. CDN is mandatory (offload most reads from origin)
    2. Compression (gzip) reduces by ~3-5× → 1-1.7 GB/sec at origin
    3. Individual 10 Gbps NIC = 1.25 GB/sec → multiple servers or 25Gbps NICs needed
    4. AWS data transfer costs: ~$0.08/GB → 5 GB/sec × 86,400 sec × $0.08 = $34,560/day in egress alone
       (CDN caching with 90% cache hit rate: reduces to $3,456/day)

→ Capacity estimation reveals the CDN is not optional — it's a cost necessity.
```

---

## Cache Sizing

```text
Formula: cache_size = hot_dataset_size × cache_hit_ratio_target

How to estimate hot dataset size:
  - What fraction of data accounts for 80% of reads? (Pareto principle)
  - Usually: 20% of data handles 80% of reads (for social feeds, products, profiles)

Example: User profile cache
  Total users: 100M
  Profile size: 500 bytes/user
  Total dataset: 100M × 500 = 50 GB
  
  Hot dataset (20% of users account for 80% of reads): 10M users × 500 bytes = 5 GB
  
  To achieve 80% cache hit ratio: cache the hot 20% → 5 GB Redis
  To achieve 95% cache hit ratio: cache ~50% → 25 GB Redis
  To achieve 99% cache hit ratio: nearly full dataset → 50 GB Redis
  
  Redis memory overhead: ~overhead_factor of 2-3× raw data (pointers, metadata)
  5 GB hot data → ~10-15 GB Redis allocated memory
  
  → One Redis node (r7g.large = 16 GB) handles the 80% hit ratio target.
  → For 95%: r7g.xlarge (32 GB) or Redis Cluster with 3 × 16 GB = 48 GB.
```

---

## Kafka Partition Estimation

```text
Formula: partitions = max(target_throughput ÷ producer_throughput_per_partition,
                          target_throughput ÷ consumer_throughput_per_partition)

Kafka throughput per partition:
  Producer: ~10-50 MB/sec per partition (compression helps)
  Consumer: ~50 MB/sec per partition
  Events/sec per partition: ~10,000-50,000 (depending on event size)
  
  Safe rule of thumb: 1 partition per 10,000 events/sec (at ~1KB/event)

Example: Payment events (IAM audit)
  Target: 300,000 events/sec average
  Event size: 300 bytes
  
  Throughput: 300,000 × 300 = 90 MB/sec

  Partitions needed (producer-bound, 10 MB/sec per partition):
    90 MB/sec ÷ 10 MB/sec = 9 partitions
  
  Add headroom (1.5× buffer): 9 × 1.5 = 13.5 → 14 partitions
  Round to power of 2 for even distribution: 16 partitions

  Consumer parallelism:
    16 partitions → max 16 consumers in a consumer group
    If each consumer processes 20,000 events/sec, 16 consumers = 320,000 events/sec ✓

→ Operational: 3 Kafka brokers, 16 partitions, replication factor 3 (each partition on 3 brokers)
  Total partition replicas: 16 × 3 = 48 partition replicas across 3 brokers = 16 per broker
```

---

## DB Connection Pool Sizing

```text
Formula: connections = concurrent_requests × avg_latency_sec

Why: Little's Law. Concurrent requests in the system = throughput × avg service time.
If each DB query takes 10ms and you have 10,000 QPS:
  Concurrent DB queries = 10,000 × 0.010 = 100 concurrent connections

PostgreSQL practical limits:
  Sweet spot: 100-300 connections
  Max before overhead: ~500 connections
  Above 500: use PgBouncer (connection pooler)

PgBouncer pool sizing:
  PgBouncer maintains N server connections to PostgreSQL.
  API nodes maintain connections to PgBouncer (many, cheap).
  PgBouncer multiplexes app connections to a smaller pool of real DB connections.
  
  50 API nodes × 20 app connections = 1,000 app connections to PgBouncer
  PgBouncer → PostgreSQL: 100 server connections
  PostgreSQL handles 100 concurrent queries, not 1,000. Sustainable.
```

---

## Putting It All Together: IAM Platform Sizing

```text
Given:
  50M active users
  5,000 logins/sec peak
  500,000 token introspections/sec peak
  1,000 admin/provisioning actions/sec

Compute tier:
  Login: 5,000 req/sec, ~50ms per request
    Concurrent threads needed: 5,000 × 0.050 = 250 threads
    4 nodes × 64 threads each = 256 threads (tight; use 6 nodes)
    
  Token introspection: 500,000 req/sec, ~2ms per request
    Fully served from Redis — no dedicated compute tier, API gateway validates JWT inline

Data tier:
  Credentials DB (PostgreSQL):
    Write: 5,000 logins/sec × 1 DB write (session) = 5,000 writes/sec
    PostgreSQL r7g.2xlarge (8 vCPU, 64 GB): handles 15,000-20,000 writes/sec ✓
    Read replicas: 2 (for admin queries, non-critical reads)
    PgBouncer: 100 connections to PostgreSQL, 500 from API tier

  Token Cache (Redis):
    500,000 req/sec → Redis Cluster, 3 primary shards
    Each shard handles 167,000/sec (well within 1M/sec Redis capacity)
    Memory: 50M sessions × 200 bytes = 10 GB raw → ~30 GB with overhead
    3 × r7g.large (16 GB each) = 48 GB cluster → comfortable ✓

  Audit Log (Kafka + S3):
    1,000 events/sec (IAM actions) × 300 bytes = 300 KB/sec
    2 Kafka partitions with replication factor 3 → sufficient
    Debezium CDC → S3 archival → Athena for compliance queries

Monthly infrastructure estimate:
  6 × c7g.xlarge (API): 6 × $150 = $900
  2 × r7g.2xlarge (PostgreSQL primary + replica): 2 × $600 = $1,200
  3 × r7g.large (Redis Cluster): 3 × $150 = $450
  Kafka (MSK, 3 brokers): ~$500
  S3 + Athena: ~$200
  Total: ~$3,250/month at 50M users
```

---

*End of Chapter 79: Capacity Estimation Cheat Sheet*
# Part 1 Supplement: Real-Time Communication and Flexible Data Fetching

*These chapters extend Part 1 (Networking Foundations). They cover topics that appear constantly in system design interviews and production systems: the full spectrum of real-time communication patterns and GraphQL as an alternative API paradigm.*

---

## Chapter 8b: Short Polling and Long Polling

### Interview Question

> "Before WebSockets existed, how did web applications implement real-time updates? What are the tradeoffs, and when would you still choose polling today?"

---

### Why Should You Care?

Polling is not a legacy curiosity. It is still the right choice in many scenarios, particularly when clients are behind restrictive firewalls, when the update frequency is low, or when operational simplicity outweighs efficiency. Understanding polling — and its failure modes — lets you make the choice deliberately rather than defaulting to WebSockets for everything.

---

### Short Polling

**What it is**: The client sends a request to the server on a fixed timer interval. The server responds immediately, whether or not there is new data.

```text
Client                        Server
  │                              │
  │──── GET /notifications ─────→│
  │←─── 200 OK { data: [] } ─────│  (no new data)
  │                              │
  │  [wait 5 seconds]            │
  │                              │
  │──── GET /notifications ─────→│
  │←─── 200 OK { data: [] } ─────│  (still nothing)
  │                              │
  │  [wait 5 seconds]            │
  │                              │
  │──── GET /notifications ─────→│
  │←─── 200 OK { data: [evt] } ──│  (finally something)
```

**The problem**: At 1 request per 5 seconds per user, with 100,000 users:
```text
Request rate = 100,000 ÷ 5 = 20,000 requests/sec
99% of those requests return empty responses.
20,000 req/sec × 99% waste = 19,800 useless requests/sec.
Server CPU, DB queries, log lines — all for nothing.
```

**When to use short polling:**
- Update frequency is low (once per minute is fine)
- Simplicity is paramount
- Client is a simple script or CLI with no persistent connection support
- Behind extremely restrictive proxies/firewalls that kill long-lived connections
- Building a quick prototype

**Implementation (Spring Boot)**:
```java
@GetMapping("/notifications")
public ResponseEntity<List<Notification>> poll(
        @RequestParam(defaultValue = "0") long since) {
    List<Notification> events = notificationService.findSince(since);
    return ResponseEntity.ok(events);
}
```

Client calls this every 5 seconds with `since=<last_timestamp>`. Simple. No special infrastructure.

---

### Long Polling

**What it is**: The client sends a request. The server *holds the connection open* — not responding immediately — until new data is available or a timeout is reached. As soon as the server responds, the client immediately sends another request.

```text
Client                        Server
  │                              │
  │──── GET /notifications ─────→│
  │                              │  [server holds connection open]
  │                              │  [waits for new data... 15 seconds]
  │                              │  [new notification arrives]
  │←─── 200 OK { data: [evt] } ──│
  │                              │
  │──── GET /notifications ─────→│  (immediately reconnects)
  │                              │  [server holds... 22 seconds]
  │←─── 200 OK { data: [] } ─────│  (timeout, empty response)
  │                              │
  │──── GET /notifications ─────→│  (immediately reconnects)
```

The timeout response (empty data after N seconds) is a heartbeat — it proves the connection is still alive and triggers an immediate reconnect.

**Why it's better than short polling:**
- Near-real-time delivery (event delivered within 10-100ms of occurrence, not within the poll interval)
- Far fewer requests (one per event, plus one per timeout, vs continuous timer)
- Server controls the latency (can respond instantly when data arrives)

**The catch — server resource consumption:**

Each held connection occupies:
- A thread (thread-per-connection model): 100,000 users = 100,000 threads. JVM with 512 MB stack per thread = 50 GB RAM for threads alone. Infeasible.
- A file descriptor: 100,000 open connections requires OS tuning (`ulimit -n`)

**Solution**: Async/non-blocking server model. Each held connection is a suspended coroutine or callback — no dedicated thread. Spring WebFlux, Node.js, and Netty are designed for this.

```java
// Spring WebFlux: long polling endpoint, holds connection up to 30s
@GetMapping("/notifications")
public Mono<ResponseEntity<List<Notification>>> longPoll(
        @RequestParam long since) {
    return notificationService.waitForNew(since, Duration.ofSeconds(30))
        .map(events -> ResponseEntity.ok(events))
        .defaultIfEmpty(ResponseEntity.ok(List.of()))  // timeout → empty
        .timeout(Duration.ofSeconds(30), Mono.just(ResponseEntity.ok(List.of())));
}
```

**When to use long polling:**
- Need near-real-time but can't use WebSockets (restrictive proxies)
- Unidirectional: server pushes to client, client never pushes back
- Load is moderate (not millions of simultaneous connections)
- Chat applications used this pattern (Comet) before WebSockets existed

**IAM example**: An admin console polls for "is my bulk provisioning job done yet?" The job takes 10-120 seconds. Long polling: client connects, server holds until job completes or 60s timeout, client reconnects. User sees completion almost immediately when it happens, without hammering the server with short polls.

---

### Comparison: Polling vs Long Polling vs SSE vs WebSockets

```text
Technique        │ Server push? │ Bidirectional? │ Connections │ Overhead    │ Best for
─────────────────┼─────────────┼───────────────┼─────────────┼─────────────┼──────────────────────
Short Polling    │ No (pull)   │ No            │ Many/brief  │ High (waste)│ Low-freq updates, simplicity
Long Polling     │ Simulated   │ No            │ 1 per user  │ Medium      │ Near-RT, proxy-constrained
SSE              │ Yes (HTTP)  │ No (srv→clt)  │ 1 per user  │ Low         │ Notifications, live feeds
WebSockets       │ Yes         │ Yes           │ 1 per user  │ Lowest/msg  │ Chat, collab, gaming
```

---

### Failure Modes

**Short polling — server overload under scale**: 1M users polling every 2 seconds = 500,000 req/sec. Mostly empty. Fix: increase poll interval, or migrate to SSE/WebSockets.

**Long polling — connection draining on deployment**: Deploying a new version drains all held connections simultaneously. All clients reconnect at the same moment → thundering herd. Fix: rolling deployments with connection draining; stagger the reconnect with jitter.

**Long polling — proxy timeouts**: Corporate proxies often close idle connections after 30-60 seconds. A long polling connection held for 60 seconds is killed silently. The client doesn't know. Fix: server-side timeout shorter than the proxy timeout (e.g., 25 seconds), always returning a heartbeat response before the proxy kills the connection.

**Short polling — polling the wrong endpoint**: Client polls `/payments/status?id=123` every second. The handler queries the database every time. 1000 users × 1/sec = 1000 DB queries/sec for status checks. Fix: cache the status with a short TTL (5s); the polling is now served from Redis, not the DB.

---

### Staff-Level Thinking

Polling is often the correct choice when the update frequency is low. A payment status check that updates once every 30 seconds doesn't need a WebSocket. A polling interval of 10 seconds delivers updates with <10 second latency, uses no persistent connections, works through every proxy, and requires no special infrastructure.

The question is always: **"What is the acceptable latency for this update, and what does maintaining a persistent connection cost?"** When the latency requirement is seconds (not milliseconds) and connection count is large, polling with a smart interval is often better than the operational complexity of WebSockets.

---

## Chapter 8c: Server-Sent Events (SSE)

### Interview Question

> "Compare SSE to WebSockets. When would you choose SSE? What does it look like under the hood?"

---

### Why Should You Care?

SSE is the underrated middle ground between polling and WebSockets. For the most common real-time requirement — server pushing updates to a browser client — SSE is simpler to implement, works over standard HTTP/2, and auto-reconnects. Engineers who don't know SSE default to WebSockets even when SSE is a better fit.

---

### Plain English Explanation

SSE is a one-way, persistent HTTP connection where the server streams a sequence of events to the client indefinitely. The client cannot send data back over the SSE connection (it uses separate REST calls for that).

Think of it as a radio broadcast: the station (server) transmits continuously; your radio (client) receives. You can't broadcast back to the station over the same channel — but you can call them on the phone (a separate HTTP request) if you want to send something.

---

### Internal Mechanics

#### The SSE Protocol

SSE is plain HTTP. The response has `Content-Type: text/event-stream` and the body is never-ending:

```text
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

data: {"type":"payment_updated","id":"pay-123","status":"completed"}\n\n

data: {"type":"notification","message":"Your login was detected in Mumbai"}\n\n

id: 1700000045\n
data: {"type":"role_revoked","role":"admin"}\n\n

event: heartbeat\n
data: {}\n\n
```

**Event format** (each event separated by a blank line `\n\n`):
```text
id: <event-id>           (optional; client uses for reconnect resume)
event: <event-type>      (optional; custom event name, default is "message")
data: <payload>          (required; can be multi-line with multiple data: lines)
retry: 3000              (optional; tells client reconnect delay in ms)
```

#### Client Connection (Browser)

```javascript
const source = new EventSource('/sse/notifications');

source.addEventListener('message', (event) => {
    const data = JSON.parse(event.data);
    updateUI(data);
});

source.addEventListener('role_revoked', (event) => {
    // Custom event type
    logout();
});

source.onerror = (err) => {
    // Browser auto-reconnects; this fires on each reconnect attempt
    console.log('SSE connection error, reconnecting...');
};
// No close() needed; browser handles reconnect automatically
```

**Auto-reconnect**: The browser's `EventSource` API automatically reconnects when the connection is lost. If the server sent an `id:` field, the browser sends `Last-Event-ID: <id>` on reconnect. The server uses this to replay missed events.

#### Server Implementation (Spring Boot)

```java
@GetMapping(value = "/sse/notifications", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> streamNotifications(
        @RequestHeader(value = "Last-Event-ID", required = false) String lastEventId,
        Authentication auth) {
    
    String userId = auth.getName();
    
    return notificationService
        .streamForUser(userId, lastEventId)  // replay from lastEventId if reconnect
        .map(notification -> ServerSentEvent.<String>builder()
            .id(notification.getId())
            .event(notification.getType())
            .data(toJson(notification))
            .retry(Duration.ofSeconds(3))
            .build())
        .mergeWith(heartbeat());  // periodic heartbeat to keep proxies alive
}

private Flux<ServerSentEvent<String>> heartbeat() {
    return Flux.interval(Duration.ofSeconds(25))
        .map(tick -> ServerSentEvent.<String>builder()
            .event("heartbeat")
            .data("{}")
            .build());
}
```

#### SSE vs WebSockets: Choosing

```text
Choose SSE when:
  ✓ Server pushes to client only (notifications, live feeds, progress bars)
  ✓ HTTP/2 is available (multiplexes multiple SSE streams over one TCP connection)
  ✓ You want automatic reconnect with event replay built in
  ✓ Standard HTTP infrastructure (CDN, load balancers) without special config
  ✓ Client is a browser (native EventSource API)
  ✓ Simplicity: SSE is pure HTTP; no protocol upgrade, no WebSocket frames

Choose WebSockets when:
  ✓ Client ALSO sends data to server at high frequency (chat, collaborative editing, gaming)
  ✓ Sub-10ms bidirectional latency required
  ✓ Native (non-browser) client with WebSocket library support
  ✓ Binary data streaming (audio, video chunks)
```

**The key insight**: Most "real-time" features are actually server-to-client only. Live sports scores, notification bells, payment status updates, dashboard metrics — none of these require the client to send data continuously. SSE handles all of them more simply than WebSockets.

---

### Production Example: IAM Security Event Stream

When an admin is logged into the IAM console, security events must appear in near-real-time: failed login attempts, suspicious sessions, privilege escalations.

```text
Admin browser → SSE connection: GET /sse/security-events
  Authorization: Bearer <admin_token>

IAM Backend:
  1. Validate admin token (must have SECURITY_ANALYST role)
  2. Open SSE stream
  3. Subscribe to Kafka topic: iam.security-events
  4. Filter for events relevant to admin's scope (their tenant)
  5. Push each event to the SSE stream

Events received by browser (automatically displayed):
  { "type": "login_failure", "user": "user-123", "ip": "192.168.1.1", "timestamp": "..." }
  { "type": "suspicious_session", "user": "user-456", "reason": "new_country", "timestamp": "..." }

If the network drops:
  Browser EventSource auto-reconnects after 3 seconds.
  Sends Last-Event-ID: <last_received_event_id>
  Server replays events since that ID from Kafka (consumer seek to offset).
  No events missed.
```

---

### ASCII Diagram

```text
Browser                   IAM Server                    Kafka
   │                           │                           │
   │ GET /sse/security-events  │                           │
   │──────────────────────────→│                           │
   │                           │ subscribe: security-events│
   │                           │──────────────────────────→│
   │ 200 OK (stream starts)    │                           │
   │←──────────────────────────│                           │
   │                           │ event: login_failure      │
   │                           │←──────────────────────────│
   │ data: {login_failure}     │                           │
   │←──────────────────────────│                           │
   │                           │ event: suspicious_session │
   │                           │←──────────────────────────│
   │ data: {suspicious_sess}   │                           │
   │←──────────────────────────│                           │
   │ [heartbeat every 25s]     │                           │
   │←──────────────────────────│                           │
   │                           │                           │
   │ [network drop]            │                           │
   │ [EventSource: reconnect]  │                           │
   │ GET /sse/security-events  │                           │
   │ Last-Event-ID: evt-789    │                           │
   │──────────────────────────→│                           │
   │                           │ seek to offset after evt-789
   │                           │──────────────────────────→│
   │ 200 OK (stream resumes)   │                           │
   │←──────────────────────────│ replay missed events      │
```

---

### Common Failure Modes

**1. Proxy timeouts killing SSE connections**: Corporate HTTP proxies close idle connections after 60-90 seconds. A heartbeat sent every 25 seconds prevents this.

**2. HTTP/1.1 connection limit**: Browsers allow 6 concurrent connections per domain over HTTP/1.1. Three open SSE streams = half the browser's connection budget gone. Fix: use HTTP/2, which multiplexes all SSE streams over one TCP connection. No limit.

**3. Missing Last-Event-ID on reconnect**: Server doesn't use the `Last-Event-ID` on reconnect. Events during the disconnect are lost. Fix: always implement reconnect replay; store recent events in Redis with TTL.

**4. Memory leak in long-lived streams**: Server holds a reference to the SSE emitter for every connected client. When clients disconnect without the server knowing (browser tab closed), the server keeps the emitter alive. Fix: implement client heartbeat detection; remove dead emitters on timeout.

---

### Staff-Level Thinking

SSE is HTTP. That means everything that works for HTTP works for SSE: CDN edge caching of the stream is possible for public broadcast streams (same content to all clients), standard authentication headers, standard load balancer routing, standard TLS. WebSockets require special handling at every infrastructure layer. Choose the simpler protocol until you have a clear reason not to.

---

## Chapter 8d: WebRTC — Peer-to-Peer Communication

### Interview Question

> "How does video calling work in a browser? How does the connection get established if both users are behind NAT? Where does a server still play a role?"

---

### Why Should You Care?

WebRTC is the protocol underlying browser-based video calls (Google Meet, Jitsi), screen sharing, and peer-to-peer file transfer. Engineers designing any real-time audio/video feature — or any system where high-bandwidth, low-latency data flows between peers — need to understand WebRTC's connection establishment, the NAT traversal problem, and when a TURN relay server is unavoidable.

---

### Plain English Explanation

WebRTC (Web Real-Time Communication) enables **direct browser-to-browser** (or app-to-app) connections for audio, video, and data — without routing through a central server.

The key challenge: both browsers are usually behind NAT (Network Address Translation). They don't have public IP addresses. They can't simply connect to each other by IP. WebRTC solves this with a signalling server + ICE (Interactive Connectivity Establishment) protocol.

---

### Internal Mechanics

#### The Three Components

**1. Signalling**: How two peers find each other and exchange connection parameters. WebRTC does NOT define the signalling protocol — you implement it (WebSocket or REST).

**2. ICE (Interactive Connectivity Establishment)**: The protocol that finds a network path between two peers, including through NAT.

**3. DTLS/SRTP**: Security layer. All WebRTC media is encrypted end-to-end. Non-negotiable.

#### The Connection Establishment Flow

```text
Alice (Browser)              Signalling Server           Bob (Browser)
     │                              │                         │
     │  Create RTCPeerConnection    │                         │
     │  Gather ICE candidates       │                         │
     │  (my local IPs, STUN-derived │                         │
     │   public IP, TURN relay)     │                         │
     │                              │                         │
     │  Create SDP Offer            │                         │
     │  (codec preferences,         │                         │
     │   ICE candidates)            │                         │
     │──── sendOffer ──────────────→│                         │
     │                              │──── forwardOffer ──────→│
     │                              │                         │
     │                              │   Create SDP Answer     │
     │                              │   (Bob's ICE candidates)│
     │                              │←── sendAnswer ──────────│
     │←── forwardAnswer ────────────│                         │
     │                              │                         │
     │  ICE connectivity checks:    │                         │
     │  Try each candidate pair ────────────────────────────→ │
     │                              │              (direct P2P)│
     │←──────────────────────────────────────────────────────│
     │  Best path selected          │                         │
     │                              │                         │
     │←─────────── Audio/Video data (P2P, no server) ────────│
     │                              │                         │
     │  Signalling server no longer involved in media         │
```

#### NAT Traversal: STUN, TURN, ICE

**The problem**: Alice has private IP 192.168.1.5. Bob has private IP 10.0.0.3. Neither knows the other's public IP. They can't connect directly.

**STUN (Session Traversal Utilities for NAT)**: A public server that tells you your public IP and port as seen from outside your NAT.

```text
Alice → STUN server (public): "What is my public IP?"
STUN → Alice: "You are 203.0.113.10:54321"

Now Alice knows her public address and includes it in the ICE offer.
Bob does the same: learns his public IP.
They can try direct connection via public IPs.
```

**TURN (Traversal Using Relays around NAT)**: When direct connection fails (symmetric NAT, firewall), a TURN server relays all media.

```text
Without TURN (symmetric NAT, direct connection fails):
  Alice ─────X──── Bob  (NAT blocks it)

With TURN:
  Alice ──→ TURN server ──→ Bob
  (media flows through the relay, not peer-to-peer)
  Higher latency, server bandwidth cost, but always works
```

**ICE**: The algorithm that tries all possible connection methods (host, STUN-derived, TURN relay) and picks the best working path.

```text
ICE Candidate Priority:
  1. Host candidate: direct LAN connection (lowest latency)
  2. Server Reflexive (STUN): public NAT-traversed connection
  3. Relay (TURN): fallback relay (highest latency, always works)
```

#### SDP (Session Description Protocol)

The "offer" and "answer" are SDP documents describing what each peer supports:

```text
SDP Offer (simplified):
  v=0
  o=alice 123456 654321 IN IP4 192.168.1.5
  m=audio 9 UDP/TLS/RTP/SAVPF 111  (audio: codec 111 = opus)
  m=video 9 UDP/TLS/RTP/SAVPF 96   (video: codec 96 = VP8)
  a=ice-ufrag:abc123
  a=ice-pwd:xyz789
  a=candidate:1 1 UDP 2122260223 192.168.1.5 54321 typ host
  a=candidate:2 1 UDP 1686052607 203.0.113.10 54321 typ srflx raddr 192.168.1.5
  a=candidate:3 1 UDP 1 203.0.113.20 3478 typ relay raddr 203.0.113.10
```

SDP negotiation determines: codec (Opus for audio, VP8/VP9/H.264 for video), encryption keys, ICE credentials.

#### Data Channels

WebRTC is not just audio/video. `RTCDataChannel` allows arbitrary binary or text data to flow peer-to-peer:

```javascript
const dataChannel = peerConnection.createDataChannel("file-transfer");
dataChannel.send(fileBuffer);  // Binary peer-to-peer, no server
```

Use cases: file transfer, game state sync, collaborative document editing.

#### Topology Models

**Mesh (P2P)**: Every peer connects to every other peer. Works for small groups (≤4 participants). Each peer encodes and sends N-1 video streams. CPU explodes at scale.

```text
3 participants (mesh):
  Alice ←→ Bob ←→ Carol ←→ Alice
  (3 P2P connections, each encodes 2 outgoing streams)
```

**SFU (Selective Forwarding Unit)**: A server receives each participant's stream and forwards it to others without decoding. Scales to large meetings. Used by Google Meet, Jitsi, Twilio.

```text
100 participants via SFU:
  All participants → SFU server → All participants
  SFU forwards streams, doesn't decode.
  Each participant sends 1 stream to SFU.
  SFU sends N streams to each participant (or adaptive, based on capacity).
```

**MCU (Multipoint Control Unit)**: A server decodes all streams, mixes them into one composite video, re-encodes, and sends one stream to everyone. Lowest bandwidth to clients, highest server CPU. Used for recording and legacy systems.

---

### Production Example

A bank uses WebRTC for the KYC (Know Your Customer) video verification flow. A customer's browser connects directly to a bank agent's browser:

```text
Components:
  1. Signalling Server (Spring Boot WebSocket): exchanges SDP offer/answer + ICE candidates
  2. STUN server (public: stun.l.google.com or self-hosted coturn)
  3. TURN server (self-hosted coturn): required because ~15-20% of users are behind
     symmetric NAT or strict firewalls where direct P2P fails
  4. Recording server (SFU with recording capability): stores the KYC session as proof

Flow:
  Customer opens KYC URL → connects to signalling server
  Agent opens agent console → connects to signalling server
  Signalling pairs them: exchange SDP + ICE candidates
  WebRTC establishes P2P connection (85% of cases) or TURN relay (15%)
  Video call starts — audio/video flows P2P (low latency, encrypted E2E)
  Recording server receives a copy via SFU for compliance retention
```

---

### Common Failure Modes

**1. Forgetting TURN servers**: STUN-only configuration works for 70-80% of users. The remaining 20-30% (behind symmetric NAT, corporate firewalls) fail silently. Fix: always deploy TURN; monitor TURN relay usage rate.

**2. ICE candidate gathering latency**: STUN/TURN server is slow or unreachable. ICE gathering hangs for 10+ seconds. Fix: `iceCandidatePoolSize` pre-gathers candidates; set a gathering timeout.

**3. Signalling server scalability**: At scale, the signalling server handles session setup for all active calls. At 100k concurrent calls: signalling server is the bottleneck, not the media (which is P2P). Design signalling as a stateless cluster.

**4. NAT rebinding during mobile calls**: Mobile switches from WiFi to cellular. IP changes. WebRTC's ICE restart mechanism handles this — but must be explicitly triggered on network change detection, or the call drops.

---

### Staff-Level Thinking

WebRTC is the right tool for peer-to-peer media (video calls, voice, low-latency data). For anything that flows through your servers (API calls, chat messages, notifications), it is not the right tool — it adds ICE/STUN/TURN complexity for no benefit.

TURN server bandwidth cost is real: relaying HD video at 1.5 Mbps for 10,000 concurrent calls = 15 Gbps of relay bandwidth. Size your TURN fleet accordingly, and monitor the relay rate. A spike in TURN usage (from 15% to 80%) indicates a network change (ISP blocking UDP, new firewall rule) that's forcing more users to relay.

---

## Chapter 8e: GraphQL — Flexible Data Fetching

### Interview Question

> "When would you choose GraphQL over REST? What problems does it solve, and what new problems does it introduce? Design the GraphQL schema for an IAM platform."

---

### Why Should You Care?

GraphQL is now a mainstream API technology used by Facebook, GitHub, Shopify, Twitter, and many large financial systems. Engineers who can only design REST APIs are leaving a significant tool out of their kit. More importantly, the tradeoffs of GraphQL (flexibility vs complexity, N+1 query risk, caching complexity) appear directly in system design interviews.

---

### Plain English Explanation

GraphQL is a query language for APIs. Instead of the server deciding what data to return (REST), the **client specifies exactly what data it needs** in a typed query. The server returns exactly that — no more, no less.

```text
REST (server decides):
  GET /users/123         → { id, name, email, address, roles, lastLogin, createdAt, ... }
  (returns everything, even if you only wanted the name)
  
  GET /users/123/payments  → { payments: [...100 fields each...] }
  (separate request, separate round trip)

GraphQL (client decides):
  query {
    user(id: "123") {
      name                        ← only this field
      payments(last: 5) {         ← and this
        amount                    ← only this subfield
        status                    ← and this
      }
    }
  }
  
  → One request. Returns exactly { name, payments: [{amount, status} × 5] }
  → No extra fields. No second round trip.
```

---

### Internal Mechanics

#### Schema Definition Language (SDL)

A GraphQL schema is a strongly-typed contract:

```graphql
type Identity {
  id: ID!
  externalId: String
  displayName: String!
  status: IdentityStatus!
  roles: [Role!]!
  sessions(active: Boolean): [Session!]!
  auditEvents(limit: Int = 20, before: String): AuditEventConnection!
}

type Role {
  id: ID!
  name: String!
  permissions: [Permission!]!
}

type Session {
  id: ID!
  createdAt: DateTime!
  expiresAt: DateTime!
  ipAddress: String
  isActive: Boolean!
}

enum IdentityStatus {
  ACTIVE
  SUSPENDED
  DEPROVISIONED
}

type Query {
  identity(id: ID!): Identity
  identities(
    status: IdentityStatus
    role: String
    limit: Int = 20
    cursor: String
  ): IdentityConnection!
}

type Mutation {
  provisionIdentity(input: ProvisionIdentityInput!): Identity!
  grantRole(identityId: ID!, roleId: ID!): Identity!
  revokeRole(identityId: ID!, roleId: ID!): Identity!
  suspendIdentity(id: ID!, reason: String!): Identity!
}

type Subscription {
  securityEvent(tenantId: ID!): SecurityEvent!
}
```

#### Query Execution

```graphql
query AdminDashboard {
  identities(status: ACTIVE, limit: 10) {
    edges {
      node {
        id
        displayName
        status
        roles {
          name
        }
        sessions(active: true) {
          createdAt
          ipAddress
        }
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

The server resolves each field via **resolvers** — functions that know how to fetch a specific field's data.

#### The N+1 Problem

The most critical GraphQL performance issue:

```text
Query: identities(limit: 10) { roles { name } }

Naive resolver execution:
  1. Resolve identities: SELECT * FROM identities LIMIT 10  → 10 identities
  2. For each identity, resolve roles:
     SELECT * FROM roles JOIN identity_roles ON ... WHERE identity_id = ?  × 10
  
  Total: 11 queries (1 + 10). For 100 identities: 101 queries.
  This is O(n) queries for n identities — the N+1 problem.
```

**Solution: DataLoader (batching)**

```javascript
// DataLoader batches all role lookups for one request into one query
const roleLoader = new DataLoader(async (identityIds) => {
    // Called ONCE with all 10 identity IDs
    const roles = await db.query(
        'SELECT * FROM identity_roles WHERE identity_id = ANY($1)',
        [identityIds]
    );
    // Group by identity_id and return in same order as input
    return identityIds.map(id => roles.filter(r => r.identity_id === id));
});

// Resolver:
roles: (identity) => roleLoader.load(identity.id)
// All 10 load() calls are batched into ONE SQL query by DataLoader
```

DataLoader is non-negotiable for any GraphQL implementation with nested entity relationships.

#### Mutations

```graphql
mutation SuspendUser {
  suspendIdentity(id: "user-123", reason: "Suspicious activity detected") {
    id
    status
    displayName
  }
}
```

Returns the updated object — the client immediately gets the new state without a separate query.

#### Subscriptions (Real-time)

GraphQL subscriptions push data to clients over a persistent connection (WebSocket or SSE):

```graphql
subscription SecurityFeed {
  securityEvent(tenantId: "tenant-456") {
    eventType
    actorId
    targetId
    timestamp
    severity
  }
}
```

The server pushes each matching event to subscribed clients. Common implementation: WebSocket transport with a Kafka consumer backing the subscription resolver.

#### Introspection

GraphQL schemas are self-documenting. Any client can query the schema itself:

```graphql
{
  __schema {
    types { name }
    queryType { fields { name, type { name } } }
  }
}
```

This powers tools like GraphiQL (browser-based playground) and auto-generated SDK clients. The schema IS the API documentation.

---

### REST vs GraphQL Decision Guide

```text
Choose GraphQL when:
  ✓ Multiple client types (mobile, web, partner) with very different data needs
  ✓ Complex, deeply nested domain models (IAM: identities → roles → permissions → policies)
  ✓ Rapid client-driven product iteration (frontend can add fields without backend changes)
  ✓ Strong tooling investment (persisted queries, schema registry, codegen)
  ✓ Public API with diverse third-party consumers (GitHub, Shopify model)
  ✓ Team has GraphQL experience

Choose REST when:
  ✓ Simple CRUD operations with predictable data shapes
  ✓ Public API that must be maximally interoperable (every HTTP client speaks REST)
  ✓ Aggressive HTTP caching required (GraphQL POST requests are not cached by default)
  ✓ Team unfamiliar with GraphQL (learning curve is real)
  ✓ File upload heavy (GraphQL multipart is complex)
  ✓ Simple, stable client (one client type, data needs are well-understood)
```

---

### Production Example: IAM Admin Console

The IAM admin console has three client types:
- **Web console**: needs rich data (identity + roles + recent sessions + audit summary)
- **Mobile companion app**: needs minimal data (identity status + count of active sessions)
- **Partner API**: needs stable, specific fields for automated provisioning

```text
REST approach (pain points):
  Web console: GET /identities/123 + GET /identities/123/roles + GET /identities/123/sessions
               = 3 round trips, over-fetching (roles response includes unused fields)
  
  Mobile: same 3 requests, but uses 10% of the data returned
  
  Partner: requires a dedicated /v1/ API with versioned contracts

GraphQL approach:
  Web console query:
    { identity(id:"123") { displayName, status, roles { name }, sessions(active:true) { ip } } }
    → 1 request, exactly the data needed
    
  Mobile query:
    { identity(id:"123") { status, sessions(active:true) { id } } }
    → 1 request, 10 fields instead of 300
    
  Partner mutation:
    mutation { provisionIdentity(input: {...}) { id, status } }
    → Same schema, client picks what it needs
    
  All three served from ONE GraphQL endpoint.
  Adding a new field to the web query doesn't require a backend deployment.
```

---

### GraphQL Caching (The Hard Part)

REST: GET requests are cacheable by URL. HTTP layer caches automatically.

GraphQL: typically POST requests with the query in the body. URL never changes (`/graphql`). HTTP caching doesn't work out of the box.

Solutions:
- **Persisted queries**: hash the query, register it as a GET endpoint (`GET /graphql?queryId=abc123`). Now cacheable.
- **Client-side caching**: Apollo Client / Relay cache entity graphs in-memory by ID. Subsequent queries served from cache.
- **CDN for public GraphQL**: query allowlisting (only pre-approved queries allowed) + GET-based persisted queries enables CDN edge caching.
- **Field-level caching**: `@cacheControl(maxAge: 300)` directives on slow/expensive fields.

---

### Common Failure Modes

**1. N+1 queries without DataLoader**: The most common GraphQL production incident. A query for 50 users with their roles fires 51 database queries. Under load: 10,000 req/sec × 51 queries = 510,000 DB queries/sec. Database melts. Fix: DataLoader is mandatory, not optional.

**2. Overly complex queries**: A malicious or careless client submits a deeply nested query with high cost. Server executes it and saturates the database:
```graphql
{ identities { roles { permissions { identities { roles { ... } } } } } }
```
Fix: query depth limiting (max depth = 5), query complexity analysis (assign costs to fields, reject over-budget queries), query timeouts.

**3. Schema design leaking internals**: Designing the GraphQL schema as a thin wrapper over the database schema. The client can now construct queries that map 1:1 to expensive SQL joins. Fix: design the schema for the domain, not the database. Explicitly restrict which relationships are queryable.

**4. Subscription scaling**: WebSocket subscriptions for millions of concurrent users. Each subscription filter requires a topic subscription on the backing message bus. Fix: aggregate subscriptions per topic (not one Kafka consumer per connected client); use a subscription multiplexer.

---

### Staff-Level Thinking

GraphQL is a product-facing API technology. Its primary value is **developer experience for API consumers** — they get exactly what they need in one request, with full type safety and auto-complete in their IDE. The cost is **server-side complexity**: DataLoader, depth limiting, complexity analysis, caching strategy, schema governance.

For internal service-to-service APIs, gRPC is almost always better than GraphQL (typed, binary, streaming, faster). GraphQL's strengths shine at the boundary between your system and external consumers (including your own frontend team).

The schema is a public contract. Breaking changes (removing fields, changing types) require deprecation cycles. Invest in schema governance tools (schema registry, breaking-change detection in CI) before your GraphQL API has hundreds of consumers.

---

*End of Part 1 Supplement: Real-Time Communication and Flexible Data Fetching*
