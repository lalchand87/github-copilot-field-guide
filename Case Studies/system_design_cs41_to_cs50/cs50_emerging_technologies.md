# Case Study 50 — Emerging Technologies: Passkeys, Decentralised Identity, CAEP, ITDR

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Where these are production-ready vs experimental, Your IAM context.

---

## Business Context

The IAM landscape is rapidly evolving. Passkeys are replacing passwords in 2024–2025. Decentralised identity (W3C DIDs + Verifiable Credentials) is emerging for regulatory compliance. CAEP (Continuous Access Evaluation Profile) enables real-time session revocation across federated systems. ITDR (Identity Threat Detection and Response) combines identity signals with security operations.

A Staff Engineer needs to know: which emerging technology is hype vs ready, when to adopt vs wait, and how each fits into the existing payment platform architecture.

---

## Recognition Framework

### Technology Readiness Tiers

```
Production-ready NOW (adopt):
  Passkeys (FIDO2/WebAuthn): Apple, Google, Microsoft all support natively
  Hardware security keys (YubiKey): mature; use for high-privilege access

Production-ready, limited adoption:
  CAEP / RISC: implemented by Google, Microsoft; others catching up
  SCIM 2.0: mature standard; widely supported

Early production (selective use):
  Decentralised Identity (W3C DID + VC): used in specific regulated industries
  ITDR: emerging category; Okta, CrowdStrike, Microsoft have early products

Research / future:
  Zero-knowledge proofs for identity
  Post-quantum cryptography for PKI (NIST standardised 2024; transition begins)
```

---

## Passkeys (Production-Ready NOW)

### What Passkeys Change
```
Password problem: 80% of breaches involve stolen/weak passwords
Traditional solution: MFA (password + OTP) — still requires a password + phishable OTP
Passkey solution: replaces password entirely with public-key cryptography bound to your device

How passkeys differ from FIDO2 security keys:
  Security key (FIDO2 2FA): hardware token; second factor; still requires password as first factor
  Passkey: first AND only factor; stored on device (phone, Mac); synced via iCloud/Google account
  
  Passkey = FIDO2 credential + sync across devices + no password required
  
UX comparison:
  Password login: type email → type password → OTP → enter OTP
  Passkey login: enter email → device prompts Face ID / fingerprint → done
  No password to forget, steal, or phish

Platform support (2024):
  iOS 16+: passkeys in Safari and apps
  macOS Ventura+: passkeys in Safari
  Android 9+: passkeys via Google Password Manager
  Windows 11: passkeys via Windows Hello or iCloud Keychain
  Chrome, Edge, Firefox: WebAuthn support
  
  → Any modern device can use passkeys today
```

### Passkey Implementation for Payment Platform
```java
// Extending Case 44 (MFA) — passkeys replace password + MFA combined

// Registration (same WebAuthn API as Case 44; passkey-specific settings)
@PostMapping("/api/v1/passkeys/register/options")
public PublicKeyCredentialCreationOptions registerOptions(@AuthUser String userId) {
    return PublicKeyCredentialCreationOptions.builder()
        .rp(new RelyingPartyIdentity("platform.com", "Payment Platform"))
        .user(getUserIdentity(userId))
        .challenge(generateChallenge())
        .pubKeyCredParams(List.of(
            new PublicKeyCredentialParameters(Algorithm.ES256),
            new PublicKeyCredentialParameters(Algorithm.RS256)
        ))
        .authenticatorSelection(
            AuthenticatorSelectionCriteria.builder()
                .residentKey(ResidentKeyRequirement.REQUIRED)   // REQUIRED for passkeys
                .userVerification(UserVerificationRequirement.REQUIRED)  // biometric/PIN
                .build()
        )
        .build();
}

// Key differences for passkeys vs Case 44 FIDO2:
//   residentKey: REQUIRED (credential stored on device; not just a key for a hardware token)
//   This enables: login without username (device presents passkey identity)
//   sync: handled by platform (iOS/Google); not your responsibility

// Login flow (passwordless):
//   User opens app → taps "Sign in with passkey"
//   Device shows: "Use Face ID for platform.com?" → user authenticates
//   App: sends assertion to server; server verifies; session created
//   No password field shown; no OTP sent
```

---

## Decentralised Identity (W3C DID + Verifiable Credentials)

### Concepts
```
Current identity model (centralised):
  You prove identity by: presenting credential issued by central authority
  Example: PaymentPlatform verifies your identity via OTP to their own user DB
  Problem: every platform stores your identity; you can't control it; data breaches
  
Decentralised Identity model:
  DID (Decentralised Identifier): your globally unique identifier (like email but on blockchain or distributed storage)
  Format: did:web:platform.com:users:alice OR did:ion:abc123 (Microsoft ION)
  Verifiable Credential (VC): a claim about you, signed by an issuer
  Example: Bank issues VC: "DID:alice has verified KYC; Indian citizen; age > 18"
  
  You present VCs to platforms; platforms verify the issuer's signature
  Platforms don't need to store your data; they just verify the credential

Payment platform use case:
  User: has KYC VC from their bank (Aadhaar-linked)
  Payment Platform: instead of doing KYC itself, accepts bank's KYC VC
  User: presents VC (signed by bank) → platform verifies bank's signature → trusted
  
  Benefits:
    Platform: no need to store KYC data (GDPR simplification)
    User: one-time KYC; reuse across platforms ("reusable KYC")
    Privacy: selective disclosure (prove age > 18 without revealing exact birthdate)

India context: NPCI eRUPI, Account Aggregator framework, and DigiLocker are steps toward this
```

### Verifiable Credential Flow
```
Issuer (HDFC Bank):
  Creates VC: {
    "@context": "https://www.w3.org/2018/credentials/v1",
    "type": ["VerifiableCredential", "KYCCredential"],
    "issuer": "did:web:hdfc.bank.com",
    "credentialSubject": {
      "id": "did:web:platform.com:users:alice",
      "name": "Alice Kumar",
      "kycLevel": "FULL",
      "isIndianCitizen": true,
      "isAdult": true     // selective disclosure: age > 18, not exact birthdate
    }
  }
  Signs VC with bank's private key

Alice stores VC in her digital wallet (app or browser)

Payment Platform (Verifier):
  Requests: "please present a KYC credential proving you're an adult Indian citizen"
  Alice: selects and presents the HDFC KYC VC from her wallet
  Platform: verifies:
    Signature: valid against HDFC bank's DID document public key
    Issuer: HDFC bank is a trusted KYC issuer (from trust registry)
    Claim: isAdult=true AND isIndianCitizen=true → requirements met
  Platform: creates account (no need to re-do KYC)

Current reality (2024):
  W3C DID/VC: production in EU eIDAS 2.0 regulation (mandatory for EU member states)
  India: DigiLocker is proto-VC; Account Aggregator is proto-VC for financial data
  Adoption: still early; not yet interoperable across all platforms
  Recommendation: watch; pilot in regulatory sandbox; not yet default architecture
```

---

## CAEP: Continuous Access Evaluation Profile

### Problem CAEP Solves
```
Current session model problem:
  User authenticates → gets 1-hour access token
  During that 1 hour: user is terminated from company; IT revokes access in IdP
  BUT: their access token is still valid for remaining time (JWT cannot be revoked)
  Result: 30-59 minutes of unrevoked access after termination
  
  For payment platforms: terminated employee with access token → 30-59 min of risk

CAEP (OpenID CAEP): RFC / OpenID specification for real-time session signals
  IdP pushes signals to RP (Relying Party) when session state changes
  Signal types:
    session_revoked: "Alice's session was revoked; stop accepting her tokens"
    credential_change: "Alice changed her password; invalidate old sessions"
    token_claims_change: "Alice's role changed; refresh her claims"
    assurance_level_change: "Alice's MFA level dropped; require re-auth"

CAEP architecture:
  Signal Provider (IdP): ForgeRock AM / Okta / Azure AD
  Signal Receiver (RP): your payment platform
  
  On session revocation event:
    IdP: POST https://platform.com/.well-known/caep { subject: alice@email.com, event: session_revoked }
    Platform: immediately revoke Alice's sessions (Case 46 pattern)
    Alice: next request → 401 → must re-authenticate

ForgeRock AM: implements CAEP as both provider and receiver
  Your context: can configure AM to push CAEP signals to downstream apps
              and receive signals from upstream IdPs (e.g., Azure AD)
```

### CAEP Implementation
```java
// CAEP signal receiver endpoint (your platform)
@PostMapping("/.well-known/caep")
public void handleCaepSignal(@RequestBody CaepSignal signal,
                              @RequestHeader("Authorization") String signature) {
    // Verify signal authenticity (HMAC or JWT-signed by IdP)
    if (!signalVerifier.verify(signal, signature)) {
        return; // reject unauthenticated signals
    }
    
    String subjectId = signal.getSubject().getUser();
    
    switch (signal.getEventType()) {
        case "session_revoked":
        case "credential_change":
            // Immediate: revoke ALL sessions for this user
            sessionService.revokeAllSessions(subjectId);
            // Effective within milliseconds: next request fails
            break;
        
        case "token_claims_change":
            // Refresh claims: mark user's current tokens as needing refresh
            sessionService.markClaimsStale(subjectId);
            // Next token refresh: re-fetches claims from IdP
            break;
        
        case "assurance_level_change":
            // Downgrade event: require step-up auth for sensitive operations
            sessionService.requireStepUp(subjectId, "HIGH_VALUE_PAYMENT");
            break;
    }
    
    auditLog.record("CAEP_SIGNAL_RECEIVED", signal.getEventType(), subjectId);
}

// This closes the gap: employee terminated → IdP sends CAEP → platform revokes → immediate
```

---

## ITDR: Identity Threat Detection and Response

### What ITDR Is
```
ITDR = bringing Security Operations Center (SOC) practices to identity

Traditional SOC: monitors network traffic, endpoint logs, application logs
ITDR: monitors identity events specifically
  Sources: IdP authentication logs, OAuth token issuance, directory changes, MFA events
  
  Detection:
    Impossible travel (same user: login Mumbai + login London within 2 hours)
    Credential stuffing (many failed logins across many accounts)
    Token theft (token used from unexpected IP/device after issuance)
    Privilege escalation (user suddenly granted high-privilege role at unusual time)
    Shadow IT (OAuth authorisation to unknown application)
    MFA bypass attempts (many MFA failures before success)
  
  Response (automated or human-in-loop):
    Risk-based: low confidence → challenge with MFA; high confidence → revoke session
    Block: device blocked; user account locked
    Alert: security analyst investigates
    Playbook: trigger ForgeRock AM intelligent authentication tree

ITDR vendors (2024):
  Microsoft Defender for Identity: strong AD/Entra integration
  Okta Identity Threat Protection: Okta-native
  CrowdStrike Falcon Identity Protection: endpoint + identity combined
  ForgeRock (Ping One Protect): risk-based intelligent authentication
  
Your context: ForgeRock AM's "Intelligent Authentication" feature set
  is the ForgeRock approach to ITDR: real-time risk signals → authentication tree decisions
```

---

## Post-Quantum Cryptography (Planning Horizon)

```
Current PKI: RSA and EC cryptography
  Secure today against classical computers
  Vulnerable to quantum computers running Shor's algorithm (breaks RSA/EC)
  
  Quantum timeline: 2030s likely for cryptographically relevant quantum computers
  Risk: "harvest now, decrypt later" — attackers store encrypted traffic today; decrypt when quantum arrives
  
NIST Post-Quantum Standards (2024):
  ML-KEM (CRYSTALS-Kyber): key encapsulation (replaces RSA/EC for key exchange)
  ML-DSA (CRYSTALS-Dilithium): digital signatures (replaces RSA-PSS, ECDSA)
  SLH-DSA (SPHINCS+): hash-based signature (backup option)
  
  Finalised 2024: now available in OpenSSL 3.x, BouncyCastle, AWS
  
Migration timeline for payment platform:
  2024-2026: inventory all cryptographic usage (TLS certs, JWT signing, API signing)
  2026-2028: begin dual-algorithm support (old + quantum-safe together)
  2028-2030: migrate critical systems (card vault, audit log signing)
  2030+: deprecate classical algorithms
  
  Your most sensitive case: card vault HSM keys and JWT signing keys
  These are most sensitive to "harvest now, decrypt later"
  
  Practical now: implement crypto agility (abstraction layer that can swap algorithms)
  Not practical now: full migration (still early; finalised standards just arrived)
```

---

## Architecture Decisions: Adopt Now vs Wait

```
ADOPT NOW:
  ✓ Passkeys: production-ready; major platform support; phishing-resistant
    Add passkey registration as option alongside password
    Don't force: still need fallback for legacy devices
  
  ✓ CAEP for enterprise customers: ForgeRock AM supports; close the revocation gap
    Implement CAEP receiver endpoint
    Integrate with ForgeRock AM CAEP provider configuration

PILOT / SELECTIVE:
  ~ Decentralised Identity: adopt for KYC use case in regulatory sandbox
    India: DigiLocker VC is available now (Aadhaar-linked)
    Start: accept DigiLocker credentials for KYC (reduce KYC ops cost)
    Don't: bet entire identity architecture on DID standard (still evolving)
  
  ~ ITDR: implement as analytics on top of existing audit logs
    Start: build anomaly detection on ForgeRock AM audit log events (CS49 techniques)
    Don't: buy expensive dedicated ITDR platform yet; build with existing tooling first

PLAN / PREPARE:
  ◻ Post-quantum cryptography: inventory now; don't migrate yet
    Create: cryptographic algorithm inventory
    Implement: crypto agility layer (abstract all crypto calls behind interface)
    Timeline: first migration in 2026-2028 for most sensitive systems

IGNORE FOR NOW:
  ✗ Blockchain-based identity: no clear advantage over existing PKI for payment platforms
  ✗ Zero-knowledge proofs: interesting but complex; wait for practical tooling
```

---

## Integration with Existing Architecture

```
How emerging tech maps to what you've built:

Passkeys (CS44 MFA → extend):
  Add passkey registration flow (WebAuthn API)
  ForgeRock AM: WebAuthn node in authentication tree
  Replaces: password + TOTP for user-facing app
  Keeps: recovery codes, SAML for enterprise (CS42)

CAEP (CS46 Session Management → extend):
  Add CAEP receiver endpoint
  On signal: triggers CS46 session revocation
  ForgeRock AM: configure as CAEP signal provider + receiver
  Closes: the JWT TTL revocation gap

DID/VC (CS45 Identity Lifecycle → complement):
  Accept VC as alternative to internal KYC for onboarding
  DigiLocker VC: Indian citizen identity verified by UIDAI
  Platform: verify signature; create account; no need to store KYC data
  Keeps: internal user DB; just adds verification source

ITDR (CS10 Audit Log + CS22 Fraud Detection → combine):
  Audit log (CS10) → feed to anomaly detection (CS22 techniques applied to auth events)
  ForgeRock AM authentication events → risk engine → step-up decisions
  This is ITDR built on existing architecture
```

---

## Observability
```
Emerging tech adoption metrics:
  passkey_adoption_rate                   (% users with passkeys registered)
  passkey_authentication_success_rate     (should be > 99%)
  caep_signals_received_total             (by signal type, by IdP source)
  caep_session_revocations_triggered      (effectiveness of real-time revocation)
  did_credential_verifications_total      (DID/VC adoption rate)
  post_quantum_crypto_readiness_score     (% of crypto calls using quantum-safe algorithms; track toward 100%)
```

---

## Architecture Review Checklist
```
□ Passkeys:
  → WebAuthn API implemented (same as Case 44 FIDO2)
  → Resident key required (passkey-specific; enables usernameless login)
  → Sync: handled by platform (don't build your own sync)
  → Fallback: keep password + backup codes for legacy device recovery

□ CAEP:
  → CAEP receiver endpoint: /.well-known/caep
  → Signal verification: HMAC or JWT-signed from trusted IdP
  → Revocation handler: calls CS46 session revocation on signal
  → ForgeRock AM: configured as CAEP provider for your downstream apps

□ DID/VC:
  → Trust registry: define which issuers' VCs you accept (e.g., UIDAI for DigiLocker)
  → Claim mapping: VC claims → platform user attributes
  → Privacy: don't store the VC itself; only store the derived claims you need
  → Expiry: VCs have expiry; require fresh VC on expiry

□ Post-quantum:
  → Inventory: list all cryptographic operations (TLS, JWT, card vault encryption)
  → Agility: wrap all crypto in interfaces (easy algorithm swap later)
  → Priority: card vault HSM keys and JWT signing first (most sensitive)

□ SCC LENS for emerging tech:
  Passkeys: no new STATE (same WebAuthn DB); same COORDINATION (challenge-response)
  CAEP: new coordination channel (IdP → RP push); STATE change (session revoked)
  DID: new STATE (VC verification cache); new COORDINATION (DID resolver)
  
□ IAM EXPERTISE TIE-IN (your field's frontier):
  All 5 emerging topics are directly in your IAM domain:
  - Passkeys: evolution of FIDO2 (CS44); you understand the protocol
  - CAEP: ForgeRock AM feature; you likely know its configuration
  - DID: OIDC/SAML for self-sovereign identity; same protocol concepts
  - ITDR: IAM analytics; extends your identity event knowledge
  - PQ crypto: future of the PKI you currently manage
  
  Staff interview on "future of IAM": this case study is your answer
  Differentiation: most candidates know OAUTH/SAML; few know CAEP/passkeys/DID
```

---

*End of Volume 2 Expert+ Track — Case Studies 41–50 Complete*

```
COMPLETE HANDBOOK STRUCTURE (50 Case Studies):
  CS01–10: Foundational (URL shortener, Cache, Rate limiter, CDN, API Gateway, ...)
  CS11–20: Intermediate (Notification, News feed, Chat, Slack, Collab editor, ...)
  CS21–30: Advanced (Payment gateway, Fraud, UPI, Ledger, Multi-tenant, ZT, ...)
  CS31–40: Expert (Service mesh, K8s, CI/CD, Secrets, Idempotency, Sharding, ...)
  CS41–50: IAM Expert+ (OAuth2, SAML SSO, Fine-grained authz, MFA, JML, Sessions,
           Compliance, Attack vectors, AI/ML in payments, Emerging: Passkeys/CAEP/DID)

CS41–50 is uniquely positioned: your ForgeRock/IAM background makes this your
strongest area relative to other Staff Engineer candidates. These are the cases
where you have genuine expert insight, not just textbook knowledge.
```
