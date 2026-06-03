# Case Study 48 — IAM Attack Vectors and Threat Modelling

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Credential stuffing defence, OAuth token attacks, JWT weaknesses.

---

## Business Context

Understanding how IAM systems are attacked is inseparable from designing them correctly. Every security control in this handbook — MFA, short-lived tokens, PKCE, refresh token rotation — exists because there is a specific, real attack it defends against. A Staff Engineer who knows the attack model knows WHY the controls are designed the way they are, and can reason about novel threats.

**Business goals:**
- Understand the attack model for every IAM control you build
- Threat model new features before they're shipped
- Respond to incidents with clear attribution of attack vector
- Design controls that are proportionate to the threat level

---

## Recognition Framework

### The Attack Surface of IAM

```
IAM attacks target four things:
1. Credentials (steal the password/token)
2. The authentication protocol (exploit weaknesses in OAuth, SAML, JWT)
3. The authorisation model (escalate privileges)
4. The session (hijack an active session)
```

---

## Attack 1: Credential Stuffing

```
Attack: attacker has 100M (email, password) pairs from previous breaches
  Tries each pair against YOUR login form
  Users who reuse passwords → compromised
  Success rate: 0.1–2% (enough to compromise millions of accounts)

Indicators:
  Login success rate drops (many failed attempts before successful one)
  Many logins from residential proxy IP ranges (attack uses rotating IPs)
  Login attempts for non-existent accounts (using leaked email lists)

Defences:

1. Rate limiting per IP (Case 25 pattern):
   Soft limit: 10 failures/min per IP → CAPTCHA required
   Hard limit: 50 failures/min per IP → 429 for 10 minutes
   
2. Rate limiting per account:
   5 consecutive failures on alice@email.com → lock account; send reset email
   (Lockout threshold: 5 is typical; too low causes DoS on valid users)

3. Breached password check:
   On login: check password against HaveIBeenPwned API (k-anonymity; safe)
   k-anonymity: SHA1(password), send first 5 chars to HIBP API, compare against returned hashes
   If match: force password reset before allowing login

4. Device fingerprinting:
   Store device_id for trusted devices (after MFA verification)
   New device + correct password: require MFA (credential stuffed account → MFA blocks attacker)
   Known device + correct password: no MFA (legitimate user; not stuffed)

5. Behavioral signals:
   Login from unexpected location/time: flag for MFA even on correct password
   Velocity: 100 accounts logged in from same IP in 5 min → block IP

Implementation: login attempts counter in Redis
  INCR login:failed:{ip}:{minute}
  INCR login:failed:account:{email}:{hour}
  If either exceeds threshold: apply friction (CAPTCHA, lockout, forced MFA)
```

---

## Attack 2: OAuth2 Attacks

### Authorization Code Interception
```
Attack (without PKCE):
  1. Victim clicks "Login with Google"
  2. Auth server redirects to: app://callback?code=abc123
  3. Malicious app installed on same device intercepts the redirect
  4. Attacker exchanges code=abc123 for tokens

Defence: PKCE (Case 41 covers mechanics)
  PKCE: code only usable by app that generated code_verifier
  Without code_verifier: code is useless
  No legitimate public client (mobile/SPA) should use authorization code without PKCE
```

### Token Leakage via Redirect URI
```
Attack: attacker registers malicious redirect URI
  1. Attacker registers: evil.com/callback
  2. Crafts URL: /authorize?redirect_uri=https://evil.com/callback&...
  3. If auth server allows arbitrary redirect URIs: tokens go to attacker

Defence: strict redirect URI validation
  Auth server: only allow pre-registered redirect URIs
  No: prefix matching, subdomain matching (attacker registers evil.legit.com)
  Yes: exact match only (https://app.platform.com/callback exactly)
  ForgeRock AM: redirect URIs registered per OAuth2 client; no wildcards
```

### Access Token in Browser History / Referrer
```
Attack: access token in URL fragment or query param → browser history → referrer header
  Implicit grant: access_token in URL fragment (DEPRECATED for this reason)
  Bad code: /dashboard?access_token=eyJ... → token in URL → browser history
  
Defence: 
  Never put tokens in URL parameters
  Authorization Code flow: tokens only in response body (not URL)
  SPA: store tokens in memory (not localStorage); use HttpOnly cookie for refresh token
```

### CSRF on OAuth Callback
```
Attack: no state parameter → attacker forges login completion
  1. Attacker starts OAuth flow for their account
  2. Gets auth code: code=attacker_code
  3. Tricks victim into submitting: POST /callback?code=attacker_code
  4. Victim's session is now linked to attacker's account → attacker can see victim's data

Defence: state parameter (Case 41 covers mechanics)
  state = random_value stored in user's session before redirect
  On callback: verify state matches → confirms callback is from the flow the user started
  SameSite=Lax cookies also mitigate CSRF in most modern browsers
```

---

## Attack 3: JWT Attacks

### Algorithm Confusion (alg: none)
```
Attack: attacker modifies JWT to use algorithm "none"
  Original JWT: {"alg":"RS256"}.payload.signature
  Tampered JWT: {"alg":"none"}.modified_payload.(no signature)
  Vulnerable server: accepts alg=none → no signature verification → arbitrary claims

Defence:
  NEVER accept alg=none (or only if explicitly configured)
  Explicitly whitelist accepted algorithms: only RS256 or ES256
  
  Java (JJWT):
  Jwts.parser()
      .verifyWith(publicKey)
      .build()
      .parseSignedClaims(token);
  // JJWT: does not accept alg=none by default; explicit key required
```

### Key Confusion Attack (RS256 → HS256)
```
Attack: server uses RS256 (asymmetric); attacker switches to HS256 (symmetric)
  HS256 expects a shared secret for verification
  Attacker: uses the PUBLIC key as the HS256 secret
  Vulnerable server: "HS256? Let me verify with my public key (which attacker knows)"
  → Verification passes; attacker made their own valid token

Defence:
  Server: explicitly configure expected algorithm (do not trust alg header)
  Never: auto-detect algorithm from the JWT header
  
  ```java
  // WRONG: trust JWT header algorithm
  Jwts.parser().parse(token);  // alg comes from JWT header
  
  // CORRECT: enforce algorithm at server
  Jwts.parser()
      .verifyWith(publicKey)  // forces RS256 with this key; HS256 would fail
      .build()
      .parseSignedClaims(token);
  ```
```

### JWT Secret Brute Force
```
Attack: HS256 JWT signed with weak secret ("password123")
  Attacker: takes any valid JWT → tries to crack the signing secret
  If secret is short/predictable: cracked in seconds (hashcat GPU cracking)
  Attacker: signs their own JWT with cracked secret → arbitrary claims

Defence:
  Use asymmetric algorithms (RS256 / ES256): public key for verification, private key for signing
  If HS256: secret must be cryptographically random, >= 256 bits (32 bytes)
  Rotate signing keys regularly (key rotation via JWKS endpoint, Case 41)
```

### JWT Claim Injection
```
Attack: server trusts JWT claims without validation
  JWT: { "sub": "alice", "is_admin": false }
  Attacker modifies (if signature not validated): { "sub": "alice", "is_admin": true }
  Vulnerable server: reads is_admin from JWT without signature check → admin access

Defence: always validate JWT signature before reading any claim
  Order of operations:
    1. Verify signature (reject if invalid; do not proceed)
    2. Verify expiry (reject if expired)
    3. Verify audience (reject if wrong recipient)
    4. THEN read claims
  Never read claims from unvalidated JWT
```

---

## Attack 4: SAML Attacks

### XML Signature Wrapping (XSW)
```
Attack: SAML assertions contain signed elements
  Attacker: duplicates the signed element; modifies the unsigned copy
  Positions the modified copy where the SP reads from (not the signed location)
  
  Example:
    Original (signed): <Assertion ID="A"><AuthnStatement>Alice</AuthnStatement></Assertion>
    Tampered: <Assertion ID="A-tampered"><AuthnStatement>Admin</AuthnStatement></Assertion>
              <Assertion ID="A"><AuthnStatement>Alice</AuthnStatement></Assertion>  (signed)
    
    SP reads the first (unsigned) assertion: "This says Admin; assertion is signed (the second one)"
    
Attack lets attacker authenticate as any user if SP is vulnerable

Defence:
  Use a well-tested SAML library (not custom XML parsing)
  Verify: the signed assertion is the SAME one being used for authentication
  ForgeRock AM: verified to be resistant to XSW (do not write custom SAML parsing)
```

---

## Attack 5: Session Attacks

### Session Fixation
```
Attack:
  1. Attacker obtains session_id from unauthenticated access to app
  2. Tricks victim into authenticating with that session_id
  3. Now attacker's session_id is authenticated → attacker is logged in as victim

Defence:
  On successful authentication: ALWAYS generate a NEW session_id
  Discard the pre-authentication session_id
  ForgeRock AM: does this correctly; custom code must explicitly regenerate session
```

### Session Riding (CSRF for session-based apps)
```
Attack: malicious site makes requests to your app using victim's session cookie
  <img src="https://platform.com/api/v1/payments/pay-123/refund?amount=5000">
  If GET requests have side effects: victim's browser makes request with their cookie

Defence:
  SameSite=Strict cookies: cookie not sent on cross-site requests (most effective)
  SameSite=Lax: sent on top-level navigation only (weaker but better UX)
  CSRF tokens: check anti-CSRF token on state-changing requests
  Modern apps: API-only (no cookies) → CSRF not applicable if using Authorization header
```

---

## Attack 6: Privilege Escalation

```
Attack: user A accesses resource belonging to user B
  Example: GET /api/v1/payments/pay-123 where pay-123 belongs to Merchant B
  Attacker is Merchant A but increments payment_id to access pay-123

Defence: Insecure Direct Object References (IDOR) prevention
  Never: trust user-provided IDs without checking ownership
  Always: SELECT payment WHERE id=? AND tenant_id = current_user.tenant_id
  OPA policy: payment.tenant_id must equal user.tenant_id (Case 43)
  
OAuth scope escalation:
  Attacker: has token with scope payments:read
  Tries: POST /payments (requires payments:create)
  Defence: enforce scope at gateway; reject if required scope not in token

Role escalation via JWT manipulation:
  Attacker: modifies role claim in JWT from USER to ADMIN
  Defence: verify JWT signature (Case 41); signature prevents modification
```

---

## Threat Modelling Process (STRIDE)

```
For every new feature, run STRIDE analysis:
  S - Spoofing: can attacker pretend to be someone else?
      → MFA, JWT signature, SAML assertion validation
  T - Tampering: can attacker modify data in transit or at rest?
      → TLS, JWT signature, database permissions, immutable logs
  R - Repudiation: can attacker deny their actions?
      → Audit log (Case 10), non-repudiation (signed payment confirmations)
  I - Information Disclosure: can attacker see data they shouldn't?
      → Tenant isolation (Case 26), GDPR pseudonymisation, access control (Case 43)
  D - Denial of Service: can attacker prevent legitimate use?
      → Rate limiting (Case 25), DDoS protection, circuit breakers
  E - Elevation of Privilege: can attacker gain more access than granted?
      → Least privilege, SoD (Case 45), IDOR prevention, scope enforcement

Example STRIDE on "Merchant API key":
  S: Can attacker forge an API key?
     → Hmac-based key; key hash in DB; cannot forge without secret
  T: Can attacker modify a payment request in transit?
     → TLS 1.3; HMAC request signing (AWS Signature V4 style for sensitive ops)
  R: Can merchant deny creating a payment?
     → API key in audit log; payment_id with timestamp → non-repudiable
  I: Can merchant see another merchant's payments?
     → tenant_id in every query; OPA enforcement; RLS
  D: Can merchant flood our API?
     → Rate limiting per API key (Case 25); quota by tier
  E: Can merchant access admin endpoints?
     → Scope enforcement; admin scope not grantable to merchant clients
```

---

## Observability + Security Alerts
```
Attack detection metrics:
  login_failure_rate_spike by {ip}             (credential stuffing indicator)
  login_success_rate by {device_type}          (new device high failure → stuffing)
  impossible_travel_detections                 (session hijack indicator)
  jwt_validation_failures by {reason}          (alg confusion, signature failure)
  saml_assertion_replay_attempts               (SAML replay attack)
  idor_denied_total                            (cross-tenant access attempts)
  privilege_escalation_blocked_total           (scope violation attempts)
  api_key_brute_force_attempts by {key_prefix} (key guessing attack)

Security incident thresholds:
  login_failure_rate > 1000/min from one IP: → block IP; alert security
  impossible_travel_detected: → email user; require MFA on suspicious session
  jwt_algorithm_confusion_detected: → immediate alert security; audit JWT validation code
  saml_replay_attempt: → alert security; check for broader compromise
```

---

## Architecture Review Checklist
```
□ Credential stuffing defences:
  → Rate limiting per IP and per account
  → CAPTCHA after N failures
  → Breached password check on login
  → MFA as final backstop (stolen password + no MFA = still blocked)

□ OAuth2 attack surface:
  → PKCE on all public clients (mobile, SPA)
  → Exact redirect URI matching (no wildcards)
  → state parameter on all flows (CSRF protection)
  → No tokens in URL parameters or logs

□ JWT security:
  → Explicit algorithm whitelist (no alg=none; no HS256/RS256 confusion)
  → Validate: signature → expiry → audience → issuer → THEN read claims
  → Asymmetric keys (RS256/ES256) for production JWT signing

□ SAML security:
  → Use battle-tested SAML library (no custom XML parsing)
  → Validate: signature → expiry → audience → InResponseTo → replay check
  → Replay prevention: AssertionID in Redis for validity window

□ Session security:
  → Regenerate session_id on authentication (fixation prevention)
  → SameSite=Strict/Lax cookies (CSRF protection)
  → Immediate revocation capability (Redis session store)

□ SCC LENS:
  STATE: attack counters (Redis), blocklists (Redis), threat intelligence feeds
  COORDINATION: anomaly detection → MFA step-up; credential stuffing → IP block
  CONCENTRATION: authentication service is the highest-value target → HA + WAF + DDoS protection
```

---

*Next: Case Study 49 — AI/ML in Payments (Fraud, Anomaly, Forecasting)*
