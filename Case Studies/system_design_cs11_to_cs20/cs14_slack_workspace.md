# Case Study 14 — Slack-Like Workspace Messaging

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Workspace isolation, Channel threading, Search architecture.

---

## Business Context

A Slack-like system differs from general chat (Case 13) in key ways: messages belong to named channels within isolated workspaces, threads allow focused conversations within a channel, and powerful search across all workspace history is a core feature. Enterprise customers have strict data residency requirements (workspace data must stay in a specific region).

**Business goals driving architecture:**
- Workspace isolation: Tenant A cannot see Tenant B's messages
- Search: find messages matching text, author, date within workspace
- Threads: reply to a message without cluttering the main channel
- Retention policies: enterprise workspaces may require 7-year message retention
- Data residency: EU workspaces must be stored in EU; US in US

---

## Recognition Framework

### Signals from the Problem
```
- Multi-tenant (workspace isolation) → data isolation is a first-class concern
- Channel-based (not conversation-based) → different addressing model than 1:1 chat
- Full-text search → Elasticsearch or similar (Cassandra/PostgreSQL cannot do FTS efficiently)
- Threads → messages have parent-child relationships → additional complexity
- Retention policies vary per workspace → TTL must be configurable per tenant
- Enterprise: audit log, compliance export, eDiscovery → Case 10 architecture applies
- Read-heavy: channel history frequently browsed; search is frequent
```

### Key Difference from Case 13 (Chat)

```
Chat (Case 13):          Slack (Case 14):
  1:1 and group chat       Channel-based (public, private, DM)
  Contact list             Workspace directory
  Push for offline         Same + in-app notification center
  No search (or basic)     Full-text search is a core feature
  Simple retention         Configurable retention per workspace
  Social product           Enterprise product (SSO, SCIM, compliance)
  Message expiry optional  Compliance retention required
  No threading             Threading is first-class
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| Workspace-scoped data | All queries scoped by workspace_id; no cross-workspace queries |
| Elasticsearch | Full-text search across channel history; faceted by author/date/channel |
| Channel Fan-out (Kafka) | New message → all channel subscribers notified (not just 1:1) |
| Thread Reply Model | Parent message stores reply_count; replies stored with parent_id |
| Kafka → Elasticsearch Sync | Async indexing; search index eventually consistent with message store |

### Patterns Rejected
| Pattern | Reason Rejected |
|---|---|
| PostgreSQL full-text search (tsvector) | Does not scale to billions of messages; no semantic search; limited faceting |
| Shared DB for all workspaces | Compliance violation; noisy tenant; data residency impossible |
| Real-time search indexing (sync) | Indexing in write path adds latency; async is correct |

---

## Problem Statement

Workspace messaging system with channels, threading, and full-text search. Enterprise-grade: workspace isolation, configurable retention, compliance export.

---

## Functional Requirements
- Workspaces with isolated member directories
- Public and private channels; DMs between members
- Send messages with mentions (@user, #channel), reactions, file attachments
- Thread replies on any message
- Full-text search within workspace (by content, author, date, channel)
- Notification center (in-app + push + email digest)
- Retention policies per workspace (7-day to 10-year)
- SSO login (SAML/OIDC) for enterprise workspaces

## Non-Functional Requirements
- Message delivery: **< 100ms** for channel members (same as Case 13)
- Search latency: **< 2 seconds** for workspace-wide queries
- Search freshness: **< 30 seconds** (message indexed within 30s of sending)
- Scale per workspace: up to 100K members; 1M channels; 10B messages

---

## Capacity Estimation
```
Global scale:
  10M workspaces × avg 50 members = 500M users
  10B messages/day (across all workspaces)
  Messages per workspace: varies from 10/day (small) to 10M/day (large enterprise)

Search index:
  10B messages/day × 200 bytes = 2 TB/day new data to index
  90-day hot search index: ~180 TB
  Elasticsearch: 3-5× storage overhead (inverted index) → ~900 TB
  Partition: one Elasticsearch cluster per large workspace or shared per region
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Workspace Isolation | All data keyed by workspace_id; separate DBs for large enterprise |
| Cassandra (messages) | Same as Case 13; time-series; wide row per channel |
| Elasticsearch | Full-text search; faceted; relevance ranking |
| Kafka → ES Sync | Async message indexing; search is eventually consistent |
| Thread Model | parent_message_id links replies; channel shows parent + reply count |
| Redis Channel Membership | Fast lookup: who subscribes to channel; for fan-out |

---

## Data Model

```cql
-- Messages (Cassandra; same pattern as Case 13 but channel-scoped)
CREATE TABLE channel_messages (
  workspace_id     UUID,
  channel_id       UUID,
  seq_num          BIGINT,           -- per-channel sequence
  message_id       UUID,
  sender_id        TEXT,
  parent_message_id UUID,            -- null for top-level; set for thread replies
  message_type     TEXT,             -- TEXT / FILE / SYSTEM
  content          TEXT,
  reactions        MAP<TEXT, FROZEN<SET<TEXT>>>,  -- {"👍": {"u-123","u-456"}}
  reply_count      INT DEFAULT 0,    -- updated on thread reply
  edited_at        TIMESTAMP,
  deleted_at       TIMESTAMP,        -- soft delete; content replaced with "This message was deleted"
  sent_at          TIMESTAMP,
  PRIMARY KEY ((workspace_id, channel_id), seq_num)
) WITH CLUSTERING ORDER BY (seq_num DESC)
  AND default_time_to_live = 2592000;  -- default 30 days; overridden per workspace
```

```json
// Elasticsearch document (per message)
{
  "workspace_id": "ws-abc",
  "channel_id": "ch-xyz",
  "message_id": "msg-uuid",
  "sender_id": "u-123",
  "sender_name": "Alice",              // denormalized for display
  "content": "We need to review the payment gateway architecture",
  "content_analyzed": "review payment gateway architecture",  // stemmed/analyzed
  "sent_at": "2024-01-01T14:00:00Z",
  "has_attachment": false,
  "thread_reply_count": 3,
  "reactions_summary": ["👍", "🎉"],
  "is_thread_reply": false,
  "parent_message_id": null
}
// ES index: workspace_messages_{workspace_id} or shared with workspace_id as field
```

```sql
-- Workspace and channel metadata (PostgreSQL per region)
CREATE TABLE workspaces (
  workspace_id     UUID         PRIMARY KEY,
  name             VARCHAR(256) NOT NULL,
  plan             VARCHAR(32),          -- free / pro / enterprise
  region           VARCHAR(16),          -- us-east / eu-west / ap-south
  retention_days   INTEGER      DEFAULT 90,
  sso_provider     VARCHAR(64),          -- null for non-SSO workspaces
  created_at       TIMESTAMP    DEFAULT NOW()
);

CREATE TABLE channels (
  channel_id       UUID         PRIMARY KEY,
  workspace_id     UUID,
  name             VARCHAR(80),
  is_private       BOOLEAN      DEFAULT FALSE,
  topic            TEXT,
  member_count     INTEGER,
  created_at       TIMESTAMP    DEFAULT NOW()
);
```

---

## API Design

```
WebSocket (same protocol as Case 13; extended event types):
  { "type": "send_channel_message",
    "workspace_id": "ws-abc",
    "channel_id": "ch-xyz",
    "content": "Hello everyone!",
    "parent_message_id": null }   // set to reply in thread

  { "type": "add_reaction",
    "message_id": "msg-uuid",
    "emoji": "👍" }

REST API:
  GET /api/v1/workspaces/{ws_id}/channels/{ch_id}/messages?cursor=&limit=50&thread_of=
  GET /api/v1/workspaces/{ws_id}/search?q=payment+gateway&channel=ch-xyz&from=2024-01-01
  POST /api/v1/workspaces/{ws_id}/channels/{ch_id}/messages/{msg_id}/reactions

Search API response:
  {
    "results": [
      {
        "message_id": "msg-uuid",
        "content_highlight": "We need to review the <em>payment gateway</em> architecture",
        "sender": { "id": "u-123", "name": "Alice" },
        "channel": { "id": "ch-xyz", "name": "engineering" },
        "sent_at": "2024-01-01T14:00:00Z",
        "score": 0.87
      }
    ],
    "total": 234,
    "query_time_ms": 340
  }
```

---

## High-Level Architecture

```
Client → WebSocket → Channel Message Handler
  │
  ├── 1. Write to Cassandra (channel_messages; seq_num from Redis INCR)
  ├── 2. Fan-out to channel members (same as Case 13 routing)
  ├── 3. Async: PUBLISH to Kafka "channel-messages" topic
  │
  └── Kafka Consumer: Message Indexer
        Read from Kafka → normalize → index to Elasticsearch
        Batch: 1,000 messages per bulk index call
        Lag: < 30 seconds from message write to search availability

Search Service:
  POST /search → Elasticsearch query
    {
      "query": {
        "bool": {
          "must": [
            { "term":  { "workspace_id": "ws-abc" } },
            { "match": { "content_analyzed": "payment gateway" } }
          ],
          "filter": [
            { "range": { "sent_at": { "gte": "2024-01-01" } } }
          ]
        }
      },
      "highlight": { "fields": { "content": {} } }
    }

Workspace Isolation Strategy:
  Option A: Separate ES index per workspace
    Pro: data isolation; easy deletion (delete entire index)
    Con: ES shard overhead for small workspaces (10K shards for 10M workspaces)
    Use: large enterprise workspaces (> 100K messages)
  
  Option B: Shared ES index with workspace_id filter
    Pro: efficient for small workspaces; fewer shards
    Con: noisy neighbor; workspace_id filter on every query
    Use: small/free workspaces

  Hybrid: workspaces < 1M messages → shared index
          workspaces > 1M messages → dedicated index
```

---

## Detailed Components

### Thread Architecture
```
Threads in Slack: a message can have a "thread" of replies
  Top-level channel message: parent_message_id = null; shown in channel
  Thread reply: parent_message_id = {parent_msg_id}; shown in thread view only

  Thread view: GET /messages?thread_of={parent_msg_id}
    Returns all messages WHERE parent_message_id = ? AND channel_id = ?
    Ordered by sent_at ASC

  Channel view pagination:
    Returns only top-level messages (WHERE parent_message_id IS NULL)
    Shows reply_count on each message ("5 replies")
    Parent message's reply_count is incremented on each thread reply (Redis counter; sync to DB)

  Unread tracking:
    Per-user, per-channel: last_read_seq (for channel)
    Per-user, per-thread: last_read_thread_seq (for thread they've participated in)
    "New replies" badge: reply_count > user's last_read_thread_seq
```

### Retention Policy Enforcement
```
Enterprise requirement: workspace can set 90-day, 1-year, or 7-year retention.
Messages older than retention period must be deleted.

Implementation:
  1. Cassandra TTL: set default_time_to_live per workspace
     CREATE TABLE with workspace-specific TTL or per-row TTL override
  2. Elasticsearch: use ILM (Index Lifecycle Management) policy
     Delete index (or rollover to cold tier) after retention period
  3. S3 attachments: S3 Object Lifecycle rule: delete after retention period
     (unless legal hold applied — Case 10 pattern)
  4. PostgreSQL metadata: soft delete; purge after retention period
  
  Legal hold override:
    Specific messages (or entire conversations) can be placed on legal hold
    Legal hold exempts from retention policy deletion
    Same pattern as Case 10 (Audit Logging Service)
```

### Notification Center (In-App)
```
Unlike Case 13 (push only for offline), Slack uses a notification center:
  - @mentions: always notify (push + in-app badge)
  - Keyword alerts: user sets keywords; notified on match
  - Thread replies: notify all thread participants
  - DMs: always notify

In-app notification center:
  notifications:{user_id} → Redis sorted set (score = timestamp)
  Each notification: { type, from, channel, message_id, preview, timestamp }
  GET /api/v1/notifications?unread_only=true → last 100 notifications
  Mark read: update unread_count in Redis

Push notification (for mobile/desktop offline):
  Same as Case 11 (Notification Service)
  Payload includes workspace + channel context for deep linking
```

---

## Scaling Strategy
```
Phase 1 (small team, 1K workspaces):
  PostgreSQL for everything (messages, channels, workspaces)
  Basic full-text search with PostgreSQL tsvector
  No Elasticsearch (complexity not yet justified)

Phase 2 (10K workspaces, search demanded):
  Cassandra for messages (hot path)
  Elasticsearch for search (shared index; workspace_id filter)
  Kafka for async indexing

Phase 3 (100K workspaces, enterprise customers):
  Dedicated ES indices for large enterprise workspaces
  Data residency: regional deployments (US, EU, AP)
  SSO/SCIM integration (Okta, Azure AD)

Phase 4 (Slack scale, 10M workspaces):
  Workspace-to-shard mapping (consistent routing)
  Elasticsearch per region; large workspaces get dedicated clusters
  Cassandra clusters per region
  Global user directory (cross-workspace user lookup)
```

---

## Reliability + Security

```
Workspace data isolation:
  Every query MUST include workspace_id as partition key
  API layer: every request is scoped to authenticated workspace
  Elasticsearch: every query includes mandatory workspace_id filter
  Even super-admin queries must be scoped to workspace (audit log captures)

SSO + SCIM:
  SAML/OIDC: user authenticates with enterprise IdP (Okta, Azure AD)
  SCIM provisioning: users added/removed from workspace automatically
  JIT provisioning: first SSO login creates workspace account
  (Directly applies your ForgeRock IAM background)

Encryption:
  Messages encrypted at rest (Cassandra + ES disk encryption)
  Messages encrypted in transit (TLS)
  Enterprise: customer-managed encryption keys (BYOK); AWS KMS CMK per workspace
```

---

## Observability
```
Key metrics beyond Case 13:
  search_latency_p99              (target < 2s; alert > 5s)
  search_indexing_lag_seconds     (message write → ES availability; target < 30s)
  es_index_health by workspace    (green/yellow/red per dedicated index)
  thread_reply_latency            (reply_count update delay)
  workspace_message_volume        (top workspaces; noisy tenant detection)
```

---

## Chaos Testing
```
Experiment 1: Elasticsearch Cluster Unavailable
  Inject:    Stop Elasticsearch cluster
  Expected:  Message sending and receiving: unaffected (no ES in write path)
  Expected:  Search API: returns 503 with "search temporarily unavailable"
  Expected:  Kafka consumer lag grows (indexing paused); catches up on recovery
  Red flag:  Message delivery fails because ES is in critical path

Experiment 2: Large Workspace Message Blast
  Inject:    1M-member workspace: all admins send simultaneously
  Expected:  Fan-out handled by Kafka fan-out workers (not in WebSocket hot path)
  Expected:  Delivery latency within 1s for online members
  Expected:  Kafka consumer lag briefly spikes; recovers within 30s
  Red flag:  WebSocket servers OOM from trying to fan-out 1M messages synchronously

Experiment 3: Retention Policy Enforcement
  Inject:    Set workspace retention to 1 day; wait 24 hours
  Expected:  Messages older than 1 day deleted from Cassandra (TTL)
  Expected:  ES documents deleted by ILM policy
  Expected:  Legal hold messages NOT deleted despite retention policy
  Red flag:  Legal hold messages deleted (retention overrides legal hold)
```

---

## Monthly Cost Estimate
```
Scale: 1M active workspaces, 10B messages/day

Elasticsearch (search):
  30 nodes × r6g.2xlarge ($500/mo) [180 TB storage]   = $15,000/mo

Cassandra (messages):
  20 nodes × r6g.4xlarge ($800/mo)                    = $16,000/mo

Kafka (fan-out + indexing):
  6 × kafka.m5.2xlarge ($300/mo)                      = $1,800/mo

WebSocket Servers:
  500 × c5.2xlarge ($280/mo)                          = $140,000/mo

Redis (presence + channels + notifications):
  10 × cache.r6g.2xlarge ($500/mo)                    = $5,000/mo

PostgreSQL (metadata):
  db.r6g.4xlarge ($1,600/mo)                          = $1,600/mo
─────────────────────────────────────────────────────────────────
Total: ~$179,400/month
Dominant cost: WebSocket servers (78%) — same issue as Case 13
Elasticsearch is 8% of cost but provides the highest-value feature (search)
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → Cassandra: message content (authoritative; append-only; TTL-managed per workspace).
  → Elasticsearch: search index (derived from Cassandra; eventually consistent; expendable).
  → Redis: presence, channel membership cache, notification queues (ephemeral).
  → PostgreSQL: workspace/channel metadata (authoritative; small).

□ Consistency model?
  → Messages: strong (Cassandra quorum write; seq_num via Redis INCR).
  → Search index: eventual (Kafka → ES; < 30s lag).
  → Channel member list: eventual (Redis cache; 5-min TTL from PostgreSQL).

□ Failure mode?
  → Elasticsearch down: search returns 503; messaging unaffected.
  → Cassandra node down: RF=3 absorbs; no message loss.
  → Redis down: channel membership rebuilt from PostgreSQL; presence temporarily lost.

□ IAM TIE-IN:
  Enterprise SSO: SAML/OIDC integration (ForgeRock AM as IdP)
  SCIM provisioning: user lifecycle in workspace mirrors enterprise directory
  Fine-grained authz: OPA policy for "can user X post to private channel Y"
  Per-workspace customer-managed encryption keys (BYOK): KMS CMK integration
```

---

## Staff Engineer Discussion Points

**"How does search respect channel privacy?"**
Every Elasticsearch query MUST include the workspace_id AND a filter for channels the requesting user has access to: public channels + private channels where user is a member. The channel membership list is fetched from PostgreSQL/Redis as part of search authorization. This is enforced server-side — the client cannot bypass it. A search for "salary raise" in a private #executive channel by a non-member returns zero results (the channel is simply not in their accessible channel list).

**"How do you handle data residency for EU workspaces?"**
Regional deployment: EU workspaces are routed to EU Cassandra, EU Elasticsearch, EU Kafka, and EU WebSocket servers. The routing decision is made at the API gateway level based on workspace_id → region mapping (cached in Redis). EU user's WebSocket connection lands on an EU WebSocket server. No EU user data crosses to US infrastructure. This is the same pattern as multi-region database architecture — the workspace_id is the tenant isolation key.

---

## Cheat Sheet Tie-in

```
SLACK vs CHAT (Case 13) KEY DIFFERENCES:
  Channels (not conversations) → different addressing; public/private channel model
  Full-text search → Elasticsearch (not Cassandra; not PostgreSQL FTS)
  Threading → parent_message_id relationship; separate thread view
  Enterprise → SSO, SCIM, data residency, BYOK, retention policies
  Multi-tenant isolation → workspace_id on every query, table, and index

ELASTICSEARCH FOR SEARCH:
  Index async (Kafka → ES consumer): search never in write path
  Workspace isolation: per-workspace index (large) or shared with workspace_id filter (small)
  Query: bool must [workspace_id term + content match] + filter [date range, channel]
  ILM policy: auto-delete old indices to enforce retention

SCC LENS:
  STATE: Cassandra (messages/authoritative), ES (search/derived), Redis (presence/cache)
  COORDINATION: seq counter (Redis INCR per channel), Kafka fan-out + indexing
  CONCENTRATION: large workspace broadcast → Kafka fan-out workers handle; not in WS hot path

IAM PLATFORM CONNECTION:
  Enterprise workspaces = your primary persona:
  SAML SSO with ForgeRock as IdP (or ForgeRock as SP for customer IdPs)
  SCIM provisioning: sync workspace members from enterprise directory (OpenDJ/AD)
  Fine-grained authz: OPA/Cerbos for "can user X post to private channel Y"
  Audit log: every message, search, and admin action → Case 10 architecture
```

---

*Next: Case Study 15 — Collaborative Editor*
