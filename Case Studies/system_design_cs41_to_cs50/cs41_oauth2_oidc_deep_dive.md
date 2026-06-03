# Case Study 41 — OAuth2 and OIDC Deep Dive

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Grant type selection, Token validation, Refresh token rotation.

---

## Business Context

OAuth2 is the authorisation framework that powers every "Login with Google" button, every API key system, and every microservice authentication pattern. OIDC (OpenID Connect) adds identity on top of OAuth2's authorisation layer. For a payment platform, OAuth2 controls which merchants can access which payment APIs, which users can see which accounts, and how microservices authenticate to each other. Getting OAuth2 wrong means either over-permissive access (security breach) or under-permissive access (broken integrations).

**Business goals driving architecture:**
- Merchant API access: scoped tokens (payments:create but not refunds:create)
- User-facing app: short-lived access tokens; refresh without re-login
- Service-to-service: machine-to-machine without user involvement
- Token introspection: validate tokens without calling auth server on every request
- ForgeRock AM integration: your existing platform expertise directly applies here

---

## Recognition Framework

### OAuth2 Grant Types — When to Use Which

```
Grant Type 1: Authorization Code (+ PKCE)
  WHO: user-facing applications (browser, mobile)
  FLOW: user → auth server → redirect with code → exchange code for tokens
  PKCE (Proof Key for Code Exchange): prevents code interception on mobile
  
  Use when:
    - Browser/mobile app where user authenticates
    - "Login with Google/ForgeRock" flows
    - End-user grants permissions to merchant app to access their data
  
  Tokens issued: access_token + refresh_token + id_token (OIDC)
  
  Why not Implicit: implicit grant is DEPRECATED (tokens exposed in URL fragment)

Grant Type 2: Client Credentials
  WHO: machine-to-machine (no user involved)
  FLOW: client_id + client_secret → auth server → access_token
  
  Use when:
    - payment-service calling fraud-service
    - Merchant's backend server calling payment API (server-side, no user)
    - Batch jobs, cron jobs, background services
  
  Tokens issued: access_token only (no refresh_token; just re-authenticate)
  
  Your ForgeRock context: this is exactly what SPIFFE/SPIRE replaces at the infra layer

Grant Type 3: Device Code
  WHO: limited-input devices (smart TVs, IoT, CLI tools)
  FLOW: device shows user code + URL → user logs in on phone → device polls for token
  
  Use when:
    - CLI tool needs user authentication (kubectl, AWS CLI)
    - TV app needing user login
  
  Rarely needed for payment platforms

Grant Type 4: Refresh Token
  Not a standalone grant; used to exchange refresh_token for new access_token
  Enables: long sessions without re-login; short-lived access tokens (15 min)
  Rotation: every refresh token use → new refresh token issued (old one invalidated)

DECISION MATRIX:
  Human uses app       → Authorization Code + PKCE
  Service calls API    → Client Credentials
  CLI with user auth   → Device Code
  Renewing session     → Refresh Token (combined with above)
```

---

## Problem Statement

OAuth2/OIDC implementation for a payment platform. Merchant API access (client credentials). User-facing app login (authorization code + PKCE). Scoped token validation at gateway. Token introspection and JWKS.

---

## Token Architecture

```
Access Token (short-lived; 15 minutes):
  JWT format: header.payload.signature
  Payload:
    {
      "iss": "https://auth.platform.com",      // issuer
      "sub": "u-12345",                         // subject (user ID)
      "aud": "payment-api",                     // audience (who should accept this)
      "exp": 1712345678,                        // expiry (Unix timestamp)
      "iat": 1712344778,                        // issued at
      "jti": "unique-token-id",                 // JWT ID (for revocation)
      "scope": "payments:create payments:read", // granted scopes
      "tenant_id": "merch-abc",                 // custom claim (tenant context)
      "user_tier": "ENTERPRISE",                // custom claim (rate limits)
      "acr": "urn:mace:incommon:iap:silver"    // authentication context reference
    }
  
  Validation (gateway; per request):
    1. Signature: verify against JWKS (JSON Web Key Set) from auth server
    2. Expiry: exp > now
    3. Audience: aud == "payment-api" (not a token for another service)
    4. Issuer: iss == known trusted issuer
    5. Scope: requested operation matches granted scopes
    
  Gateway extracts claims → injects as headers (X-User-ID, X-Scopes, X-Tenant-ID)
  Services: trust gateway-injected headers (no JWT library needed in each service)

Refresh Token (long-lived; 30 days):
  Opaque (not JWT) → reference token → stored server-side in token store
  On use: new access_token + new refresh_token issued; old refresh_token invalidated
  Why opaque: can be revoked server-side (JWT access tokens can't be revoked before expiry)
  Refresh token rotation: prevents refresh token theft (old token becomes invalid immediately)

ID Token (OIDC; user identity):
  JWT with user identity claims:
    {
      "sub": "u-12345",
      "email": "user@example.com",
      "name": "Alice",
      "email_verified": true,
      "phone_verified": true,
      "updated_at": 1712300000
    }
  Only issued for authorization code flow (human users)
  Purpose: tell the application WHO the user is (not what they can do)
  Access token: what they can do (authorisation)
  ID token: who they are (authentication/identity)
```

---

## Authorization Code + PKCE Flow

```
Mobile App (Payment Platform App)                  Auth Server (ForgeRock AM)

1. User opens app; taps "Login"

2. App generates:
   code_verifier = random 32 bytes (base64url)
   code_challenge = SHA256(code_verifier) (base64url)

3. App redirects browser to:
   GET /authorize
     ?client_id=payment-app
     &redirect_uri=com.platform.app://callback
     &response_type=code
     &scope=openid payments:create payments:read
     &state=random-state-value    (CSRF protection)
     &code_challenge=abc123...
     &code_challenge_method=S256

4. Auth server: shows login page
5. User: enters credentials + MFA
6. Auth server: validates credentials

7. Auth server redirects to:
   com.platform.app://callback
     ?code=auth-code-xyz
     &state=random-state-value

8. App: verifies state matches step 3's state (CSRF check)

9. App: exchanges code for tokens:
   POST /token
     code=auth-code-xyz
     &code_verifier=original-verifier-from-step-2   ← PKCE proof
     &grant_type=authorization_code
     &redirect_uri=com.platform.app://callback
     &client_id=payment-app

10. Auth server: verifies SHA256(code_verifier) == code_challenge from step 3
                 If match: issues tokens
                 If not: rejects (prevents code interception attack)

11. Response: {
      access_token: "eyJ...",   // 15-min JWT
      refresh_token: "opaque",  // 30-day opaque
      id_token: "eyJ...",       // user identity
      expires_in: 900
    }

PKCE prevents:
  Attacker intercepts auth code in redirect → cannot exchange without code_verifier
  code_verifier never transmitted in step 3 (only code_challenge = hash of it)
  Without original code_verifier: cannot get tokens from stolen code
```

---

## Client Credentials Flow (Service-to-Service)

```java
// payment-service authenticating to fraud-service's protected API
// (In practice: SPIFFE/SPIRE handles this at infra layer; OAuth2 is for external APIs)

@Service
public class FraudApiClient {
    
    private final OAuth2ClientCredentialsTokenProvider tokenProvider;
    
    // Token cached; refreshed when near expiry
    public FraudDecision evaluate(PaymentRequest payment) {
        String accessToken = tokenProvider.getToken(
            clientId = "payment-service",
            clientSecret = System.getenv("FRAUD_API_CLIENT_SECRET"),  // from Vault
            scope = "fraud:evaluate",
            tokenEndpoint = "https://auth.platform.com/token"
        );
        
        return fraudApiTemplate.post()
            .uri("/api/v1/fraud/evaluate")
            .header("Authorization", "Bearer " + accessToken)
            .body(payment)
            .retrieve()
            .bodyToMono(FraudDecision.class)
            .block();
    }
}

// Token caching (don't fetch new token on every API call):
// Token has 15-min TTL; cache it; refresh 2 min before expiry
// CachedTokenProvider: stores token + expiry; fetches new when TTL - 2min reached
```

---

## Token Introspection vs JWT Validation

```
Two ways to validate a token:

Option A: Local JWT validation (preferred for high-throughput)
  Gateway: load JWKS from auth server once (and every 1 hour)
  Per request: validate JWT signature locally using cached public key
  Latency: < 1ms (no network call)
  Limitation: cannot detect revoked tokens within the token's TTL window
  
  JWKS endpoint: GET /jwks
    {"keys": [{"kty": "RSA", "use": "sig", "kid": "key-1", "n": "...", "e": "AQAB"}]}
  
  Cache JWKS: 1-hour TTL; refresh on "kid not found" (key rotation event)

Option B: Token Introspection (RFC 7662)
  Per request: POST /introspect with token → auth server responds {active: true/false}
  Latency: 5-20ms (network call to auth server)
  Benefit: real-time revocation check (compromised token → revoked → introspection returns false)
  
  CHOSEN: local JWT validation (Option A) for most tokens
  Introspection (Option B) for: admin operations, high-risk operations (large transfers)

JWT Key Rotation:
  Auth server rotates signing keys every 30 days
  New key: new "kid" (key ID) in JWT header
  JWKS endpoint: serves both old and new key during rotation period
  Gateway: on unknown "kid" → refresh JWKS cache → retry validation
  This is transparent to clients (no token re-issuance needed)
```

---

## Scope Design for Payment Platform

```
Scope hierarchy (coarse-grained → fine-grained):
  payments:*          → all payment operations (admin only)
  payments:create     → create a new payment
  payments:read       → read payment details
  payments:refund     → issue refunds
  payments:void       → void authorisations
  
  merchant:read       → read merchant profile
  merchant:manage     → update merchant settings
  
  accounts:read       → read account balance and history
  
  webhooks:manage     → register/update webhook URLs

Scope enforcement (OPA policy or gateway):
  POST /payments → requires payments:create scope
  GET /payments/{id} → requires payments:read scope
  POST /payments/{id}/refund → requires payments:refund scope
  
  Tenant isolation via claims (not scopes):
    scope = payments:read allows reading ANY payment → too broad
    Correct: scope = payments:read + tenant_id claim → gateway scopes query to tenant
    OPA policy: allow { input.scope contains "payments:read"; input.tenant_id == input.payment.tenant_id }

OIDC Claims for Payment Platform:
  Standard OIDC: sub (user ID), email, name
  Custom claims:
    tenant_id:     "merch-abc"          (determines data scope)
    user_tier:     "ENTERPRISE"         (determines rate limits)
    kyc_level:     "FULL"              (KYC completion status)
    payment_limit: 1000000             (single payment limit in paise)
    mfa_verified:  true                (MFA completed in this session)
    acr:           "urn:platform:silver" (authentication strength)
  
  Step-up authentication: if payment > ₹10,000 and acr < "gold":
    Require: re-authentication with higher acr
    How: return 401 with WWW-Authenticate: Bearer scope="payments:create" acr_values="urn:platform:gold"
    App: re-initiates authorization code flow with acr_values parameter
```

---

## Refresh Token Management

```
Refresh Token Rotation (Security Best Practice):
  Problem: stolen refresh token → attacker refreshes indefinitely
  Solution: every refresh → new refresh token issued; old invalidated
  
  Theft detection:
    If attacker uses old (invalidated) refresh token:
      Auth server detects: this token was already rotated
      Auth server: revoke ENTIRE refresh token family
      User is logged out; must re-authenticate
    
    This works because:
      Legitimate client: uses new token after rotation; never uses old
      Attacker with old token: tries to use it → detects theft → full revocation

Refresh token store (required; refresh tokens are opaque reference tokens):
  Redis (fast lookup; TTL matches refresh token lifetime):
    Key: refresh_token_id → { user_id, tenant_id, scope, family_id, created_at }
    TTL: 30 days
  
  On refresh:
    1. Lookup old refresh_token in Redis
    2. Verify token matches stored record
    3. Mark old token as ROTATED (don't delete yet; detect reuse)
    4. Issue new access_token + new refresh_token
    5. Store new refresh_token in Redis; link to same family_id
  
  On detected reuse:
    1. Old token lookup → status = ROTATED
    2. Find family_id → revoke ALL tokens in this family
    3. User must re-authenticate

Session limits:
  Max active refresh tokens per user: 5 (one per device)
  On 6th login: revoke oldest refresh token (session limit)
  User can see active sessions and revoke any session (CIAM best practice)
```

---

## ForgeRock AM Integration (Your Platform Context)

```
ForgeRock AM acts as the OAuth2/OIDC Authorization Server:
  Token endpoint: https://am.internal.platform.com/oauth2/token
  Authorization endpoint: https://am.internal.platform.com/oauth2/authorize
  JWKS endpoint: https://am.internal.platform.com/oauth2/jwks
  Introspection: https://am.internal.platform.com/oauth2/introspect
  
ForgeRock-specific configurations:
  OAuth2 Provider: configured in AM admin console
  Scope validation: custom scripted scope validation (Groovy/JavaScript)
  Custom claims: ForgeRock's Claims Provider scripting
  Token lifetime: configurable per OAuth2 client (payment-app: 15min; batch-service: 60min)
  
  Realm isolation: different realms for different tenant types
    /realms/root/realms/consumers → consumer-facing OAuth2
    /realms/root/realms/merchants → merchant API OAuth2
    /realms/root/realms/internal  → service-to-service OAuth2
  
  Session binding: OIDC session linked to AM session
    AM session invalidation → OIDC session invalidation → access token unusable after TTL
    Step-up: AM policy requires fresh authentication for high-risk operations
  
  SAML-to-OAuth2 bridge:
    Enterprise merchants use SAML SSO internally
    ForgeRock AM: converts SAML assertion → OAuth2 access token (token exchange)
    RFC 8693 Token Exchange: used for this bridge

  OpenDJ (LDAP) integration:
    User profile claims (email, name, phone) fetched from OpenDJ at token issuance
    Custom claim mappings in AM: LDAP attributes → JWT claims
    Sync: SCIM provisioning keeps AM + OpenDJ user data consistent
```

---

## Observability
```
OAuth2 / OIDC metrics:
  token_issuance_total by {grant_type, client_id}
  token_validation_latency_p99                  (JWT validation at gateway; target < 1ms)
  token_introspection_latency_p99               (for high-security ops; target < 20ms)
  refresh_token_rotation_total
  token_reuse_detected_total                    (stolen token detection; alert on any)
  scope_denied_total by {scope, client_id}      (misconfigured client scopes)
  jwks_refresh_total                            (key rotation detection)

Security alerts:
  token_reuse_detected > 0 → stolen refresh token; immediate investigation
  scope_denied spike → client misconfiguration or authorization bypass attempt
  unknown_kid on JWT → key rotation happening; refresh JWKS cache
  expired_token_accepted > 0 → gateway clock skew or validation bug; CRITICAL
```

---

## Architecture Review Checklist
```
□ Grant type selection:
  → User-facing app: Authorization Code + PKCE (never Implicit)
  → Service-to-service: Client Credentials (or SPIFFE for infra layer)
  → CLI with user: Device Code
  → Never: Resource Owner Password Credentials (deprecated; insecure)

□ Token security:
  → Access token TTL: 15 minutes (balance security vs refresh frequency)
  → Refresh token TTL: 30 days (usable period)
  → Refresh token rotation: enabled (theft detection)
  → Refresh token: opaque (revokable); access token: JWT (local validation)
  → PKCE: mandatory for all public clients (mobile, SPA)

□ Scope design:
  → Principle of least privilege: merchant gets only scopes they need
  → Scope + tenant_id: scope alone is not enough (add tenant claim for data isolation)
  → Admin scopes: separate OAuth2 client; not available to merchant API clients

□ JWT validation:
  → Verify: signature, expiry, audience, issuer
  → JWKS cached: refresh on unknown kid
  → Never: trust without verifying all four fields

□ SCC LENS:
  STATE: token store (refresh tokens), JWKS (public keys), session store (AM sessions)
  COORDINATION: refresh token rotation (atomic: invalidate old + issue new in one operation)
  CONCENTRATION: auth server is single point for all token issuance → HA (ForgeRock AM cluster)

□ IAM DEEP TIE-IN:
  This entire case study IS your domain expertise:
  ForgeRock AM = the auth server implementation
  OAuth2 scopes = the authorisation model
  OIDC claims = the identity layer
  Refresh tokens in OpenDJ/Redis = your token management experience
  Staff interview: "design an OAuth2 system for a payment platform" → this is your answer
```

---

*Next: Case Study 42 — SAML and Enterprise SSO*
