# Case Study 33 — CI/CD Pipeline and GitOps

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: GitOps model, Progressive delivery, Secret management in pipelines.

---

## Business Context

A payment platform deploying 30+ microservices needs a deployment system that is: fast (< 10 minutes from commit to production), safe (no bad deployments reach customers), and auditable (who deployed what, when, with which change). GitOps makes Git the single source of truth for what should be running in production — not Slack messages, not manual kubectl commands.

**Business goals driving architecture:**
- Code merged → production in < 15 minutes (developer velocity)
- Any bad deployment auto-rolled back within 5 minutes
- Git history = complete audit of every production change
- Secrets never in Git or CI logs (PCI-DSS requirement)
- Multiple environments: dev → staging → production (gate at each stage)

---

## Recognition Framework

### GitOps Model

```
Traditional deployment (imperative):
  Developer runs: kubectl apply -f deployment.yaml
  Or CI pipeline runs: kubectl set image deployment/payment-service ...
  Problem:
    No audit trail of who ran what (kubectl history is not a log)
    Manual process → human error → different environments diverge
    "What's actually running?" → hard to answer

GitOps (declarative):
  Git repo contains: desired state of all K8s manifests
  ArgoCD/Flux: continuously syncs cluster to match Git
  "Desired state" → push to Git
  "Actual state" → what's running in K8s
  ArgoCD: ensures actual = desired at all times
  
  On commit to Git:
    ArgoCD detects change → applies to cluster
    If cluster diverges (manual kubectl applied): ArgoCD reverts (self-healing)
  
  Audit: who made what change? → git log (complete, immutable, signed)
  Rollback: git revert → ArgoCD deploys previous state
  
  Security: developers never have kubectl access to production
             CI never has kubectl access → only ArgoCD does
```

---

## Pipeline Architecture

```
Code commit → GitHub (or GitLab)
  │
  ├── CI Pipeline (GitHub Actions / Jenkins):
  │     1. Unit tests + integration tests
  │     2. Security scan: SAST (Semgrep), SCA (dependencies), container scan (Trivy)
  │     3. Build Docker image: tag with git SHA
  │     4. Push image to registry (ECR / GCR / Harbor)
  │     5. Update K8s manifest: sed -i "s|image:.*|image: payment-service:$GIT_SHA|" k8s/deployment.yaml
  │     6. Commit updated manifest to gitops-repo (separate repo from source)
  │
  └── CD Pipeline (ArgoCD watching gitops-repo):
        1. Detects manifest change in gitops-repo
        2. Applies to target cluster (auto or manual approval gate)
        3. Monitors rollout: waits for Deployment rollout to complete
        4. Health check: verifies new pods pass readiness/liveness
        5. If health check fails: automatic rollback (ArgoCD reverts to previous)
        6. Sends Slack/PagerDuty notification

Two-repo pattern (best practice):
  app-repo:     source code, Dockerfile, tests (developer-owned)
  gitops-repo:  K8s manifests (platform team controlled; stricter access)
  
  Separation: breaking a build doesn't affect production (different repos)
              gitops-repo is protected: direct push blocked; PR required
```

---

## CI Pipeline Detail

### GitHub Actions Workflow
```yaml
name: Payment Service CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  IMAGE: payment-service
  REGISTRY: 123456789.dkr.ecr.ap-south-1.amazonaws.com

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Run unit tests
      run: ./gradlew test

    - name: Run integration tests
      run: ./gradlew integrationTest
      # Spins up PostgreSQL + Redis in Docker Compose for integration tests

    - name: SAST security scan
      uses: returntocorp/semgrep-action@v1
      with:
        config: p/java p/secrets p/owasp-top-ten

    - name: Dependency vulnerability scan
      run: ./gradlew dependencyCheckAnalyze
      # Fails if CVSS score > 7.0 in any dependency

  build-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    outputs:
      image_tag: ${{ steps.meta.outputs.version }}
    steps:
    - name: Build Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ env.REGISTRY }}/${{ env.IMAGE }}:${{ github.sha }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

    - name: Scan container image
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE }}:${{ github.sha }}
        severity: CRITICAL,HIGH
        exit-code: 1    # fail CI if critical/high CVE found in final image

  update-gitops:
    needs: build-push
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
      with:
        repository: org/gitops-repo
        token: ${{ secrets.GITOPS_DEPLOY_TOKEN }}

    - name: Update image tag in manifest
      run: |
        yq e -i '.spec.template.spec.containers[0].image = "${{ env.REGISTRY }}/${{ env.IMAGE }}:${{ github.sha }}"' \
          apps/payments/payment-service/deployment.yaml
        git config user.email "ci@platform.com"
        git commit -am "chore: deploy payment-service ${{ github.sha }}"
        git push
    # ArgoCD detects this commit → deploys to staging automatically
    # Production: requires manual approval in ArgoCD UI or Slack approval workflow
```

### Progressive Delivery (Argo Rollouts)
```yaml
# Argo Rollouts: canary deployment with automatic analysis
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-service
spec:
  replicas: 10
  strategy:
    canary:
      steps:
      - setWeight: 5         # Step 1: 5% canary traffic
      - pause: {duration: 5m}  # Hold for 5 minutes; monitor metrics
      - analysis:            # Step 2: automated analysis
          templates:
          - templateName: payment-success-rate
      - setWeight: 20        # Step 3: increase to 20%
      - pause: {duration: 10m}
      - setWeight: 50
      - pause: {duration: 5m}
      - setWeight: 100       # Promote to full production
---
# AnalysisTemplate: auto-rollback if error rate spikes
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: payment-success-rate
spec:
  metrics:
  - name: success-rate
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(payment_requests_total{status="success",version="canary"}[5m]))
          /
          sum(rate(payment_requests_total{version="canary"}[5m]))
    successCondition: result[0] >= 0.99   # 99% success rate required
    failureLimit: 3         # fail if metric fails 3 consecutive evaluations
    interval: 1m
  # If success-rate < 99% for 3 consecutive minutes: Rollout auto-aborts → v1 restored
```

---

## Secret Management (No Secrets in Git)
```yaml
# External Secrets Operator: pulls secrets from AWS Secrets Manager into K8s
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-service-secrets
  namespace: payments
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: payment-service-secret  # K8s Secret created/updated
  data:
  - secretKey: db-password
    remoteRef:
      key: production/payment-service/db-credentials
      property: password
  - secretKey: card-vault-key
    remoteRef:
      key: production/payment-service/vault-api-key

# What NEVER goes in Git:
#   - DB passwords
#   - API keys (NPCI, card network)
#   - PCI vault credentials
#   - Signing keys
#
# What CAN go in Git (gitops-repo):
#   - K8s manifest structure (deployment, service, ingress)
#   - Config values (max_retries=3, timeout_ms=5000)
#   - References to secrets (ExternalSecret resource names)
```

---

## Observability + Audit
```
Deployment audit trail:
  Every production change: PR in gitops-repo with code review
  PR merge = deployment approval (git log = complete audit)
  ArgoCD: deployment history with git SHA, deployer, timestamp
  ArgoCD sync status dashboard: what's deployed vs what's in Git

Deployment metrics:
  deployment_frequency                 (deploys per day; DORA metric; target > 1/day)
  change_failure_rate                  (% deploys that cause incident; target < 5%)
  mean_time_to_restore                 (after incident; target < 1 hour)
  lead_time_for_changes               (commit → production; target < 1 day)
  rollback_count                       (auto-rollbacks from Argo Rollouts)
```

---

## Chaos Testing
```
Experiment 1: Bad Deployment Auto-Rollback
  Inject:    Deploy version with intentional high error rate (mock 503 for 20% requests)
  Expected:  Argo Rollouts: canary at 5%; success-rate drops below 99%
  Expected:  Analysis fails 3 consecutive checks → rollout aborted
  Expected:  Stable version restored automatically; no manual intervention
  Expected:  Alert: "Rollout payment-service aborted; reverted to previous version"
  Red flag:  Bad version reaches 100% traffic before auto-rollback

Experiment 2: GitOps Self-Healing
  Inject:    Manually apply kubectl change to production (bypass GitOps)
  Expected:  ArgoCD detects: actual state ≠ desired state (OutOfSync)
  Expected:  After 3 minutes (default sync interval): ArgoCD reverts manual change
  Expected:  Alert: "Unmanaged change detected in payment-service deployment"
  Red flag:  Manual change persists (ArgoCD not enforcing desired state)

Experiment 3: Secret Rotation
  Inject:    Rotate DB password in AWS Secrets Manager
  Expected:  External Secrets Operator: detects change within 1h (refreshInterval)
  Expected:  K8s Secret updated; pods with environment variables need restart
  Expected:  Rolling restart triggered automatically (annotate Deployment with checksum)
  Red flag:  Services still using old password after rotation (secret not propagated)
```

---

## Architecture Review Checklist
```
□ GitOps invariants:
  → All production changes go through Git PR (no direct kubectl)
  → CI has push access to gitops-repo only (not cluster)
  → ArgoCD has cluster access only (syncs gitops-repo → cluster)
  → Rollback = git revert PR (< 2 minutes)

□ Secret hygiene:
  → No secrets in Git, CI logs, or Docker image layers
  → External Secrets Operator pulls from Secrets Manager
  → CI uses short-lived IAM roles (not long-lived keys)
  → Secret rotation automated; rotation triggers rolling restart

□ Progressive delivery:
  → New versions never go 0% → 100% immediately
  → Canary: 5% → analyze → 20% → analyze → promote
  → Analysis gates: error rate + latency must stay within bounds
  → Auto-rollback: no human required for most bad deploys

□ DORA metrics tracked:
  → Deployment frequency, Lead time, Change failure rate, MTTR
  → Alert if Change Failure Rate > 10% (deployment quality issue)

□ SCC LENS:
  STATE: gitops-repo (desired state), K8s etcd (actual state), ArgoCD DB (sync status)
  COORDINATION: ArgoCD continuous reconciliation (desired = actual at all times)
  CONCENTRATION: ArgoCD application controller → all cluster syncs through it → HA required
```

---

*Next: Case Study 34 — Secrets Management and PKI*
