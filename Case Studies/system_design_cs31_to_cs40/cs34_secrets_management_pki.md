# Case Study 34 — Secrets Management and PKI

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Dynamic secrets, Certificate lifecycle, Vault roles, Secret sprawl.

---

## Business Context

A payment platform has hundreds of secrets: DB passwords, API keys, TLS certificates, NPCI credentials, card vault keys. Secret sprawl — secrets in environment variables, hardcoded in code, stored in spreadsheets — is the most common root cause of breaches. HashiCorp Vault solves this by centralising secret storage, enforcing access policies, generating dynamic short-lived credentials, and auditing every secret access.

**Business goals driving architecture:**
- No long-lived static credentials (everything is dynamic or rotated frequently)
- Audit trail: who accessed which secret, when, from which service
- Least privilege: payment-service gets DB credentials; not fraud-service
- Rotation: DB password rotation without service restart (or with minimal restart)
- PCI-DSS: card vault key access logged and restricted to vault-service only

---

## Recognition Framework

### Static Secrets vs Dynamic Secrets

```
Static secrets (the problem):
  DB_PASSWORD = "supersecretpassword123"
  → Stored in: .env file, K8s ConfigMap (wrong!), CI environment, developer laptop
  → Never rotated (too painful: update N services simultaneously)
  → When leaked: attacker has indefinite access (no expiry)
  → Who accessed it? Nobody knows.

Dynamic secrets (the solution):
  Vault generates a unique credential per service per request
  DB credential for payment-service:
    username: v-payment-svc-k9x2m
    password: A3xj-random-generated
    TTL: 1 hour (auto-revoked after 1 hour)
  
  If leaked: attacker has 1 hour max; auto-revoked on expiry
  If service compromised: revoke that service's lease immediately
  Audit: every credential generation logged (service identity, timestamp, IP)
  Rotation: automatic (Vault rotates root DB credential; generates new dynamic creds)
  
Vault dynamic secret engines:
  Database: generates DB users with custom role on demand
  AWS: generates IAM credentials on demand
  PKI: generates TLS certificates on demand
  SSH: generates SSH OTPs or signed certificates on demand
```

### HashiCorp Vault Architecture

```
Vault Server (HA cluster):
  Active node: handles all requests
  Standby nodes: replicate via Raft; take over if active fails
  Storage backend: Raft (integrated) or Consul or PostgreSQL
  Seal: master key from AWS KMS or HSM (Vault is encrypted at rest; decrypted on unseal)

Vault clients authenticate via auth methods:
  K8s auth: service account token → Vault verifies with K8s API → issues short-lived token
  AWS auth: instance identity document → Vault verifies with AWS → issues token
  AppRole: role_id + secret_id → for CI pipelines
  OIDC: human engineers authenticate via SSO (Okta/ForgeRock → Vault)

Access control:
  Policies (HCL): define which paths a role can access (read/write/list)
  Role: maps auth method identity → policy
  Token: issued after authentication; short TTL; used for subsequent secret reads
```

---

## Problem Statement

Centralised secrets management for a payment platform. HashiCorp Vault for dynamic DB credentials, TLS certificates, API keys. Kubernetes-native auth. Audit logging. No secrets in Git or environment variables.

---

## Core Vault Configuration

### Kubernetes Auth Setup
```hcl
# Enable K8s auth method
vault auth enable kubernetes

# Configure K8s auth to verify service account tokens
vault write auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc:443" \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_ca_cert="$(cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt)"

# Create role: payment-service can access DB secrets and PKI
vault write auth/kubernetes/role/payment-service \
    bound_service_account_names=payment-service \
    bound_service_account_namespaces=payments \
    policies=payment-service-policy \
    ttl=1h    # token expires after 1 hour; must re-authenticate

# Policy: what payment-service can access
cat > payment-service-policy.hcl << 'EOF'
# Read dynamic DB credentials
path "database/creds/payment-service-role" {
  capabilities = ["read"]
}
# Generate TLS certificates
path "pki/issue/payment-service" {
  capabilities = ["create", "update"]
}
# Read static secrets (card vault API key)
path "secret/data/payment/card-vault-key" {
  capabilities = ["read"]
}
# DENY: payment-service cannot read fraud-service secrets
# (no path = deny; Vault is default-deny)
EOF
vault policy write payment-service payment-service-policy.hcl
```

### Dynamic Database Credentials
```hcl
# Enable database secrets engine
vault secrets enable database

# Configure PostgreSQL connection (Vault has root access to rotate)
vault write database/config/payment-db \
    plugin_name=postgresql-database-plugin \
    connection_url="postgresql://{{username}}:{{password}}@payment-db.internal:5432/payments" \
    allowed_roles="payment-service-role" \
    username="vault-root" \
    password="vault-root-password"

# Define role: what SQL to run when generating credentials
vault write database/roles/payment-service-role \
    db_name=payment-db \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';
                          GRANT SELECT, INSERT, UPDATE ON payments TO \"{{name}}\";
                          GRANT SELECT, INSERT ON payment_events TO \"{{name}}\";" \
    revocation_statements="DROP ROLE IF EXISTS \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"

# Generate credentials:
vault read database/creds/payment-service-role
# Key                Value
# lease_id           database/creds/payment-service-role/xyz
# lease_duration     1h
# username           v-payment-svc-k9x2m
# password           A3xj-random-generated

# Vault renews lease before expiry (if service is still running)
# Vault revokes when lease expires or revoke is called explicitly
```

### PKI (Certificate Authority in Vault)
```hcl
# Enable PKI engine for internal certificates
vault secrets enable -path=pki pki
vault secrets tune -max-lease-ttl=87600h pki  # 10 year root CA TTL

# Generate root CA
vault write pki/root/generate/internal \
    common_name="Payment Platform Root CA" \
    ttl=87600h

# Create issuing role for payment-service
vault write pki/roles/payment-service \
    allowed_domains="payment-platform.internal" \
    allow_subdomains=true \
    allow_bare_domains=true \
    max_ttl="720h"    # max 30-day cert; most services use 1h (SPIFFE pattern)

# Issue a certificate:
vault write pki/issue/payment-service \
    common_name="payment-service.payment-platform.internal" \
    ttl="1h"
# Returns: certificate, private_key, issuing_ca, serial_number
# Cert expires in 1 hour; service must request new one before expiry

# Cert revocation (if service compromised):
vault write pki/revoke serial_number="39:dd:2e:90:..."
# All clients presenting this cert: rejected immediately (CRL/OCSP)
```

---

## K8s Integration Pattern

### Vault Agent Injector (Zero-Code Secret Injection)
```yaml
# Deployment with Vault annotations: secrets injected as files
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "payment-service"
        
        # Inject DB credentials as file
        vault.hashicorp.com/agent-inject-secret-db-credentials: "database/creds/payment-service-role"
        vault.hashicorp.com/agent-inject-template-db-credentials: |
          {{- with secret "database/creds/payment-service-role" -}}
          PGUSER={{ .Data.username }}
          PGPASSWORD={{ .Data.password }}
          PGHOST=payment-db.internal
          {{- end }}
        
        # Inject card vault API key
        vault.hashicorp.com/agent-inject-secret-card-vault: "secret/data/payment/card-vault-key"
        vault.hashicorp.com/agent-inject-template-card-vault: |
          {{- with secret "secret/data/payment/card-vault-key" -}}
          CARD_VAULT_API_KEY={{ .Data.data.api_key }}
          {{- end }}

# Vault Agent (injected sidecar):
#   1. Authenticates to Vault using K8s service account token
#   2. Reads secrets and writes to /vault/secrets/ files
#   3. Renews secrets before TTL expiry (dynamic credentials)
#   4. On lease expiry: refreshes file content; sends SIGHUP to app (if configured)

# Service code reads from /vault/secrets/db-credentials (a file, not env var)
# NEVER environment variables: env vars can be dumped in logs; files are safer
```

### External Secrets Operator (Alt Pattern for K8s)
```yaml
# ESO is simpler than Vault Agent for static secrets
# ESO: pulls from Vault/AWS SM → creates K8s Secret object
# Good for: static secrets (API keys, TLS certs); not for dynamic credentials

apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: npci-credentials
  namespace: payments
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: npci-credentials-secret
  data:
  - secretKey: client-id
    remoteRef:
      key: secret/data/payment/npci
      property: client_id
  - secretKey: client-secret
    remoteRef:
      key: secret/data/payment/npci
      property: client_secret
```

---

## Audit and Compliance

```
Vault audit log (enabled by default; write to file + syslog):
  Every request logged:
    timestamp, operation (read/write/list), path, caller_token, response_code
  Example:
    {"type":"request","auth":{"client_token":"hmac-sha256:abc","entity_id":"payment-service-v2","policies":["payment-service-policy"]},"request":{"operation":"read","path":"database/creds/payment-service-role"}}
  
  PCI requirement: all access to cardholder data must be logged
  Vault audit log covers: who (service identity), what (secret path), when (timestamp)
  
  SIEM integration: audit logs → Splunk/Elasticsearch for alerting
    Alert: unexpected service accessing card-vault-key path → immediate incident

Secret lifecycle metrics:
  vault_secret_kv_count             (total secrets stored; trend)
  vault_token_count_by_policy       (active tokens per service)
  vault_lease_renewal_rate          (dynamic creds being renewed)
  vault_audit_log_request_failed    (audit backend failure → CRITICAL: Vault stops serving if audit fails)
  vault_core_active                 (leader status; alert if changes unexpectedly)
```

---

## Scaling + Reliability
```
Vault HA (Raft integrated storage):
  3-node cluster (1 active + 2 standby)
  Leader election via Raft; failover < 5 seconds
  No external storage dependency (unlike Consul backend)

Seal/Unseal:
  Vault is sealed at rest (master key from AWS KMS)
  Auto-unseal on startup (KMS decrypts master key)
  If KMS unavailable: Vault cannot unseal → CRITICAL
  Mitigation: KMS HA (multiple KMS regions); Vault auto-unseal config across regions

Secret caching:
  Vault Agent: caches secrets in memory; reduces Vault server load
  Cache TTL: shorter than secret TTL (force periodic refresh)
  
Performance secret engine:
  For very high-throughput secret reads: Vault Performance Replication
  Primary Vault → Performance Replica (read-only; serves read requests)
  50K+ secret reads/sec: use performance replica
```

---

## Architecture Review Checklist
```
□ No static long-lived secrets:
  DB credentials: Vault dynamic (1h TTL; auto-revoked)
  TLS certs: Vault PKI (1h for internal; 30d for external partners)
  API keys (NPCI, card network): Vault KV with rotation every 90 days
  
□ Secret access is least-privilege:
  Payment-service policy: only its DB role + card vault key
  Fraud-service policy: only its DB role + feature store key
  No policy: never access each other's secrets (Vault default-deny)
  
□ Audit log integrity:
  Audit backend failure: Vault stops accepting requests (safe-by-default)
  Audit logs shipped to immutable storage (S3 Glacier / SIEM)
  Daily review of unexpected access patterns

□ Rotation is automatic:
  Dynamic credentials: Vault handles rotation automatically
  Static credentials: rotation runbook; alert 30 days before expiry

□ SCC LENS:
  STATE: Vault Raft (secret data + policies + audit log)
  COORDINATION: Raft consensus (leader election + replication)
  CONCENTRATION: Vault active node = single write point → HA Raft cluster mandatory
  
□ IAM TIE-IN (deep):
  Vault roles ≡ IAM roles (service identity → access policy)
  Vault policies ≡ IAM policies (what paths/resources are accessible)
  Vault K8s auth ≡ OIDC auth (K8s SA token ≡ OIDC JWT)
  Vault PKI ≡ ForgeRock certificate services
  This IS identity and access management — applied to secrets instead of users
```

---

*Next: Case Study 35 — API Gateway Design (Advanced)*
