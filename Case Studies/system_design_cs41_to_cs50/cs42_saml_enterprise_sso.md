# Case Study 42 — SAML and Enterprise SSO

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: SP-initiated vs IdP-initiated flow, Assertion validation, SAML-to-OIDC bridge.

---

## Business Context

SAML 2.0 is the enterprise identity federation standard. When a bank's employees need to access your payment dashboard using their corporate credentials (Active Directory / Okta / Azure AD), they use SAML. You don't store their passwords — their corporate Identity Provider (IdP) authenticates them, then sends a signed assertion to your Service Provider (SP) confirming who they are.

For a payment platform serving enterprise merchants: they refuse to manage separate credentials in your system. They want single sign-on with their existing enterprise IdP. SAML is how you integrate with that IdP, and ForgeRock AM is how you build the SP or IdP side of this integration.

**Business goals:**
- Enterprise merchants authenticate with their corporate IdP (no password in your system)
- Just-In-Time (JIT) provisioning: user accounts created on first login
- SCIM: user lifecycle sync (deprovisioning when employee leaves)
- SAML assertions mapped to your platform roles and tenant context
- ForgeRock AM as SAML SP receiving assertions from enterprise IdPs

---

## Recognition Framework

### SAML vs OIDC — When to Use Which

```
SAML 2.0:
  Protocol: XML-based; heavyweight; mature (2005)
  Transport: HTTP POST (base64-encoded XML assertion in form post)
  Use when: enterprise federation; legacy IdPs; large banks; government
  Strengths: rich attribute assertions; federation metadata; well-understood in enterprise
  Weaknesses: verbose XML; complex parsing; not mobile-friendly (no redirect flow for apps)
  Tools: Okta, Azure AD, ADFS, PingFederate, ForgeRock AM

OIDC:
  Protocol: JSON/JWT-based; lightweight; modern (2014)
  Transport: HTTP redirect + JSON token
  Use when: consumer apps; modern cloud IdPs; mobile; microservices
  Strengths: compact tokens; mobile-friendly; REST-native; easy to implement
  Weaknesses: less enterprise-mature; fewer legacy IdP integrations

Decision:
  Enterprise bank or corporation with ADFS/Azure AD: SAML (that's what they have)
  Consumer user with Google/Apple: OIDC
  Modern enterprise with Okta/Auth0: either (Okta supports both)
  Internal microservices: OIDC (or SPIFFE for infra layer)
  
  ForgeRock context: ForgeRock AM supports BOTH; you can receive SAML from enterprise IdP
                     and issue OIDC tokens to your downstream services (bridge pattern)
```

### SAML Roles

```
Identity Provider (IdP): authenticates the user; issues assertions
  Examples: Okta, Azure AD, ADFS, Google Workspace, ForgeRock AM (as IdP)
  Has: user directory, authentication policies, MFA enforcement
  Issues: signed XML assertion ("Alice is authenticated; she's in role MERCHANT_ADMIN")

Service Provider (SP): the application being accessed; consumes assertions
  Examples: your payment dashboard, Salesforce, JIRA
  Does NOT: authenticate the user; trust the IdP's assertion
  Validates: assertion signature, expiry, audience, conditions
  ForgeRock AM can also act as SP (receives assertions from enterprise IdP)

Metadata Exchange:
  Before federation: SP and IdP exchange XML metadata files
  SP metadata: entityID, ACS URL, signing certificate
  IdP metadata: entityID, SSO URL, signing certificate
  This is the "trust establishment" step (done once; updated on cert rotation)
```

---

## Problem Statement

SAML SSO integration for enterprise merchant employees accessing the payment dashboard. ForgeRock AM as SP. JIT provisioning. Attribute-to-role mapping. SAML-to-OIDC bridge for downstream services.

---

## SP-Initiated SSO Flow (Standard)

```
Employee (Alice, HDFC Bank) → payment dashboard (your SP)

Step 1: Access protected resource
  Alice: https://dashboard.platform.com/merchant/hdfc/payments
  Dashboard: not authenticated → redirect to SAML SSO flow

Step 2: SP generates AuthnRequest
  Dashboard (SP) creates:
    <samlp:AuthnRequest
      ID="req-123"
      Version="2.0"
      IssueInstant="2024-01-01T14:00:00Z"
      AssertionConsumerServiceURL="https://dashboard.platform.com/saml/acs"
      Destination="https://hdfc-idp.bank.com/sso"
      ForceAuthn="false"
      IsPassive="false">
      <saml:Issuer>https://dashboard.platform.com</saml:Issuer>
      <samlp:NameIDPolicy Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress"/>
    </samlp:AuthnRequest>
  
  Base64-encode + URL-encode → redirect Alice to:
  https://hdfc-idp.bank.com/sso?SAMLRequest=PHNhbWxwOkF1...&RelayState=original-url

Step 3: IdP authenticates Alice
  HDFC's IdP (Azure AD / ADFS):
    Alice logs in with corporate credentials + MFA
    IdP verifies: Alice is active employee; in MERCHANT_ADMIN role

Step 4: IdP creates and signs SAML Assertion
  <samlp:Response Destination="https://dashboard.platform.com/saml/acs">
    <saml:Assertion>
      <saml:Issuer>https://hdfc-idp.bank.com</saml:Issuer>
      <ds:Signature>...</ds:Signature>    ← signed with IdP's private key
      <saml:Subject>
        <saml:NameID>alice@hdfc.com</saml:NameID>
      </saml:Subject>
      <saml:Conditions
        NotBefore="2024-01-01T13:59:00Z"
        NotOnOrAfter="2024-01-01T14:05:00Z">   ← 5-minute validity window
        <saml:AudienceRestriction>
          <saml:Audience>https://dashboard.platform.com</saml:Audience>
        </saml:AudienceRestriction>
      </saml:Conditions>
      <saml:AttributeStatement>
        <saml:Attribute Name="email"><saml:AttributeValue>alice@hdfc.com</saml:AttributeValue></saml:Attribute>
        <saml:Attribute Name="displayName"><saml:AttributeValue>Alice Kumar</saml:AttributeValue></saml:Attribute>
        <saml:Attribute Name="role"><saml:AttributeValue>MERCHANT_ADMIN</saml:AttributeValue></saml:Attribute>
        <saml:Attribute Name="department"><saml:AttributeValue>Treasury</saml:AttributeValue></saml:Attribute>
        <saml:Attribute Name="tenantId"><saml:AttributeValue>merch-hdfc-001</saml:AttributeValue></saml:Attribute>
      </saml:AttributeStatement>
    </saml:Assertion>
  </samlp:Response>
  
  POST to ACS URL: https://dashboard.platform.com/saml/acs
  (HTTP POST form with SAMLResponse=base64-encoded-XML and RelayState=original-url)

Step 5: SP validates assertion (CRITICAL)
  Validation checklist — ALL must pass:
    □ Signature: valid? (signed by HDFC IdP's certificate from SP metadata)
    □ Issuer: matches registered HDFC IdP entityID
    □ NotBefore / NotOnOrAfter: current time is within validity window
    □ Audience: includes your SP's entityID (prevents assertion replay to other SPs)
    □ InResponseTo: matches your AuthnRequest ID (prevents unsolicited assertions)
    □ AssertionID: not seen before (replay prevention; store for 5-minute validity window)
  
  If any check fails: REJECT; do not authenticate user

Step 6: JIT provisioning + session creation
  User alice@hdfc.com exists in your DB? → update last_login; map roles
  User does NOT exist? → create account (JIT provisioning):
    INSERT INTO users (email, name, tenant_id, role, idp_source)
    VALUES ('alice@hdfc.com', 'Alice Kumar', 'merch-hdfc-001', 'MERCHANT_ADMIN', 'hdfc-saml')
  
  Create session; redirect to original URL (RelayState)
```

---

## IdP-Initiated SSO Flow

```
Less common; enterprise portals use it:
  HDFC employee opens: https://hdfc-portal.bank.com
  Clicks: "Payment Dashboard" link
  HDFC portal (IdP): generates assertion WITHOUT prior AuthnRequest from SP
  POST assertion to SP ACS URL directly

Security risk: no InResponseTo check (no prior AuthnRequest)
Mitigation: validate all other assertion fields strictly; accept only from registered IdPs
SP configuration: explicitly allow IdP-initiated from trusted IdPs only
```

---

## Attribute Mapping to Platform Roles

```java
@Component
public class SamlAttributeMapper {

    private final Map<String, String> ROLE_MAPPING = Map.of(
        "MERCHANT_ADMIN",     "PLATFORM_MERCHANT_ADMIN",
        "MERCHANT_VIEWER",    "PLATFORM_MERCHANT_READ_ONLY",
        "TREASURY_OFFICER",   "PLATFORM_MERCHANT_FINANCE",
        "COMPLIANCE_OFFICER", "PLATFORM_MERCHANT_COMPLIANCE"
    );

    public UserPrincipal map(SamlAssertion assertion, String tenantId) {
        String email = assertion.getAttribute("email");
        String name  = assertion.getAttribute("displayName");
        String rawRole = assertion.getAttribute("role");

        // Map IdP role to platform role
        String platformRole = ROLE_MAPPING.getOrDefault(rawRole, "PLATFORM_MERCHANT_READ_ONLY");
        // Fallback: unknown roles get read-only access (safe default)

        // Tenant from assertion (or from SP metadata for that IdP)
        String assertedTenantId = assertion.getAttribute("tenantId");
        if (!tenantId.equals(assertedTenantId)) {
            // SECURITY: assertion claims different tenant than expected → reject
            throw new SecurityException("Tenant mismatch in SAML assertion");
        }

        return UserPrincipal.builder()
            .email(email)
            .name(name)
            .tenantId(tenantId)
            .role(platformRole)
            .idpSource("saml:" + assertion.getIssuer())
            .build();
    }
}
```

---

## SAML-to-OIDC Bridge (ForgeRock Pattern)

```
Problem: your internal services use OIDC (modern); enterprise IdP speaks SAML
Solution: ForgeRock AM acts as the bridge

Flow:
  Alice → SP (ForgeRock AM) → SAML exchange with HDFC IdP → AM issues OIDC token
  
  ForgeRock AM receives SAML assertion → validates → creates AM session
  Alice's app: exchanges AM session for OIDC access token (standard OIDC flow)
  Internal services: receive OIDC JWT (no SAML knowledge needed)

ForgeRock AM configuration:
  SAML SP: configure HDFC IdP as remote IdP (import their metadata)
  Identity Mapping: map SAML NameID (email) to AM user identity
  JIT Provisioning: create AM user from SAML attributes on first login
  OAuth2 Provider: AM issues access_token after SAML authentication
  Token claims: SAML attributes (role, tenantId) → JWT custom claims

This is the exact pattern for payment platforms:
  External enterprise SAML → ForgeRock AM SP → OIDC tokens for internal use
  Developers never deal with SAML XML after AM → all downstream is clean JWT
```

---

## SCIM Provisioning (User Lifecycle)

```
Problem: Alice leaves HDFC. SAML only prevents her from getting NEW sessions.
  Her existing session or unexpired token still works.
  More critically: her account still exists in your system with MERCHANT_ADMIN role.

Solution: SCIM (System for Cross-domain Identity Management) — RFC 7642/7643/7644
  HDFC's directory (Okta, Azure AD) pushes user lifecycle events to your SCIM endpoint
  
  Provisioning (create user when hired):
    POST /scim/v2/Users
    { "userName": "alice@hdfc.com", "active": true, "roles": ["MERCHANT_ADMIN"] }
    → JIT provisioning can be SCIM-driven instead of at login time
  
  Deprovisioning (disable when fired):
    PATCH /scim/v2/Users/alice-id
    { "Operations": [{"op": "replace", "path": "active", "value": false}] }
    → IMMEDIATELY disable Alice's account and revoke active sessions
    → THIS is what SAML SSO alone cannot do (no deprovisioning signal)
  
  Group sync:
    PUT /scim/v2/Groups/merchant-admins
    { "members": [...] }
    → Role changes propagate automatically

  ForgeRock OpenIDM: SCIM server implementation; also handles connectors to AD/LDAP
  ForgeRock AM + OpenIDM = full SSO + lifecycle management for enterprise merchants
  
SCIM vs SAML JIT:
  SAML JIT: creates user at first login; no deletion signal
  SCIM: full lifecycle (create, update, disable, delete); recommended for enterprise
  Best practice: SCIM + SAML together; SCIM for lifecycle; SAML for authentication
```

---

## Security Considerations

```
SAML-specific attacks:

1. XML Signature Wrapping (XSW):
   Attack: attacker modifies unsigned parts of assertion; signature still valid
   Mitigation: validate signature over ENTIRE assertion body; use canonicalization
   ForgeRock AM: handles this correctly by default; don't implement custom SAML parsing

2. Replay Attack:
   Attack: capture valid assertion; replay to SP after 5 minutes
   Mitigation: check NotOnOrAfter; store AssertionID in Redis for validity window
   Redis: SET assertion-id "1" EX 300 (5-minute window); reject if key already exists

3. Open Redirect via RelayState:
   Attack: craft malicious RelayState URL; SP redirects user to attacker's site
   Mitigation: validate RelayState against allowlisted origins
   Code: assert(relayState.startsWith("https://dashboard.platform.com"))

4. Assertion Forgery (Unsolicited):
   Attack: attacker sends forged assertion (not in response to SP's request)
   Mitigation: InResponseTo check; store pending AuthnRequest IDs; verify match
   For IdP-initiated: additional source IP validation; only allow from registered IdP

5. Certificate Expiry:
   Risk: IdP rotates their signing certificate; old assertions rejected
   Mitigation: monitor IdP certificate expiry; alert 90 days before; update metadata
   ForgeRock: certificate expiry alerts in AM console; automated metadata refresh
```

---

## Observability
```
SAML-specific metrics:
  saml_authentication_success_total by {idp_entity_id}
  saml_authentication_failure_total by {failure_reason, idp_entity_id}
  saml_assertion_validation_errors by {error_type}  (signature, expired, audience, replay)
  saml_jit_provisioning_total                       (new users created via SAML)
  saml_session_duration_seconds                     (how long SAML sessions last)
  idp_certificate_expiry_days by {idp_entity_id}    (alert if < 90 days)
  
Alert:
  saml_assertion_validation_errors type=replay > 0 → possible replay attack; investigate
  idp_certificate_expiry_days < 30 → urgent: update IdP metadata before cert expires
  saml_authentication_failure_rate > 5% → misconfiguration or IdP issue
```

---

## Architecture Review Checklist
```
□ Assertion validation (ALL must pass before authenticating user):
  → Signature: valid against IdP certificate from trusted metadata
  → Expiry: NotOnOrAfter > now
  → Audience: your SP entityID is in AudienceRestriction
  → InResponseTo: matches your pending AuthnRequest (SP-initiated)
  → AssertionID: not seen before (replay prevention)
  → Issuer: matches registered IdP entityID

□ JIT provisioning:
  → Create user account on first SAML login
  → Map SAML attributes → platform roles (safe default for unknown roles)
  → Tenant isolation: tenantId from assertion must match configured tenant for that IdP

□ Full lifecycle:
  → SAML: handles authentication (first login, session)
  → SCIM: handles provisioning/deprovisioning (hire/fire events)
  → Without SCIM: ex-employees' accounts persist indefinitely

□ SAML-to-OIDC bridge:
  → ForgeRock AM: SP receives SAML; issues OIDC tokens downstream
  → Internal services: see only OIDC JWT (no SAML complexity)

□ SCC LENS:
  STATE: ForgeRock AM (session store, user profiles, IdP metadata)
  COORDINATION: assertion validation (stateful: must remember pending AuthnRequest IDs)
  CONCENTRATION: ForgeRock AM is the federation hub → HA cluster mandatory

□ IAM EXPERTISE TIE-IN:
  This is your core domain: ForgeRock AM as SAML SP
  HDFC ADFS / Azure AD as enterprise IdP
  OpenIDM for SCIM provisioning / JIT user creation
  The interview answer to "how do you integrate enterprise SSO" = this case study
```

---

*Next: Case Study 43 — Fine-Grained Authorization (OPA / ReBAC)*
