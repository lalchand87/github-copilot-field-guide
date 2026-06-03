# Case Study 27 — Zero Trust Network Architecture

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: SPIFFE/SPIRE, mTLS everywhere, Policy engine (OPA), BeyondCorp.

---

## Business Context

Traditional perimeter security assumes "inside the network is safe." Zero Trust assumes "never trust, always verify." Every request — whether from a customer, an employee, or a microservice — must prove its identity and be authorised before accessing any resource. This matters especially for a payment platform where a compromised internal service can mean unrestricted access to financial data.

The seminal paper is Google's BeyondCorp (2014), which eliminated VPNs for Google employees. The key insight: network location is not a proxy for trust. A laptop on the corporate network and a laptop in a coffee shop should have the same trust level — which is zero until they prove identity.

**Business goals driving architecture:**
- Eliminate "implicit trust" from internal network
- Every service-to-service call is authenticated and authorised
- Lateral movement after a breach is minimised (one compromised service ≠ everything is compromised)
- Compliance: PCI-DSS, SOC2 require access controls on all internal traffic
- Observability: know exactly which service called which service with which identity

---

## Recognition Framework

### Zero Trust Principles

```
Principle 1: Verify explicitly
  Every access request must present credentials
  Never assume identity from network location (IP address ≠ identity)
  
  Implementation: SPIFFE/SPIRE for service identity; OIDC for user identity
  No more: "this service is on the internal network, so it must be us"

Principle 2: Use least privilege access
  Each service can only access the resources it needs for its job
  "Payment service can call fraud service, but not user profile service"
  
  Implementation: OPA (Open Policy Agent) policies per service pair
  Granular: not just "can access" but "can call specific endpoints with specific methods"

Principle 3: Assume breach
  Design as if a service is already compromised
  Limit blast radius: compromised payment service cannot access HR data
  
  Implementation: network microsegmentation; per-service DB credentials; audit log

How it maps to your IAM background:
  BeyondCorp = your CIAM experience applied to internal services
  SPIFFE = OIDC for machines (service identity ≡ user identity in OIDC)
  OPA = ABAC for service-to-service (same as user-to-resource ABAC)
  mTLS = mutual authentication (client presents cert ≡ user presents JWT)
```

### SPIFFE/SPIRE — Service Identity

```
SPIFFE (Secure Production Identity Framework for Everyone):
  Defines a standard for service identity: the SPIFFE ID
  Format: spiffe://{trust_domain}/{path}
  Example: spiffe://payment-platform.internal/payment-service
  
  Properties:
    Cryptographically verifiable (X.509 certificate or JWT-SVID)
    Short-lived (certificates expire in 1 hour → auto-rotated)
    Workload-bound (tied to Kubernetes pod identity, not IP)
    Trust domain separation (dev, staging, prod are separate trust domains)

SPIRE (SPIFFE Runtime Environment):
  The reference implementation that issues and rotates SPIFFE IDs
  Components:
    SPIRE Server: the CA; issues SVIDs (SPIFFE Verifiable Identity Documents)
    SPIRE Agent: runs on every node; provides SVIDs to workloads on that node
    Workload API: gRPC endpoint workloads use to get their SVID

How it works:
  1. payment-service starts in K8s pod
  2. SPIRE agent on the node attests the pod's identity (K8s API + pod UID)
  3. SPIRE agent fetches SVID from SPIRE server for: spiffe://payment/payment-service
  4. SVID is an X.509 cert: subject = spiffe://payment/payment-service; TTL = 1 hour
  5. payment-service presents this cert in every outbound TLS connection
  6. Receiving service (e.g. fraud-service) validates: cert is signed by SPIRE CA
     AND subject = expected SPIFFE ID for payment-service
  7. Connection is mutually authenticated (both sides proved identity)
  8. SPIRE agent auto-rotates cert every 1 hour (transparent to service)

Why better than static TLS certificates:
  Static cert: 1-year TTL; stored in secrets; rotated manually (often forgotten)
  SPIRE SVID: 1-hour TTL; auto-rotated; revocation is implicit (cert expires)
  No secret sprawl: services never store long-lived credentials
```

---

## Problem Statement

Zero trust network architecture for a payment platform. Service-to-service mTLS via SPIFFE/SPIRE. Fine-grained authorisation via OPA. Employee access without VPN (BeyondCorp-lite). Network microsegmentation.

---

## Functional Requirements
- Service-to-service authentication via mTLS (SPIFFE X.509 SVIDs)
- Fine-grained authorisation: service A can call service B endpoint C but not D
- Employee access to internal tools without VPN (BeyondCorp device trust)
- Audit log: every service-to-service call logged with identities
- Certificate rotation: automatic, zero-downtime, every 1 hour
- Revocation: compromise detected → SPIRE can stop issuing certs for a workload

## Non-Functional Requirements
- mTLS overhead: **< 1ms** added latency per connection (after TLS session resumption)
- Certificate rotation: **zero-downtime** (connection reuse during rotation)
- Policy evaluation: **< 1ms** (OPA in-process; not network call)
- Zero VPN dependency for employee access

---

## Core Patterns
| Pattern | Why |
|---|---|
| SPIFFE/SPIRE | Cryptographic service identity; auto-rotating certs |
| mTLS | Mutual authentication; both sides prove identity |
| OPA (Envoy ext_authz) | Per-call authorisation policy; fine-grained access control |
| Service Mesh (Istio/Envoy) | mTLS transparent to application; sidecar handles TLS |
| Network Policy (K8s) | Network-level microsegmentation; defence in depth |
| BeyondCorp (Access Proxy) | Employee access without VPN; device posture check |
| Audit Log (OPA decision log) | Every access decision logged; compliance evidence |

---

## Architecture

```
Service-to-Service Zero Trust:

┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  payment-service              fraud-service                         │
│  ┌─────────────────┐          ┌─────────────────────────────┐       │
│  │  App Code       │          │  App Code                   │       │
│  │  (no TLS code)  │          │  (no TLS code)              │       │
│  ├─────────────────┤          ├─────────────────────────────┤       │
│  │  Envoy Sidecar  │──mTLS──►│  Envoy Sidecar             │       │
│  │  SPIFFE SVID:   │          │  Validates:                 │       │
│  │  payment-svc    │          │  - cert signed by SPIRE CA  │       │
│  └────────┬────────┘          │  - subject = payment-svc    │       │
│           │                   │  OPA check:                 │       │
│    SPIRE Agent                │  - can payment-svc call     │       │
│    (issues SVID)              │    /evaluate ?              │       │
│                               └─────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────────┘

Flow:
  1. payment-service calls fraud-service → goes to Envoy sidecar (transparent)
  2. Envoy uses SPIFFE SVID (auto-fetched from SPIRE agent) for mTLS
  3. fraud-service's Envoy terminates mTLS → validates cert (is it really payment-service?)
  4. Envoy calls OPA (via ext_authz): "can spiffe://payment/payment-service call POST /evaluate?"
  5. OPA evaluates policy → ALLOW or DENY
  6. If ALLOW: request forwarded to fraud-service app
  7. All decisions logged to audit (OPA decision log → Kafka → Case 10)
```

### OPA Policy for Service-to-Service
```rego
# fraud_service_authz.rego
package fraud_service.authz

import future.keywords.in

# Default: deny all
default allow := false

# Allow payment-service to call fraud evaluate
allow {
    input.caller_spiffe_id == "spiffe://payment-platform.internal/payment-service"
    input.method == "POST"
    input.path == "/api/v1/fraud/evaluate"
}

# Allow fraud-service to call itself (internal callbacks)
allow {
    input.caller_spiffe_id == "spiffe://payment-platform.internal/fraud-service"
}

# Allow monitoring to scrape metrics
allow {
    input.caller_spiffe_id == "spiffe://payment-platform.internal/prometheus"
    input.method == "GET"
    input.path == "/metrics"
}

# DENY: payment-service cannot call admin endpoints
# (no rule matches → default deny applies)
# This means:
#   payment-service → POST /evaluate → ALLOW
#   payment-service → GET /admin/model → DENY (no matching rule)
#   unknown-service → any → DENY
```

### BeyondCorp for Employee Access
```
Traditional VPN model:
  Employee → VPN → any internal tool
  Problem: VPN credentials stolen → attacker has full internal access

BeyondCorp model:
  Employee → Access Proxy → specific tool
  Access Proxy checks:
    1. Identity: is this a valid employee? (OIDC from IdP: Okta/Azure AD)
    2. Device posture: is this device managed? (device cert from MDM)
    3. Context: is this normal access time/location for this user?
    4. Authorization: does this user's role allow this tool?
  
  Implementation:
    Cloudflare Access / Google IAP (Identity-Aware Proxy) / self-built
    Every internal tool is behind the proxy: Grafana, internal APIs, admin consoles
    No VPN required; employee authenticates per-tool, not per-network
    
  Device trust:
    MDM (Mobile Device Management) installs device certificate
    Access Proxy requires: OIDC token (who you are) + device cert (what device)
    BYOD devices get read-only access to non-sensitive tools; full access requires managed device

Example: engineer accessing Grafana
  1. Engineer opens grafana.internal.company.com
  2. Access proxy intercepts; redirects to Okta OIDC login
  3. Engineer authenticates with MFA → JWT token
  4. Proxy checks: JWT claims include "engineering" group → access to Grafana allowed
  5. Proxy checks: device has valid MDM certificate → device is managed → full access
  6. Proxy issues short-lived (1h) token for Grafana
  7. Access logged: who accessed Grafana at what time from which device
```

### Network Microsegmentation (Defence in Depth)
```yaml
# Kubernetes NetworkPolicy: payment-service can only talk to specific services
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-service-egress
  namespace: payments
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
  - Egress
  egress:
  # Allow: payment-service → fraud-service
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: fraud
    - podSelector:
        matchLabels:
          app: fraud-service
    ports:
    - port: 8080
      protocol: TCP

  # Allow: payment-service → PostgreSQL
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: data
    ports:
    - port: 5432
      protocol: TCP

  # Allow: DNS
  - to:
    ports:
    - port: 53
      protocol: UDP

  # DENY ALL OTHER EGRESS (implicit; no rule = deny)
  # payment-service cannot call: HR service, auth admin, user-service (no rule)
  # Even if mTLS credential is valid: network policy blocks at lower level

# This provides defence in depth:
# mTLS + OPA: application layer isolation
# NetworkPolicy: network layer isolation
# If Envoy/OPA is bypassed: network policy still blocks
```

---

## Detailed Components

### mTLS Certificate Lifecycle
```
SPIRE Server (1 cluster-wide; HA with Raft):
  Issues SVIDs (certificates) to SPIRE Agents
  Validates agent attestation (K8s API: does this pod really exist?)
  Certificate chain: SPIRE Root CA → SPIRE Intermediate CA → SVID

SPIRE Agent (1 per K8s node):
  Attests workloads: is this pod who it claims to be? (K8s pod UID + SA token)
  Fetches SVID from SPIRE Server on behalf of workload
  Serves SVID to workload via Unix socket (Workload API)
  Rotates SVID before expiry (every 45 min for 1h cert)

Envoy integration:
  Envoy configured with SDS (Secret Discovery Service) endpoint
  Envoy fetches certificate from SPIRE Agent via SDS
  Envoy uses cert for TLS handshake on every new connection
  On cert rotation: Envoy gets new cert via SDS; existing connections use old cert until close
  Zero-downtime rotation: TLS session resumption + graceful connection drain

Connection reuse:
  After TLS handshake: HTTP/2 multiplexing reuses the connection
  TLS handshake overhead (1ms) paid once; subsequent requests on same connection: 0.1ms
  payment-service → fraud-service: connection pooled; not re-established per request
```

### OPA Performance (In-Process)
```
OPA can run in two modes:
  1. Sidecar/External (network call):
     Envoy → HTTP call to OPA sidecar → decision
     Latency: 5–20ms per call (network + OPA eval)
     Problem: adds latency to every request
  
  2. In-process (embedded):
     OPA compiled to WebAssembly or Go library → runs in Envoy process
     Latency: < 1ms (in-process evaluation; no network)
     
  3. Envoy ext_authz → OPA sidecar (chosen for most cases)
     Envoy's ext_authz: first request → check OPA; cache decision for connection
     Cache: same SPIFFE ID + endpoint → cached for 60s
     Effective latency: 0.1ms (cache hit); 5ms (cache miss)
     
Policy distribution:
  OPA policies are YAML files stored in Git
  OPA Bundle Server serves policies as bundles
  Each OPA instance polls bundle server every 60s for updates
  Policy update propagation: < 60s globally
  New policy takes effect without restart (hot-reload)
```

---

## Scaling Strategy
```
Phase 1 (baseline; no zero trust):
  VPN for engineers
  Implicit trust within network
  Shared DB credentials

Phase 2 (basic mTLS):
  Istio service mesh (mTLS by default)
  Basic RBAC (all services in same role can call each other)
  No SPIFFE/SPIRE yet (Istio manages certs)

Phase 3 (full zero trust):
  SPIFFE/SPIRE (replace Istio cert management)
  OPA policies per service pair (fine-grained)
  BeyondCorp for engineer access (eliminate VPN)
  Network policies in K8s

Phase 4 (mature zero trust):
  Cross-cluster trust (SPIRE federation between prod/staging/dev)
  Just-in-time access for production (engineers get temp elevated access; auto-revoke)
  Continuous posture monitoring (anomaly detection on service call patterns)
  SASE (Secure Access Service Edge) for branch offices
```

---

## Reliability Strategy
- **SPIRE HA**: SPIRE Server in HA mode (Raft consensus; 3 nodes); agent continues to serve cached SVIDs if server unreachable for < 5 min
- **OPA policy failure mode**: on OPA timeout → deny by default (fail-closed); NOT fail-open (unlike rate limiter)
- **Certificate rotation**: auto-rotate 15 min before expiry; if rotation fails → alert; service degraded but not down (cert still valid for 45 min)
- **mTLS overhead**: HTTP/2 multiplexing means TLS handshake is one-time per connection; subsequent calls within connection add ~0.1ms

---

## Security Considerations (IAM Platform Tie-in)
- **SPIFFE = machine identity**: same concept as user identity in OIDC; just for services
- **OPA = service-to-service ABAC**: same as user-to-resource ABAC; subject is SPIFFE ID not user ID
- **ForgeRock integration**: ForgeRock AM can be the IdP for BeyondCorp access proxy (OIDC)
- **Credential minimisation**: no service has username/password for DB; K8s service account tokens + Vault dynamic secrets
- **Lateral movement prevention**: even if payment-service is compromised, attacker cannot call HR service (NetworkPolicy + OPA)
- **Audit completeness**: OPA decision log + mTLS access log → full trace of "what called what with what identity"

---

## Observability
```
Zero Trust specific metrics:
  mtls_handshake_failures_total           (cert validation failures; alert > 0)
  spire_svid_rotation_failures_total      (cert rotation issues; alert > 0)
  opa_policy_evaluation_latency_p99       (target < 1ms in-process; < 10ms sidecar)
  opa_policy_deny_rate by {caller, callee} (detect unexpected denials)
  opa_policy_updates_total                (track policy deployment)
  beyondcorp_access_denied_rate           (employee access denials; investigate spikes)
  tls_certificate_expiry_seconds          (alert if < 3600s remaining on any cert)

Security metrics:
  inter_service_calls_by_spiffe_id        (baseline; anomalies = possible compromise)
  new_service_identity_detected           (unknown SPIFFE ID → alert immediately)
  certificate_chain_validation_errors     (MITM attempt indicator)
```

---

## Chaos Testing
```
Experiment 1: SPIRE Server Failure
  Inject:    Kill SPIRE Server (all 3 HA nodes)
  Expected:  SPIRE Agents cache existing SVIDs; services continue with cached certs
  Expected:  No new cert rotations for ~45 min (until certs near expiry)
  Expected:  Alert fires immediately: "SPIRE Server unreachable"
  Expected:  SPIRE Server recovers (Raft restores); rotations resume
  Red flag:  Services start failing after SPIRE outage (not using cached certs)

Experiment 2: Unauthorized Service-to-Service Call
  Inject:    Configure HR service to call payment-service's admin endpoint
             (no OPA policy exists for this pair)
  Expected:  OPA: no matching rule → default deny → 403 Forbidden
  Expected:  Deny logged: "HR service denied access to payment admin endpoint"
  Expected:  NetworkPolicy also blocks (defence in depth)
  Red flag:  Call succeeds (OPA policy missing; default is allow)
  Note: DEFAULT MUST BE DENY in OPA for zero trust to work

Experiment 3: Certificate Rotation Under Load
  Inject:    Trigger forced SVID rotation while payment-service is processing 10K TPS
  Expected:  SPIRE agent provides new cert; Envoy picks up via SDS
  Expected:  Existing HTTP/2 connections continue on old cert until natural close
  Expected:  New connections use new cert
  Expected:  Zero dropped requests during rotation; zero latency spike
  Red flag:  10K requests fail during cert rotation (connections not properly drained)

Experiment 4: BeyondCorp Device Trust Verification
  Inject:    Engineer attempts to access Grafana from unmanaged (personal) laptop
  Expected:  Access proxy: OIDC succeeds (valid employee); device cert missing → access denied
  Expected:  Access denied message: "Device not enrolled in MDM; use a managed device"
  Expected:  Access attempt logged with device fingerprint
  Red flag:  Unmanaged device gets full access (device cert not checked)
```

---

## Monthly Cost Estimate
```
SPIRE Infrastructure:
  3 × SPIRE Server (m5.large, $75/mo)               = $225/mo
  (Agents run as DaemonSets; no additional cost)

Envoy Service Mesh:
  Sidecar containers: ~10% CPU overhead per pod
  At 100 pods × 0.1 vCPU × $0.04/vCPU-hr × 720hr   = $288/mo
  (Istio control plane: 3 × t3.medium $34/mo)         = $102/mo

OPA Bundle Server:
  2 × t3.small ($15/mo)                              = $30/mo

BeyondCorp Access Proxy:
  Cloudflare Access: ~$3/user/mo × 100 engineers     = $300/mo
  OR self-hosted: 2 × c5.large ($130/mo)              = $260/mo

Monitoring for ZT metrics:
                                                      = $100/mo
─────────────────────────────────────────────────────────────────
Total: ~$1,045/month (using Cloudflare Access)

Zero Trust infrastructure is remarkably cheap relative to the security value.
The real investment is engineering time: designing policies, migrating services.
ROI: preventing one breach (avg cost $4.2M per IBM report) >> years of ZT infrastructure cost.
```

---

## Architecture Review Checklist
```
□ Zero Trust principles verified:
  □ Every service-to-service call is authenticated (mTLS with SPIFFE SVID)
  □ Every service-to-service call is authorised (OPA policy; default deny)
  □ Employee access is identity-based, not network-based (BeyondCorp)
  □ Credentials are short-lived (1h SVIDs; auto-rotated)
  □ No implicit trust from network location

□ Defence in depth:
  □ Application layer: OPA authorization
  □ Transport layer: mTLS (both sides prove identity)
  □ Network layer: K8s NetworkPolicy (block at IP level)
  □ Three independent controls; all must be bypassed for lateral movement

□ Blast radius minimisation:
  □ Compromised payment-service: can it call HR? → No (NetworkPolicy + OPA)
  □ Can it escalate DB privileges? → No (DB role is read-only for non-admin)
  □ Can it exfiltrate audit logs? → No (audit log service requires specific SPIFFE ID)

□ SCC LENS:
  STATE: SPIRE (cert issuance state), OPA (policy state), audit log (access decisions)
  COORDINATION: cert rotation (SPIRE auto-manages), policy updates (OPA bundle pull)
  CONCENTRATION: SPIRE Server is a critical dependency → HA (Raft 3 nodes) mandatory

□ IAM PLATFORM TIE-IN (your core strength):
  SPIFFE = OIDC for machines (same token model; different subject type)
  OPA = ABAC for services (same as user-level ABAC you know from Zanzibar/ReBAC)
  BeyondCorp IdP = ForgeRock AM (OIDC issuer for employee identity)
  Just-in-time access = PAM pattern (CyberArk/Vault) applied to internal services
  This case study is where your IAM expertise most directly applies to infrastructure
```

---

## Staff Engineer Discussion Points

**"How is SPIFFE different from just issuing TLS certificates from a corporate CA?"**
Three key differences: (1) Workload binding — SPIFFE certificates are tied to the workload identity (Kubernetes pod, VM instance), not just "a service called payment-service." SPIRE validates the actual running workload before issuing. A corporate CA issues based on who asks, not who is running. (2) Auto-rotation — SPIFFE SVIDs expire in 1 hour and auto-rotate. Corporate CA certs are typically 1–2 years; rotation requires manual processes and often gets forgotten. (3) Trust federation — SPIFFE defines how different trust domains (prod/staging; company A/company B) can federate to verify each other's identities without sharing a CA. Corporate CAs don't have a standard federation model.

**"Why does Zero Trust matter more for a payment platform than for a generic SaaS?"**
Financial data is the highest-value target. A breach of your payment platform exposes card numbers, account balances, and transaction histories — immediately monetisable by attackers. The lateral movement attack pattern is: compromise any internet-facing service → move to internal services → reach the payment database. Zero Trust specifically breaks the lateral movement step: even if the attacker owns a compromised service, they cannot call the payment DB service (OPA denies + NetworkPolicy blocks) because the compromised service's SPIFFE ID doesn't have a policy for that call. Contrast with a generic SaaS: data breach is bad, but not as immediately monetisable as payment data.

---

## Cheat Sheet Tie-in

```
ZERO TRUST PRINCIPLES:
  1. Verify explicitly: every call carries cryptographic identity (SPIFFE SVID)
  2. Least privilege: OPA policy (default deny); only what's explicitly permitted
  3. Assume breach: NetworkPolicy limits blast radius even if OPA is bypassed

SPIFFE/SPIRE MECHANICS:
  SPIFFE ID: spiffe://{trust_domain}/{workload_path}
  SVID: X.509 cert; subject = SPIFFE ID; TTL = 1h; auto-rotated by SPIRE agent
  Validation: cert signed by SPIRE CA → binds identity to cryptographic proof
  
OPA PATTERN:
  Default: deny all (not allow all)
  Policy: allow {explicit conditions}
  Evaluated per-call; result cached 60s; decision logged to audit

BEYONDCORP (employee access without VPN):
  Old: employee → VPN → all internal tools (stolen VPN = game over)
  New: employee → Access Proxy → specific tool (per-app auth; device posture check)
  Identity: OIDC from IdP (ForgeRock AM / Okta)
  Device: MDM certificate (managed device required for sensitive tools)

SCC LENS:
  STATE: SPIRE (cert state), OPA (policy state), audit log (decision state)
  COORDINATION: short-lived certs force periodic re-validation; OPA policy pull every 60s
  CONCENTRATION: SPIRE Server → HA (Raft); every service depends on cert issuance

IAM TIE-IN (the key bridge between your IAM expertise and systems design):
  User IAM → Machine IAM:
    OIDC tokens (users) ≡ SPIFFE SVIDs (services)
    RBAC (users) ≡ OPA service policies (services)
    MFA (users) ≡ mTLS (services) — proof of possession of a key/cert
    ForgeRock AM (user IdP) ≡ SPIRE (service IdP)
  
  Your IAM knowledge is directly transferable:
    "How does a service prove identity?" → SPIFFE SVID (X.509 cert from SPIRE)
    "How is service-to-service access controlled?" → OPA ABAC policy
    "How does a new service get credentials?" → SPIRE agent attestation
    This IS IAM, just for machines instead of humans
```

---

*Next: Case Study 28 — Event-Driven Architecture with CQRS and Event Sourcing*
