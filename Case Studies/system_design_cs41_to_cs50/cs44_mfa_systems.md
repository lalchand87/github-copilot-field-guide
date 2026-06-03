# Case Study 44 — Multi-Factor Authentication (MFA) Systems

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: TOTP mechanics, FIDO2/Passkeys, Risk-based MFA, Recovery flows.

---

## Business Context

MFA is the single most effective control against credential stuffing and account takeover. For a payment platform, MFA is both a security requirement and a UX challenge: too much friction → users abandon checkout; too little → account takeover → financial fraud. Risk-based MFA solves this by asking for second factors only when the risk is high.

**Business goals:**
- Reduce account takeover by 99%+ (MFA blocks stolen password attacks)
- Minimal friction for low-risk operations (browsing payment history)
- Step-up authentication for high-risk operations (large transfers)
- FIDO2/Passkeys: phishing-resistant; no OTP codes to intercept
- Recovery: MFA device lost → secure account recovery without creating a backdoor

---

## Recognition Framework

### MFA Factor Types — Security vs Usability

```
Factor 1 (Something you know): Password, PIN, security question
  Security: weak (phishable, reusable, guessable)
  Usability: high (everyone knows passwords)
  For payments: minimum baseline; not sufficient alone

Factor 2 (Something you have):
  SMS OTP: 6-digit code via SMS
    Security: moderate (SIM swap attacks; SS7 attacks; SMS interception)
    Usability: high (everyone has a phone)
    For payments: good enough for low-risk; not for high-risk
    RBI mandate: most UPI and card transactions require OTP

  TOTP (Time-based OTP): Google Authenticator, Authy
    Security: good (not SMS-based; no SIM swap risk)
    Usability: moderate (requires app; 30-second window)
    For payments: good for merchant portal login; not for consumer checkout
    Phishable: OTP can be phished in real-time (attacker relays OTP to real site)

  FIDO2 / Passkeys (WebAuthn):
    Security: excellent (phishing-resistant; cryptographic; biometric-bound)
    Usability: high (Face ID, fingerprint; no codes to type)
    For payments: best option for consumer (passkeys) and enterprise (security keys)
    NOT phishable: origin-bound (attacker's site gets a different credential)

Factor 3 (Something you are): Biometrics
  Fingerprint, face recognition
  For payments: device-side biometric (unlocks FIDO2 credential; biometric never leaves device)

Risk-based (adaptive): no second factor asked UNLESS risk signals trigger it
  Low risk: login from known device + known location → password only
  Medium risk: new device → TOTP or SMS OTP
  High risk: impossible travel, new country, suspicious amount → always step-up
```

---

## Problem Statement

MFA system for a payment platform. TOTP for merchant portal. SMS OTP for consumer transactions (RBI mandate). FIDO2/Passkeys for high-security users. Risk-based step-up for payment authorisation.

---

## TOTP (Time-Based OTP) Implementation

### How TOTP Works
```
Setup:
  1. Server generates random 20-byte secret: totp_secret = random.bytes(20)
  2. Server stores totp_secret (encrypted) linked to user
  3. Server presents QR code: otpauth://totp/Platform:alice@email.com?secret=BASE32(totp_secret)
  4. User scans with Authenticator app → app stores the secret

Code generation (same algorithm on server and app):
  HOTP_counter = floor(current_unix_time / 30)  # changes every 30 seconds
  HMAC_value  = HMAC-SHA1(totp_secret, HOTP_counter)
  offset      = HMAC_value[19] & 0x0f
  truncated   = HMAC_value[offset:offset+4] & 0x7fffffff
  OTP         = truncated % 1_000_000  # 6 digits
  
Validation:
  Server computes OTP for: current_counter - 1, current_counter, current_counter + 1
  (± 1 window allows for clock skew of up to 30 seconds)
  If user's code matches any of the 3: valid
  
  Anti-replay: each valid code can only be used ONCE (store used codes; TTL 90 sec)
```

```java
@Service
public class TotpService {

    // Enrollment
    public TotpEnrollmentResponse enroll(String userId) {
        byte[] secret = SecureRandom.getInstanceStrong().generateSeed(20);
        String secretBase32 = Base32.encode(secret);

        // Store encrypted secret
        userRepo.storeTotpSecret(userId, encrypt(secret));

        String otpAuthUri = String.format(
            "otpauth://totp/%s:%s?secret=%s&issuer=%s&algorithm=SHA1&digits=6&period=30",
            URLEncoder.encode("Payment Platform", UTF_8),
            URLEncoder.encode(getUserEmail(userId), UTF_8),
            secretBase32,
            "platform.com"
        );

        return TotpEnrollmentResponse.of(secretBase32, generateQrCode(otpAuthUri));
    }

    // Verification
    public boolean verify(String userId, String code) {
        byte[] secret = decrypt(userRepo.getTotpSecret(userId));
        long counter = Instant.now().getEpochSecond() / 30;

        // Check ±1 window for clock skew
        for (long t = counter - 1; t <= counter + 1; t++) {
            String expected = computeHotp(secret, t);
            if (MessageDigest.isEqual(code.getBytes(), expected.getBytes())) {
                // Anti-replay: mark this code as used
                String usedKey = "totp:used:" + userId + ":" + t;
                if (!redis.setIfAbsent(usedKey, "1", Duration.ofSeconds(90))) {
                    return false;  // already used (replay attack)
                }
                return true;
            }
        }
        return false;
    }
}
```

---

## FIDO2 / WebAuthn / Passkeys

### How FIDO2 Works
```
Registration (one-time; device to server):

1. Server: generates challenge (random 32 bytes)
   registerOptions = {
     challenge: random_bytes,
     rp: { id: "platform.com", name: "Payment Platform" },
     user: { id: user_id, name: "alice", displayName: "Alice" },
     pubKeyCredParams: [{ type: "public-key", alg: -7 }]  // ES256
   }

2. Authenticator (device):
   Creates key pair: (privateKey, publicKey)
   privateKey: stored in device's secure enclave (TPM / Secure Element)
   Binds to: rpId (platform.com) + user gesture (biometric/PIN)
   Signs: challenge + rpId with privateKey
   Returns to server: { credentialId, publicKey, attestation }

3. Server stores: { user_id, credentialId, publicKey }
   (publicKey: safe to store; used for future verification only)

Authentication (per login):

1. Server: generates new challenge
   authOptions = { challenge: new_random_bytes, rpId: "platform.com" }

2. Authenticator: user provides biometric/PIN
   Device: signs challenge + rpId + counter with privateKey
   Returns: { credentialId, signature, counter }

3. Server verifies:
   a. Signature: valid against stored publicKey? YES
   b. Origin: from platform.com (not evil.com)?  YES (phishing-resistant!)
   c. Counter: greater than stored counter?       YES (cloned device detection)
   → Authentication successful

WHY PHISHING-RESISTANT:
  Origin is cryptographically bound to the credential
  On evil-platform.com: authenticator signs rpId="evil-platform.com"
  Server: expected rpId="platform.com" → mismatch → verification fails
  Attacker gets a valid signature for evil-platform.com but not for platform.com
  → Phished credentials are useless
```

```java
@RestController
@RequestMapping("/api/v1/passkeys")
public class PasskeyController {

    @PostMapping("/register/options")
    public PublicKeyCredentialCreationOptions registerOptions(@AuthUser String userId) {
        byte[] challenge = secureRandom.generateSeed(32);
        redis.set("fido2:challenge:" + userId, challenge, Duration.ofMinutes(5));

        return PublicKeyCredentialCreationOptions.builder()
            .rp(new RelyingPartyIdentity("platform.com", "Payment Platform"))
            .user(getUserIdentity(userId))
            .challenge(challenge)
            .pubKeyCredParams(List.of(new PublicKeyCredentialParameters(Algorithm.ES256)))
            .authenticatorSelection(
                AuthenticatorSelectionCriteria.builder()
                    .residentKey(ResidentKeyRequirement.REQUIRED)  // passkey (discoverable)
                    .userVerification(UserVerificationRequirement.REQUIRED)  // biometric/PIN
                    .build()
            )
            .timeout(60_000L)
            .build();
    }

    @PostMapping("/register/verify")
    public void verifyRegistration(@AuthUser String userId,
                                   @RequestBody PublicKeyCredential credential) {
        byte[] challenge = redis.get("fido2:challenge:" + userId);
        RegistrationResult result = webAuthnServer.finishRegistration(credential, challenge);
        passkeyRepo.save(userId, result.getCredentialId(), result.getPublicKeyCose());
        redis.delete("fido2:challenge:" + userId);
    }
}
```

---

## Risk-Based (Adaptive) MFA

```
Goal: minimal friction for normal operations; step-up for suspicious ones

Risk Signal Categories:
  Device signals:
    new_device: first time this device_fingerprint seen for this user
    device_compromised: device flagged by MDM or threat intel
  
  Behavioral signals:
    impossible_travel: previous login in Mumbai 2 min ago; now London
    unusual_login_time: 3am login for user who always logs in 9am-6pm
    new_ip_country: India user now logging in from Eastern Europe
  
  Transaction signals:
    large_payment: amount > user's 30-day average × 5
    new_payee: first time sending to this merchant
    rapid_payments: 5 payments in 10 minutes (unusual for this user)

Risk Score → MFA Decision:
  Risk 0–30 (LOW):    No MFA required (known device, normal behavior, small amount)
  Risk 31–60 (MEDIUM): TOTP or SMS OTP required
  Risk 61–80 (HIGH):   FIDO2 (phishing-resistant) required; or step-up with biometric
  Risk 81–100 (CRITICAL): Block and require full re-authentication + manual review

Implementation:
  PaymentRiskEngine.score(user, transaction, device, context) → risk_score
  ForgeRock AM: adaptive authentication tree evaluates risk score
    → RISK_LOW: proceed with existing session
    → RISK_MEDIUM: invoke TOTP/SMS node
    → RISK_HIGH: invoke WebAuthn node
    → RISK_CRITICAL: block; notify on-call fraud team
```

---

## SMS OTP (RBI Mandated for Payments)

```
RBI mandate: 2FA required for all card-not-present transactions > ₹2,000
OTP via SMS to registered mobile number

Implementation:
  1. Payment API: amount > ₹2,000 → trigger OTP flow
  2. OTP Service: generate 6-digit OTP; store in Redis (TTL 10 minutes)
     OTP = random 6-digit number (NOT TOTP; no shared secret)
  3. Notification Service (Case 11): send SMS via Twilio/Sinch
  4. User: enters OTP in app/browser
  5. OTP Service: validate:
     a. OTP matches stored value (constant-time comparison; prevent timing attacks)
     b. Not expired (TTL check)
     c. Attempt count < 3 (prevent brute force: 3-attempt limit; lockout if exceeded)
     d. Not already used (mark as USED after first successful verification)

Security considerations:
  OTP never stored in plaintext: hash with HMAC(otp, user_phone_hash) before storage
  Not: store_otp("123456") → attacker reads Redis → knows OTP
  Yes: store_otp(HMAC("123456", user_phone_hash)) → cannot reverse without phone number
  
  SIM swap: OTP SMS to SIM-swapped number → attacker receives OTP
  Mitigation: monitor for SIM swap events (Twilio SIM swap API); delay OTP if recent swap

OTP delivery SLA:
  SMS delivery: < 10 seconds (Twilio SLA)
  If OTP not received: resend after 30 seconds; max 3 resends
  Resend rate limiting: max 3 OTPs per phone number per 10 minutes
```

---

## MFA Recovery (The Hardest Problem)

```
Scenario: Alice loses her phone (lost TOTP app, lost passkey device)
  How does Alice get back into her account without MFA?
  
  Bad recovery: "email a reset link" → attacker compromises email → bypasses MFA entirely
  Better recovery: backup codes + identity verification

Recovery options (in decreasing preference):

Option 1: Backup Codes (recommended)
  At MFA enrollment: generate 8 backup codes (random 8-character alphanumeric)
  Show ONCE to user; tell them to save securely
  Store: HASH of each code (not plaintext)
  On recovery: user enters backup code → validated against hash → one-time use
  After use: backup code invalidated; generate new set

Option 2: Trusted Recovery Contact
  User designates a trusted person; that person must authenticate to vouch
  Complex to implement; good for enterprise

Option 3: In-Person / Video KYC Verification
  User contacts support with government ID
  Support verifies identity; resets MFA
  Last resort; high operational cost; high security

Option 4: Account Recovery Grace Period
  New device: "you have 24 hours to verify via backup code or trusted device before account locked"
  During grace: limited access (view-only; no transactions)
  After 24h without verification: account locked; manual recovery

Implementation (backup codes):
  On enrollment:
    codes = [random_alphanumeric(8) for _ in range(8)]
    code_hashes = [bcrypt(code) for code in codes]
    store(user_id, code_hashes)
    present(codes)  # shown once; user must save

  On recovery use:
    for stored_hash in user.backup_code_hashes:
        if bcrypt_verify(entered_code, stored_hash):
            invalidate(stored_hash)
            issue_session()
            return  # success; one code consumed
    fail("Invalid backup code")
```

---

## Observability
```
MFA metrics:
  mfa_enrolment_rate by {method}              (TOTP, FIDO2, SMS)
  mfa_challenge_success_rate by {method}      (failed MFA attempts; alert on spike)
  mfa_challenge_failure_rate by {user_tier}   (high failure → UX issue or attack)
  sms_otp_delivery_latency_p99                (target < 5s; alert if > 10s)
  fido2_challenge_latency_p99                 (target < 2s including device interaction)
  backup_code_usage_rate                      (spike → MFA device loss incident)
  risk_score_distribution                     (calibration check)
  step_up_mfa_triggered_rate                  (how often adaptive MFA fires)
```

---

## Architecture Review Checklist
```
□ Factor selection by risk level:
  SMS OTP: regulatory compliance (RBI); moderate security; high availability
  TOTP: good for merchant portal; no SMS dependency; slightly more secure
  FIDO2: best security; phishing-resistant; use for high-risk users and operations
  Risk-based: apply friction proportional to risk; not uniform MFA on everything

□ Anti-replay on every factor:
  TOTP: mark used code in Redis (90s TTL)
  SMS OTP: mark as USED after first valid use; TTL 10 min
  FIDO2: counter increment; reject if counter didn't increase (cloned device)
  Backup codes: one-time use; hash-stored; invalidate on use

□ Recovery is not a backdoor:
  Email reset alone: NOT acceptable (email compromise bypasses MFA)
  Backup codes + strong validation: YES
  KYC verification for lost device: YES (high friction; appropriate for lost authenticator)

□ Rate limiting on MFA:
  OTP attempts: max 3 before lockout (prevent brute force on 6-digit OTP)
  OTP resends: max 3 per 10 min per phone number
  Backup code attempts: max 5 before account lock

□ SCC LENS:
  STATE: TOTP secrets (encrypted in DB), FIDO2 public keys (DB), OTP hashes (Redis TTL)
  COORDINATION: challenge-response: server issues challenge; user responds; server verifies
  CONCENTRATION: SMS OTP depends on Twilio; failover to secondary SMS provider (Case 11)
```

---

*Next: Case Study 45 — Identity Lifecycle Management (JML, SCIM, IGA)*
