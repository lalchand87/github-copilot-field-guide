# Case Study 15 — Collaborative Editor (Google Docs-Like)

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Operational Transformation vs CRDT, Conflict Resolution, Cursor Presence.

---

## Business Context

A collaborative editor allows multiple users to edit the same document simultaneously. The fundamental challenge: two users type at the same time → both changes must be preserved, ordered, and converge to the same document state on all clients. This is a distributed systems problem disguised as a UI feature.

For a fintech context: collaborative SLAs/contracts, shared compliance documents, audit templates, or collaborative financial models.

**Business goals driving architecture:**
- Edits feel instant (applied locally before server confirmation)
- Conflicts auto-resolved (no "you can't edit while someone else is editing")
- All clients converge to identical document state
- Complete version history (who changed what, when)
- Offline editing: changes sync when reconnected

---

## Recognition Framework

### Signals from the Problem
```
- Multiple concurrent writers to same data → conflict is expected, not exceptional
- Clients need immediate feedback (optimistic UI) → apply locally before server ack
- Network latency is 20–200ms → cannot wait for server before applying local edits
- Clients may go offline → edits must be buffered and synced later
- Every edit must be preserved (no "last write wins") → requires operation-based conflict resolution
- Version history → append-only operation log
```

### OT vs CRDT — The Core Decision

```
Operational Transformation (OT):
  Model:  Operations are transformed against each other before application
  How:    Server serializes all operations; clients send ops to server;
          server transforms concurrent ops and broadcasts
  Pros:   Well-understood; Google Docs uses OT; space-efficient
  Cons:   Requires central server to serialize operations; hard to implement correctly;
          complex transformation functions; not peer-to-peer

CRDT (Conflict-free Replicated Data Type):
  Model:  Data structure mathematically guarantees convergence without coordination
  How:    Operations are designed so that any order of application yields same result
          (commutativity + idempotency)
  Types:  RGA (Replicated Growable Array) for text; LWW-Register for key-value
  Pros:   Peer-to-peer capable; no server serialization required; simpler failure modes
  Cons:   Higher memory overhead (tombstones for deleted characters);
          more complex merge logic for some data types

For a typical collaborative doc editor (not peer-to-peer):
  OT: used by Google Docs, Etherpad — server-centric; works well with central authority
  CRDT: used by Figma, Notion — works well for offline-first, P2P scenarios

Choice depends on:
  Need offline-first + P2P?     → CRDT (Automerge, Yjs)
  Central server is acceptable? → OT (simpler; Google's approach)
  Most SaaS collaborative tools: OT with central server
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| OT (Operational Transformation) | Server-centric; simpler conflict resolution; established approach |
| Optimistic Concurrency | Client applies edit locally immediately; server confirms async |
| Operation Log (append-only) | Complete history; replay for version control; undo/redo |
| WebSocket | Real-time bidirectional; same as Case 13 |
| Redis (operation buffer) | Buffer pending operations; order them before DB persistence |
| Cursor Presence | Show where other editors are in real-time (ephemeral; TTL-based) |

---

## Problem Statement

Real-time collaborative text document editor. Multiple users edit simultaneously. Changes applied instantly (optimistic). Conflicts auto-resolved. Complete version history. Offline editing syncs on reconnect.

---

## Functional Requirements
- Create, read, update, delete documents
- Concurrent editing by multiple users (real-time)
- Show other users' cursors and selections in real-time
- Version history: view document at any point in time
- Comments and suggestions (non-destructive annotations)
- Offline support: edit disconnected; sync on reconnect
- Permissions: view, comment, edit, admin per document

## Non-Functional Requirements
- Local edit latency: **< 10ms** (applied locally immediately, before server confirmation)
- Remote edit latency (from commit to appearing on other screens): **< 200ms**
- Convergence: **all clients reach same state** after network partition resolves
- Document size: up to **10 MB** (text + structure; not media)

---

## Capacity Estimation
```
Documents: 100M documents; active editing: 1M concurrent
Concurrent editors per document: avg 2-3; peak 50
Operations per document per second: avg 10 ops/sec (fast typist: ~5 chars/sec)
Total operations/sec: 1M active docs × 3 editors × 10 ops = 30M ops/sec
  → This is peak; sustained lower; buffer in Kafka

Operation size: ~50 bytes (type, position, character, revision)
Operation log storage: 30M ops/sec × 50 bytes = 1.5 GB/sec
  → Compressed, batched: ~150 MB/sec
  → 90-day retention: ~1.2 TB (compressed)

Document snapshots (for fast load, not full replay):
  Every 100 operations: create snapshot
  Snapshot size: avg 20 KB per document
  100M documents × 20 KB = 2 TB snapshots
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Operational Transformation (OT) | Conflict resolution for concurrent text edits |
| Optimistic UI | Apply locally immediately; server corrects if needed |
| Append-Only Operation Log | Version history; undo/redo; replay |
| WebSocket | Real-time op broadcast; cursor positions |
| Periodic Snapshots | Fast document load (not full replay from op 1) |
| Redis Presence | Cursor positions; active editors (ephemeral; TTL 5s) |

---

## Data Model

```sql
-- Documents (PostgreSQL)
CREATE TABLE documents (
  doc_id          UUID         PRIMARY KEY,
  title           VARCHAR(256),
  owner_id        VARCHAR(64),
  current_rev     BIGINT       DEFAULT 0,  -- latest committed revision
  snapshot_rev    BIGINT       DEFAULT 0,  -- revision of last snapshot
  content_type    VARCHAR(32)  DEFAULT 'TEXT',
  created_at      TIMESTAMP    DEFAULT NOW(),
  updated_at      TIMESTAMP    DEFAULT NOW()
);

-- Operation log (Cassandra — append-only; time-series)
CREATE TABLE doc_operations (
  doc_id          UUID,
  revision        BIGINT,         -- server-assigned; sequential per document
  client_id       TEXT,           -- which client submitted
  user_id         TEXT,
  op_type         TEXT,           -- INSERT / DELETE / RETAIN / FORMAT
  position        INT,            -- character index in document
  content         TEXT,           -- for INSERT: character(s) to insert
  length          INT,            -- for DELETE/RETAIN: number of chars affected
  attributes      MAP<TEXT,TEXT>, -- for FORMAT: bold, italic, color
  created_at      TIMESTAMP,
  PRIMARY KEY (doc_id, revision)
) WITH CLUSTERING ORDER BY (revision ASC);

-- Document snapshots (S3 + metadata in PostgreSQL)
-- Snapshot every 100 operations; stored as JSON in S3
-- Load: fetch latest snapshot + replay operations since snapshot_rev
CREATE TABLE doc_snapshots (
  doc_id          UUID,
  snapshot_rev    BIGINT,
  s3_key          TEXT,           -- s3://docs-bucket/{doc_id}/snapshot-{rev}.json
  size_bytes      INT,
  created_at      TIMESTAMP       DEFAULT NOW(),
  PRIMARY KEY (doc_id, snapshot_rev)
);
```

```
Redis keys (ephemeral, real-time state):
  doc_rev:{doc_id}           → current server revision (INCR; atomic)
  cursor:{doc_id}:{user_id}  → JSON {position, selection_start, selection_end, color}
                               TTL: 5 seconds (refreshed on every cursor move)
  active_editors:{doc_id}    → Set of user_ids currently editing (TTL: 10s per member)
  pending_ops:{doc_id}       → List of operations waiting to be committed in order
```

---

## API Design

```
WebSocket protocol (same connection infrastructure as Case 13):

Client → Server (op submission):
  {
    "type": "submit_op",
    "doc_id": "doc-uuid",
    "client_revision": 47,     // client's current revision when op was created
    "operations": [             // Quill Delta format (OT operations)
      { "retain": 15 },         // keep first 15 characters
      { "insert": "Hello" },    // insert "Hello" at position 15
      { "retain": 100 }         // keep remaining characters
    ]
  }

Server → Client (committed op broadcast):
  {
    "type": "op_committed",
    "doc_id": "doc-uuid",
    "server_revision": 48,     // server-assigned revision
    "transformed_ops": [...],  // ops transformed against concurrent ops
    "user_id": "u-123",
    "username": "Alice"
  }

Client → Server (cursor update):
  { "type": "cursor_update",
    "doc_id": "doc-uuid",
    "position": 150,
    "selection_start": 148,
    "selection_end": 152 }

Server → Client (cursor broadcast):
  { "type": "cursor_positions",
    "doc_id": "doc-uuid",
    "cursors": [
      { "user_id": "u-456", "username": "Bob", "position": 200,
        "color": "#FF6B6B", "selection_start": 198, "selection_end": 205 }
    ]
  }

REST API:
  GET /api/v1/documents/{doc_id}         → load document (snapshot + ops since snapshot)
  GET /api/v1/documents/{doc_id}/history → operation history for version timeline
  POST /api/v1/documents                 → create document
  GET /api/v1/documents/{doc_id}/version/{revision} → document at specific revision
```

---

## High-Level Architecture

```
Client (Browser/Desktop App)
  │
  ├── Local document state: in-memory (Quill / CodeMirror / custom)
  ├── On user types:
  │     1. Apply operation locally IMMEDIATELY (optimistic)
  │     2. Send op to server via WebSocket (with client_revision)
  │     3. Wait for server ACK with server_revision
  │
  └── On receive op from server:
        If op is from self: update local revision tracking; no reapplication
        If op is from other user:
          Transform received op against any unacknowledged local ops (OT)
          Apply transformed op to local document
          → Document state converges

WebSocket Server (Doc Session Server):
  One WebSocket server instance manages all editors of one document
  (or a doc is pinned to one server via consistent hashing)
  
  On receive "submit_op" from client:
    1. Get current server revision: GET doc_rev:{doc_id}
    2. If client_revision < server_revision:
         Transform client's ops against all ops since client_revision (OT transform)
    3. INCR doc_rev:{doc_id} → assign new server_revision
    4. Persist operation to Cassandra (with server_revision)
    5. Broadcast transformed op to all other editors of this doc
    6. ACK submitting client: { server_revision: 48 }
    7. Async: if revision % 100 == 0 → create snapshot

Snapshot Creation (async, background):
  Apply all ops from last snapshot to current → serialize → upload to S3
  Update doc_snapshots table
  Old operations before snapshot_rev can be archived to cold storage
```

---

## Detailed Components

### Operational Transformation (Simplified Text Example)
```
Scenario:
  Document: "Hello World"  (revision 5)
  
  User A (at revision 5): types "!" at position 11 → op_a: INSERT(11, "!")
  User B (at revision 5): deletes "World" (pos 6-10) → op_b: DELETE(6, 5)
  
  Both submit at same time. Server receives op_a first (revision 6).
  Server must transform op_b against op_a before applying.

OT Transform function: transform(op_b, op_a) → op_b'
  op_b: DELETE(6, 5)  -- delete 5 chars starting at position 6
  op_a: INSERT(11, "!")  -- insert at position 11 (AFTER position 6)
  
  Transform rule: op_a inserted AFTER op_b's range → op_b is unchanged
  op_b' = op_b = DELETE(6, 5)  -- no adjustment needed
  
  Result after both ops applied: "Hello!"
  (Deletes "World", keeps "!" which was inserted after the deletion range)

Harder case:
  op_a: INSERT(6, "Beautiful ")  -- insert before "World"
  op_b: DELETE(6, 5)             -- delete "World" (which is now at position 16)
  
  Transform op_b against op_a:
    op_a inserted 10 chars before op_b's target position
    op_b' = DELETE(16, 5)  -- position shifts by +10
  
  Result: "Hello Beautiful "
  (Correct: both users' intent preserved; just at adjusted positions)

Real implementation:
  Quill Delta format: operations are INSERT/DELETE/RETAIN sequences
  Transform function must handle all combinations
  Libraries: ot.js, ShareDB, Automerge
```

### Client Offline Sync
```
Scenario: User edits document while offline for 30 minutes
  Client: buffers all operations locally (IndexedDB)
  On reconnect:
    1. Client sends: { "type": "reconnect", "doc_id": "x", "client_revision": 30 }
    2. Server: fetches all ops from revision 30 to current (server_revision=150)
    3. Server sends: all 120 ops to client
    4. Client: transforms buffered local ops against received server ops (OT)
    5. Client: sends transformed local ops to server
    6. Server: applies them; document converges

  Result: both online edits (revisions 31-150) AND offline edits are preserved
  
  If conflict is irresolvable (rare): server's ops win; client gets a "diff" view
  showing what changed remotely that conflicts with their edit
```

### Version History and Undo
```
Version history:
  Every operation is stored in Cassandra with user_id + timestamp
  "View at version N": fetch snapshot before N + replay ops from snapshot_rev to N
  "Restore to version N": create new operation that reverses all ops from N to current
    This preserves the history (audit trail); restoration is a new operation, not a delete

Undo/Redo (per client, not global):
  Client maintains local undo stack of submitted operations
  Undo: sends "inverse operation" to server (INSERT ↔ DELETE inverse)
  This is broadcast to all; all editors see the undo

Named versions / version checkpoint:
  User explicitly saves "Version 1.0 - For Review"
  Creates a snapshot with a label; linked from doc_snapshots
  Anyone can access the labeled version via URL
```

---

## Scaling Strategy
```
Phase 1 (small team, 1K concurrent docs):
  Single WebSocket server handles all docs
  PostgreSQL for operations (small scale; fine)
  No snapshots (full replay is fast at low revision count)

Phase 2 (10K concurrent docs):
  Multiple WS servers; document-to-server routing (consistent hashing on doc_id)
  Cassandra for operation log (replaces PostgreSQL)
  Periodic snapshots (every 100 ops; prevents slow load for long-lived docs)

Phase 3 (1M concurrent docs):
  Horizontal WS server scaling
  Cassandra clusters per region
  CDN for initial document load (snapshot from S3)
  Redis Cluster for presence

Phase 4 (Google Docs scale):
  CRDT migration consideration: CRDT eliminates need for central server per document
  (peer-to-peer convergence; no single point of failure per document)
  Yjs library: CRDT-based; used by linear.app, Tiptap
  Trade: higher memory per document vs no central server required
```

---

## Reliability Strategy
- **Operation persistence**: Cassandra RF=3 before ACK to client; no op loss
- **Snapshot recovery**: if WebSocket server crashes mid-session, clients reconnect and resync from last known revision
- **Client local buffer**: unconfirmed ops stored client-side; retransmitted on reconnect
- **Document recovery**: worst case — reload from latest snapshot + replay Cassandra ops

---

## Security Considerations
- **Document access control**: every WebSocket subscription and REST call validates user has access to doc_id
- **Operation injection**: server validates each op is well-formed before applying; malformed ops rejected
- **Content scanning**: document content scanned for malware/PII if shared externally
- **Audit log**: every edit, comment, share logged to Case 10 audit service

---

## Observability
```
Metrics:
  op_commit_latency_p99         (time from submit_op to client receiving op_committed)
  ot_transform_latency_ms       (time to transform and apply concurrent ops)
  concurrent_editors_per_doc    (gauge; alert if > 50 on single doc)
  snapshot_creation_lag         (how far behind snapshots are from current revision)
  websocket_reconnect_rate      (high rate = network instability)
  offline_sync_op_count         (how many ops need syncing on reconnect)
```

---

## Chaos Testing
```
Experiment 1: Concurrent Edit Conflict Resolution
  Inject:    Two clients simultaneously insert at same position in document
  Expected:  Both inserts preserved (OT transforms positions)
  Expected:  Both clients converge to same document state
  Expected:  No character loss; correct ordering after OT
  Red flag:  One edit lost (last-write-wins instead of OT)

Experiment 2: Network Partition and Reconnect
  Inject:    Disconnect client B for 5 minutes while client A edits heavily
  Expected:  Client B's buffered edits preserved in IndexedDB
  Expected:  On reconnect: server sends 5 min of A's ops to B; B transforms and sends its ops
  Expected:  Both clients converge to same document state
  Red flag:  Client B's offline edits lost on reconnect

Experiment 3: WebSocket Server Crash During Edit Session
  Inject:    Kill WebSocket server; 100 clients are editing documents on it
  Expected:  Clients auto-reconnect to another server
  Expected:  All ops committed before crash: in Cassandra (persisted before ACK)
  Expected:  Pending ops (submitted but not yet ACKed): client retransmits on reconnect
  Red flag:  Ops lost that were committed to Cassandra but not yet ACKed to clients
             (these should be safe; client retransmit + server dedup by client_id+revision)
```

---

## Monthly Cost Estimate
```
Scale: 10M documents, 1M concurrent editors

WebSocket Doc Servers:
  500 × c5.2xlarge ($280/mo) [2K concurrent docs each]  = $140,000/mo

Cassandra (op log):
  15 nodes × r6g.4xlarge ($800/mo)                      = $12,000/mo

S3 (snapshots):
  100M docs × 20 KB = 2 TB × $0.023/GB                  = $47/mo

Redis (presence + revision counters):
  5 × cache.r6g.xlarge ($260/mo)                        = $1,300/mo

PostgreSQL (document metadata):
  db.r6g.2xlarge ($500/mo)                              = $500/mo
─────────────────────────────────────────────────────────────────
Total: ~$153,847/month
WebSocket servers dominate (91%) — high connection count is the cost driver
CRDT approach (Yjs) could reduce server requirement significantly (peer-to-peer sync)
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → Cassandra: operation log (authoritative; append-only).
  → S3 snapshots: derived from Cassandra (accelerates load; not authoritative alone).
  → Redis: real-time state (cursors, active editors, revision counter) — ephemeral.
  → If Redis is lost: revision counter rebuilt from MAX(revision) in Cassandra.

□ Consistency model?
  → Operations: serialized by server (revision counter → strict ordering).
  → OT: server transforms concurrent ops → all clients converge to same state.
  → Cursor positions: eventual (Redis TTL 5s; acceptable for UI-only data).

□ Failure mode?
  → WS server crash: clients reconnect; unconfirmed ops retransmitted; converge.
  → Cassandra node down: RF=3; operations continue; no data loss.
  → Redis down: revision counter rebuilt from Cassandra; presence lost (acceptable).

□ OT vs CRDT DECISION:
  OT: server-centric; complex transform functions; established (Google Docs)
  CRDT: no central coordination; offline-first; more memory per document (tombstones)
  Choose OT when: server is always available; text editing is primary; team is familiar
  Choose CRDT when: offline-first; P2P; Figma-style multi-type collaboration

□ SCC LENS:
  STATE: Cassandra (op log/authoritative), S3 (snapshots/derived), Redis (cursors/ephemeral)
  COORDINATION: revision INCR (Redis/atomic) serializes all ops; OT transform handles concurrency
  CONCENTRATION: popular document → many editors → single WS server hot; shard popular docs
```

---

*Next: Case Study 16 — Search Engine (Typeahead + Full Text)*
