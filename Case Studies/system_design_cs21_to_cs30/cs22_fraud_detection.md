# Case Study 22 — Fraud Detection System

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Real-time vs Batch signals, Rule Engine vs ML, False Positive cost.

---

## Business Context

A fraud detection system decides in milliseconds whether to allow or block a payment. A missed fraud is a financial loss. A false positive (blocking a legitimate transaction) is a lost sale and an angry customer. The cost of each error is asymmetric and context-dependent: blocking a ₹5M wire transfer at a 0.1% FPR is acceptable; blocking 1 in 1,000 ₹50 UPI payments is devastating for conversion.

**Business goals driving architecture:**
- Fraud decision in < 100ms (payment authorisation SLA is 2 seconds; fraud is one step)
- Catch > 95% of fraud (recall); false positive rate < 0.5% (precision)
- Explainable decisions (regulatory requirement; customer disputes; model audit)
- Adaptive: fraud patterns change; model must update within hours
- Diverse signals: transaction history + device fingerprint + network signals + behavioural

---

## Recognition Framework

### Signals from the Problem
```
- Real-time decision required (< 100ms → no batch ML inference)
- High volume (50K TPS) → inference must be in-memory, not DB query
- Features change per request (new transaction context) → online feature computation
- Historical patterns needed (is this user's 50th payment or first?) → feature store
- Rule-based for known patterns (velocity, geo-impossible) → fast and explainable
- ML for subtle patterns (device + behaviour + network graph) → high recall
- False positives are expensive (conversion loss) → calibrate threshold carefully
- Model drift: fraudsters adapt; model degrades → continuous retraining
```

### Rule Engine vs ML — Architecture Decision

```
Rule Engine (Rule-Based System):
  How: IF velocity > 5 payments in 10 minutes THEN flag
  Pros: fast (microseconds), explainable, easy to add known fraud patterns
  Cons: cannot detect novel fraud; requires manual rule creation; brittle
  Use: known fraud patterns; regulatory required rules (velocity, geo)

ML Model (Gradient Boosted Trees / Neural Network):
  How: 200 features → model → fraud_probability_score 0.0–1.0
  Pros: detects novel patterns; adapts to new fraud types; high recall
  Cons: black-box (explainability required for disputes); slower (10–50ms); model drift
  Use: complement rules; catch subtle fraud patterns; anomaly detection

Decision architecture (CHOSEN): two-stage pipeline
  Stage 1: Rule Engine (< 1ms)
    Hard block rules: impossible geo travel, known fraud device, blocklisted card
    If BLOCK: return immediately (no ML inference needed)
  
  Stage 2: ML Scoring (< 50ms for non-blocked)
    Feature extraction → GBM model → fraud_score
    Threshold: fraud_score > 0.8 → BLOCK; 0.5–0.8 → REVIEW; < 0.5 → ALLOW
    3D Secure challenge for REVIEW (push MFA to cardholder)
  
  Benefits: rules handle 20% of fraud instantly (free); ML handles the rest
  Explainability: SHAP values on GBM features → "flagged due to unusual location + new device"
```

---

## Problem Statement

Real-time fraud detection for payments. < 100ms decision. Two-stage: rule engine + ML model. Feature store for historical signals. Adaptive retraining on new fraud patterns.

---

## Functional Requirements
- Real-time fraud score per payment attempt (< 100ms)
- Rule engine: velocity limits, geo-impossible travel, blocklists
- ML model: GBM fraud score based on 200+ features
- Feature store: real-time features (last 5 min) + historical features (last 90 days)
- Case management: REVIEW queue for human analysts
- Feedback loop: analyst decisions feed back into model retraining
- Explainability: reason codes for every BLOCK/REVIEW decision

## Non-Functional Requirements
- Latency: **< 100ms p99** (end-to-end fraud decision)
- Throughput: **50,000 RPS** (matches payment gateway)
- Precision: **> 99.5%** (false positive rate < 0.5%)
- Recall: **> 95%** (catch 95% of actual fraud)
- Model staleness: **< 4 hours** (retrain on new fraud patterns)

---

## Capacity Estimation
```
Feature store size:
  100M users × 200 features × 4 bytes = 80 GB for user features
  Real-time features: 50K TPS × 5-min window = 15M recent events in memory
  At 200 bytes/event: 3 GB for real-time feature window

ML model:
  GBM with 500 trees, depth 6: ~50 MB (fits in memory on serving nodes)
  Inference time: ~5ms on single core (fast for GBM)

Rule engine:
  ~100 rules; each evaluation: O(features) lookups in feature store
  Redis lookup: ~1ms per feature; 10 features = 10ms → batch fetch in pipeline
  Total rule engine: < 5ms
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Two-Stage Pipeline | Rules (cheap, fast) before ML (expensive, slow) |
| Feature Store (Redis + PostgreSQL) | Real-time + historical features; < 2ms feature retrieval |
| GBM Model (LightGBM) | High recall for tabular fraud features; fast inference; SHAP explainability |
| Risk Score Threshold | ALLOW / REVIEW / BLOCK based on score; not binary |
| 3D Secure Challenge | REVIEW → step-up auth; legitimate users complete; fraudsters often don't |
| Model Retraining Pipeline | Daily on confirmed fraud labels; alerts on model drift |
| Feedback Loop | Analyst decisions → labels → model training data |

---

## Data Model

```
Feature Store (Redis — real-time, TTL-based):
  user_features:{user_id}           → Hash: recent_txn_count_5min, total_5min_amount,
                                       distinct_merchants_5min, last_txn_country, etc.
  device_features:{device_id}       → Hash: first_seen, txn_count_24h, fraud_history
  card_features:{card_token}        → Hash: first_txn_at, country_history, velocity_1h
  network_features:{ip_hash}        → Hash: txn_count_1h, country, proxy_flag, vpn_flag
  (All TTLs: 10 minutes for velocity; 24h for device features; 90 days for history)

Feature Store (PostgreSQL — historical, persistent):
  user_fraud_history: past fraud incidents, dispute rate, chargeback history
  merchant_risk_profile: chargeback rate, dispute rate, category risk
  card_velocity_90d: rolling 90-day transaction patterns per card token

Fraud Cases (for analyst review queue):
  CREATE TABLE fraud_cases (
    case_id        UUID PRIMARY KEY,
    payment_id     UUID,
    fraud_score    DECIMAL(5,4),  -- 0.0000 to 1.0000
    risk_level     VARCHAR(16),   -- ALLOW / REVIEW / BLOCK
    decision       VARCHAR(16),   -- ALLOW / BLOCK (final human decision; null if pending)
    reason_codes   TEXT[],        -- ['HIGH_VELOCITY', 'NEW_DEVICE', 'GEO_MISMATCH']
    shap_values    JSONB,         -- feature importance for this specific decision
    analyst_id     VARCHAR(64),   -- who reviewed (if REVIEW)
    reviewed_at    TIMESTAMP,
    is_confirmed_fraud BOOLEAN,   -- label for model retraining
    created_at     TIMESTAMP
  );
```

---

## API Design

```
POST /api/v1/fraud/evaluate (called by payment gateway synchronously)
Body: {
  "payment_id": "pay-uuid",
  "amount": 50000,
  "currency": "INR",
  "user_id": "u-12345",
  "card_token": "tok-abc",
  "merchant_id": "merch-xyz",
  "merchant_category": "ELECTRONICS",
  "device_id": "dev-abc",
  "ip_address_hash": "sha256-of-ip",
  "user_agent": "...",
  "billing_country": "IN",
  "shipping_country": "IN",
  "is_new_card": false,
  "is_new_device": true
}
Response 200 (< 100ms): {
  "payment_id": "pay-uuid",
  "decision": "REVIEW",               // ALLOW / REVIEW / BLOCK
  "fraud_score": 0.67,
  "risk_level": "MEDIUM",
  "reason_codes": ["NEW_DEVICE", "UNUSUAL_AMOUNT"],
  "recommended_action": "3DS_CHALLENGE",
  "evaluation_time_ms": 45
}

POST /api/v1/fraud/feedback  (analyst submits decision on REVIEW case)
Body: { "case_id": "...", "decision": "ALLOW", "analyst_notes": "Verified with customer" }

GET /api/v1/fraud/cases?status=PENDING_REVIEW&risk_level=HIGH
  → Returns analyst review queue (sorted by fraud_score DESC)
```

---

## High-Level Architecture

```
Payment Gateway → Fraud Evaluation API (synchronous; in payment authorisation path)
  │
  ├── Stage 1: Rule Engine (< 5ms)
  │     Load features from Redis (HMGET pipeline; 1 round-trip for all features)
  │     Evaluate rules in priority order:
  │       1. Hard blocks (BLOCK immediately; no ML needed):
  │          - Blocklisted card/device/IP
  │          - Impossible geographic travel (prev txn Mumbai 2 min ago; now London)
  │          - Velocity limit: > 10 payments in 5 minutes
  │          - Amount > 5× historical avg (outlier detection)
  │       2. High-risk signals (flag for ML evaluation):
  │          - New device (is_new_device=true)
  │          - First transaction > ₹10,000
  │          - Unusual merchant category for user
  │     If any hard block → return BLOCK immediately; log to fraud_cases
  │     If no hard block → proceed to Stage 2
  │
  ├── Stage 2: ML Scoring (< 50ms)
  │     Extract 200 features (from Stage 1 feature load + additional computation)
  │     Load model (GBM): in-memory on each serving node (50 MB)
  │     Run inference: LightGBM predict_proba → fraud_score
  │     Compute SHAP values for top 5 features (explainability)
  │     
  │     Decision threshold:
  │       fraud_score > 0.80 → BLOCK
  │       fraud_score 0.50–0.80 → REVIEW (trigger 3D Secure challenge)
  │       fraud_score < 0.50 → ALLOW
  │
  ├── Feature Update (async; after decision returned):
  │     Update feature store: increment velocity counters, update last_txn fields
  │     This is async (fire-and-forget): feature update does NOT delay the response
  │
  └── Write to fraud_cases table (async; for audit + retraining):
        INSERT fraud_cases with decision, score, reason_codes, SHAP values

Model Retraining Pipeline (offline; triggered every 4 hours):
  1. Collect labels: confirmed fraud (chargebacks) + analyst decisions
  2. Feature extraction for labelled transactions
  3. Train new LightGBM model
  4. Validate: precision, recall, AUC vs previous model
  5. If improved: deploy to model serving nodes (hot-reload; no restart)
  6. If degraded: alert + keep previous model

Feature Engineering (real-time; runs in fraud evaluation request):
  Velocity features: COUNT(txn) last 5min, 1h, 24h per user/card/device/IP
  Pattern features: is_new_device, geo_match, time_of_day, day_of_week
  Historical features: avg_txn_amount_30d, merchant_category_diversity, past_disputes
  Network features: ip_is_proxy, ip_is_vpn, ip_country != billing_country
```

---

## Detailed Components

### Feature Engineering Pipeline
```python
class FeatureExtractor:
    def extract(self, payment: PaymentRequest, redis: Redis) -> Dict[str, float]:
        # Batch Redis fetch (1 round-trip for all real-time features)
        pipe = redis.pipeline()
        pipe.hgetall(f"user_features:{payment.user_id}")
        pipe.hgetall(f"device_features:{payment.device_id}")
        pipe.hgetall(f"card_features:{payment.card_token}")
        pipe.hgetall(f"network_features:{payment.ip_hash}")
        user_f, device_f, card_f, network_f = pipe.execute()

        return {
            # Velocity features
            "user_txn_count_5min":        int(user_f.get("txn_count_5min", 0)),
            "user_amount_5min":           float(user_f.get("total_amount_5min", 0)),
            "card_velocity_1h":           int(card_f.get("txn_count_1h", 0)),

            # Deviation features
            "amount_vs_user_avg":         payment.amount / max(float(user_f.get("avg_amount_30d", payment.amount)), 1),
            "is_new_device":              int(device_f.get("first_seen_today", 0)),
            "is_new_merchant_category":   self._is_new_category(user_f, payment.merchant_category),

            # Geographic features
            "geo_country_match":          int(network_f.get("country", "IN") == user_f.get("home_country", "IN")),
            "geo_impossible_travel":      self._check_impossible_travel(user_f, payment),
            "ip_is_vpn":                  int(network_f.get("is_vpn", 0)),

            # Historical features
            "user_dispute_rate_90d":      float(user_f.get("dispute_rate_90d", 0)),
            "merchant_chargeback_rate":   self._get_merchant_risk(payment.merchant_id),
            "card_fraud_history":         int(card_f.get("ever_fraud", 0)),

            # Context features
            "amount":                     payment.amount / 100,  # normalised to rupees
            "is_weekend":                 int(datetime.now().weekday() >= 5),
            "hour_of_day":                datetime.now().hour,
            # ... 185 more features
        }
```

### GBM Model Serving
```python
class FraudModelServer:
    def __init__(self):
        self.model = lgb.Booster(model_file="fraud_model_v42.lgb")
        self.feature_names = self.model.feature_name()
        # Model is loaded once; hot-reloaded on new version without restart

    def score(self, features: Dict) -> Tuple[float, Dict]:
        X = np.array([[features.get(f, 0) for f in self.feature_names]])
        fraud_prob = self.model.predict(X)[0]

        # SHAP values for explainability
        explainer = shap.TreeExplainer(self.model)
        shap_vals = explainer.shap_values(X)[0]
        top_features = sorted(
            zip(self.feature_names, shap_vals),
            key=lambda x: abs(x[1]),
            reverse=True
        )[:5]

        return fraud_prob, {name: float(val) for name, val in top_features}

    def hot_reload(self, new_model_path: str):
        """Zero-downtime model update."""
        new_model = lgb.Booster(model_file=new_model_path)
        self.model = new_model  # atomic swap (GIL protects in CPython)
        logger.info(f"Model reloaded: {new_model_path}")
```

### 3D Secure (3DS) Challenge Flow
```
REVIEW decision triggers 3D Secure challenge:

1. Payment Gateway returns "3DS_CHALLENGE_REQUIRED" to merchant
2. Merchant redirects customer to card network's 3DS page
3. Customer receives OTP on registered phone
4. Customer enters OTP on 3DS page
5. Card network validates OTP → returns 3DS authentication result
6. Merchant retries payment with 3DS authentication token
7. Fraud service re-evaluates: is_3ds_authenticated=true → score drops significantly
8. Payment proceeds if score now < 0.5

Why 3DS reduces fraud:
  Fraudsters have stolen card details but NOT the cardholder's phone
  3DS OTP requires control of the phone → most fraudsters cannot complete
  3DS shifts liability: if 3DS completed but still fraud → liability shifts to card issuer
  Legitimate customers: slightly annoyed but complete the flow
  False positive handling: legitimate customer completes 3DS → payment succeeds
```

---

## Scaling Strategy
```
Phase 1 (0 → 1K TPS):
  Simple rule engine only (velocity, blocklist)
  Rules in PostgreSQL; checked on every transaction
  No ML; pattern-based only

Phase 2 (1K → 10K TPS):
  Redis feature store (velocity counters)
  Basic GBM model (50 features)
  Model retrained weekly

Phase 3 (10K → 50K TPS):
  Feature store sharding (Redis Cluster by user_id)
  GBM model replicated on all serving nodes (50 MB each)
  Model retrained every 4 hours on new fraud labels
  Case management system for analyst review

Phase 4 (50K+ TPS, graph fraud):
  Graph neural network for account linkage fraud
    (shared devices/IPs across multiple accounts → fraud ring detection)
  Real-time streaming feature computation (Flink)
  Shadow mode: new model evaluated alongside production model
```

---

## Reliability Strategy
- **Fail-open on timeout**: if fraud evaluation > 150ms → ALLOW with flag (payment processed; fraud flagged for post-hoc review)
- **Model serving redundancy**: 3 serving nodes; round-robin; any node can serve any request
- **Feature store HA**: Redis Cluster; feature retrieval continues if 1 node fails
- **Fallback rule-only mode**: if ML model unavailable → rule engine only; higher false negatives but system operates

---

## Security Considerations (IAM Tie-in)
- **Feature data sensitivity**: user transaction history in feature store is PII; encrypted at rest; access via mTLS + service identity
- **Model explainability for disputes**: customer challenges a blocked payment → fraud reason codes provided in dispute response; SHAP values are internal only
- **Analyst access control**: fraud analysts can view case details including masked card data; cannot see plaintext PAN; access logged
- **Model poisoning**: training data integrity — confirmed fraud labels from chargebacks (ground truth); analyst labels weighted by analyst accuracy score
- **Rate limiting on feedback API**: prevent analyst from accidentally labelling thousands of cases programmatically

---

## Observability
```
Real-time fraud metrics:
  fraud_score_distribution_p50_p90_p99    (model calibration; alert if shifts)
  fraud_decision_distribution             (ALLOW/REVIEW/BLOCK %; alert on unusual BLOCK spike)
  false_positive_rate                     (BLOCK decisions later confirmed legitimate)
  true_positive_rate                      (BLOCK decisions confirmed as fraud)
  rule_engine_block_rate                  (% blocked at Stage 1)
  ml_review_rate                          (% sent to 3DS)
  evaluation_latency_p99                  (target < 100ms)

Model health:
  model_precision_24h                     (chargebacks vs BLOCK decisions; alert if < 99%)
  model_recall_24h                        (fraud caught vs total fraud confirmed)
  model_auc_rolling                       (alert if drops > 2% from baseline)
  feature_staleness_seconds               (age of features in feature store; alert if > 60s)

Business metrics:
  fraud_loss_inr_per_day                  (confirmed fraud that passed through)
  chargeback_rate_by_merchant             (identify high-risk merchants)
  analyst_queue_depth                     (REVIEW cases pending; alert if > 500)
  analyst_review_accuracy                 (% of analyst decisions later confirmed correct)
```

---

## Chaos Testing
```
Experiment 1: ML Model Serving Node Failure
  Inject:    Kill one of 3 ML serving nodes mid-traffic
  Expected:  Load balancer routes to remaining 2 nodes
  Expected:  Latency increases slightly (33% more load per remaining node)
  Expected:  No fraud decisions degraded; model still available
  Expected:  Alert fires: "fraud-ml-node-1 is unhealthy"
  Red flag:  All decisions fall back to rules-only (ML not HA)

Experiment 2: Redis Feature Store Slowdown
  Inject:    Inject 50ms latency into Redis operations
  Expected:  Feature retrieval now 50ms → total latency ~100ms (at limit)
  Expected:  Alert fires: feature_retrieval_latency > 20ms
  Expected:  If latency exceeds budget: fail-open (ALLOW with flag)
  Red flag:  Payment gateway times out waiting for fraud decision

Experiment 3: Sudden Fraud Pattern Shift
  Inject:    Simulate new fraud pattern: bulk of fraud now comes from a new merchant category
  Expected:  Rule engine: no rule for this pattern; passes through
  Expected:  ML model: initially misses (trained on old patterns) → chargebacks appear
  Expected:  Chargeback spike detected by monitoring → alert fires
  Expected:  Retraining triggered with new chargeback labels → model updated within 4 hours
  Red flag:  No alert on chargeback spike; model not retrained for days

Experiment 4: Impossible Geo Travel Rule
  Inject:    User transacts in Mumbai at 14:00; next transaction attempts from London at 14:05
  Expected:  Rule engine: time_between_txns = 5 min; distance_km = 7,200; impossible
  Expected:  BLOCK returned in < 5ms (rule fires; no ML needed)
  Expected:  Reason code: "GEO_IMPOSSIBLE_TRAVEL"
  Red flag:  Transaction allowed (geo rule not evaluating correctly)
```

---

## Monthly Cost Estimate
```
Scale: 50K TPS, 432M evaluations/day

Fraud API serving (16 nodes):
  16 × c5.2xlarge ($280/mo)                        = $4,480/mo

Redis Feature Store (Cluster):
  6 × cache.r6g.2xlarge ($500/mo) [80 GB cluster]  = $3,000/mo

ML Model Training (GPU; 4× daily):
  4 × g4dn.xlarge ($0.526/hr × 2hr × 4 × 30)      = $506/mo

PostgreSQL (fraud cases + historical features):
  db.r6g.2xlarge ($500/mo)                         = $500/mo

Case Management UI + Analyst tooling:
  2 × t3.large ($60/mo)                            = $120/mo

Monitoring + Kafka (fraud events):
                                                    = $800/mo
─────────────────────────────────────────────────────────────────
Total: ~$9,406/month

Context: fraud detection prevents far more than it costs.
At 50K TPS, even 0.1% fraud rate = 50 fraud attempts/sec.
If avg fraud amount ₹1,000, catching 95% saves ₹47.5M/sec = ₹4.1B/day.
Infrastructure cost ($9,406/mo) vs prevented fraud ($125B/mo) → clear ROI.
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → fraud_cases: every evaluation, decision, and label (authoritative).
  → feature_store (Redis): real-time operational features (ephemeral; rebuilt from events).
  → PostgreSQL historical features: durable; rebuilt from payment history if lost.
  → ML model: versioned artifact in S3; serving nodes load from S3.

□ Consistency model?
  → Fraud decision: synchronous, consistent (in-request path; blocking).
  → Feature updates: async (after decision returned); up to 100ms stale.
  → Model updates: hot-reload on all serving nodes within 5 minutes of new version.

□ Failure mode?
  → ML node down → route to other nodes; alert.
  → Feature store slow → timeout → fail-open (ALLOW with flag; post-hoc review).
  → Model training failure → keep previous model; alert; human review of new fraud patterns.

□ False positive vs false negative tradeoff:
  → Threshold tuning: lower threshold → more REVIEW → higher recall but lower precision.
  → For high-value transactions: lower threshold (₹50K+: prefer false positive over miss).
  → For micro-transactions (₹10 UPI): higher threshold (conversion matters more than fraud).
  → Per-merchant-category threshold tuning (electronics = lower; grocery = higher).

□ SCC LENS:
  STATE: Redis (real-time features), PostgreSQL (historical/cases), S3 (model artifacts)
  COORDINATION: two-stage pipeline (rules gate ML); 3DS challenge coordinates with card network
  CONCENTRATION: Redis feature store (every transaction reads it) → cluster + HA mandatory
```

---

*Next: Case Study 23 — UPI Payment System*
