# Case Study 26 — Multi-Tenant SaaS Architecture

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Tenant isolation models, Data residency, Noisy neighbour.

---

## Business Context

A multi-tenant SaaS platform serves thousands of businesses (tenants) from shared infrastructure. Each tenant believes they have a dedicated system — their data is private, their performance is predictable, their configuration is independent. The engineering challenge is delivering that experience at 1/100th the cost of actually providing dedicated infrastructure.

For a payment platform: enterprise bank (Tenant A) cannot see retail startup's (Tenant B) data. A viral moment at Tenant B cannot degrade Tenant A's checkout experience.

**Business goals driving architecture:**
- Strong tenant isolation (data and performance)
- Tenant-aware pricing: small tenants on shared; large tenants on dedicated
- Data residency: EU tenants' data stays in EU (GDPR)
- Onboarding: new tenant live in minutes (not days of provisioning)
- Operational simplicity: one codebase; not N dedicated deployments

---

## Recognition Framework

### Tenant Isolation Models

```
Model 1: Shared Everything (Lowest Isolation, Lowest Cost)
  DB:     Shared DB; tenant_id column on every table; Row-Level Security (RLS)
  Compute: Shared application servers
  Cache:  Shared Redis; tenant key prefixes
  
  Pros: cheapest; simplest; instant onboarding
  Cons: noisy neighbour (one tenant's heavy query affects others)
        data breach risk (RLS bug → cross-tenant exposure)
        cannot provide per-tenant SLA
  Best for: free tier; small tenants; < 1K tenants

Model 2: Shared Compute, Separate Schemas/DBs (Medium Isolation)
  DB:     Separate PostgreSQL schema per tenant (same server, different schema)
          OR separate DB per tenant (same cluster, different DB)
  Compute: Shared application servers (connection pooling challenges)
  Cache:  Separate Redis keyspace per tenant
  
  Pros: data isolation at DB level; RLS bugs can't leak across tenants
        schema migration per tenant (different tenants can be on different versions)
  Cons: connection pool explosion (1K tenants × 10 connections = 10K connections)
        migration complexity (1K schema updates)
  Best for: 10–1K tenants; compliance-sensitive customers

Model 3: Separate Everything (Highest Isolation, Highest Cost)
  DB:     Dedicated PostgreSQL cluster per tenant
  Compute: Dedicated application tier per tenant (Kubernetes namespace or cluster)
  Cache:  Dedicated Redis per tenant
  Network: Dedicated VPC per tenant (for highest security)
  
  Pros: perfect isolation; guaranteed SLA; simplest data residency
  Cons: expensive; slow onboarding (minutes to provision); ops complexity
  Best for: enterprise; large tenants; regulated industries; willing to pay premium

CHOSEN: Tiered isolation (not all-or-nothing)
  Free/Starter: Model 1 (shared everything)
  Business: Model 2 (shared compute, separate DB schema)
  Enterprise: Model 3 (dedicated infrastructure)
  
  Same codebase serves all tiers; routing layer decides which model to use
  This is how Salesforce, HubSpot, and Stripe work.
```

---

## Problem Statement

Multi-tenant payment SaaS platform. Serves free, business, and enterprise tenants from shared infrastructure. Tenant isolation appropriate to their tier. Data residency for EU tenants. Noisy neighbour prevention.

---

## Functional Requirements
- Tenant onboarding: new tenant operational within 5 minutes
- Data isolation: tenant A cannot access tenant B's data
- Configuration isolation: each tenant has independent settings, webhooks, API keys
- Performance isolation: one tenant's traffic surge cannot degrade others
- Data residency: EU tenants' data processed and stored in EU
- Custom domains: tenants can use their own domain (api.merchantbank.com → platform)
- Audit log: per-tenant audit trail (Case 10)

## Non-Functional Requirements
- Tenant onboarding: **< 5 minutes**
- Data isolation: **zero cross-tenant data leakage** (RLS + application-level enforcement)
- Noisy neighbour: **P99 latency degradation < 10%** when adjacent tenant has 10× traffic
- Data residency: **EU data never leaves EU infrastructure**

---

## Capacity Estimation
```
Tenants:
  Free: 10K tenants (low volume; shared everything)
  Business: 1K tenants (medium volume; separate schema)
  Enterprise: 50 tenants (high volume; dedicated)

Shared DB (for free tier):
  10K tenants × 1K payments/day avg = 10M payments/day
  Payments table: 10M × 1 KB = 10 GB/day → manageable on shared cluster

Connection pool problem (Model 2):
  1K business tenants; each needs 5–10 connections
  Direct: 1K × 10 = 10K connections → PostgreSQL max_connections exceeded
  Solution: PgBouncer per schema group (50 tenants share a PgBouncer pool)
  PgBouncer: 50 tenants × 10 connections = 500 backend connections per PgBouncer
  20 PgBouncer instances handle 1K business tenants
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Tenant Context (request middleware) | Every request carries tenant_id; never trust client claim |
| Row-Level Security (RLS) | PostgreSQL-enforced data isolation for shared DB |
| Schema-Per-Tenant (PgBouncer) | Business tier: full isolation without dedicated cluster |
| Tenant Router (API Gateway) | Route to correct infrastructure tier based on tenant |
| Connection Pool Management | PgBouncer prevents connection explosion |
| Data Residency Router | EU tenant → EU infra; never touches non-EU services |
| Noisy Neighbour Detection | Per-tenant resource tracking; circuit break heavy tenants |

---

## Data Model

```sql
-- Tenant registry (control plane; PostgreSQL; shared across all regions)
CREATE TABLE tenants (
  tenant_id        UUID         PRIMARY KEY,
  name             VARCHAR(256) NOT NULL,
  slug             VARCHAR(64)  UNIQUE NOT NULL,  -- used in subdomains: slug.platform.com
  tier             VARCHAR(16)  NOT NULL,          -- free / business / enterprise
  region           VARCHAR(16)  NOT NULL,          -- us-east / eu-west / ap-south
  db_schema        VARCHAR(64),                   -- for business tier: schema name in shared cluster
  db_cluster       VARCHAR(64),                   -- for enterprise tier: dedicated cluster ID
  namespace        VARCHAR(64),                   -- for enterprise tier: K8s namespace
  status           VARCHAR(16)  DEFAULT 'ACTIVE',
  created_at       TIMESTAMP    DEFAULT NOW()
);

-- Global API keys (control plane; links key to tenant)
CREATE TABLE api_keys (
  key_id           UUID         PRIMARY KEY,
  api_key_hash     VARCHAR(64)  UNIQUE NOT NULL,  -- SHA-256 of API key; never store plaintext
  tenant_id        UUID         NOT NULL REFERENCES tenants(tenant_id),
  name             VARCHAR(128),
  scopes           TEXT[],
  is_active        BOOLEAN      DEFAULT TRUE
);

-- Free tier: all tenants in shared schema with tenant_id column
-- Row-Level Security (PostgreSQL RLS):
ALTER TABLE payments ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON payments
  USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
-- SET app.current_tenant_id = '<tenant_id>' at connection start
-- → queries automatically filtered to current tenant

-- Business tier: separate schema per tenant
-- Schema name: tenant_{tenant_id_first_8_chars}
-- Each schema has identical table structure (managed by migration tool)
-- Connection: SET search_path = tenant_a1b2c3d4;

-- Enterprise tier: dedicated cluster
-- Tenant's application tier connects to their dedicated PostgreSQL cluster
-- No shared infrastructure at all
```

---

## API Design

```
Multi-tenant API routing:
  Option A: Subdomain routing  → tenant1.api.platform.com / tenant2.api.platform.com
  Option B: Path routing       → api.platform.com/v1/{tenant_slug}/payments
  Option C: Header routing     → X-Tenant-ID: {tenant_id}
  
  CHOSEN: Subdomain routing for enterprise (custom domains) + header routing for API keys
  
  Flow:
    api.platform.com → API Gateway → extract tenant from subdomain or X-API-Key header
    → tenant lookup → determine tier and region → route to correct infrastructure

Custom domain setup (enterprise):
  Merchant's DNS: api.merchantbank.com → CNAME → platform-lb.platform.com
  SNI routing: platform LB reads SNI; matches to tenant_id
  Certificate: platform manages wildcard cert; merchant provides custom domain cert
  
Tenant onboarding API (internal):
  POST /internal/v1/tenants
  Body: { name, tier, region, admin_email }
  Response 201: { tenant_id, slug, api_key, webhook_secret, onboarding_steps_completed: 5/5 }
  (Automated: create DB schema, provision infra if enterprise, send welcome email, etc.)
```

---

## High-Level Architecture

```
Client Request
  │
  ▼
Global Load Balancer (anycast; routes to nearest regional cluster)
  │
  ├── EU Region (eu-west):
  │     EU tenants' requests stay here (GDPR data residency)
  │     Dedicated EU API gateway, DB cluster, Redis
  │
  └── US Region (us-east):
        US/default tenants processed here

Per-Region API Gateway (extends Case 05):
  ├── Step 1: Tenant resolution
  │     Extract tenant identifier: subdomain OR X-API-Key header
  │     If API key: lookup api_keys table → tenant_id (in-process LRU cache; 5 min)
  │     If subdomain: lookup tenants.slug → tenant_id (in-process cache)
  │
  ├── Step 2: Data residency check
  │     tenant.region must match request's region
  │     If EU tenant reaches US gateway: redirect to EU gateway (302 or TCP routing)
  │     EU data NEVER processed by US services
  │
  ├── Step 3: Tier routing
  │     Free tier:        route to shared application pool
  │     Business tier:    route to shared application pool + inject DB schema context
  │     Enterprise tier:  route to tenant's dedicated namespace
  │
  ├── Step 4: Set tenant context
  │     Inject into request context: tenant_id, tier, db_connection_hint
  │     Application layer uses this to scope all DB queries
  │
  └── Step 5: Rate limiting
        From Case 25; now tenant-aware
        Free tier limits < Business < Enterprise
        Enterprise tenants have burstable limits (paid for dedicated capacity)

Application Layer (shared for free/business; dedicated for enterprise):
  Every DB query MUST include tenant filter:
    Free tier:   PostgreSQL RLS handles this automatically (session variable)
    Business:    connection pool points to correct schema (search_path)
    Enterprise:  connection pool points to dedicated cluster (no shared infra)
  
  Code never changes between tiers — tenant isolation is infrastructure-level

Tenant Isolation Enforcement:
  1. PostgreSQL RLS (free tier): DB engine enforces tenant_id filter on every query
  2. Schema-per-tenant (business): wrong schema → table doesn't exist → query fails
  3. Dedicated cluster (enterprise): wrong cluster → connection fails
  4. Application middleware: assert tenant_id in context before any DB operation
  5. Audit log: every API call includes tenant_id (Case 10)
```

---

## Detailed Components

### Tenant Context Middleware
```java
@Component
public class TenantContextFilter implements Filter {

    @Override
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) {
        String apiKey = extractApiKey((HttpServletRequest) req);
        TenantInfo tenant = resolveTenant(apiKey);

        // Set tenant context for this thread
        TenantContext.set(tenant);

        // For free tier: set PostgreSQL session variable (RLS)
        if (tenant.getTier() == FREE) {
            DatabaseContext.setTenantId(tenant.getTenantId());
            // Translates to: SET app.current_tenant_id = '{uuid}';
            // PostgreSQL RLS policy uses this to filter all queries
        }

        try {
            chain.doFilter(req, resp);
        } finally {
            TenantContext.clear();  // MUST clear ThreadLocal after request
        }
    }
}

// Assertion in every repository method:
@Repository
public class PaymentRepository {
    public Payment findById(UUID paymentId) {
        // Guard: ensure tenant context is set
        UUID tenantId = TenantContext.requireTenantId();  // throws if not set

        // For free tier: RLS handles filtering (tenantId in session variable)
        // For business tier: search_path already set to correct schema
        // For enterprise: connected to dedicated cluster
        return jdbcTemplate.queryForObject(
            "SELECT * FROM payments WHERE payment_id = ?",
            PAYMENT_ROW_MAPPER, paymentId
        );
        // RLS automatically adds: AND tenant_id = current_setting('app.current_tenant_id')
    }
}
```

### Noisy Neighbour Prevention
```
Problem: Tenant B runs a bulk export query that takes 30 seconds.
         Tenant A's checkout payments slow down because of shared DB connection pool.

Solutions:
  1. Statement timeout per tenant tier:
     Free: SET statement_timeout = '5s';
     Business: SET statement_timeout = '30s';
     Enterprise: no statement timeout (dedicated cluster)
     
  2. Connection pool priority:
     PgBouncer: priority queuing — transactional queries > analytical queries
     Detect long-running queries: any query > 5s → cancel + log
     
  3. Per-tenant resource tracking:
     Track CPU + I/O per tenant per minute (PostgreSQL pg_stat_activity)
     If tenant exceeds quota: throttle their queries (queue behind others)
     
  4. Read replica routing:
     Write queries → primary
     Analytical/read queries → read replicas (isolated from write load)
     Free tier: shared read replica; Enterprise: dedicated read replica
     
  5. Query cost estimation (PostgreSQL EXPLAIN):
     Before executing: EXPLAIN (ANALYZE FALSE) query → estimate cost
     If estimated cost > threshold for tier → reject (tell user to upgrade)
```

### Tenant Onboarding Automation
```python
class TenantOnboardingService:

    def provision_tenant(self, name: str, tier: str, region: str, admin_email: str) -> TenantInfo:
        tenant_id = uuid4()
        slug = self._generate_slug(name)

        with db_transaction():
            # 1. Create tenant record
            tenant = Tenant(id=tenant_id, name=name, slug=slug, tier=tier, region=region)
            db.insert(tenant)

            # 2. Tier-specific provisioning
            if tier == 'FREE':
                # No extra provisioning needed; shared DB + RLS
                pass

            elif tier == 'BUSINESS':
                # Create dedicated schema
                schema_name = f"tenant_{str(tenant_id)[:8].replace('-', '')}"
                db.execute(f"CREATE SCHEMA {schema_name}")
                migration_runner.run_migrations(schema_name)
                tenant.db_schema = schema_name
                db.update(tenant)

            elif tier == 'ENTERPRISE':
                # Trigger K8s namespace + dedicated DB provisioning
                namespace = k8s_provisioner.create_namespace(tenant_id)
                db_cluster = rds_provisioner.create_cluster(tenant_id, region)
                migration_runner.run_migrations(db_cluster)
                tenant.namespace = namespace
                tenant.db_cluster = db_cluster
                db.update(tenant)

            # 3. Create API keys
            api_key, api_key_hash = generate_api_key_pair()
            db.insert(ApiKey(tenant_id=tenant_id, hash=api_key_hash, scopes=['full']))

            # 4. Welcome email
            email_service.send_welcome(admin_email, api_key, slug)

        # Duration:
        # FREE: < 1 second
        # BUSINESS: ~10 seconds (schema creation + migrations)
        # ENTERPRISE: 3–5 minutes (RDS cluster provisioning)
        return TenantInfo(tenant_id, api_key, slug)
```

---

## Scaling Strategy
```
Phase 1 (0 → 100 tenants):
  Single DB, no tenant isolation
  tenant_id column; no RLS (trust the application)
  Simple; works at small scale; don't over-engineer early

Phase 2 (100 → 1K tenants, compliance needed):
  Add RLS on PostgreSQL (free tier)
  Schema-per-tenant for paying customers
  PgBouncer for connection pooling
  Regional deployment for data residency

Phase 3 (1K → 10K tenants):
  Dedicated application tier for enterprise
  Automated provisioning (< 5 min for any tier)
  Per-tenant usage monitoring
  Noisy neighbour detection + throttling

Phase 4 (global, regulated markets):
  Regional control planes (per region tenant registry)
  Data sovereignty tooling (verify data never leaves region)
  SOC2 / ISO 27001 / PCI-DSS per-tenant compliance reports
  Tenant-level encryption keys (BYOK per tenant)
```

---

## Reliability + Security

```
Security layers per tier:
  Free:       Application RLS + tenant_id in all queries + rate limiting
  Business:   Schema isolation (DB-level) + above
  Enterprise: Dedicated cluster + VPC + BYOK + above + dedicated WAF

Data leakage prevention checklist:
  □ RLS policy verified in unit tests (try to access another tenant's data)
  □ RLS policy survives DB superuser operations (BYPASSRLS prevention)
  □ Error messages never reveal other tenant existence ("not found" not "belongs to other tenant")
  □ Pagination cursors signed (prevent tenant from incrementing cursor to reach other data)
  □ File uploads: S3 bucket policy per tenant (no cross-tenant read)
  □ Webhooks: validated against tenant's registered URLs
  □ Logs: tenant_id is primary tag; filtered before access by support engineers

GDPR / Data residency verification:
  Automated test: create EU tenant → run payment → verify:
    All DB writes go to EU cluster
    All event logs are in EU Kafka cluster
    No data appears in US S3 / US CloudWatch
```

---

## Observability
```
Per-tenant metrics (aggregated, not individual):
  request_count_by_tier             (free/business/enterprise distribution)
  error_rate_by_tenant              (identify struggling tenants)
  p99_latency_by_tier               (verify isolation; enterprise should be consistently low)
  noisy_neighbour_detections        (how often one tenant affects others)
  tenant_provisioning_duration      (onboarding speed by tier)

Tenant health dashboard (per tenant, visible to support team):
  recent_api_calls, error_rate, top_endpoints, rate_limit_hits
  Used by support for debugging customer issues without exposing raw DB
```

---

## Chaos Testing
```
Experiment 1: Cross-Tenant Data Access Attempt
  Inject:    With tenant A's API key, attempt to access tenant B's payment:
             GET /v1/payments/{tenant_B_payment_id}
  Expected:  RLS / schema isolation: query returns empty or 404
  Expected:  No tenant B data visible
  Expected:  Audit log: access attempt logged with tenant A context
  Red flag:  Tenant B data returned to tenant A

Experiment 2: Noisy Neighbour Simulation
  Inject:    Free tier tenant runs long-running bulk query (1M rows export)
  Expected:  Statement timeout (5s) kicks in; query cancelled
  Expected:  Tenant A's checkout latency: unaffected (< 10% degradation)
  Expected:  Tenant B's support team notified: "Query exceeded free tier limit"
  Red flag:  Tenant B's 5-minute query causes 200ms latency increase for other tenants

Experiment 3: Tenant Onboarding Under Load
  Inject:    Provision 100 new business tenants simultaneously
  Expected:  Each provisioning creates separate schema; migrations run independently
  Expected:  Existing tenants unaffected during provisioning
  Expected:  All 100 tenants provisioned within 60 seconds (parallel)
  Red flag:  Sequential provisioning takes 100 × 10s = 16 minutes

Experiment 4: EU Data Residency Violation Detection
  Inject:    Route EU tenant's request through US gateway (simulated misconfiguration)
  Expected:  US gateway detects tenant.region = 'eu'; rejects or redirects
  Expected:  No EU tenant data processed in US infrastructure
  Expected:  Alert fires: "EU tenant routed to wrong region"
  Red flag:  EU tenant data lands in US S3 bucket without detection
```

---

## Monthly Cost Estimate
```
Scale: 10K free, 1K business, 50 enterprise tenants

Shared infrastructure (free + business tiers):
  PostgreSQL cluster (free tier): db.r6g.4xlarge + 2 replicas = $4,800/mo
  Business tier schemas: same cluster + PgBouncer                = $500/mo
  Shared app servers (30 × c5.2xlarge):                         = $8,400/mo
  Shared Redis, Kafka, monitoring:                               = $3,000/mo

Enterprise tenants (50 × dedicated):
  Each enterprise: dedicated RDS + K8s namespace + Redis
  Per enterprise: ~$2,000/mo infrastructure
  50 × $2,000:                                                   = $100,000/mo
  (Offset by enterprise pricing: avg $10K/mo/tenant = $500K/mo revenue)

Control plane:
  Tenant registry DB, provisioning service                       = $1,000/mo
─────────────────────────────────────────────────────────────────
Total: ~$117,700/month
Enterprise is 85% of infra cost but provides highest revenue (margin at scale).
Free tier serves as acquisition channel; Business is expansion; Enterprise is retention.
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → Control plane DB: tenant registry (which tenant maps to which infra).
  → Data plane DB: tenant's actual data (isolated per tier model).
  → Tenant context: derived from API key at request time; not trusted from client.

□ Isolation model by tier:
  → Free: RLS (application-enforced via session variable + DB policy).
  → Business: schema separation (DB-level; misconfig → different schema = different tenant).
  → Enterprise: dedicated cluster (network-level isolation; no shared hardware).
  → Each tier's isolation cannot be bypassed by the tier below.

□ Noisy neighbour:
  → Statement timeout per tier (free = 5s; business = 30s).
  → Connection pool per tenant group (not shared across groups).
  → Read replicas absorb analytical load (checkout queries unaffected by exports).

□ Data residency:
  → Enforced at gateway level (redirect if wrong region).
  → Enforced at application level (tenant.region check before any external call).
  → Verified by automated test in CI pipeline.

□ SCC LENS:
  STATE: Control plane (tenant registry), Data plane (tenant data — isolated per tier)
  COORDINATION: API key → tenant → infra tier (routing decision at every request)
  CONCENTRATION: Shared DB (free tier) → noisy neighbour risk → statement timeout + read replicas
```

---

*Next: Case Study 27 — Zero Trust Network Architecture*
