# Case Study 13 — Chat System (WhatsApp/Slack Scale)

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: WebSocket connection management, Message ordering, Offline delivery.

---

## Business Context

A real-time chat system must deliver messages instantly when the recipient is online and reliably when they are offline. For a payment platform, chat enables user-to-merchant communication, support conversations, and transaction dispute resolution — all requiring a complete, ordered, tamper-evident message history.

The fundamental tension: WebSockets are stateful (server maintains connection state) but backends must be horizontally scalable (stateless). Routing a message to the correct WebSocket server instance is the core coordination problem.

**Business goals driving architecture:**
- Message delivery < 100ms when recipient is online
- 100% delivery guarantee (no message loss, even if recipient offline)
- Message ordering: messages in a conversation must arrive in send order
- Exactly-once: no duplicate messages on retry
- Read receipts: sender knows when message was read

---

## Recognition Framework

### Signals from the Problem
```
- Real-time delivery requires persistent connection (WebSocket; not HTTP polling)
- Server must maintain N×M connections (N users × M average sessions)
- Messages must be delivered in order within a conversation
- Offline recipients need messages queued until they reconnect
- Multiple devices per user: message appears on all of a user's devices
- Connection state is per-server: routing must direct to correct server
- High read/write: messages are written once, read many times (history)
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| WebSocket | Real-time bidirectional; one persistent connection per client |
| Presence Service (Redis) | Which WebSocket server holds a user's connection; route messages |
| Message Queue (per conversation) | Ordering guarantee; at-least-once delivery |
| Inbox (per user, per device) | Offline delivery; fan-out to all user devices |
| Sequence Numbers | Total ordering of messages within a conversation |
| Read Receipts | Sender acknowledgment of delivery + read status |

### Patterns Rejected
| Pattern | Reason Rejected |
|---|---|
| HTTP Long-polling | Higher latency; server resource waste; WebSocket is correct for real-time |
| Server-Sent Events (SSE) | One-way only; cannot receive messages (send requires separate HTTP POST) |
| Polling (client asks every N sec) | High latency; battery drain on mobile; poor UX |
| In-memory message store (no persistence) | Message lost if server restarts; must persist |
| Ordering by client timestamp | Clocks not synchronized; easy to manipulate; server-assigned is correct |

---

## Problem Statement

Real-time messaging system. Users send messages in 1:1 and group conversations. Messages delivered instantly if recipient is online; queued if offline. Complete message history accessible. Read receipts visible to sender.

---

## Functional Requirements
- 1:1 and group conversations (up to 256 members)
- Send/receive text, images, files
- Online/offline presence indicators
- Read receipts (delivered + read)
- Message history (last 90 days online; full history in cold storage)
- Multi-device: messages appear on all connected devices
- Push notification for offline messages (from Case 11)

## Non-Functional Requirements
- Delivery latency (online recipient): **< 100ms**
- Delivery latency (offline → push notification): **< 5 seconds**
- Message ordering: **guaranteed within a conversation**
- Message loss: **zero** (at-least-once with dedup)
- Scale: **2B users**, **100B messages/day**

---

## Capacity Estimation
```
Messages:
  100B messages/day = ~1.15M messages/sec
  Average message size: 200 bytes (text)
  Media messages: stored in S3; message contains URL only

WebSocket connections:
  Daily active users: 1B
  Average session: 3 hours
  Concurrent connections: 1B × (3/24) = 125M concurrent WebSocket connections
  Each connection: ~2 KB state → 125M × 2 KB = 250 GB memory fleet-wide
  1 WebSocket server handles 50K connections → 2,500 WebSocket servers

Message storage:
  1.15M messages/sec × 200 bytes = 230 MB/sec
  90-day hot storage: 230 MB/sec × 90×86400 = ~1.8 PB
  Use Cassandra (time-series, wide row per conversation)
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| WebSocket Server | Persistent connection for real-time delivery |
| Redis Presence Map | user_id → WebSocket server ID; enables message routing |
| Cassandra Message Store | Time-series; wide row per conversation; efficient range scans |
| Sequence Number (per conversation) | Total ordering; client uses to detect gaps |
| Inbox Queue (per device) | Offline delivery; per-device queue in Redis/Cassandra |
| Push Notification Fallback | Offline user → push notification → trigger app open → sync messages |

---

## Data Model

```sql
-- Conversation metadata (PostgreSQL; small; frequently updated)
CREATE TABLE conversations (
  conversation_id  UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
  type             VARCHAR(16)  NOT NULL,  -- ONE_TO_ONE / GROUP
  name             VARCHAR(128),           -- null for 1:1
  created_at       TIMESTAMP    DEFAULT NOW(),
  last_message_at  TIMESTAMP,
  last_seq_num     BIGINT       DEFAULT 0
);

CREATE TABLE conversation_members (
  conversation_id  UUID,
  user_id          VARCHAR(64),
  role             VARCHAR(16)  DEFAULT 'MEMBER',  -- MEMBER / ADMIN
  joined_at        TIMESTAMP    DEFAULT NOW(),
  last_read_seq    BIGINT       DEFAULT 0,         -- for read receipts
  PRIMARY KEY (conversation_id, user_id)
);
```

```cql
-- Messages in Cassandra (time-series; wide row per conversation)
CREATE TABLE messages (
  conversation_id  UUID,
  seq_num          BIGINT,           -- monotonically increasing; server-assigned
  message_id       UUID,             -- globally unique; for dedup
  sender_id        TEXT,
  message_type     TEXT,             -- TEXT / IMAGE / FILE / SYSTEM
  content          TEXT,             -- text or S3 URL for media
  sent_at          TIMESTAMP,
  PRIMARY KEY (conversation_id, seq_num)
) WITH CLUSTERING ORDER BY (seq_num ASC)
  AND default_time_to_live = 7776000;  -- 90 days (seconds)

-- Partition key: conversation_id → all messages for a conversation together
-- Clustering key: seq_num → messages returned in order; efficient range scan
-- Range query: SELECT * FROM messages WHERE conversation_id = ? AND seq_num > ?
```

```
Redis keys:
  presence:{user_id}           → ws_server_id (TTL: 90s; heartbeat refreshes)
  inbox:{user_id}:{device_id}  → list of undelivered message_ids (for offline users)
  seq:{conversation_id}        → current sequence number (INCR for atomic increment)
  typing:{conversation_id}:{user_id} → "1" (TTL: 5s; typing indicator)
```

---

## API Design

```
WebSocket connection:
  wss://chat.platform.com/ws
  On connect: authenticate (JWT token in first message or query param)
  Protocol: JSON-over-WebSocket (or binary: Protocol Buffers for efficiency)

WebSocket message types:
  Client → Server:
    { "type": "send_message",
      "conversation_id": "conv-abc",
      "idempotency_key": "client-generated-uuid",
      "message_type": "TEXT",
      "content": "Hello!" }

    { "type": "mark_read",
      "conversation_id": "conv-abc",
      "seq_num": 42 }

    { "type": "typing_start", "conversation_id": "conv-abc" }
    { "type": "heartbeat" }

  Server → Client:
    { "type": "new_message",
      "conversation_id": "conv-abc",
      "seq_num": 43,
      "message_id": "msg-uuid",
      "sender_id": "u-789",
      "content": "Hello!",
      "sent_at": "2024-01-01T14:00:01Z" }

    { "type": "message_ack",
      "idempotency_key": "client-uuid",
      "message_id": "msg-uuid",
      "seq_num": 43 }  // confirms message was stored and assigned seq_num

    { "type": "read_receipt",
      "conversation_id": "conv-abc",
      "user_id": "u-789",
      "seq_num": 43 }

    { "type": "typing_indicator",
      "conversation_id": "conv-abc",
      "user_id": "u-789",
      "is_typing": true }

REST API (for history and conversation management):
  GET /api/v1/conversations/{id}/messages?from_seq=&limit=50
  GET /api/v1/conversations (list user's conversations)
  POST /api/v1/conversations (create new conversation)
  POST /api/v1/conversations/{id}/members (add member to group)
```

---

## High-Level Architecture

```
Client (Mobile App / Browser)
  │
  └── WebSocket connection to WebSocket Server (load-balanced; sticky session)
        │
        ├── Connection map: user_id → this server_id → SET presence:{user_id} {server_id} EX 90

WebSocket Server Cluster (2,500 servers; each handles 50K connections)
  │
  ├── On receive "send_message" from client:
  │     1. Assign seq_num: INCR seq:{conversation_id} (atomic; Redis)
  │     2. Write message to Cassandra (with assigned seq_num)
  │     3. ACK client: send "message_ack" with seq_num
  │     4. Fan-out to all conversation members:
  │          For each member:
  │            GET presence:{member_user_id}
  │            → If has presence: forward to their WebSocket server
  │            → If no presence (offline): queue in inbox:{member_user_id}:{device_id}
  │                                        + trigger push notification (Case 11)
  │
  ├── Message routing between WebSocket servers:
  │     Redis Pub/Sub OR internal gRPC
  │     WS Server A receives message for user on WS Server B:
  │       → Publish to Redis channel ws:{server_b_id}
  │       → Server B receives and pushes to client's WebSocket
  │
  └── Heartbeat: client sends heartbeat every 30s
        → Server refreshes presence:{user_id} TTL (keeps connection alive in presence map)

On app open (reconnect after offline period):
  Client → GET /api/v1/conversations/{id}/messages?from_seq={last_seen_seq}
  → Returns all missed messages
  → Client processes inbox:{user_id}:{device_id} (pending messages)
```

---

## Detailed Components

### Message Sequencing (Ordering Guarantee)
```java
// WebSocket server receives "send_message" from client
public MessageAck handleSendMessage(String conversationId, String content,
                                    String idempotencyKey) {
    // Step 1: Assign sequence number atomically
    long seqNum = redis.incr("seq:" + conversationId);

    // Step 2: Generate globally unique message ID
    String messageId = UUID.randomUUID().toString();

    // Step 3: Persist to Cassandra
    cassandraRepo.insertMessage(conversationId, seqNum, messageId,
                                currentUserId, content, Instant.now());

    // Step 4: ACK sender immediately (before fan-out completes)
    // Client knows message was durably stored with this seq_num
    websocket.send(MessageAck.of(idempotencyKey, messageId, seqNum));

    // Step 5: Async fan-out to recipients
    fanOutWorker.dispatch(conversationId, seqNum, messageId, content);

    return MessageAck.of(idempotencyKey, messageId, seqNum);
}

// Why server-assigned seq_num?
//   Client clocks: not synchronized; skewed by 100ms to 1 second typically
//   Client seq: could be manipulated (set seq_num to 1 to inject at start of history)
//   Server seq: INCR is atomic; Redis guarantees sequential; monotonically increasing
//   Gap detection: client receives seq 5 after seq 3 → knows seq 4 is missing → request resync
```

### Online/Offline Routing
```
Message fan-out to conversation members:

for member_id in conversation.member_ids:
  server_id = redis.get("presence:" + member_id)  # e.g., "ws-server-47"

  if server_id is None:
    # User is offline
    # Option 1: Queue in inbox for when user reconnects
    redis.lpush(f"inbox:{member_id}:{device_id}", message_json)
    redis.expire(f"inbox:{member_id}:{device_id}", 7 * 86400)  # 7 days

    # Option 2: Trigger push notification (Case 11)
    notif_service.send_push(member_id,
                            "New message from " + sender_name,
                            notification_type="CHAT_MESSAGE")
  else:
    # User is online on server_id
    if server_id == MY_SERVER_ID:
      # User is on this server; direct WebSocket push
      websocket_connections[member_id].send(new_message_event)
    else:
      # User is on a different server; route via Redis Pub/Sub
      redis.publish(f"ws:{server_id}", json.dumps({
          "user_id": member_id,
          "event": new_message_event
      }))
      # Other server subscribes to ws:{its_server_id} → delivers to client
```

### Multi-Device Sync
```
One user has 3 devices: phone, tablet, desktop.
Message sent to this user:
  1. Fan-out to each device_id registered for user_id
     GET user_devices:{user_id} → ["device-phone", "device-tablet", "device-desktop"]
  2. For each device:
     IF device is connected (has presence): send via WebSocket
     IF device is offline:   queue in inbox:{user_id}:{device_id}
                              + push notification to that device token

On device reconnect:
  Device connects WebSocket → authenticates → sends "sync" message
  Server: GET inbox:{user_id}:{device_id} → send all queued messages
  Server: GET /messages?from_seq={last_seen_seq} → send any missed messages

Cross-device read receipt:
  User reads message on phone:
    → POST mark_read(conversation_id, seq_num) via WebSocket on phone
    → Server: UPDATE conversation_members SET last_read_seq = seq_num WHERE user_id = ?
    → Server: fan-out read_receipt event to all user's OTHER devices
              (so tablet and desktop also show "read" status)
```

---

## Scaling Strategy
```
Phase 1 (0 → 1M users):
  Single WebSocket server cluster
  PostgreSQL for messages
  Redis for presence

Phase 2 (1M → 50M users):
  Multiple WebSocket servers; Redis Pub/Sub for cross-server routing
  Cassandra for messages (PostgreSQL can't handle 50M+ message time-series)
  Redis Cluster for presence

Phase 3 (50M → 500M users):
  WebSocket servers behind L4 LB with consistent hashing per user_id
    (same user always routes to same server — reduces cross-server routing)
  Dedicated presence service (separate from WebSocket servers)
  Separate fan-out service (async; Kafka-based)

Phase 4 (global, 2B users):
  Regional WebSocket clusters
  Global message routing (cross-region if users are in different regions)
  Cassandra global replication
  Regional Kafka clusters with cross-region replication for global presence
```

---

## Reliability Strategy
- **At-least-once delivery**: ACK client only after Cassandra write; fan-out can retry
- **Inbox queue**: offline messages persist in Redis + push notification; sync on reconnect
- **Sequence gap recovery**: client detects gap (seq 4 missing after seq 3 and 5) → REST API resync
- **WebSocket reconnect**: client auto-reconnects with exponential backoff; server restores presence on reconnect
- **Cassandra replication**: RF=3; no message loss on node failure

---

## Security Considerations
- **End-to-end encryption (E2EE)**: message encrypted client-side; server stores ciphertext; server cannot read messages
- **Authentication**: JWT validated on WebSocket connect; re-validated on every sensitive operation
- **Group conversation access**: server validates membership before fan-out; client cannot inject messages to conversations they're not in
- **Message deletion**: marks deleted in DB but encrypted content still stored (E2EE means server can't verify deletion); client hides from UI
- **Rate limiting**: max 10 messages/sec per user (prevent spam via WebSocket)

---

## Observability
```
Metrics:
  ws_connections_active           (gauge; per server; target < 50K)
  message_delivery_latency_p99    (time from send to recipient receive; target < 100ms)
  offline_message_queue_depth     (inbox depth per user; alert if > 1K)
  presence_map_size               (total online users; business metric)
  fan_out_latency_p99             (time to route message to all recipients)
  seq_gap_recovery_rate           (how often clients need to resync)

Alerts:
  ws_connections_active > 45K     → server approaching capacity; scale
  message_delivery_latency > 500ms → routing bottleneck; Redis Pub/Sub lag
  offline_queue_depth > 10K users → push notification service down
  seq_gap_rate > 1%               → message ordering issue
```

---

## Chaos Testing
```
Experiment 1: WebSocket Server Crash
  Inject:    Kill one WebSocket server (50K connections drop)
  Expected:  Clients auto-reconnect to another server (LB redistributes)
  Expected:  During reconnect: clients receive missed messages via inbox sync
  Expected:  Presence map updated: old server_id replaced with new server_id
  Red flag:  Messages sent during reconnect window are permanently lost

Experiment 2: Message Ordering Under Concurrency
  Inject:    5 clients send messages to same conversation simultaneously
  Expected:  All 5 messages assigned unique, sequential seq_nums (INCR is atomic)
  Expected:  All recipients receive messages in seq_num order
  Expected:  No two messages have same seq_num
  Red flag:  Duplicate seq_nums; race condition in seq counter

Experiment 3: Offline User Message Queue
  Inject:    User A offline for 1 hour; User B sends 50 messages to A
  Expected:  50 messages queued in inbox:{A}:{device}
  Expected:  Push notifications sent for first N messages (throttled to avoid spam)
  Expected:  When A opens app: all 50 messages delivered in order
  Red flag:  Messages lost after 1 hour offline (inbox TTL too short)
```

---

## Monthly Cost Estimate
```
Scale assumption: 100M daily active users, 10B messages/day

WebSocket Servers:
  250 × c5.2xlarge ($280/mo) [50K connections each]    = $70,000/mo

Cassandra (message store):
  20 nodes × r6g.4xlarge ($800/mo)                     = $16,000/mo

Redis Cluster (presence + inbox + seq counters):
  10 × cache.r6g.xlarge ($260/mo)                      = $2,600/mo

Kafka (fan-out):
  6 × kafka.m5.xlarge ($250/mo)                        = $1,500/mo

S3 (media storage):
  ~10 TB/month media × $0.023/GB                       = $235/mo

PostgreSQL (conversation metadata):
  db.r6g.2xlarge ($500/mo)                             = $500/mo
─────────────────────────────────────────────────────────────────
Total: ~$90,835/month (~$1.09M/year)
Cost driver: WebSocket servers (77%) — connection state is memory-intensive
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → Cassandra: message content (ordered by conversation + seq_num; authoritative).
  → Redis: presence (ephemeral; TTL-based), inbox queue (7-day TTL), seq counters.
  → Redis is NOT authoritative for messages; Cassandra is.

□ Consistency model?
  → Message storage: strong (Cassandra RF=3, quorum writes).
  → Fan-out: async (eventual; < 100ms for online, < 5s for offline via push).
  → Presence: eventual (TTL-based; up to 90s stale after disconnect).

□ Failure mode?
  → WebSocket server crash: clients reconnect; inbox sync catches up.
  → Redis presence loss: presence unavailable; all fan-outs go to inbox (offline path).
  → Cassandra node down: RF=3 absorbs; reads continue.
  → Seq counter lost (Redis crash): reset to MAX(seq_num) from Cassandra + 1.

□ SCC LENS:
  STATE: Cassandra (messages), Redis (presence + inbox), PostgreSQL (conversation metadata)
  COORDINATION: seq counter (Redis INCR) — serializes message ordering across all servers
  CONCENTRATION: popular group conversation — all members' servers must route; fan-out scales with member count
```

---

*Next: Case Study 14 — Slack-Like Workspace Messaging*
