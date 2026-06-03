# Case Study 31 — Service Mesh (Istio / Envoy)

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Sidecar pattern, Traffic management, Observability without code changes.

---

## Business Context

A payment platform with 30+ microservices has a cross-cutting concerns problem: every service needs mTLS, retry logic, circuit breaking, load balancing, distributed tracing, and rate limiting. Without a service mesh, each team re-implements these in every service. With a service mesh, these become infrastructure — configured once, applied everywhere, without changing service code.

The service mesh is the infrastructure layer that gives you zero-trust security (Case 27), observability (Case 29), and resilience patterns — all from configuration, not code.

**Business goals driving architecture:**
- Zero-code mTLS across all services (security without developer burden)
- Uniform retry, timeout, and circuit-breaker policies
- Traffic splitting for canary deployments (1% → 10% → 100%)
- Automatic distributed tracing headers propagation
- Unified traffic observability across 30+ services

---

## Recognition Framework

### What a Service Mesh Solves

```
Without service mesh (each team implements independently):
  Team A (payment-service): has retry logic; uses custom circuit breaker; no mTLS
  Team B (fraud-service): no retry; custom timeout; partial mTLS
  Team C (ledger-service): has mTLS; different retry strategy; no tracing
  
  Problems:
    Inconsistent resilience: one service missing retry = cascade failure
    Security gaps: some services do mTLS, others don't = partial zero trust
    No uniform observability: each service instruments differently
    Upgrade burden: change retry policy = update code in 30 services

With service mesh:
  Every service gets a sidecar proxy (Envoy) injected automatically
  Sidecar handles: mTLS, retries, circuit breaking, load balancing, tracing
  Service code: knows nothing about any of this (writes to localhost; sidecar handles rest)
  Policy change: update one config; applies to all services immediately

The Sidecar Pattern:
  Application container: runs service code; listens on :8080
  Envoy sidecar container: intercepts all inbound AND outbound traffic
  From application's view: it's just calling localhost:9080 (intercepted by Envoy)
  From Envoy's view: every call goes through it; can apply any policy

Data plane (Envoy sidecars): the proxies that handle actual traffic
Control plane (Istiod): distributes configuration to all sidecars; manages certificates
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| Sidecar Proxy (Envoy) | Zero-code networking; every service gets proxy injected by K8s |
| mTLS (transparent) | Zero-trust (Case 27) without service code changes |
| VirtualService / DestinationRule | Traffic management: canary, A/B, fault injection |
| Circuit Breaker | Envoy-level outlier detection; no library needed in service code |
| Retry + Timeout | Configured in YAML; consistent across all services |
| Istio Telemetry | Automatic distributed traces + metrics without instrumentation |

---

## Problem Statement

Service mesh for a 30-service payment platform. mTLS everywhere. Traffic management for canary deployments. Circuit breaking. Uniform observability. Zero service code changes.

---

## Functional Requirements
- mTLS between all services (transparent; no service code changes)
- Retry and timeout policies per service
- Circuit breaker (outlier detection)
- Canary deployments: route X% of traffic to new version
- Fault injection: inject latency/errors for testing
- Traffic observability: requests/errors/latency per service pair

## Non-Functional Requirements
- Sidecar overhead: **< 1ms additional latency** per hop (Envoy is extremely fast)
- Control plane update propagation: **< 10 seconds** to all sidecars
- mTLS: **automatic certificate rotation** (no ops burden)

---

## Core Components

```
Istio Control Plane (Istiod):
  Pilot:     distributes routing config (VirtualService, DestinationRule) to sidecars
  Citadel:   certificate authority; issues certs to each workload (SPIFFE SVIDs)
  Galley:    validates and distributes Istio config

Istio Data Plane (Envoy sidecars):
  Injected into every pod in labelled namespaces
  Handles: TLS termination, retries, circuit breaking, load balancing, observability
  
  Inbound: external request → Envoy intercepts → applies policy → forwards to :8080
  Outbound: service calls :9080 → Envoy intercepts → mTLS → forwards to destination Envoy
```

---

## Configuration Patterns

### mTLS (PeerAuthentication + RequestAuthentication)
```yaml
# Enable mTLS for entire namespace (all services must use mTLS)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: payments
spec:
  mtls:
    mode: STRICT    # STRICT: all traffic must be mTLS; PERMISSIVE: allow both
---
# Define allowed source services for fraud-service (authorization)
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: fraud-service-authz
  namespace: fraud
spec:
  selector:
    matchLabels:
      app: fraud-service
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/payments/sa/payment-service"]
    to:
    - operation:
        methods: ["POST"]
        paths: ["/api/v1/fraud/evaluate"]
  # All other traffic: implicitly denied (default deny)
```

### Traffic Management: Canary Deployment
```yaml
# VirtualService: route 5% to v2, 95% to v1 (canary deployment)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
  - payment-service
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: payment-service
        subset: v2
      weight: 100            # sticky: force-route canary header to v2
  - route:
    - destination:
        host: payment-service
        subset: v1
      weight: 95
    - destination:
        host: payment-service
        subset: v2
      weight: 5              # 5% of remaining traffic goes to v2
---
# DestinationRule: define subsets (v1 and v2)
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    outlierDetection:       # circuit breaker
      consecutive5xxErrors: 5          # open circuit after 5 consecutive errors
      interval: 30s                    # evaluation window
      baseEjectionTime: 30s            # eject unhealthy instance for 30s
      maxEjectionPercent: 50           # never eject more than 50% of instances
```

### Retry and Timeout Policy
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: fraud-service
spec:
  hosts:
  - fraud-service
  http:
  - route:
    - destination:
        host: fraud-service
    timeout: 150ms          # fraud check must complete in 150ms (payment SLA budget)
    retries:
      attempts: 2           # retry up to 2 times
      perTryTimeout: 50ms   # each attempt gets 50ms
      retryOn: "gateway-error,connect-failure,retriable-4xx"
      # NOT retrying on: 5xx from fraud (could be intentional BLOCK decision)
```

### Fault Injection for Chaos Testing
```yaml
# Inject 100ms delay on 10% of calls to fraud-service (test payment timeout handling)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: fraud-service-fault-test
spec:
  hosts:
  - fraud-service
  http:
  - fault:
      delay:
        percentage:
          value: 10.0
        fixedDelay: 100ms
    route:
    - destination:
        host: fraud-service

# Inject 5% HTTP 503 errors (test circuit breaker)
  - fault:
      abort:
        percentage:
          value: 5.0
        httpStatus: 503
```

---

## Detailed Components

### Envoy Sidecar Traffic Interception
```
How Envoy intercepts traffic without service code changes:
  1. K8s admission webhook: on pod creation, Istio injects Envoy container
  2. Init container: sets iptables rules to redirect ALL traffic through Envoy
     iptables -t nat -A OUTPUT -p tcp -j REDIRECT --to-port 15001
     (Envoy listens on 15001 for outbound; 15006 for inbound)
  3. Service code: calls "http://fraud-service:8080" → actually hits Envoy at 15001
  4. Envoy: applies policies, does mTLS handshake, forwards to destination Envoy
  5. Destination Envoy: terminates mTLS, applies inbound policy, forwards to :8080
  
  Service code never knows about Envoy; no SDK import; no config file change.
  The interception is at the iptables (network) layer, below the application.

Envoy listener for payment-service calling fraud-service:
  Outbound listener (0.0.0.0:15001):
    Route: fraud-service:8080 → cluster: fraud-service-cluster
    Cluster: fraud-service-cluster → endpoints: [pod IPs of fraud-service instances]
    TLS context: use SPIFFE SVID cert for mTLS
    Retry policy: 2 attempts, 150ms total
```

### Istio Observability (Automatic Metrics + Tracing)
```
Automatic metrics per service pair (no instrumentation needed):
  istio_requests_total{
    source_workload="payment-service",
    destination_workload="fraud-service",
    response_code="200",
    ...
  }
  → Request rate, error rate, latency per (source, destination, response_code)
  → Available in Prometheus immediately; pre-built Grafana dashboards

Automatic distributed tracing:
  Envoy injects/propagates W3C traceparent headers on every call
  BUT: service code must forward headers it receives to downstream calls
    Envoy cannot do this automatically (requires reading request body intent)
    Spring Boot OTEL starter: automatically forwards trace headers
    Without OTEL: manually forward: b3, x-request-id, traceparent
  
  Result: traces show full service graph without OTEL SDK (just header forwarding)

Kiali (Istio service map):
  Real-time visualization of service-to-service traffic
  Shows: request rate, error rate, latency per edge
  Highlights: traffic anomalies, configuration issues
  No configuration needed; derives from Envoy metrics
```

---

## Scaling Strategy
```
Phase 1 (5–10 services):
  Basic Istio install; permissive mTLS (allows non-mTLS for migration)
  Default retry/timeout policies
  Envoy metrics → Prometheus

Phase 2 (10–30 services):
  Strict mTLS (all traffic encrypted and authenticated)
  Per-service DestinationRules (circuit breakers, connection limits)
  AuthorizationPolicies (which services can call which)
  Canary deployment automation

Phase 3 (30+ services, multi-cluster):
  Multi-cluster mesh (services in different K8s clusters can call each other)
  Egress gateway: all external calls (card network, NPCI) go through egress gateway
  Ingress gateway: all inbound traffic through single controlled entry point
  Global traffic management (route to nearest healthy region)

Phase 4 (ambient mesh):
  Istio ambient mode: removes per-pod sidecar (lower CPU/memory overhead)
  Uses node-level ztunnel (L4) + waypoint proxy (L7) instead
  Better for: high-density pod deployments; teams sensitive to sidecar overhead
```

---

## Reliability + Observability
```
Control plane HA:
  Istiod: 3 replicas; if all fail → sidecars continue with last config (fail-safe)
  New pods: won't get config until Istiod recovers (brief period; acceptable)

Sidecar overhead:
  CPU: ~0.5% extra per request (Envoy is C++ and extremely optimised)
  Memory: ~50 MB per Envoy sidecar (300 services × 50 MB = 15 GB fleet-wide)
  Latency: < 1ms per hop (Envoy's P99 overhead is typically 0.5–0.8ms)

Key metrics:
  istio_requests_total (by source/dest/code)
  istio_request_duration_milliseconds_p99
  pilot_k8s_cfg_events_total (config propagation rate)
  envoy_cluster_upstream_cx_active (connection pool health)
```

---

## Chaos Testing
```
Experiment 1: Service-to-Service Call Blocked by AuthorizationPolicy
  Inject:    New service (test-service) tries to call fraud-service without policy
  Expected:  AuthorizationPolicy: no matching rule → 403 RBAC denied
  Expected:  Audit log: unauthorized access attempt from test-service
  Red flag:  Unauthorized service gets through (default allow instead of default deny)

Experiment 2: Canary Deployment Traffic Split
  Inject:    Deploy payment-service v2; VirtualService set to 5% v2
  Expected:  Approximately 5% of requests routed to v2 (verify via Prometheus labels)
  Expected:  Error rate on v2 visible separately from v1
  Expected:  Rollback: change weight to 0% v2 → all traffic back to v1 (< 5s)
  Red flag:  v2 receives 0% or 100% traffic (weight not applied correctly)

Experiment 3: Circuit Breaker Opens
  Inject:    fraud-service returns 503 for 5 consecutive calls
  Expected:  Outlier detection: fraud-service pod ejected for 30s
  Expected:  payment-service: calls fail fast (circuit open) → fail-open to ALLOW
  Expected:  After 30s: pod re-admitted; circuit closes; requests resume
  Red flag:  Circuit never opens; all 5xx retried until timeout
```

---

## Monthly Cost Estimate
```
Istio control plane (Istiod, 3 replicas):
  3 × m5.large ($75/mo)                             = $225/mo

Envoy sidecar memory overhead:
  300 pods × 50 MB sidecar × $0.004/GB-hr × 720hr = $432/mo (memory cost)
  CPU: 300 pods × 0.01 vCPU extra × $0.04/hr × 720hr = $864/mo

Prometheus extra metrics (Istio golden signals):
  +30% metrics volume vs app-only: ~$500/mo additional

Total service mesh overhead: ~$2,021/month
Value: replaces N custom retry/circuit-breaker libraries; uniform mTLS; canary deployments.
```

---

## Architecture Review Checklist
```
□ SCC LENS:
  STATE: Envoy xDS cache (routing config), Istio secret store (certs), K8s etcd (policy)
  COORDINATION: Istiod pushes xDS config to all Envoys on change (< 10s propagation)
  CONCENTRATION: Istiod is the config distribution hub → 3 HA replicas mandatory

□ Failure mode?
  → Istiod down: sidecars keep last known config; no new config pushes; new pods: no mesh
  → Envoy sidecar OOM: K8s restarts sidecar; brief traffic drop for that pod
  → mTLS cert expiry: Istiod renews before expiry; if renewal fails → traffic drops

□ Key trade-offs:
  Pro: zero-code mTLS, observability, retry, circuit breaking
  Con: sidecar overhead (memory + CPU + latency); complex debugging (traffic layer added)
  → Envoy logs show full request path; Kiali shows service graph
  → Debug: istioctl proxy-config routes <pod> to see Envoy routing config
```

---

*Next: Case Study 32 — Kubernetes Platform Engineering*
