# Case Study 35 — API Gateway Design (Advanced)

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Edge auth vs service auth, Aggregation layer, BFF pattern.

---

## Business Context

Case Study 05 covered the basic API gateway. This case study covers advanced patterns needed for a mature payment platform: Backend-for-Frontend (BFF) for different client types, GraphQL aggregation, request/response transformation, and the hard problem of where authentication happens (edge vs service).

A payment platform has many clients: mobile app (iOS/Android), merchant web dashboard, merchant REST API, partner webhooks, internal tools. Each has different needs — mobile needs compressed aggregated responses; merchant API needs strict versioning; internal tools need broader access.

**Business goals driving architecture:**
- Mobile app: single API call returns everything needed (reduce round trips)
- Merchant API: stable versioned contracts; backward compatibility
- Partner integrations: separate auth and rate limits
- Edge auth: JWT validated once at gateway, not in every service

---

## Recognition Framework

### Why One API Gateway is Not Enough

```
Single gateway for all clients (naive):
  Mobile app: needs lightweight responses; different auth (mobile tokens)
  Merchant API: needs stable versioning; REST with strict contracts
  Merchant dashboard (browser): needs session auth; CORS; aggregated views
  Partner webhook consumers: needs webhook verification; async response model
  
  Problem:
    API contract changes for merchant API → breaks mobile (different release cycle)
    Mobile payload optimization → bloats merchant API response
    Adding CORS headers for browser → irrelevant for partner API
    One gateway becomes a mega-config that no one understands

Backend for Frontend (BFF) pattern:
  Separate gateway per client type:
    Mobile BFF: optimized responses; mobile push; offline sync
    Merchant Dashboard BFF: session auth; CORS; rich queries; aggregation
    Merchant REST API BFF: stable versioning; API key auth; strict contracts
    Partner API BFF: webhook auth; async; partner-specific rate limits
  
  Each BFF: owned by the team that owns that client experience
  Each BFF: calls the same underlying microservices
  Shared: authentication infrastructure, rate limiting, observability
```

---

## Problem Statement

Advanced API gateway layer for a payment platform. BFF pattern for multiple clients. Edge authentication. Request aggregation (reduce mobile round trips). GraphQL for dashboard. API versioning strategy.

---

## Architecture

```
Clients:
  Mobile App (iOS/Android)     → Mobile BFF        → Underlying microservices
  Merchant Dashboard (Browser) → Dashboard BFF      → Underlying microservices
  Merchant REST API            → Public API Gateway → Underlying microservices
  Partner Integrations         → Partner API Gateway→ Underlying microservices
  Internal Tools               → Internal Gateway   → Underlying microservices

Shared infrastructure (Kong / Nginx / Envoy-based):
  TLS termination, DDoS protection, IP allowlisting → all BFFs use this
  Rate limiting (per API key, per tier) → all use Case 25 rate limiter
  JWT validation → all use same JWKS endpoint
  Observability → all BFFs emit to same Prometheus

The BFFs are separate deployable services:
  mobile-bff: Kotlin + Spring Boot; handles mobile-specific logic
  dashboard-bff: Node.js; serves dashboard-specific GraphQL queries
  public-api-gateway: Kong or AWS API Gateway; Merchant REST API
  partner-api: separate Kong instance; different rate limits and auth
```

---

## Edge Authentication Pattern

```
Where does JWT validation happen?

Option A: Every microservice validates JWT
  Pros: defense in depth; no single point of trust
  Cons: every service imports JWT library; JWKS cache per service;
        service gets full token; must implement claims checking
  When to use: high-security; zero-trust; when services need token claims

Option B: Gateway validates JWT; forwards claims as headers
  Gateway: validates JWT signature (JWKS), extracts claims, forwards:
    X-User-ID: u-12345
    X-User-Tier: ENTERPRISE
    X-Scopes: payments:create payments:read
    X-Tenant-ID: tenant-abc
  Services: trust gateway-injected headers (no JWT library needed)
  Risk: if a request bypasses gateway, service trusts fake headers
  Mitigation: network policy (services only accept traffic from gateway/service mesh)
  When to use: standard setup; services behind service mesh mTLS

CHOSEN: Gateway validates JWT + Service mesh as backup
  Gateway (Kong): validates JWT, strips original Authorization header,
                  injects X-User-ID etc. as verified headers
  Services: read from X-User-ID headers (no JWT library)
  Service mesh (Istio): rejects requests NOT originating from gateway (source SPIFFE ID check)
  Defence in depth: both layers must be bypassed for unauthorized access
```

---

## Mobile BFF (Request Aggregation)

```
Problem: Mobile app checkout flow needs:
  1. Payment methods (GET /payment-methods)
  2. User balance (GET /accounts/{id}/balance)
  3. Recent transactions (GET /transactions?limit=5)
  4. Current offers (GET /offers)
  
  4 serial round trips on mobile = 4 × (network latency + server latency)
  On 4G: 100ms × 4 = 400ms just for network
  
Solution: Mobile BFF aggregates in parallel:
  1. Client: POST /api/mobile/v1/checkout-context
  2. Mobile BFF: parallel calls to all 4 services (CompletableFuture.allOf)
  3. Mobile BFF: merge, transform, compress (gzip or protobuf)
  4. Client receives single response with all data in < 200ms

Aggregation code pattern:
```java
@GetMapping("/checkout-context")
public CheckoutContext getCheckoutContext(@RequestHeader("X-User-ID") String userId) {
    // All 4 calls in parallel
    CompletableFuture<List<PaymentMethod>> paymentMethodsFuture =
        CompletableFuture.supplyAsync(() -> paymentService.getPaymentMethods(userId));
    
    CompletableFuture<Balance> balanceFuture =
        CompletableFuture.supplyAsync(() -> ledgerService.getBalance(userId));
    
    CompletableFuture<List<Transaction>> transactionsFuture =
        CompletableFuture.supplyAsync(() -> transactionService.getRecent(userId, 5));
    
    CompletableFuture<List<Offer>> offersFuture =
        CompletableFuture.supplyAsync(() -> offerService.getActiveOffers(userId));
    
    CompletableFuture.allOf(paymentMethodsFuture, balanceFuture, 
                            transactionsFuture, offersFuture).join();
    
    return CheckoutContext.builder()
        .paymentMethods(paymentMethodsFuture.join())
        .balance(balanceFuture.join())
        .recentTransactions(transactionsFuture.join())
        .activeOffers(offersFuture.join())
        .build();
}
```

---

## Dashboard BFF: GraphQL

```graphql
# Merchant dashboard queries are flexible: sometimes needs payment list,
# sometimes payment + merchant + user details, sometimes just totals.
# REST: separate endpoints or over-fetching; GraphQL: exact data per query.

type Query {
  merchant(id: ID!): Merchant
  payments(
    merchantId: ID!
    dateFrom: String
    dateTo: String
    status: PaymentStatus
    first: Int
    after: String
  ): PaymentConnection
  dashboardSummary(merchantId: ID!, period: DateRange): DashboardSummary
}

type Payment {
  id: ID!
  amount: Int!
  currency: String!
  status: PaymentStatus!
  customer: Customer      # resolved separately; not always needed
  merchant: Merchant      # resolved separately
  createdAt: DateTime!
}

# N+1 problem: DataLoader batches customer lookups
# Without DataLoader: 100 payments → 100 GET /customers/{id} calls
# With DataLoader: 100 payments → 1 GET /customers?ids=id1,id2,...,id100

type Mutation {
  issueRefund(paymentId: ID!, amount: Int!, reason: String!): RefundResult
  updateWebhookUrl(merchantId: ID!, url: String!): Merchant
}
```

---

## API Versioning Strategy

```
Problem: merchant REST API v1 exists; want to add breaking change → v2
Options:

Option A: URL versioning (CHOSEN for public API)
  /api/v1/payments  →  /api/v2/payments
  Pros: clear; cacheable; visible in logs
  Cons: URL changes; clients must update

Option B: Header versioning
  Accept: application/vnd.platform.v2+json
  Pros: URL stable; content negotiation
  Cons: hidden; harder to debug; browser can't do it

Option C: Query param versioning
  /api/payments?version=2
  Cons: not RESTful; pollutes query params; not cacheable

CHOSEN: URL versioning for REST API; date-based versioning for webhooks

Version lifecycle:
  v1: generally available (GA)
  v2: beta (breaking changes; opt-in for early access)
  v3: alpha (experimental; no SLA)
  
  Deprecation policy:
    v1 deprecated: announce 12 months ahead; sunset header in every response
    Sunset header: "Sunset: Sat, 01 Jan 2026 00:00:00 GMT"
    After sunset: v1 returns 410 Gone with migration guide URL

Kong API versioning (upstream routes):
  Route: /api/v1/* → upstream: payment-service-v1
  Route: /api/v2/* → upstream: payment-service-v2
  Both versions run simultaneously during transition period
```

---

## Partner API (Webhook Authentication)

```
Partners receive webhooks (payment events → partner's server).
How does partner verify the webhook is really from us?

HMAC Webhook Signature (same as Case 21):
  1. Platform signs payload: signature = HMAC-SHA256(payload, partner_secret)
  2. Platform includes: X-Signature-256: sha256=abc123... in request
  3. Partner verifies: compute HMAC with their secret → compare
  4. If match: genuine; if not: reject

Partner API key auth (incoming requests FROM partner):
  Partner has: API key + secret (like AWS access key + secret key)
  Request signing: every request signed with HMAC using their secret
  Gateway verifies: recomputes HMAC; validates timestamp (prevent replay within 5 min)
  
Rate limits for partners (per-partner, not per-IP):
  BRONZE partner: 100 req/min; 10K/day
  SILVER partner: 1,000 req/min; 100K/day
  GOLD partner: 10,000 req/min; unlimited daily
```

---

## Observability
```
Gateway metrics per BFF:
  requests_total by {bff, route, status_code}
  upstream_latency_p99 by {bff, upstream_service}
  auth_failures by {reason: expired_token, invalid_signature, insufficient_scope}
  rate_limit_hits by {client_tier, limit_type}
  aggregation_timeout_count (mobile BFF parallel calls; timeout on one service)

API versioning metrics:
  api_version_usage by {version, endpoint}   (detect v1 usage before sunset)
  sunset_warning_requests                     (how many clients still on deprecated version)
```

---

## Architecture Review Checklist
```
□ BFF isolation:
  Mobile BFF code change → only mobile clients affected
  Merchant REST API change → only merchant API clients affected
  Each BFF independently deployable; different tech stacks possible

□ Edge auth correctness:
  JWT validation at gateway: signature + expiry + issuer check
  Forwarded headers: X-User-ID, X-Tenant-ID, X-Scopes — stripped from inbound requests
  Network policy: services only accept from known sources (service mesh SPIFFE ID)
  Services never accept X-User-ID from unknown sources

□ Aggregation resilience:
  Mobile BFF parallel calls: timeout per upstream (50ms fraud, 100ms ledger)
  If one upstream times out: return partial response (degrade gracefully)
  User sees: checkout loads without offers (better than no response)

□ SCC LENS:
  STATE: API gateway config (routes, rate limit rules), JWKS (JWT validation keys)
  COORDINATION: JWT validation (centralized at edge), rate limiting (Redis, Case 25)
  CONCENTRATION: Gateway is the single ingress → HA + global CDN + DDoS protection mandatory
```

---

*Next: Case Study 36 — Idempotency Patterns (Deep Dive)*
