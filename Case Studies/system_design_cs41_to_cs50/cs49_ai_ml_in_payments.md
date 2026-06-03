# Case Study 49 — AI/ML in Payments (Fraud, Anomaly, Forecasting)

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Real-time inference constraints, Model governance, Responsible AI in finance.

---

## Business Context

AI/ML transforms payment platform operations: fraud detection (Case 22 used ML), but ML also powers anomaly detection for system health, cash flow forecasting for merchants, customer churn prediction, and dynamic pricing of financial products. The unique challenge for payments ML: models make decisions with real financial consequences in < 100ms. A wrong fraud model can block legitimate transactions (lost revenue); a wrong anomaly model can miss a system failure (outage).

**Business goals:**
- Fraud detection: < 100ms inference; recall > 95%; FPR < 0.5%
- Anomaly detection: catch unusual merchant behaviour before it becomes a chargeback spike
- Merchant cash advance: predict merchant's future revenue to offer credit
- Responsible AI: decisions explainable to regulators; no discriminatory patterns

---

## Recognition Framework

### ML Domains in Payments

```
Real-time inference (in payment critical path):
  Fraud scoring: < 100ms; GBM or neural network; features from Redis feature store
  Risk scoring: at account creation; identity verification
  Routing optimisation: which card network/bank has best success rate right now?
  
  Constraint: inference must be fast; model must be in memory; features pre-computed

Near-real-time (within minutes):
  Anomaly detection: merchant suddenly has 10× normal chargeback rate → alert
  Device reputation: new device flagged by threat intelligence → check before auth
  AML (Anti-Money Laundering): transaction pattern analysis within 1 hour
  
  Constraint: Flink streaming aggregation; model can be slightly larger; minutes acceptable

Batch (daily/weekly):
  Merchant credit scoring: predict revenue for cash advance product
  Customer churn: which merchants likely to leave? (proactive outreach)
  Cohort analysis: payment behaviour clustering for product decisions
  
  Constraint: Spark on S3 data lake; can use large models; days acceptable
```

---

## Problem Statement

ML platform for payment intelligence. Real-time fraud scoring (extends Case 22). Near-real-time anomaly detection. Merchant revenue forecasting. Model governance and responsible AI.

---

## Real-Time ML Inference Infrastructure

### Feature Store (Central to Real-Time ML)
```
Feature store: the missing piece in most ML systems
  Problem: training uses historical features (from data warehouse)
            serving uses real-time features (from live transactions)
            Often different code paths → training-serving skew → model degradation
  
  Solution: online/offline feature store
    Offline store (Spark/Delta Lake): historical features for training
    Online store (Redis/DynamoDB): low-latency features for serving
    Same feature definitions: no skew
    
    Feature computation:
      Time-window features (last 5 min txn count): Flink streaming → Redis INCR
      Historical features (90-day avg amount): Spark batch → Redis HSET
      External features (device reputation, IP reputation): sync from threat intelligence API

Feature store schema (Redis Hash per entity):
  user_features:{user_id}: {
    txn_count_1min: 3,
    txn_count_5min: 7,
    txn_count_1h: 14,
    total_amount_5min: 15000,
    avg_amount_30d: 2500,
    distinct_merchants_24h: 5,
    last_txn_timestamp: 1712345600,
    last_txn_country: "IN",
    dispute_rate_90d: 0.001,
    preferred_categories: "FOOD|SHOPPING"
  }
  
  device_features:{device_id}: {
    first_seen_days: 30,
    fraud_incidents: 0,
    txn_count_24h: 3,
    is_emulator: false,
    os_version: "iOS 17.2"
  }
```

### Model Serving Architecture
```
Payment request arrives → Fraud API (Case 22 extended here)

Step 1: Feature retrieval (1 Redis round-trip; batch fetch all entities)
  HMGET user:{uid} [all feature keys]
  HMGET device:{did} [all feature keys]
  HMGET ip:{ip_hash} [all feature keys]
  Parallelised: all 3 Redis calls in pipeline → < 2ms total

Step 2: Feature engineering (in-memory; no I/O)
  Computed features:
    amount_vs_avg_30d = current_amount / user.avg_amount_30d
    velocity_spike = txn_count_5min / max(txn_count_5min_baseline, 1)
    time_since_last_txn = now - user.last_txn_timestamp
  < 0.5ms

Step 3: Model inference (in-process; loaded in memory)
  LightGBM predict_proba:
    Input: 200-feature vector
    Output: fraud probability 0.0-1.0
    Latency: 5-15ms (LightGBM is very fast on CPU)
  
  SHAP values for top 5 features: 10ms additional
  Total model inference: < 25ms

Step 4: Decision
  score > 0.80: BLOCK
  score 0.50-0.80: REVIEW (3DS challenge)
  score < 0.50: ALLOW
  
Total: < 30ms for fraud scoring in payment hot path
```

---

## Near-Real-Time Anomaly Detection

### Merchant Anomaly Detection (Streaming)
```
Use case: detect unusual merchant behaviour that precedes chargeback spikes
  Signal: merchant refund rate suddenly 5× normal → potential fraud ring
  Signal: merchant transaction velocity 10× normal at 3am → possible account takeover
  Signal: merchant chargebacks > 1% → Visa/Mastercard fines incoming

Flink streaming job: merchant_anomaly_detector
  Input: Kafka topic payment.events + fraud.decisions
  
  Per merchant, per 1-hour window:
    COUNT transactions
    SUM refunds
    SUM chargebacks
    AVG transaction amount
    COUNT distinct customer_ids
    COUNT transactions by country
  
  Anomaly triggers (rule-based first; then ML):
    chargeback_rate > 1%: → alert fraud team + account review
    refund_rate spike > 3× 30-day avg: → alert + investigation
    velocity spike > 5× 30-day avg at off-hours: → alert
    new_customer_concentration > 80% (mostly new customers): → potential first-time-buyer fraud ring
  
  ML anomaly score:
    Isolation Forest trained on 90-day merchant behaviour
    Anomaly score > threshold → alert (even if no specific rule fires)
    Handles: novel fraud patterns not covered by explicit rules

Output: MerchantAnomalyEvent → alert_service → fraud operations dashboard
```

---

## Merchant Revenue Forecasting (Cash Advance Product)

```
Product: "Merchant Cash Advance" — advance merchant ₹10 lakh; they repay from future sales
  Risk: merchant defaults → we lose the advance amount
  Model: predict next 3 months revenue → determine creditworthiness + advance amount

Features for revenue forecasting:
  Transaction history (12 months): monthly revenue, volume, growth trend
  Seasonality: Diwali uplift, year-end patterns
  Category: restaurant revenue more stable than fashion (seasonal)
  Merchant age: newer merchants have less predictable revenue
  External: macroeconomic indicators (optional; add complexity)

Model choice: LSTM (Long Short-Term Memory) for time-series
  Input: 12 months of features (daily granularity)
  Output: predicted next 90 days daily revenue
  
  Why LSTM (not ARIMA):
    Non-linear patterns (Diwali, new merchant growth curve)
    Multiple input features (not just univariate time series)
    Handles irregular patterns better than ARIMA
  
  Why not Transformer (LLM-style):
    Unnecessary complexity for univariate time-series
    LSTM still excellent for structured time-series prediction
    Less data needed to train well

Forecasting accuracy:
  MAPE (Mean Absolute Percentage Error): target < 15% for 30-day horizon
  Coverage: prediction interval contains actual revenue 80% of the time
  
Credit decision model (on top of forecasting):
  Input: forecasted revenue + merchant characteristics
  Output: recommended_advance = min(forecast_90d_revenue × 15%, max_advance_policy)
  Risk-adjusted: higher variance forecast → lower advance percentage
```

---

## AML (Anti-Money Laundering) Detection

```
AML: detect transactions that are part of money laundering schemes
  Typologies (patterns):
    Structuring: multiple transactions just below reporting threshold (₹9,900 each to avoid ₹10K reporting)
    Layering: rapid transfer between multiple accounts (obscure source)
    Integration: legitimate-looking large purchases after layering

Graph-based AML:
  Money moves through accounts; graph connects: account → payment → account
  Suspicious: high in-degree (many accounts sending to one) in short time
  Suspicious: accounts with high betweenness centrality (hub in transfer network)
  
  Tool: graph neural network (GNN) or simpler graph analytics
  Input: transaction graph per hour
  Output: suspicious_account_score per node
  
  Flink streaming + graph database (TigerGraph or Neo4j):
    Real-time: incremental graph updates as transactions arrive
    Batch: nightly full graph analysis for long-horizon patterns
  
  RBI mandate: report cash transactions > ₹10 lakh to Financial Intelligence Unit (FIU)
  FATF guidelines: financial action task force requirements for AML controls
```

---

## Model Governance and Responsible AI

```
Responsible AI for payments (regulatory and ethical):

1. Explainability:
   Every fraud BLOCK decision: must provide reason codes
   SHAP values: feature importance per decision
   Reason codes: human-readable (not raw SHAP values):
     "Flagged due to: unusual transaction amount (5× avg), new device, high-risk merchant"
   
   Regulatory requirement: RBI, GDPR ask for explainable automated decisions
   Customer right: "why was my payment blocked?" → must answer

2. Fairness:
   Model must not discriminate based on protected characteristics
   Sensitive attributes: gender, religion, caste, disability
   Test: measure false positive rate by demographic group
   Alert: FPR for group X is 3× higher than group Y → model may be biased
   
   For payment fraud: income level can correlate with fraud rate legitimately
   BUT: income proxy should not encode caste/religion discrimination
   Mitigation: fairness constraints in training; regular fairness audits

3. Model drift detection:
   PSI (Population Stability Index): measure if input feature distribution shifted
   PSI > 0.2: significant drift → retrain or investigate
   Causes: payment behaviour shifted (post-COVID); fraud pattern changed; feature pipeline bug
   
   Monitoring: daily PSI report for all model inputs
   Automatic retraining trigger: PSI > 0.25 OR AUC drops > 3%

4. Model version management:
   Every model version: unique ID, training date, training data period, feature list, performance metrics
   Champion vs challenger: new model runs in shadow mode (doesn't affect decisions) before promotion
   A/B test: 5% traffic to new model; compare fraud catch rate and FPR
   Rollback: if new model is worse → revert to champion in < 5 minutes
```

---

## Observability
```
ML-specific metrics:
  model_inference_latency_p99 by {model_name}     (target < 30ms for fraud)
  model_score_distribution                         (monitor for drift from training distribution)
  fraud_catch_rate_7d                              (recall; alert if drops > 2%)
  false_positive_rate_7d                           (FPR; alert if rises > 0.5%)
  feature_staleness_seconds by {feature_name}      (stale features = worse predictions)
  model_auc_rolling_7d                             (overall model quality; alert if drops)
  psi_daily by {feature_name}                      (data drift detection)
  
Retraining pipeline:
  training_job_duration_minutes
  training_data_volume_records
  validation_metrics_vs_champion                   (must be better to promote)
```

---

## Architecture Review Checklist
```
□ Training-serving consistency:
  Same feature definitions in offline (Spark) and online (Redis) stores
  No training-serving skew (verify periodically with shadow mode comparison)

□ Real-time inference pipeline:
  Model loaded in memory (no disk I/O at inference time)
  Feature retrieval: Redis pipeline (batch fetch in 1 round-trip)
  Total latency: < 30ms for payment-critical models
  Fallback: if model unavailable → rule-based fallback (Case 22 Stage 1)

□ Model governance:
  Every production model: version ID, performance metrics, training data period
  Champion/challenger: new models validated in shadow mode before promotion
  Rollback: automated trigger on performance degradation
  Fairness audit: monthly FPR by demographic group

□ Explainability:
  Every BLOCK decision: SHAP-based reason codes
  Human-readable: customer can understand why they were blocked
  Regulatory: stored in fraud_cases table with model_version

□ Drift monitoring:
  PSI daily for all input features
  Automatic retraining trigger on PSI > 0.25
  Alert on AUC drop > 3% from champion

□ SCC LENS:
  STATE: feature store (Redis online, S3 offline), model registry (S3), training data (S3 Parquet)
  COORDINATION: Airflow pipeline (retraining), Flink streaming (real-time feature updates)
  CONCENTRATION: Redis feature store → every fraud check reads it → Redis cluster mandatory
```

---

*Next: Case Study 50 — Emerging Technologies: Passkeys, Decentralised Identity, CAEP*
