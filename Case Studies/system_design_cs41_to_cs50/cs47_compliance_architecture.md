# Case Study 47 — Compliance Architecture (PCI-DSS, GDPR, RBI)

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: PCI scope reduction, GDPR data subject rights, RBI mandates.

---

## Business Context

Compliance is not a checkbox. It is a set of technical architectural decisions that determine whether your system can be audited, whether your data can be forgotten, and whether your payment processing meets regulatory standards. Getting compliance architecture wrong means either a regulatory fine (GDPR: 4% of global revenue; PCI: $100K/month) or an unworkable system (deleting a user's data from 30 microservices is impossible without a unified approach).

**Business goals:**
- PCI-DSS: process card payments without storing card data (scope reduction)
- GDPR: user can request deletion; their data disappears from all systems within 30 days
- RBI: mandatory transaction limits, dispute resolution, 2FA requirements met
- SOX: financial records immutable; auditable for 7 years

---

## Recognition Framework

### The Four Compliance Domains

```
PCI-DSS (Payment Card Industry Data Security Standard):
  Scope: any system that stores, processes, or transmits cardholder data
  Key requirements: no plaintext PANs; encrypt in transit; access logs; quarterly penetration test
  Architecture impact: HUGE — tokenisation vault reduces scope dramatically
  Violation cost: $100K-$500K/month; suspension from card networks
  Your context: Case 21 (payment gateway) covers tokenisation; this case covers governance

GDPR (General Data Protection Regulation — EU):
  Scope: any EU citizen's data, regardless of where company is
  Key requirements: purpose limitation; data minimisation; right to erasure; data portability
  Architecture impact: every service that stores user data must support deletion
  Violation cost: 4% of global annual turnover or €20M (whichever is higher)
  
RBI (Reserve Bank of India) Regulations:
  Scope: any payment service operating in India
  Key requirements: data localisation; mandatory 2FA; dispute resolution; transaction limits
  Architecture impact: all payment data stored in India; UPI mandates from Case 23
  
SOX (Sarbanes-Oxley):
  Scope: public companies; financial reporting integrity
  Key requirements: immutable financial records; SoD controls; audit trail
  Architecture impact: event sourcing (Case 28); double-entry ledger (Case 24); SoD (Case 45)
```

---

## PCI-DSS Architecture

### Scope Reduction Strategy (Already in Case 21; Governance Layer Here)
```
PCI Scope = any component that stores, processes, OR transmits cardholder data (CHD)
  Primary Account Number (PAN): card number
  Full Magnetic Stripe Data
  CVV/CVC
  PIN data

Scope reduction (architectural):
  IN SCOPE (heavily controlled):
    Card Vault Service: receives, encrypts, tokenises card data
    Payment API (briefly): passes encrypted data to card network; never stores
    Network segment: vault ↔ card network only
  
  OUT OF SCOPE (normal engineering practices):
    Everything that only sees card TOKENS (not actual card numbers)
    payment-service, fraud-service, ledger-service, notification-service
    All databases that store only tok_abc123 (not 4111-1111-1111-1111)

PCI control requirements (for in-scope systems):
  Req 1-2: Firewall + no vendor defaults → network segmentation
  Req 3:   Do not store sensitive auth data → CVV never stored (even encrypted)
  Req 4:   Encrypt CHD in transit → TLS 1.2+ everywhere
  Req 5-6: Anti-malware, secure development → SAST/DAST in CI/CD (Case 33)
  Req 7:   Restrict access by need-to-know → RBAC + OPA (Case 43)
  Req 8:   Unique IDs + MFA for all access → FIDO2 for vault access (Case 44)
  Req 9:   Physical access restriction → data centre controls (out of scope for cloud)
  Req 10:  Log all CHD access → every vault operation logged (Case 10)
  Req 11:  Penetration testing quarterly → automated DAST + quarterly external pentest
  Req 12:  Security policies + annual risk assessment → documented, reviewed annually

PCI-DSS 4.0 (2024) new requirements:
  Targeted risk analysis: customised security controls based on specific risk
  Automated technical controls: security tests must be automated (not just manual review)
  Multi-factor authentication: MFA now required for all accounts (not just admin)
```

---

## GDPR Architecture

### Data Subject Rights Implementation

```
GDPR Article 17: Right to Erasure ("Right to be Forgotten")
  User requests: DELETE all my personal data
  Deadline: 30 days to complete

  Challenge: data is in 20+ services
  Solution: event-driven deletion workflow

Deletion workflow:
  1. User submits deletion request
  2. Create deletion_request record (track progress; required by GDPR for proof)
  3. Publish event: UserDeletionRequested { user_id, requested_at }
  4. Each service has a DeletionHandler subscriber:
     user-service:       DELETE user record; replace email with hash
     payment-service:    anonymise payment records (replace user_id with anonymised_id)
     notification-svc:   delete phone, email, push tokens
     fraud-service:      delete fraud features and case notes
     analytics:          replace user_id with pseudonym in event store
     audit-log:          CANNOT delete (audit logs are SOX-required immutable records)
  5. Each handler: sends DeletionCompleted event when done
  6. Orchestrator: tracks completion across all services (timeout: 29 days)
  7. Confirm deletion: user receives confirmation email; request marked complete

NOT deletion (pseudonymisation):
  Financial records (payments, ledger entries): legally cannot delete (7-year retention)
  Instead: replace real user_id with pseudonymous_id throughout financial records
  The payment happened; the user who made it is de-identified
  Regulators accept pseudonymisation for financial record retention

Data that CANNOT be deleted (overrides GDPR erasure):
  Active legal hold → criminal investigation
  Financial records → 7-year statutory retention
  Audit logs for compliance → SOX retention
  GDPR allows: compliance with legal obligation overrides right to erasure
```

```java
@Service
public class GdprDeletionOrchestrator {

    @EventHandler("UserDeletionRequested")
    public void handleDeletionRequest(UserDeletionEvent event) {
        DeletionRequest request = deletionRepo.findById(event.getUserId());
        List<String> requiredServices = serviceRegistry.getDeletionHandlers();
        
        // Publish deletion events to all relevant services
        for (String service : requiredServices) {
            kafkaProducer.send("gdpr.deletion." + service, 
                              new ServiceDeletionCommand(event.getUserId(), event.getRequestId()));
        }
        
        // Start monitoring: if any service doesn't complete in 25 days → escalate
        scheduler.schedule(() -> checkDeletionProgress(event.getRequestId()), 
                          Duration.ofDays(25));
    }
    
    @EventHandler("ServiceDeletionCompleted")
    public void handleServiceCompletion(ServiceDeletionCompleted event) {
        deletionRepo.markServiceComplete(event.getRequestId(), event.getServiceName());
        
        if (deletionRepo.allServicesComplete(event.getRequestId())) {
            notifyUser(event.getUserId(), "Your data has been deleted.");
            deletionRepo.markFullyComplete(event.getRequestId());
        }
    }
}
```

### GDPR Data Minimisation and Retention
```
Data minimisation: only collect what's needed for stated purpose
  Payment platform: collect email, phone (for OTP), payment method
  NOT collect: browsing history (unless stated), social graph, location (unless stated)
  
  Technical control: privacy-by-design checklist in PR review
    "Does this new field have a stated purpose? Is it in the privacy policy?"

Data retention policy:
  Payment transaction records: 7 years (financial regulation)
  User profile data: account lifetime + 30 days after deletion request
  Session logs: 90 days
  Fraud signals: 2 years (needed for fraud pattern analysis)
  Audit logs: 7 years (SOX + RBI)
  
  Automated enforcement:
    Each table has data_retention_days column in schema registry
    Retention job: runs nightly; deletes records past retention period
    Exception: records under legal hold are excluded from deletion

GDPR data inventory (required for compliance):
  Document every data store: what data, where, retention period, legal basis
  Data map: which services have which user attributes
  Privacy impact assessment: for new features that process personal data
```

---

## RBI Regulatory Requirements

```
RBI key mandates for payment platforms (India):

1. Data Localisation (RBI circular on payment data):
   All payment transaction data must be stored in India
   Foreign companies processing Indian payments: store in India + optional foreign copy
   Technical: India-region deployment (Case 30); data never leaves ap-south-1 for Indian payments
   Audit: annual data localisation certification

2. Transaction Limits (UPI, IMPS, NEFT):
   UPI peer-to-peer: ₹1 lakh per transaction
   UPI merchant payments: ₹5 lakh per transaction (some categories unlimited)
   Enforcement: transaction_amount > limit → reject before submitting to NPCI
   
3. Mandatory 2FA:
   All card transactions > ₹2,000: OTP required (AFA - Additional Factor Authentication)
   UPI: UPI PIN is the 2FA (handled by NPCI)
   Technical: Case 44 (MFA) covers implementation; RBI defines which transactions require it

4. Dispute Resolution (grievance redressal):
   Customer complaint → resolution within 30 days
   Failed transactions → auto-refund within T+1
   Recurring charge disputes → immediate suspension of recurring mandate
   Technical: dispute tracking system; automated refund for technical failures

5. Settlement:
   Payments settled on T+0 (same day) or T+1 (next day) per NPCI schedule
   Float management: TSP must maintain sufficient float with NPCI
   Reconciliation: daily NPCI file must match ledger (Case 23)

RBI audit trail:
  All payment events logged with timestamps, amounts, bank codes
  Accessible to RBI inspectors on request (read-only regulatory access)
  Retention: minimum 5 years (RBI requirement); our platform: 7 years (conservative)
```

---

## Unified Compliance Data Flow

```
Every payment generates compliance-relevant data at multiple layers:

Payment Created (pay-abc-123):
  ↓ PCI compliance layer:
    Card token stored (not PAN) → vault handles PAN (in-scope)
    Payment record: card_token, amount, merchant, timestamp → in DB (out of scope)
  
  ↓ RBI compliance layer:
    Amount limit check: ₹5 lakh < limit → allowed
    2FA required: > ₹2,000 → OTP verified
    Data localisation: stored in ap-south-1 (India) ✓
    Settlement: submitted to NPCI → T+0 settlement
  
  ↓ GDPR compliance layer:
    User ID stored → will be pseudonymised if user exercises deletion right
    Purpose of processing: payment fulfillment (legal basis: contract)
    Retention: 7 years (financial records)
  
  ↓ SOX compliance layer:
    Immutable ledger entries: DEBIT user, CREDIT merchant, CREDIT platform_fee
    Audit log: who processed this payment, when, with what role (Case 10)
    SoD: payment created by merchant; approved by payment gateway (not same entity)
  
  ↓ Audit log (Case 10):
    Every layer's compliance action → append-only event log
    Accessible to: RBI inspectors, PCI QSA, internal audit team, GDPR DPO
```

---

## Compliance Testing and Monitoring

```
Automated compliance checks (run in CI pipeline):
  PCI: no hardcoded card numbers in test data (regex scan on test fixtures)
  PCI: no PAN appearing in application logs (log scanning in staging)
  GDPR: new DB columns flagged if no retention policy documented
  SOX: no DELETE statements on immutable tables (ledger_entries, audit_log)

Penetration testing (quarterly, per PCI):
  Scope: in-scope PCI systems (card vault, payment API, network segmentation)
  External QSA (Qualified Security Assessor): conducts test; issues report
  Critical findings: must remediate before next quarter
  
Data Subject Request testing (monthly):
  Create test user; trigger deletion request; verify all services complete within 29 days
  Automated test: check_deletion_completeness(test_user_id, after=30days) → assert all clear

Compliance metrics:
  gdpr_deletion_completion_rate          (target 100%; alert if < 99%)
  gdpr_deletion_time_p99_days            (target < 28 days; alert if > 25 days)
  rbi_transaction_limit_rejections       (monitor for unusual patterns)
  pci_scope_boundary_violations_total    (PANs appearing in out-of-scope logs: should be 0)
  audit_log_integrity_checks_passed      (Merkle tree / hash chain verification)
```

---

## Architecture Review Checklist
```
□ PCI:
  → Card data: tokenised; only vault touches PAN
  → Scope boundary: documented; enforced by network segmentation
  → Access logs: every vault access logged with user identity and reason
  → Quarterly pentest: scheduled; findings tracked to remediation

□ GDPR:
  → Right to erasure: event-driven deletion workflow across all services
  → Pseudonymisation for financial records (cannot delete but can anonymise)
  → Data retention: automated; each table has documented retention period
  → Legal basis for each processing activity: documented in data inventory

□ RBI:
  → Data localisation: India-region only for Indian payment data (verified by audit)
  → Transaction limits: enforced at payment API; before NPCI submission
  → 2FA: OTP or UPI PIN for transactions above threshold
  → Dispute resolution: SLA tracking; automated refund for technical failures

□ SOX:
  → Immutable ledger: no UPDATE/DELETE on payment_events, ledger_entries
  → Audit trail: every financial state change logged with actor identity
  → SoD: payment creator ≠ approver (Case 45 controls)
  → 7-year retention: archived to S3 Glacier after hot period

□ SCC LENS:
  STATE: compliance DB (deletion requests, retention policies), audit log (immutable)
  COORDINATION: GDPR deletion orchestrator (multi-service fan-out), RBI limits (gateway check)
  CONCENTRATION: compliance checks run at API gateway (single enforcement point)
```

---

*Next: Case Study 48 — IAM Attack Vectors and Threat Modelling*
