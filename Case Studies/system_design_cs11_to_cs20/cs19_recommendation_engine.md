# Case Study 19 — Recommendation Engine

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Collaborative Filtering, Two-Tower Model, Cold Start Problem.

---

## Business Context

A recommendation engine for a payment platform surfaces relevant merchants, offers, and financial products to users based on their transaction history. Unlike Netflix or Amazon where bad recommendations are just annoying, in fintech a bad recommendation (suggesting a loan to someone with debt, or a merchant in a country the user can't transact in) can cause actual harm. Relevance matters; so does responsibility.

**Business goals driving architecture:**
- Increase merchant discovery and checkout conversion
- Personalized: recommendations based on user's actual transaction history
- Diverse: avoid filter bubbles (not just "pay Amazon again")
- Freshness: new merchants can appear in recommendations within 1 day
- Privacy: transaction data is sensitive; recommendation model must not expose other users' data

---

## Recognition Framework

### Signals from the Problem
```
- User has implicit feedback (transaction history) not explicit ratings
- Matrix is sparse: most users haven't transacted with most merchants
- Cold start problem: new users and new merchants have no history
- Real-time inference: recommendation must be fast at serving time (< 50ms)
- Model training is offline: cannot train in real-time; train daily/weekly
- Billions of (user, merchant) pairs: cannot compute all recommendations at query time
- Personalization vs privacy: must not reveal other users' transaction patterns
```

### Algorithm Selection

```
Option A: Collaborative Filtering (Matrix Factorization)
  Model: decompose user-merchant interaction matrix into user_embedding × merchant_embedding
  Training: ALS (Alternating Least Squares) or neural CF
  Recommendation: dot_product(user_embedding, all_merchant_embeddings) → rank
  
  Pros: handles sparse data; captures latent preferences; well-understood
  Cons: cold start problem (new user/merchant has no embeddings); requires retraining for new items
  
  Use when: large interaction history; cold start handled separately

Option B: Content-Based Filtering
  Model: recommend merchants similar to ones user has transacted with
  Features: merchant category, location, price range, tags
  
  Pros: no cold start for merchants (content features available from creation)
  Cons: filter bubble (only recommends similar to what user already does)
  
  Use when: new user cold start; new merchant cold start

Option C: Two-Tower Neural Network (CHOSEN for candidate generation)
  Architecture:
    User tower: user_id → embedding + user features → user representation vector
    Item tower: merchant_id → embedding + merchant features → item representation vector
    Training: maximize dot product for interacted pairs; minimize for non-interacted
    Inference: user_tower(user) → user_vector; ANN search → top-K nearest merchant_vectors
  
  Pros: efficient; ANN search at serving time (< 5ms); handles cold start (feature-based)
  Cons: training complexity; requires feature engineering; GPU training infrastructure

CHOSEN: Two-Tower + Collaborative Filtering hybrid
  Candidate generation: Two-Tower retrieval (fast; top 500 candidates)
  Ranking: LightGBM re-ranking model (slower; complex features; top 20 from 500)
  Cold start: Content-based fallback (no transaction history)
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| Two-Tower Neural Network | Fast candidate retrieval via ANN; handles cold start with features |
| ANN Search (FAISS/HNSW) | Efficient nearest-neighbor over 1M+ merchant embeddings |
| Candidate Generation + Ranking | Two-stage: retrieve 500 broadly relevant → rank top 20 precisely |
| Offline Training Pipeline | Models trained daily; not real-time (data volume makes online training impractical) |
| Feature Store | Shared user and merchant features; used by both training and serving |
| Exploration vs Exploitation | Thompson sampling or ε-greedy: occasionally show non-obvious recommendations |

---

## Problem Statement

Recommendation engine for merchant discovery on a payment platform. Generates top-20 personalized merchant recommendations for each user, served with < 50ms latency.

---

## Functional Requirements
- Personalized merchant recommendations for logged-in users
- New user recommendations (cold start: use demographic + location + trending)
- New merchant promotion (cold start: feature in new merchant section)
- Category-specific recommendations ("food near me", "electronics under ₹2000")
- Explanation: why is this recommended ("Based on your UPI transactions at food merchants")
- Feedback: user can dismiss recommendations; model learns from dismissals

## Non-Functional Requirements
- Serving latency: **< 50ms p99**
- Recommendation freshness: **1 day** (re-ranked daily; user's new transactions reflected next day)
- Coverage: every active user has personalized recommendations (no fallback to trending-only)
- Diversity: top-20 should span at least 5 different merchant categories

---

## Capacity Estimation
```
Users: 100M active users needing recommendations
Merchants: 1M indexed merchants

Embeddings:
  User embeddings: 100M users × 128 dimensions × 4 bytes = 51 GB
  Merchant embeddings: 1M merchants × 128 dimensions × 4 bytes = 512 MB
  Both fit in memory on a single large machine; or distributed across nodes

ANN index (FAISS):
  1M merchant vectors of dim=128: ~512 MB index (FAISS flat index)
  HNSW index: ~1.5 GB but much faster query time (< 5ms)

Pre-computed recommendations:
  100M users × 20 recommendations × 8 bytes (merchant_id) = 16 GB
  Store in Redis: 16 GB for all active users' pre-computed results
  → Pre-compute offline nightly; serve from Redis at query time
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Two-Tower Model | Candidate retrieval: user vector → ANN search → 500 candidates |
| LightGBM Ranker | Re-rank 500 candidates to top 20; rich features; fast inference |
| Pre-computed Recommendations | Nightly batch; serve from Redis; < 1ms serving latency |
| ANN (HNSW/FAISS) | O(log N) approximate nearest neighbor; < 5ms for 1M merchants |
| Feature Store | Online features (recent transactions) + offline features (user demographics) |
| Cold Start Fallback | New user → trending in region + category; new merchant → categorical ranking |

---

## Data Model

```python
# Feature Store (Redis Hashes — online features used at serving time)
# user_features:{user_id} → hash of recent features
#   preferred_categories:    "FOOD,SHOPPING,TRANSPORT"
#   avg_transaction_amount:  "1500"
#   preferred_payment_method: "UPI"
#   top_merchants:           "merch-1,merch-2,merch-5"
#   city:                    "Mumbai"
#   last_transaction_days_ago: "0"
# TTL: 24h (refreshed by daily feature computation job)

# merchant_features:{merchant_id} → hash
#   category: "FOOD"
#   avg_ticket_size: "250"
#   city: "Mumbai"
#   new_merchant: "false"
#   popularity_score: "8734"
# TTL: 24h

# Pre-computed recommendations (Redis — served directly at query time)
# recs:{user_id} → JSON list of merchant_ids with scores and reasons
# TTL: 25h (slightly longer than daily refresh; handles refresh delay gracefully)
```

```sql
-- Training data (PostgreSQL / Data Warehouse)
-- User-merchant interaction matrix
CREATE TABLE user_merchant_interactions (
  user_id         VARCHAR(64),
  merchant_id     VARCHAR(64),
  interaction_type VARCHAR(32),  -- PAYMENT / VIEWED / DISMISSED
  amount          DECIMAL(15,2),
  count           INTEGER,
  last_interaction TIMESTAMP,
  PRIMARY KEY (user_id, merchant_id)
);

-- Model artifacts (stored in S3)
-- s3://ml-models/recommendation/v{version}/
--   user_embeddings.npy       (100M × 128 float32)
--   merchant_embeddings.npy   (1M × 128 float32)
--   faiss_index.bin           (HNSW index over merchant embeddings)
--   ranker_model.lgb          (LightGBM model for ranking)
--   model_metadata.json       (version, training date, metrics)
```

---

## High-Level Architecture

```
OFFLINE PIPELINE (runs nightly; Airflow orchestrated):

1. Feature Engineering (Spark on S3 data lake)
   Input: raw transaction events (S3 Parquet)
   Output: user_merchant_interaction matrix + user/merchant features
   
2. Two-Tower Model Training (Python/PyTorch on GPU cluster)
   Input: interaction matrix + features
   Training: maximize dot_product(user_vec, merchant_vec) for interacted pairs
             contrastive loss against random negative samples
   Output: user embeddings (100M × 128) + merchant embeddings (1M × 128)
   Duration: 4-6 hours on 8 × A100 GPUs

3. Build ANN Index (FAISS HNSW)
   Input: merchant embeddings (1M × 128)
   Build HNSW index (M=32, efConstruction=200): ~30 minutes
   Upload to S3; push to serving nodes

4. Pre-compute Recommendations (Spark + FAISS)
   For each active user (100M):
     Load user embedding → FAISS ANN search → 500 candidate merchants
     Run LightGBM ranker → top 20 merchants with scores + reasons
   Write to Redis: recs:{user_id} → pre-computed list
   Duration: 100M users / Spark parallelism (~10M/hour) = ~10 hours
   → Runs 22:00–08:00 UTC (10h window)

5. Model Validation (before promoting to production)
   Offline metrics: NDCG@20, Hit Rate@5
   A/B shadow mode: run new model alongside old; compare distributions
   Promote only if NDCG@20 > previous version AND no regressions

ONLINE SERVING (< 50ms):

GET /api/v1/recommendations/{user_id}
  1. Check Redis: GET recs:{user_id}
     HIT (99% of requests): return pre-computed list in < 1ms
     MISS (new user, or Redis eviction): proceed to real-time fallback
  
  2. Real-time fallback (for cache miss):
     a. Load user features from feature store: GET user_features:{user_id}
     b. If user has embedding: FAISS ANN search → 500 candidates → LightGBM rank → top 20
     c. If no embedding (new user): cold start → trending in user's region + categories
     d. Cache result in Redis with 25h TTL
     Return in < 50ms (FAISS search: 5ms; LightGBM: 10ms; overhead: 35ms)
```

---

## Detailed Components

### Two-Tower Model Architecture
```python
class TwoTowerModel(nn.Module):
    def __init__(self, num_users, num_merchants, embedding_dim=128):
        super().__init__()
        
        # User tower
        self.user_embedding = nn.Embedding(num_users, embedding_dim)
        self.user_mlp = nn.Sequential(
            nn.Linear(embedding_dim + USER_FEATURE_DIM, 256),
            nn.ReLU(),
            nn.Linear(256, embedding_dim),
            nn.LayerNorm(embedding_dim)
        )
        
        # Merchant tower
        self.merchant_embedding = nn.Embedding(num_merchants, embedding_dim)
        self.merchant_mlp = nn.Sequential(
            nn.Linear(embedding_dim + MERCHANT_FEATURE_DIM, 256),
            nn.ReLU(),
            nn.Linear(256, embedding_dim),
            nn.LayerNorm(embedding_dim)
        )
    
    def user_forward(self, user_id, user_features):
        emb = self.user_embedding(user_id)
        x = torch.cat([emb, user_features], dim=-1)
        return self.user_mlp(x)  # normalized 128-dim vector
    
    def merchant_forward(self, merchant_id, merchant_features):
        emb = self.merchant_embedding(merchant_id)
        x = torch.cat([emb, merchant_features], dim=-1)
        return self.merchant_mlp(x)  # normalized 128-dim vector
    
    def forward(self, user_id, user_features, merchant_id, merchant_features):
        user_vec = self.user_forward(user_id, user_features)
        merchant_vec = self.merchant_forward(merchant_id, merchant_features)
        return torch.dot(user_vec, merchant_vec)  # similarity score

# Training objective: contrastive loss
# Positive: (user, merchant they transacted with)
# Negative: (user, random merchant they haven't seen)
# Loss = -log(sigmoid(pos_score - neg_score))
```

### Cold Start Problem Solutions
```
Cold Start Type 1: New User (no transaction history)
  Problem: no user embedding; no interaction history
  
  Solution tiers (in priority order):
    1. Location-based trending: top merchants in user's city this week
    2. Category interest (from onboarding): user said "I love food" → top food merchants
    3. Similar users (demographic): users with same age/income bracket → their merchants
    4. Pure trending: top 20 merchants by transaction volume this week

  Trigger for personalization:
    After 5 transactions: generate first personalized embedding
    After 20 transactions: embedding is reliable; full personalization

Cold Start Type 2: New Merchant (no transactions yet)
  Problem: no merchant embedding; ANN search won't find it
  
  Solution:
    Content-based: merchant is tagged with category, price_range, location
    Use merchant features directly (without embedding) for initial ranking
    Appears in "New on Platform" section for first 30 days
    After 100 transactions: embedding trained; integrated into personalized recommendations

Cold Start Type 3: New Category / Geography
  Edge case: existing user moves to new city
  Solution:
    Detect city change from new transactions → add location as context feature
    Blend: 50% existing preferences + 50% local trending
    After 10 local transactions: preferences updated; blend shifts back to personalized
```

### Recommendation Explanation
```python
def generate_explanation(user_id: str, merchant_id: str, score: float,
                          user_top_merchants: list, merchant_categories: dict) -> str:
    """Generate human-readable explanation for recommendation."""
    
    # Identify matching signal
    user_prefs = get_user_preferences(user_id)
    merchant_category = merchant_categories.get(merchant_id, "")
    
    if merchant_category in user_prefs.top_categories:
        return f"Because you often pay at {merchant_category} merchants"
    
    if any_merchant_in_same_network(user_top_merchants, merchant_id):
        return "Popular with users who pay at similar places"
    
    if is_trending_in_city(merchant_id, user_prefs.city):
        return f"Trending in {user_prefs.city} this week"
    
    return "Recommended for you"  # fallback
```

---

## Scaling Strategy
```
Phase 1 (1M users, 10K merchants):
  Simple collaborative filtering (cosine similarity on interaction matrix)
  Pre-compute all recommendations nightly; store in DB
  No ML infrastructure needed; numpy/sklearn sufficient

Phase 2 (10M users, 100K merchants):
  Two-tower model training on single GPU
  FAISS index on single node
  Pre-computed recommendations in Redis

Phase 3 (100M users, 1M merchants):
  Distributed training (PyTorch distributed; multi-GPU)
  Distributed FAISS (sharded by merchant_id)
  Spark for pre-computation (parallelize over user shards)

Phase 4 (real-time personalization):
  Online learning: update embeddings incrementally on new transactions
  (instead of nightly re-training)
  Use Flink to compute rolling feature updates → streaming feature store
  New transaction → feature updated within 5 minutes
```

---

## Reliability + Monitoring

```
Model quality metrics (logged for every recommendation served):
  click_through_rate (CTR):      % of recommendations clicked/acted on
  conversion_rate:               % leading to completed transaction
  coverage:                      % of merchants appearing in at least 1 recommendation
  diversity_score:               avg unique categories in top-20
  cold_start_rate:               % of users getting cold-start fallback

A/B Testing:
  Experimental framework: 10% of users → new model; 90% → old
  Holdout group: 1% of users → random recommendations (control)
  Metrics: CTR, conversion, diversity; 2-week minimum experiment duration
  Statistical significance: p < 0.05 before promoting new model

Alerting:
  ctr_drop > 20% from baseline          → new model degradation; rollback
  cold_start_rate > 30%                 → feature store pipeline failure
  recommendation_latency_p99 > 100ms    → Redis issue or serving node overload
```

---

## Chaos Testing
```
Experiment 1: Redis Pre-computed Recommendations Evicted
  Inject:    Flush all recs:{user_id} keys from Redis
  Expected:  Cache miss triggers real-time inference (FAISS + LightGBM)
  Expected:  Serving latency increases from < 1ms to < 50ms (still within SLO)
  Expected:  Redis re-populated for active users within next page load
  Red flag:  Recommendations unavailable (service returns error instead of real-time fallback)

Experiment 2: New Merchant Cold Start
  Inject:    Create new merchant; make 0 transactions; request recommendations to find this merchant
  Expected:  Merchant does NOT appear in personalized recommendations (no embedding yet)
  Expected:  Merchant appears in "New on Platform" section (content-based category match)
  Expected:  After 100 transactions (simulated): merchant embedding trained; appears in ANN results
  Red flag:  New merchant appears in personalized recs immediately (data leakage)

Experiment 3: Model Training Pipeline Failure
  Inject:    Kill GPU training job at 50%; Airflow marks as failed
  Expected:  Old model version continues serving (no rollback needed; old model in Redis)
  Expected:  Airflow retries training job with new attempt
  Expected:  Alert fires: "model training failed; recommendations will be stale after 48h"
  Red flag:  Serving layer switches to untrained/partially-trained model
```

---

## Monthly Cost Estimate
```
Scale: 100M users, 1M merchants

GPU Training (nightly; 8 × A100 × 6 hours):
  8 × A100 × $3.50/hr × 6hr × 30 days                 = $5,040/mo

Spark Pre-computation (EMR; 10h/day):
  100 × m5.2xlarge × 10h × $0.043/hr × 30 days        = $12,900/mo

Redis (pre-computed recommendations):
  16 GB data → cache.r6g.xlarge ($260/mo)              = $260/mo

FAISS Serving Nodes (3 × replicas):
  3 × r6g.2xlarge ($500/mo) [4 GB HNSW index + 50 GB user embeddings] = $1,500/mo

Feature Store (Redis):
  51 GB user embeddings: 2 × r6g.2xlarge ($500/mo)    = $1,000/mo

PostgreSQL (interaction matrix + metadata):
  db.r6g.2xlarge ($500/mo)                             = $500/mo

Model Registry (S3):
  Model artifacts ~5 GB per version × 30 versions     = negligible
─────────────────────────────────────────────────────────────────
Total: ~$21,200/month
Dominated by Spark pre-computation (61%) + GPU training (24%)

Optimisation: reduce Spark cost with smaller cluster + longer runtime
  or replace pre-computation with online FAISS serving (lower Spark cost; higher serving cost)
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → S3 data lake: raw transaction events (input to training pipeline).
  → PostgreSQL: interaction matrix (training data; authoritative).
  → S3 model registry: trained model artifacts (versioned; immutable per version).
  → Redis: pre-computed recommendations (cache; rebuilt nightly from model).

□ Consistency model?
  → Recommendations: stale by up to 24h (nightly refresh) for pre-computed.
  → Real-time fallback: uses latest model (< 50ms; for cache miss).
  → User features: eventual (24h refresh; new transactions reflected next day).

□ Failure mode?
  → Training pipeline fails: old model continues serving; alert fires; 48h before staleness matters.
  → Redis evicted: real-time FAISS fallback; within SLO.
  → FAISS nodes down: cold start fallback only; lower quality but no outage.
  → Feature store down: recommendations degrade to cold start; alert fires.

□ Cold start is a first-class concern (not an afterthought):
  → New users: location-based trending → categorical → demographic similar users
  → New merchants: "New on Platform" section; not in personalized feed until 100 transactions
  → Explicit transitions: 5 transactions → first personalized; 20 → reliable personalization

□ SCC LENS:
  STATE: S3 (training data), model registry (model artifacts), Redis (pre-computed recs), feature store
  COORDINATION: Airflow orchestrates nightly pipeline; version promotion after validation
  CONCENTRATION: FAISS index hot (100M users all query same 1M merchant vectors) → replicate FAISS
```

---

## Staff Engineer Discussion Points

**"How do you evaluate recommendation quality without explicit ratings?"**
Implicit feedback: transaction = strong positive signal; view without transaction = weak positive; dismissal = negative. Offline evaluation: NDCG@20 (did good merchants rank highly?), Hit Rate@K (was the merchant the user eventually transacted with in the top-K?). Online evaluation: CTR and conversion rate (A/B test against control). The control group gets trending recommendations (not personalized) — any CTR/conversion improvement over control = recommendation value.

**"How do you prevent the recommendation system from exposing sensitive data?"**
The two-tower model learns patterns but doesn't store individual user-merchant relationships. Differential privacy during training: add Gaussian noise to gradients (DP-SGD) — prevents model from memorizing specific user transactions. At serving time: user embedding is computed from user's own features; ANN search returns nearest merchants without exposing other users' data. The embedding space is abstract (not interpretable as "users like you bought X").

---

## Cheat Sheet Tie-in

```
TWO-TOWER + ANN PATTERN:
  Offline: train user tower + merchant tower → 128-dim embeddings
  Offline: build HNSW index on 1M merchant embeddings
  Online: user_tower(user) → 128-dim vector → ANN search → top-500 candidates
  Online: LightGBM re-rank 500 → top-20 → serve

CANDIDATE GENERATION → RANKING (two-stage):
  Stage 1: recall (retrieve broadly relevant items; fast; ANN search < 5ms)
  Stage 2: rank (precise; complex features; LightGBM < 10ms on 500 candidates)
  Why two stages: can't run complex ranking on all 1M merchants; ANN is the funnel

COLD START PYRAMID:
  New user: location trending → category interest → demographic similar → global trending
  New merchant: "New on Platform" → content-based category → wait for 100 transactions
  
SCC LENS:
  STATE: S3 training data + model artifacts, Redis pre-computed recs, feature store
  COORDINATION: Airflow nightly pipeline; A/B test before model promotion
  CONCENTRATION: FAISS serving → all users query same merchant index → replicate (3×)
```

---

*Next: Case Study 20 — Distributed Job Scheduler*
