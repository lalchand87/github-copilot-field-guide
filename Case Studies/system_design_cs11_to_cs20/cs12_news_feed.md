# Case Study 12 — News Feed

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Fan-out on Write vs Read, Celebrity Problem, Feed Ranking.

---

## Business Context

A news feed aggregates content from people and entities a user follows and presents it in a ranked order. For a fintech platform, this could be transaction updates from accounts you track, updates from merchants you follow, or financial news. The core challenge is the write amplification problem: one post by a celebrity with 10M followers requires writing to 10M feed timelines.

**Business goals driving architecture:**
- Feed loads in < 200ms (users abandon slow feeds)
- Fresh content: new posts appear in feed within 5 seconds of publishing
- Relevance: ranked by recency and engagement score, not just chronological
- Scale: 500M users; top celebrities have 100M followers

---

## Recognition Framework

### Signals from the Problem
```
- Read-heavy: users read feed 10× more than they post
- Write amplification: 1 post → N (follower count) feed inserts
- Celebrity problem: 100M followers × 1 post = 100M writes in seconds
- Pre-computation vs lazy computation: tradeoff is the whole design
- Feed is personalized: different ranking for each user (engagement, follows)
- Infinite scroll: pagination of a ranked, updating dataset
- Eventual consistency tolerable: seeing a post 5s late is fine
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| Fan-out on Write (push model) | Regular users: pre-compute feed; read is O(1) |
| Fan-out on Read (pull model) | Celebrities (>1M followers): pull at read time; avoid 100M writes |
| Hybrid (push + pull) | Best of both: push for regular users, pull for celebrities |
| Redis Sorted Set | Feed timeline: sorted by score (timestamp × engagement weight) |
| Async Fan-out (Kafka) | Fan-out is slow; post returns immediately; feed updates async |

### Patterns Rejected
| Pattern | Reason Rejected |
|---|---|
| Fan-out on Write for ALL users | Celebrity with 100M followers → 100M Redis writes; takes minutes |
| Fan-out on Read for ALL users | Read time: fetch all followed users' posts + rank → 100s of DB queries per feed load |
| Chronological only (no ranking) | Low engagement; relevant content buried by noise |
| DB as feed store | Joins across follows + posts at read time → complex query → high latency |

### Fan-out on Write vs Read — The Core Tradeoff

```
Fan-out on Write (push model):
  On post: for each follower → write post_id to their Redis sorted set
  On feed load: just read the sorted set → O(1)
  
  Pros:  Feed read is instant; no per-read computation
  Cons:  Write amplification — 1M followers = 1M Redis writes (takes ~10s)
         Wasted writes: user may not open app for days; we wrote to their feed anyway
         Memory: store feed for every user even if inactive
  
  Best for: Regular users (< 1K followers)

Fan-out on Read (pull model):
  On post: just store post in user's post list; no fan-out
  On feed load: fetch all followed user_ids → fetch latest posts for each → merge + rank

  Pros:  No write amplification; celebrities only store 1 record per post
  Cons:  Read is expensive — following 500 accounts = 500 DB queries per feed load
         Latency scales with number of follows
  
  Best for: Celebrity accounts (> 1M followers)

Hybrid approach (chosen):
  Regular users (< 10K followers): fan-out on write → feed pre-computed in Redis
  Celebrities (> 10K followers): no fan-out; pulled at read time for all followers
  
  At feed load time:
    1. Read pre-computed feed from Redis (fan-out on write entries)
    2. Fetch posts from any celebrity the user follows (fan-out on read entries)
    3. Merge + re-rank by score
    4. Return merged feed
```

---

## Problem Statement

Social-style news feed. Users follow other users and entities. Feed shows recent + relevant posts from followed accounts, ranked by a scoring function. Feed loads in < 200ms. New posts appear in feed within 5 seconds.

---

## Functional Requirements
- Follow/unfollow users and entities
- Post text/media content
- Read personalized ranked feed (latest posts from followed accounts)
- Like, comment, share (engagement signals for ranking)
- Paginate feed (infinite scroll: cursor-based, not offset)
- Feed freshness: < 5s for non-celebrity posts

## Non-Functional Requirements
- Feed load latency: **< 200ms p99**
- Write (post) to feed appearance: **< 5 seconds** for regular users
- Scale: **500M users**, **100M celebrities**
- Read/write ratio: **100:1** (feeds are read far more than posted)

---

## Capacity Estimation
```
Posts:
  500M users × 1 post/week avg = ~71M posts/day = ~823/sec

Fan-out writes:
  Average user: 200 followers → 823 posts/sec × 200 followers = 164K feed writes/sec
  Celebrity posts (10K celebrities × 10M followers avg):
    Without fan-out: 10K posts/day; fan-out deferred to read time

Redis feed storage:
  Feed stored per user: last 1,000 posts in sorted set
  Entry: post_id (8 bytes) + score (8 bytes) = 16 bytes per entry
  Active users (10%): 50M × 1,000 × 16 bytes = 800 GB
  Not all users have active feeds (LRU eviction for inactive users)

Follow graph:
  500M users × avg 200 follows = 100B follow relationships
  Follow table: stored in Cassandra (wide row per user: user_id → [followed_ids])
  Not stored in Redis (too large)
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Hybrid Fan-out | Push for regular; pull for celebrities at read time |
| Redis Sorted Set | Feed timeline: ZADD score post_id; ZRANGE for paginated read |
| Kafka (async fan-out) | Post published → Kafka event → fan-out workers update follower feeds |
| Feed Score Function | Rank by recency × engagement weight; not pure chronological |
| Cursor Pagination | ZRANGEBYSCORE with cursor; stable pagination for updating feed |

---

## Data Model

```sql
-- Posts (PostgreSQL or Cassandra)
CREATE TABLE posts (
  post_id      UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
  author_id    VARCHAR(64)  NOT NULL,
  content_type VARCHAR(32),           -- TEXT / IMAGE / VIDEO / LINK
  content_text TEXT,
  media_url    TEXT,
  like_count   BIGINT       DEFAULT 0,
  comment_count BIGINT      DEFAULT 0,
  share_count  BIGINT       DEFAULT 0,
  created_at   TIMESTAMP    DEFAULT NOW()
);

-- Follow graph (Cassandra — wide row, efficient for "get all follows for user")
-- follow_graph: (user_id, followed_id) → created_at
-- Partition key: user_id → all follows for a user on same partition

-- Fan-out config per user (determines push vs pull)
CREATE TABLE user_feed_config (
  user_id      VARCHAR(64)  PRIMARY KEY,
  follower_count BIGINT,
  is_celebrity BOOLEAN      DEFAULT FALSE,  -- recomputed daily; > 10K followers
  feed_active  BOOLEAN      DEFAULT TRUE    -- inactive users: don't pre-compute feed
);
```

```
Redis sorted sets (pre-computed feeds):
  feed:{user_id}              → sorted set of (post_id, score)
  score = post_created_at_unix * engagement_weight
  TTL: 7 days (inactive users' feeds expire; rebuilt on next app open)

  ZADD feed:{user_id} {score} {post_id}       -- add post to feed
  ZRANGE feed:{user_id} {cursor} {cursor+20}  -- paginate
  ZREMRANGEBYSCORE feed:{user_id} 0 {cutoff}  -- trim old posts

Celebrity post lists (for fan-out on read):
  celebrity_posts:{celebrity_id}  → sorted set of (post_id, timestamp)
  TTL: 7 days (enough history for any user's feed)
```

---

## API Design

```
POST /api/v1/posts
Body: { "content_type": "TEXT", "content_text": "..." }
Response 201: { "post_id": "uuid", "created_at": "..." }
(async fan-out begins after this returns)

GET /api/v1/feed
Query params: cursor=<last_score>&limit=20
Response 200: {
  "posts": [
    {
      "post_id": "uuid",
      "author": { "user_id": "u-123", "name": "Alice", "avatar_url": "..." },
      "content_text": "...",
      "like_count": 142,
      "created_at": "...",
      "score": 1712345600.87
    },
    ...
  ],
  "next_cursor": "1712345000.0",  // score of last item; use for next page
  "has_more": true
}

POST /api/v1/posts/{post_id}/like
Response 200: { "like_count": 143 }
(triggers re-scoring in fan-out pipeline — post may move up in feed)

GET /api/v1/users/{user_id}/posts?cursor=&limit=20
GET /api/v1/follows              → list of user_ids I follow
POST /api/v1/follows/{user_id}   → follow
DELETE /api/v1/follows/{user_id} → unfollow
```

---

## High-Level Architecture

```
Post Creation:
  User → POST /api/v1/posts
  API → INSERT to PostgreSQL posts table
  API → PUBLISH to Kafka topic "new-posts"
  Return 201 immediately

  Kafka "new-posts" topic:
    → Fan-out Worker (consumes; determines followers)
        If author.follower_count < 10K (regular):
          GET follow_graph:{author_id} from Cassandra → list of follower_ids
          For each follower:
            score = compute_score(post, follower_affinity)
            ZADD feed:{follower_id} score post_id
            ZREMRANGEBYRANK feed:{follower_id} 0 -1001  // keep only top 1000
        If author.follower_count >= 10K (celebrity):
          ZADD celebrity_posts:{author_id} timestamp post_id
          // No per-follower fan-out; pulled at read time

Feed Read:
  User → GET /api/v1/feed?cursor=&limit=20
  Feed Service:
    1. Get user's followed celebrities: SELECT followed_id FROM follow_graph
       WHERE user_id = ? AND followed_id IN (SELECT user_id FROM user_feed_config WHERE is_celebrity=TRUE)
    2. Fetch pre-computed feed: ZRANGEBYSCORE feed:{user_id} (-inf, cursor) LIMIT 20
    3. Fetch celebrity posts for each followed celebrity:
         ZRANGEBYSCORE celebrity_posts:{celebrity_id} (now-7days, now) LIMIT 20
    4. Merge all post sets; rank by score; take top 20
    5. Hydrate: MGET post details for all post_ids from Redis/PostgreSQL
    6. Return paginated response
```

---

## Detailed Components

### Feed Score Function
```python
def compute_score(post, follower_affinity=1.0) -> float:
    """
    Score balances recency with engagement.
    Time decay: score halves every 12 hours.
    Engagement boost: viral posts stay visible longer.
    Affinity boost: posts from close contacts rank higher.
    """
    age_hours = (now - post.created_at).total_seconds() / 3600
    time_decay = 0.5 ** (age_hours / 12)          # halves every 12 hours

    engagement = (
        post.like_count * 1.0 +
        post.comment_count * 3.0 +                 # comments signal deeper engagement
        post.share_count * 5.0                     # shares signal highest relevance
    )
    engagement_boost = math.log1p(engagement)      # log scale: prevents viral domination

    return time_decay * (1 + engagement_boost) * follower_affinity

# follower_affinity: 1.0 for most follows; > 1.0 for "close friends"
# Affinity computed from mutual interactions (likes, comments) — updated weekly
```

### Fan-out Optimization for Large Accounts
```
Boundary for celebrity threshold: 10K followers
Why 10K?
  Fan-out for 10K followers: 10K Redis writes ≈ 20ms (Redis pipeline) → acceptable
  Fan-out for 100K followers: 100K Redis writes ≈ 200ms → at post creation time, borderline
  Fan-out for 1M followers: takes > 1 second → too slow even async at the scale of all posts

Hybrid at read time:
  User follows 5 celebrities (>10K each) + 200 regular users
  Feed load:
    Read pre-computed sorted set (200 regular users' posts): 1 Redis ZRANGE call
    Fetch from 5 celebrity post lists: 5 Redis ZRANGE calls (parallel pipeline)
    Merge 5 lists + pre-computed list: in-memory sort (~0.1ms for 200 items)
    Total extra cost: 5 Redis calls + merge ≈ 5ms additional

Inactive user optimization:
  If user hasn't opened app in 7 days: expire their feed sorted set (Redis TTL)
  On next app open: rebuild feed from scratch (fan-out worker processes all followed users)
  Saves memory: don't maintain feeds for users who never read them
```

### Follow/Unfollow Fan-out
```
On follow (user A follows user B):
  1. INSERT into follow_graph Cassandra
  2. If B is not celebrity: backfill A's feed with B's last 20 posts
     (so new follows appear immediately in feed)
  3. If B is celebrity: A's feed will pull from celebrity_posts:{B} at read time

On unfollow (user A unfollows user B):
  1. DELETE from follow_graph Cassandra
  2. Remove B's posts from A's feed (background job):
     Get B's recent post_ids → ZREM feed:{A} for each post_id
  Note: this is eventually consistent (A may see B's posts for a few seconds after unfollow)
  Acceptable: immediate strong consistency would require locking A's feed during write
```

---

## Scaling Strategy
```
Phase 1 (0 → 1M users):
  Fan-out on write for all users
  Redis for feed storage
  Single feed worker

Phase 2 (1M → 50M users):
  Identify celebrity accounts (> 10K followers)
  Hybrid fan-out: push for regular, pull for celebrities
  Multiple fan-out workers (Kafka consumer group)

Phase 3 (50M → 500M users):
  Feed worker sharding by author_id (different workers handle different authors)
  Feed storage sharding by user_id (Redis Cluster)
  Follow graph in Cassandra (too large for PostgreSQL at 100B relationships)
  CDN for post media (images/video separate from feed API)

Phase 4 (global, multi-region):
  Regional feed services (US, EU, Asia)
  Celebrity posts replicated globally (small dataset; high read demand)
  Follow graph replicated per region (reads local; writes global)
```

---

## Reliability Strategy
- **Feed rebuild on Redis miss**: if user's feed sorted set expired or evicted → rebuild from Cassandra follow graph + recent posts
- **Fan-out backpressure**: if fan-out workers lag, rate limit new fan-outs by shedding lower-priority operations
- **Celebrity post caching**: `celebrity_posts:{id}` has 7-day TTL; rebuilt from DB on expiry
- **Post deduplication in feed**: sorted set ensures same post_id cannot appear twice regardless of fan-out retries

---

## Observability
```
Metrics:
  feed_load_latency_p99          (target: < 200ms; alert: > 500ms)
  fan_out_lag_seconds            (time from post → appears in all follower feeds)
  feed_cache_hit_rate            (% of feed reads served from Redis; target > 95%)
  celebrity_pull_latency_ms      (time to pull celebrity posts at read time)
  fan_out_queue_depth            (Kafka consumer lag for fan-out workers)

Alerts:
  feed_load_latency_p99 > 500ms → Redis latency or fan-out lag
  fan_out_queue_depth > 100K    → Fan-out workers not keeping up; scale
  feed_cache_hit_rate < 80%     → Redis undersized or too many cache misses
```

---

## Chaos Testing
```
Experiment 1: Celebrity Posts 10M Followers
  Inject:    Celebrity (10M followers) posts; observe fan-out behavior
  Expected:  No per-follower Redis writes (celebrity threshold prevents fan-out)
  Expected:  Post appears in ZADD celebrity_posts:{id} only
  Expected:  All 10M followers see post within 200ms of opening feed (pull at read time)
  Red flag:  10M Redis writes triggered (fan-out on write for celebrity)

Experiment 2: Redis Feed Sorted Set Expiry
  Inject:    Manually expire feed:{user_id} for a user
  Expected:  Feed API detects empty/missing sorted set
  Expected:  Background rebuild triggered: follow graph → recent posts → ZADD
  Expected:  User sees rebuilt feed on next page load (may be slightly delayed)
  Red flag:  Empty feed returned without rebuild attempt

Experiment 3: Fan-out Worker Lag
  Inject:    Kill 3 of 4 fan-out workers; create 10K new posts
  Expected:  Kafka consumer lag grows; posts delayed in follower feeds
  Expected:  Transactional alerts fire (fan_out_lag > 30s)
  Expected:  Workers restart and catch up from Kafka offset
  Red flag:  Posts permanently lost from feeds (events dropped from Kafka)
```

---

## Monthly Cost Estimate
```
Redis (feed sorted sets):
  800 GB across cluster → 10 shards × r6g.xlarge ($260/mo)  = $2,600/mo

Cassandra (follow graph):
  100B relationships at ~16 bytes = 1.6 TB → 5 nodes r6g.2xlarge = $2,500/mo

Kafka (fan-out):
  3 × kafka.m5.large ($150/mo)                                = $450/mo

Fan-out workers (16 instances):
  16 × m5.large ($75/mo)                                      = $1,200/mo

PostgreSQL (posts):
  db.r6g.2xlarge ($500/mo) + read replica                     = $1,000/mo

Feed API (8 instances):
  8 × c5.xlarge ($150/mo)                                     = $1,200/mo
─────────────────────────────────────────────────────────────────
Total: ~$8,950/month
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → PostgreSQL: canonical post records (content, metadata, engagement counts).
  → Cassandra: follow graph (too large and write-heavy for PostgreSQL).
  → Redis: pre-computed feed (cache; rebuilt from follow graph + posts on expiry).
  → Redis is expendable: if lost, rebuilt from PostgreSQL + Cassandra.

□ Consistency model?
  → Feed: eventual (fan-out async; < 5s for non-celebrity, read-time for celebrity).
  → Follow/unfollow: strong for follow graph insert; eventual for feed update.
  → Engagement counts (likes): eventual (Redis counter; periodic sync to PostgreSQL).

□ Failure mode?
  → Redis feed lost → rebuild from follow graph + posts (higher latency for one request).
  → Fan-out lag → users see older feed; recovers when lag clears.
  → Celebrity post list lost → read from PostgreSQL posts table (slower but correct).

□ Hot partitions?
  → Celebrity posting → hot Kafka partition (if partitioned by author_id).
  → Mitigation: fan-out workers partition by follower_id, not author_id.
  → Feed reads: each user's feed is a separate Redis key; no single hot key.

□ SCC LENS:
  STATE: PostgreSQL (posts), Cassandra (follows), Redis (pre-computed feeds)
  COORDINATION: Kafka fan-out async; hybrid merge at read time for celebrities
  CONCENTRATION: Celebrity post → no fan-out amplification; pull at read time
```

---

## Staff Engineer Discussion Points

**"Why Redis Sorted Set and not a simple list?"**
Lists don't support re-ranking. When a post gets 10K likes an hour after publishing, its score should increase, moving it up in feeds that haven't been refreshed. With a sorted set, we can `ZADD` with the updated score and Redis repositions the post automatically. A list would require removing and re-inserting the post at the new position — much more complex. Also, `ZRANGEBYSCORE` with a cursor enables stable pagination even as new posts are added at the top.

**"How do you handle the fan-out for a user who posts right before they gain celebrity status?"**
The transition isn't instant. When a user crosses 10K followers: (1) a daily batch job identifies new celebrities and sets `is_celebrity=TRUE`, (2) the fan-out worker starts routing their new posts to `celebrity_posts` instead of per-follower feeds, (3) existing fan-out for their old posts (in pre-computed feeds) remains. During the transition window, some followers may have their feed switch from push to pull. This is acceptable — the transition is gradual and the user experience difference is minimal.

---

## Cheat Sheet Tie-in

```
CELEBRITY PROBLEM → HYBRID FAN-OUT:
  < 10K followers:  fan-out on write (push) → feed pre-computed
  ≥ 10K followers:  fan-out on read (pull)  → pulled and merged at feed load time
  At read time: merge pre-computed feed + celebrity posts → rank → paginate

REDIS SORTED SET FOR FEED:
  ZADD feed:{user_id} {score} {post_id}
  ZRANGEBYSCORE for cursor-based pagination
  ZREMRANGEBYRANK to cap feed at 1,000 posts per user

SCC LENS:
  STATE: PostgreSQL (posts), Cassandra (follows), Redis sorted sets (feed cache)
  COORDINATION: fan-out on Kafka; hybrid merge at read time
  CONCENTRATION: celebrity → no fan-out; pull prevents 100M write amplification
```

---

*Next: Case Study 13 — Chat System*
