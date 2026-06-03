# Case Study 43 — Fine-Grained Authorization (OPA, ReBAC, Zanzibar)

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: RBAC vs ABAC vs ReBAC decision, OPA policy design, Google Zanzibar model.

---

## Business Context

Course-grained authorization ("is Alice an ADMIN?") works for simple systems. A payment platform with 100M users, 10M merchants, and complex permission hierarchies (merchant admin can see payments but not refund; support agent can view but never modify; partner can access only their own data) needs fine-grained authorization that is:
- Consistent: same policy enforced at every service
- Auditable: every access decision logged with reason
- Performant: authorization check must not add > 5ms to request latency
- Maintainable: policy changes don't require code deployments

**Business goals:**
- "Can user Alice issue a refund for payment pay-123?" → answer in < 5ms
- Policy changes take effect within 60 seconds across all services
- Every authorization decision logged for compliance
- Support Google-scale (Zanzibar) relationship-based authorization

---

## Recognition Framework

### RBAC vs ABAC vs ReBAC

```
RBAC (Role-Based Access Control):
  Model: user has ROLE; ROLE has PERMISSIONS
  Example: Alice has role MERCHANT_ADMIN → can create payments, view reports
  
  Strengths: simple to understand; easy to implement; well-known
  Weaknesses:
    Role explosion: need MERCHANT_ADMIN_WITH_REFUND_BUT_NOT_VOID → infinite roles
    No context: MERCHANT_ADMIN can see ALL merchants (not just their own)
    No resource-level: cannot say "Alice can see THIS payment but not THAT one"
  
  Good for: simple systems; < 100 roles; no resource-level permissions needed

ABAC (Attribute-Based Access Control):
  Model: policy evaluates attributes of subject (user) + resource + environment
  Example: ALLOW IF user.role == MERCHANT_ADMIN AND payment.tenant_id == user.tenant_id
           AND payment.amount < user.payment_limit AND environment.business_hours == true
  
  Strengths: very expressive; context-aware; handles complex conditions
  Weaknesses: policy can become very complex; hard to explain to users what they can do;
              performance (each check may require fetching multiple attributes)
  
  Good for: complex conditional access; regulatory compliance; payment limits by context

ReBAC (Relationship-Based Access Control — Google Zanzibar model):
  Model: access based on RELATIONSHIP graph between users and resources
  Example: Alice has OWNER relationship to payment pay-123
           Owner of payment CAN issue refund FOR that payment
           → Alice can refund pay-123 (not all payments; only ones she owns)
  
  Strengths: scales to billions of resources; intuitive (same as social graph permissions);
             handles delegation and inheritance naturally
  Weaknesses: more complex to implement than RBAC; requires relationship store
  
  Good for: multi-tenant SaaS; resource ownership; delegation; Google Docs-style sharing

CHOSEN for payment platform: layered approach
  RBAC for: coarse-grained system roles (ADMIN, MERCHANT, SUPPORT)
  ABAC for: payment-specific conditions (amount limits, time-of-day, tenant isolation)
  ReBAC for: resource ownership and delegation (Alice's payments vs Bob's payments)
```

---

## Problem Statement

Fine-grained authorization for a payment platform. OPA for ABAC policy enforcement. OpenFGA (Zanzibar-based) for relationship-based resource authorization. Unified authorization across all services.

---

## OPA (Open Policy Agent) — ABAC Layer

### Policy Design
```rego
# payment_authz.rego

package platform.payment.authz

import future.keywords.in

# Default: deny everything (secure by default)
default allow := false

# ─────────────────────────────────────────
# RULE: Merchant admin can create payments
# ─────────────────────────────────────────
allow {
    input.action == "payment:create"
    input.user.role == "MERCHANT_ADMIN"
    input.user.tenant_id == input.resource.tenant_id   # only within their tenant
    input.resource.amount <= input.user.payment_limit  # within their limit
}

# ─────────────────────────────────────────────────────
# RULE: Merchant can view their OWN payments (not others)
# ─────────────────────────────────────────────────────
allow {
    input.action == "payment:read"
    input.user.role in {"MERCHANT_ADMIN", "MERCHANT_VIEWER"}
    input.user.tenant_id == input.resource.tenant_id
}

# ─────────────────────────────────────────────────────
# RULE: Support agents can VIEW any payment (read-only)
# ─────────────────────────────────────────────────────
allow {
    input.action == "payment:read"
    input.user.role == "SUPPORT_AGENT"
    # No tenant restriction: support can view across tenants (with audit log)
    # But: support cannot modify (no create/refund/void rules for SUPPORT_AGENT)
}

# ─────────────────────────────────────────────────────
# RULE: High-value refunds require elevated auth
# ─────────────────────────────────────────────────────
allow {
    input.action == "payment:refund"
    input.user.role == "MERCHANT_ADMIN"
    input.user.tenant_id == input.resource.tenant_id
    input.resource.amount <= 100000     # refunds ≤ ₹1,000 → normal admin
}

allow {
    input.action == "payment:refund"
    input.user.role == "MERCHANT_ADMIN"
    input.user.tenant_id == input.resource.tenant_id
    input.resource.amount > 100000      # refunds > ₹1,000 → require step-up MFA
    input.user.mfa_verified_at > (time.now_ns() - 5*60*1e9)  # MFA within last 5 min
}

# ─────────────────────────────────────────────────────
# DENY: Any other combination → default false applies
# ─────────────────────────────────────────────────────

# Audit metadata: always output decision context
reason := r {
    allow
    r := {"action": input.action, "user_id": input.user.id, "resource_id": input.resource.id}
}
reason := r {
    not allow
    r := {"action": input.action, "user_id": input.user.id, "denied_at": time.now_ns()}
}
```

### OPA Integration
```java
@Component
public class OpaAuthorizationService {

    private final OpaClient opaClient;

    public AuthorizationResult authorize(AuthorizationRequest req) {
        // Build OPA input document
        Map<String, Object> input = Map.of(
            "action",   req.getAction(),             // "payment:refund"
            "user",     Map.of(
                "id",              req.getUserId(),
                "role",            req.getUserRole(),
                "tenant_id",       req.getTenantId(),
                "payment_limit",   req.getPaymentLimit(),
                "mfa_verified_at", req.getMfaVerifiedAt()
            ),
            "resource", Map.of(
                "id",        req.getResourceId(),    // "pay-abc-123"
                "type",      req.getResourceType(),  // "payment"
                "tenant_id", req.getResourceTenantId(),
                "amount",    req.getResourceAmount()
            )
        );

        // OPA decision (in-process via Wasm module for < 1ms; or HTTP for < 5ms)
        OpaDecision decision = opaClient.query("platform.payment.authz.allow", input);

        // Audit: EVERY decision logged
        auditLog.record(AuditEvent.authorization(
            req.getUserId(), req.getAction(), req.getResourceId(),
            decision.isAllowed(), decision.getReasons()
        ));

        return AuthorizationResult.of(decision.isAllowed(), decision.getReasons());
    }
}
```

---

## OpenFGA (Zanzibar-Based ReBAC) — Relationship Layer

### Why ReBAC for Payment Resource Ownership

```
Question: "Can user Alice view payment pay-123?"

ABAC approach: add rule "user can view payment IF user.id == payment.created_by"
Problem: "AND if Alice is the merchant admin for the merchant who received the payment"
         "AND if Alice has been delegated viewer access by the merchant admin"
         "AND if Alice is in a team that has access to the merchant account"
  → ABAC policy becomes a graph traversal in disguise; better to model it explicitly as a graph

ReBAC approach (Google Zanzibar / OpenFGA):
  Define relationship tuples:
    "user:alice is viewer of payment:pay-123"
    "user:alice is owner of merchant:hdfc"
    "user:owner of merchant:* is viewer of all payments owned by that merchant"
  
  Query: can alice view pay-123?
    → Is there a path: alice → (relationship) → pay-123?
    → Check: alice is owner of hdfc; hdfc received pay-123; owner has view access
    → Answer: YES
  
  Scales to: 100M users, 1B resources, delegation chains of arbitrary depth
  Google Zanzibar: powers Google Drive sharing, Google Cloud IAM, YouTube visibility
```

### OpenFGA Schema
```yaml
# Authorization model (define relationships)
model
  schema 1.1

type user

type merchant
  relations
    define owner: [user]
    define admin: [user] or owner
    define viewer: [user] or admin

type payment
  relations
    define merchant: [merchant]
    define creator: [user]
    define viewer: [user] or creator or admin from merchant
    define refunder: [user] or admin from merchant
    # "admin from merchant" means: if you are admin of the payment's merchant → you are refunder

type support_ticket
  relations
    define customer: [user]
    define assignee: [user]
    define viewer: [user] or customer or assignee or viewer from payment
    # Support agent assigned to ticket can view the associated payment
```

```java
// Write relationship tuples (when merchant admin is assigned)
openFgaClient.write(List.of(
    new TupleKey("user:alice", "admin", "merchant:hdfc-001"),   // Alice is admin of HDFC
    new TupleKey("merchant:hdfc-001", "merchant", "payment:pay-123")  // pay-123 belongs to HDFC
));

// Authorization check: can Alice refund pay-123?
CheckResponse response = openFgaClient.check(
    "user:alice",    // who
    "refunder",      // relation
    "payment:pay-123"  // on what
);
// OpenFGA resolves:
//   alice is admin of hdfc-001
//   pay-123 has merchant hdfc-001
//   admin from merchant → refunder (from schema)
//   → YES, Alice can refund pay-123

// Delegation: Alice grants Bob viewer access to pay-123
openFgaClient.write(List.of(
    new TupleKey("user:bob", "viewer", "payment:pay-123")  // direct delegation
));
// Now Bob can view pay-123 without being a merchant admin
// This is not possible cleanly in RBAC or ABAC
```

---

## Layered Authorization at the Gateway

```
Request: POST /api/v1/payments/pay-123/refund
  User: Alice (JWT: role=MERCHANT_ADMIN, tenant_id=hdfc-001, mfa_verified=true)
  Resource: pay-123 (amount=₹5,000)

Layer 1: Coarse RBAC (gateway; < 0.1ms)
  Does Alice have a role that CAN perform refunds?
  Roles that can refund: {MERCHANT_ADMIN, PLATFORM_ADMIN}
  Alice is MERCHANT_ADMIN → passes Layer 1

Layer 2: ABAC Policy (OPA; < 2ms via in-process Wasm)
  OPA input: action=payment:refund, user=Alice, resource=pay-123
  Policy evaluates:
    → Alice.tenant_id (hdfc-001) == pay-123.tenant_id (hdfc-001)? YES
    → amount (5,000) > 100,000? NO → standard refund; no step-up required
  OPA decision: ALLOW

Layer 3: ReBAC Check (OpenFGA; < 5ms)
  Can Alice refund pay-123?
  OpenFGA resolves relationship graph: Alice admin of hdfc-001 → pay-123 merchant is hdfc-001 → ALLOW

All three layers pass → Alice can refund pay-123
If ANY layer denies → 403 Forbidden

This layering:
  Layer 1 (RBAC): eliminates 95% of unauthorized requests cheaply
  Layer 2 (ABAC): handles complex business conditions (amounts, time, tenant)
  Layer 3 (ReBAC): handles resource ownership and delegation
  Total: < 8ms authorization overhead per request
```

---

## Policy Update and Distribution

```
OPA Bundle Server (policy distribution):
  Policies stored in Git (version controlled, auditable)
  Bundle server serves policy bundles: tar.gz of .rego files
  Each OPA instance: polls bundle server every 60 seconds
  Policy update: merge to Git → bundle server → all OPAs → takes effect within 60s
  
  Hot reload: no restart needed; OPA loads new policy while serving traffic
  Rollback: revert Git commit → old bundle served → OPAs revert within 60s

OpenFGA relationship tuples:
  Written at user/resource creation time (via application code)
  Deprovisioning: when Alice's SCIM deprovisioning arrives:
    DELETE all tuples where subject = "user:alice"
    Immediate effect: all Alice's permissions removed
  
  Tuple consistency:
    OpenFGA: strongly consistent within a region (Raft-based storage)
    Cross-region: eventual consistency (< 1s for relationship tuple replication)
    For check queries: read from local replica (slightly stale OK; millisecond-level lag)
```

---

## Observability + Compliance
```
Authorization metrics:
  authz_decisions_total by {action, decision, policy_layer}
  authz_latency_p99 by {policy_layer}          (OPA: target < 2ms; OpenFGA: target < 5ms)
  policy_bundle_update_lag_seconds             (time from Git commit to OPA update)
  open_fga_tuple_count                         (relationship store size; capacity planning)
  authz_deny_rate by {action, user_role}       (unusual deny rates → misconfiguration)
  
Compliance audit:
  Every authorization decision → audit log (Case 10)
  Audit record: user_id, action, resource_id, decision, policy_version, timestamp
  Regulatory query: "show all access to payment pay-123" → audit log query
  "Who had access to HDFC merchant data on 2024-01-15?" → time-range query + decision=ALLOW
```

---

## Architecture Review Checklist
```
□ Authorization model selection:
  RBAC: roles; coarse-grained; simple; use as first gate (cheapest check)
  ABAC: OPA policies; context-aware; use for business rules (amounts, tenant, time)
  ReBAC: OpenFGA; resource ownership; use for resource-level delegation

□ Default deny at every layer:
  OPA: default allow := false (explicit; code-checked)
  OpenFGA: no tuple = no access (all resources start with zero access)
  NEVER: default allow (even accidentally via misconfiguration)

□ Performance:
  OPA Wasm module (in-process): < 1ms
  OPA HTTP sidecar: 5-10ms (acceptable if cached)
  OpenFGA check: < 5ms (relationship graph cached in memory)
  Total overhead: < 10ms (well within most API SLOs)

□ Policy vs code:
  Authorization logic → OPA policies (not if-statements in service code)
  If-statements in service code: hard to audit, test, update consistently
  OPA: testable (opa test), auditable (decision log), hot-reloadable

□ SCC LENS:
  STATE: OPA policy bundle (Git + bundle server), OpenFGA tuple store (relationships)
  COORDINATION: OPA polling (60s bundle refresh), OpenFGA Raft (tuple consistency)
  CONCENTRATION: OPA in-process (no single node), OpenFGA cluster (sharded by resource type)

□ IAM EXPERTISE TIE-IN (your deepest area):
  OPA / Rego = the policy language; you likely use this in ForgeRock context
  Zanzibar / OpenFGA = Google's authorization model; maps to complex IAM graphs
  RBAC + ABAC + ReBAC together = production-grade authorization at scale
  This case study is where your IAM background most directly becomes Staff-level differentiator
  Interview answer to "how do you design authorization for a payment platform" = this
```

---

*Next: Case Study 44 — Multi-Factor Authentication (MFA) Systems*
