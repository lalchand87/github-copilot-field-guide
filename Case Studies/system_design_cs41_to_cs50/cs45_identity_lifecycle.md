# Case Study 45 — Identity Lifecycle Management (JML, SCIM, IGA)

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Joiner-Mover-Leaver flows, SCIM provisioning, Access certification, SoD.

---

## Business Context

Identity lifecycle management asks: how do the right people get the right access, and how does that access disappear when they no longer need it? For a payment platform, a former employee with lingering admin access is a direct regulatory violation. An under-provisioned new joiner cannot do their job. An employee who moved from support to engineering still having support tool access is a SOX violation.

JML (Joiner-Mover-Leaver) is the framework: provision on joining, update on moving, deprovision on leaving. IGA (Identity Governance and Administration) is the tooling: SailPoint, Saviynt, or ForgeRock Identity Governance.

**Business goals:**
- New employee: fully provisioned in < 1 business day
- Employee transfer: old access removed + new access granted within 1 hour
- Termination: ALL access revoked within 15 minutes of HR notification
- Quarterly access certification: managers certify who needs what access
- Separation of Duties: nobody can both approve and execute a payment

---

## Recognition Framework

### The Identity Lifecycle

```
JOINER (new hire / new merchant):
  Trigger: HR system creates employee record; merchant onboards
  Actions:
    Create identity in directory (LDAP/AD/OpenDJ)
    Provision access based on role/department template
    Send welcome email with credentials / enrollment link
    Notify manager to complete access request for custom tools
  
  Automated: standard role-based access (every engineer gets GitHub, Jira, Confluence)
  Manual: privileged access (production DB access, PCI-scoped systems)

MOVER (transfer, promotion, role change):
  Trigger: HR updates employee record (department change)
  Actions:
    ADD: new role's access entitlements
    REMOVE: old role's access entitlements (critical; often missed manually)
    Temporary: grace period for some tools (2 weeks to transition ongoing work)
  
  Hardest problem: remove the old access (people remember to add; forget to remove)
  Result of missed removals: "access creep" → SOX audit findings → regulatory risk

LEAVER (resignation, termination, retirement):
  Trigger: HR marks employee as terminated (or last day notification)
  Actions:
    Immediate: disable account, revoke all sessions and tokens
    Immediate: revoke MFA devices and API keys
    30-day: archive email, export data, delete or anonymise
    90-day: close all tickets, transfer ownership of resources
  
  Termination SLA: HR to IT: 15 minutes for immediate threat; 2 hours for standard
  Contractual: employee NDA enforcement; access termination documented
```

---

## Problem Statement

Identity lifecycle management for a payment platform. SCIM-based provisioning from HR system. JML automation via ForgeRock OpenIDM. Access certification via IGA. Separation of Duties enforcement.

---

## SCIM-Based Provisioning Architecture

```
HR System (Workday / SAP SuccessFactors)
  │
  ├── On new hire: SCIM POST /Users
  ├── On transfer: SCIM PATCH /Users/{id}
  └── On termination: SCIM PATCH /Users/{id} { active: false }
        │
        ▼
ForgeRock OpenIDM (Identity Governance Layer)
  │
  ├── Receives SCIM events from HR
  ├── Applies provisioning policies:
  │     New hire → role template → entitlement mapping
  │     Transfer → diff old role vs new role → add/remove entitlements
  │     Termination → immediate disable all
  ├── Orchestrates provisioning to downstream systems:
  │     OpenDJ (LDAP directory)
  │     GitHub (developer access)
  │     Jira / Confluence
  │     Payment Admin Portal
  │     PCI-scoped systems (requires manual approval)
  └── Audit: every provisioning action logged

Downstream Systems (SCIM endpoints or connectors):
  OpenDJ: LDAP connector → create/modify/disable user accounts
  GitHub: GitHub SCIM connector → add/remove org membership + team membership
  Jira: REST API connector → add/remove group membership
  Payment Portal: SCIM endpoint → create/deactivate merchant admin accounts
  PCI Systems: manual approval workflow (not auto-provisioned due to PCI-DSS scope)
```

### SCIM Provisioning Code (OpenIDM scripted connector)
```groovy
// OpenIDM provisioning script: new hire → payment platform access

def provisionNewHire(user) {
    def role = user.jobTitle  // "Payment Engineer" / "Support Agent" / "Finance Analyst"
    
    // Role to entitlement mapping
    def entitlements = ROLE_ENTITLEMENTS[role] ?: DEFAULT_ENTITLEMENTS
    
    // 1. Create OpenDJ user
    openDJ.createUser([
        uid: user.employeeId,
        cn: "${user.firstName} ${user.lastName}",
        mail: user.email,
        userPassword: generateTempPassword(),
        isMemberOf: entitlements.ldapGroups
    ])
    
    // 2. Provision payment portal access
    if ("PAYMENT_PLATFORM" in entitlements.systems) {
        paymentPortal.createUser([
            userId: user.employeeId,
            email: user.email,
            role: entitlements.paymentRole,
            tenantId: "internal"  // internal employees get internal tenant
        ])
    }
    
    // 3. Request PCI access (manual approval required)
    if (entitlements.requiresPCI) {
        workflowEngine.createApprovalTask([
            type: "PCI_ACCESS_REQUEST",
            requestedFor: user.employeeId,
            approver: user.managerId,
            justification: "Role: ${role} requires PCI access",
            deadline: tomorrow()
        ])
    }
    
    // 4. Send welcome email
    notificationService.sendWelcome(user.email, generateOnboardingLink(user.employeeId))
    
    auditLog.record("PROVISIONING", user.employeeId, entitlements, "AUTO_PROVISIONED")
}
```

---

## Access Certification (IGA Core Feature)

```
Quarterly access certification:
  Every 3 months: every manager reviews their team's access
  Manager certifies: "yes Alice still needs production DB access" OR "no, remove it"
  Uncertified access: auto-revoked after 2 weeks (no response = remove)
  
  Why quarterly?
    Employees accumulate access over time (project → project)
    Access creep: each project adds access; projects end; access stays
    Certification forces periodic review; removes accumulated excess

Access certification workflow:
  1. OpenIDM generates certification campaign: "Q1 2024 Access Review"
  2. For each manager: list of team members + their current entitlements
  3. Manager reviews in governance portal:
      ✓ Certify (keep access)   ✗ Revoke (remove access)   ? Review later
  4. On Revoke: OpenIDM deprovisions within 24h
  5. On no response after 14 days: auto-revoke (same as Revoke)
  6. Exception: some accesses are certified automatically (everyone needs Jira)

Risk-based certification:
  High-risk entitlements (PCI scope, production DB): certify every quarter
  Medium-risk (payment portal view): certify every 6 months
  Low-risk (standard tools: Jira, Slack): annual or auto-certify for standard roles
  
  This reduces certification fatigue: managers aren't asked about Slack every 3 months
```

---

## Separation of Duties (SoD)

```
SoD principle: no single person should be able to initiate AND approve a financial transaction
  Example violation: Bob creates a payment AND approves that same payment
  Result: potential for fraud without detection

SoD policies for payment platform:

Policy 1: Payment Creator ≠ Payment Approver
  CANNOT HAVE: role PAYMENT_CREATOR AND role PAYMENT_APPROVER simultaneously
  Maker-checker: payment created by Alice → must be approved by Bob (different person)
  
Policy 2: Ledger Writer ≠ Ledger Auditor
  CANNOT HAVE: access to modify ledger entries AND access to audit ledger entries
  
Policy 3: PCI Cardholder Data Access ≠ Cardholder Data Auditor
  CANNOT HAVE: access to encrypted card data AND access to audit log for card data

SoD enforcement:
  Preventive (hard block): role assignment system refuses to grant conflicting roles
    OpenIDM: before granting PAYMENT_APPROVER to Bob → check if Bob has PAYMENT_CREATOR
    If conflict detected → BLOCK assignment; create exception workflow
  
  Detective (audit): periodic report of users with conflicting role combinations
    SailPoint Identity IQ or Saviynt: "SoD violation report"
    SOX audit: these reports are evidence of SoD control effectiveness

SoD exception workflow:
  Sometimes business need requires temporary exception (e.g., sole employee of small team)
  Exception must: require senior manager approval, have an expiry date, be audited
  System: grants conflicting roles with override reason + expiry; auto-revokes at expiry
  Never: permanent SoD exception without recurring approval
```

---

## Privileged Access Management (PAM) Integration

```
Some access is so sensitive it must be:
  Checked out (not permanently assigned)
  Time-limited (auto-revoked after session)
  Recorded (video/keystroke recording)

Privileged access examples:
  Production DB superuser: needed for emergency query; not needed permanently
  PCI card vault credentials: needed only by vault-service; never by humans
  Root access to payment servers: break-glass only

Just-In-Time (JIT) Access model:
  Alice needs production DB access for incident investigation
  1. Alice requests: "I need prod DB access for incident INC-123"
  2. JIT system: manager auto-approves (incident context) OR requires approval
  3. JIT system: provisions time-limited credential (e.g., DB user with 1-hour TTL)
  4. Alice uses the credential; session recorded
  5. 1-hour TTL expires: credential auto-revoked; no manual step required
  6. Session recording: retained for 1 year (compliance)
  
  Tools: CyberArk PAM, HashiCorp Vault (dynamic secrets from Case 34)
  Your context: Vault dynamic DB credentials (Case 34) IS JIT PAM for database access

Vaulted passwords (PAM for shared accounts):
  Shared service account passwords → stored in CyberArk/Vault
  Rotation: CyberArk rotates password every 24h automatically
  Checkout: Alice checks out password → gets it for 1 hour → auto-revoked
  No permanent knowledge of service account passwords by any human
```

---

## Observability + Compliance
```
Identity lifecycle metrics:
  provisioning_time_p99 by {event_type}        (joiner: target < 1 day; leaver: < 15 min)
  access_certification_completion_rate         (target > 95%; alert if < 80%)
  uncertified_access_auto_revoked_count        (no response = removed)
  sod_violation_detected_total                 (preventive block count)
  sod_violation_exception_active_count         (current active exceptions; audit target)
  privileged_access_checkout_duration_avg      (JIT access patterns)
  orphaned_accounts_detected                  (accounts with no HR record → investigate)

Compliance reports (SOX, PCI-DSS):
  "User access review Q1 2024": certification campaign results
  "SoD violations and exceptions": quarterly report for audit
  "Privileged access usage": who checked out what, when, for how long
  "Terminated employee access": was access revoked within SLA?
  ForgeRock Identity Governance: generates all of these natively
```

---

## Architecture Review Checklist
```
□ JML completeness:
  J: new hire → provisioned → can work on day 1
  M: transfer → old access removed; new access added (within 1 hour)
  L: termination → ALL access revoked within 15 minutes
  "M" (Mover) is hardest; test specifically for access removal

□ SCIM as integration standard:
  HR system pushes SCIM events; OpenIDM consumes
  Downstream systems: SCIM endpoints or connectors
  This decouples HR system from every downstream system

□ Access certification:
  Quarterly (at minimum) for high-risk entitlements
  Auto-revoke uncertified access (no response = remove, not keep)
  Separate campaigns by risk level (avoid manager fatigue)

□ SoD enforcement:
  Preventive: block conflicting role assignments at assignment time
  Detective: quarterly SoD violation report
  Exceptions: time-limited, senior-approved, audited

□ Orphaned accounts:
  Account exists in system but no HR record → investigate immediately
  Automated detection: reconcile directory vs HR system weekly

□ SCC LENS:
  STATE: HR system (authoritative for employee state), OpenIDM (provisioning orchestrator),
         OpenDJ (directory state), downstream systems (provisioned access state)
  COORDINATION: SCIM events from HR → OpenIDM → multi-system provisioning
  CONCENTRATION: OpenIDM is the provisioning hub → HA; event queue for resilience

□ IAM EXPERTISE TIE-IN:
  This IS identity governance — your direct domain
  ForgeRock OpenIDM = the provisioning engine you likely work with
  SailPoint/Saviynt = the IGA platforms that sit on top
  JML + SCIM + SoD + PAM = the full IAM operational lifecycle
  Staff interview: "how do you manage identity lifecycle at enterprise scale?" = this
```

---

*Next: Case Study 46 — Session Management and Token Revocation*
