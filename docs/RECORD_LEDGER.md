# Kura Record/Ledger System

**Date:** 2026-02-18
**Inspired by:** [CommonGround Core](https://github.com/Intelligent-Internet/CommonGround) CardBox pattern
**Status:** Design proposal

---

## Terminology

We use plain English for internal data structures, Japanese for projects.

| CommonGround term | Our term | What it is |
|-------------------|----------|------------|
| Card | **Record** | Immutable typed entry — the atom of agent memory |
| CardBox | **Ledger** | Ordered append-only collection of Record references |
| ContextBox | **context** | Read-only input ledger for an agent turn |
| OutputBox | **output** | Write-only output ledger for an agent turn |

---

## What We're Taking (and What We're Not)

### Borrowing
1. **Record** — Immutable typed entry (the atom of agent memory)
2. **Ledger** — Ordered append-only collection of record references
3. **Dual-ledger pattern** — Read-only context + write-only output per agent turn
4. **Semantic record types** — `task.instruction`, `agent.thought`, `tool.call`, `tool.result`, `task.deliverable`

### Adapting
- **PostgreSQL → SQLite** — We run on a Pi. Single-file DB, zero server overhead.
- **NATS JetStream for persistence → NATS Core for signaling only** — State lives in SQLite, NATS is the doorbell.
- **Python Pydantic models → Go structs** — Hashi is Go. Kura is Go (Phase 5).
- **UUIDv7 → Keep it** — Time-sortable, works great for append-only logs.

### Skipping
- CAS gating (overkill at single-node scale)
- PostgreSQL async adapters
- PMO fork/join orchestrator (we have Hashi)
- Their ContextEngine (our LLM routing goes through Minato/existing skills)

---

## Architecture: Where It Fits

```
                ┌────────────────────┐
                │   Minato (Brain)   │
                │   Orchestrator     │
                └────────┬───────────┘
                         │
              ┌──────────┼──────────┐
              │    NATS (doorbell)   │
              │  "new record written"│
              │  "turn complete"     │
              └──────────┬──────────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
   ┌─────────────┐ ┌──────────┐ ┌────────────┐
   │   Hashi     │ │  Mikado  │ │   Aegis    │
   │  (Executor) │ │  (Soul)  │ │  (Shield)  │
   └──────┬──────┘ └────┬─────┘ └──────┬─────┘
          │              │              │
          └──────────────┼──────────────┘
                         ▼
              ┌─────────────────────┐
              │   Kura (Storage)    │
              │                     │
              │  ┌───────────────┐  │
              │  │ records table │  │
              │  │ ledgers table │  │
              │  │ kb_vectors    │  │
              │  │ memory_graph  │  │
              │  └───────────────┘  │
              │      SQLite         │
              └─────────────────────┘
```

**Key principle:** State and signal are separated.
- **Kura (SQLite)** = source of truth. Records, ledgers, vectors, memory.
- **NATS** = doorbell only. "Hey, a record was written" / "Turn complete" / "Tool result ready." If NATS is down, the data is still safe in SQLite. Workers poll on reconnect.

---

## Data Model (SQLite)

### Records Table

```sql
CREATE TABLE records (
    record_id   TEXT PRIMARY KEY,  -- UUIDv7 (time-sortable)
    hash        TEXT NOT NULL,     -- SHA-256(type + content + author_id) — content-addressable
    record_type TEXT NOT NULL,     -- 'task.instruction', 'agent.thought', 'tool.call', etc.
    content     TEXT NOT NULL,     -- JSON blob (text, tool args, structured output)
    author_id   TEXT NOT NULL,     -- 'user:oles', 'agent:bmo', 'agent:hashi', 'tool:github'
    metadata    TEXT,              -- JSON: {project_id, trace_id, step_id, role, ...}
    created_at  TEXT NOT NULL      -- ISO 8601
);

CREATE INDEX idx_records_type ON records(record_type);
CREATE INDEX idx_records_author ON records(author_id);
CREATE INDEX idx_records_created ON records(created_at);
```

### Ledgers Table

```sql
CREATE TABLE ledgers (
    ledger_id   TEXT PRIMARY KEY,  -- UUIDv7
    root_hash   TEXT NOT NULL,     -- Merkle root of record hashes — verify integrity in one check
    label       TEXT,              -- human-readable: 'morning-session-ctx', 'hashi-turn-3-out'
    parent_ids  TEXT,              -- JSON array of parent ledger_ids (lineage, like Git parents)
    record_ids  TEXT NOT NULL,     -- JSON array of record_ids (ordered)
    created_at  TEXT NOT NULL
);
```

### Record Types (Our Semantic Vocabulary)

```
task.instruction  — What needs to be done (from user or Minato)
task.deliverable  — Final output/result
agent.thought     — Agent reasoning (BMO's chain of thought)
agent.plan        — Structured plan before execution
tool.call         — Tool invocation request
tool.result       — Tool execution result
memory.fact       — Extracted knowledge (feeds Kura Phase 2)
memory.decision   — Architectural/design decision
sys.context       — System context snapshot
sys.error         — Error/failure record
```

This maps cleanly to Kura Phase 2 typed memory categories:
- `memory.fact` → Kura `Fact` type
- `memory.decision` → Kura `Decision` type
- `task.*` → Kura `Event` type (with fast decay)

---

## Dual-Ledger Pattern: How a Turn Works

```
┌─────────────────────────────────────────────────────────┐
│                     Agent Turn                          │
│                                                         │
│  context (READ-ONLY)              output (WRITE-ONLY)   │
│  ┌─────────────────────┐         ┌───────────────────┐  │
│  │ task.instruction    │         │ agent.thought     │  │
│  │ sys.context         │         │ tool.call         │  │
│  │ memory.fact (RAG)   │         │ tool.result       │  │
│  │ prev deliverable    │         │ task.deliverable  │  │
│  └─────────────────────┘         └───────────────────┘  │
│                                                         │
│  Agent reads context              Agent writes to       │
│  to understand the task.          output only.          │
│  Never modifies it.              This becomes the       │
│                                   next turn's context.  │
└─────────────────────────────────────────────────────────┘
```

**Flow:**
1. **Minato** receives a task → creates `task.instruction` Record → packs a **context** ledger with instruction + relevant memory (Kura search results) + previous output
2. **Hashi** picks up the turn → reads **context** → executes → writes thoughts, tool calls, results, and deliverable into **output** ledger
3. **Mikado** observes **output** → extracts `memory.fact` / `memory.decision` records → feeds them back to Kura for long-term storage
4. **Next turn** gets a new **context** built from the previous **output** + fresh context

**No mutable shared state.** Each turn is a pure function: `output = Agent(context)`.

---

## Git-Like Properties

The Record/Ledger model maps directly to Git's object model:

```
Git                          Kura
─────────────────────────────────────
blob      → Record           (immutable, content-addressed via hash)
tree      → record_ids[]     (ordered list of what's in this snapshot)
commit    → Ledger           (snapshot with parent lineage + root_hash)
ref/HEAD  → latest ledger_id (pointer to current state)
branch    → session/project  (namespace for a chain of ledgers)
```

### Content-Addressed Records

Every Record gets a `hash = SHA-256(type + content + author_id)`. This gives us:

- **Tamper detection** — if a record's content doesn't match its hash, something is wrong
- **Deduplication** — same fact extracted twice? Same hash, skip it
- **Fast equality** — compare two records without reading their content

### Merkle Root on Ledgers

Each Ledger stores a `root_hash` computed from its records' hashes. This is the same idea as a Git tree hash — you can verify the entire ledger's integrity in one comparison.

```go
func computeRootHash(records []Record) string {
    h := sha256.New()
    for _, r := range records {
        h.Write([]byte(r.Hash))
    }
    return hex.EncodeToString(h.Sum(nil))
}
```

### Sessions as Commit Chains

A session is a **chain of ledgers**, each pointing to its parent via `parent_ids` — exactly like a Git branch is a chain of commits:

```
context₁ ──→ output₁ ──→ context₂ ──→ output₂ ──→ context₃ ──→ ...
  (turn 1)                 (turn 2)                 (turn 3)
```

Each `context` ledger includes the previous `output` ledger's ID in its `parent_ids`.

### CLI Affordances

This unlocks Git-like operations on agent history:

```bash
kura log <session>                    # walk the ledger chain — like git log
kura diff <ledger_a> <ledger_b>       # what records changed between turns
kura verify <ledger_id>               # check merkle root, confirm integrity
kura show <record_id>                 # inspect a single record
kura export <session> --format=git    # export session as actual Git repo for archival
```

---

## NATS Integration

We already wanted NATS for Mikado's event queue. Now it serves double duty:

### Subjects

```
oclaw.v1.{project}.cmd.{target}.{action}   — Commands (work queue)
oclaw.v1.{project}.evt.{source}.{event}    — Events (broadcast)
```

**Examples:**
```
oclaw.v1.hashi.cmd.worker.dispatch    — "New turn ready, go execute"
oclaw.v1.hashi.evt.worker.complete    — "Turn done, output ledger written"
oclaw.v1.mikado.evt.record.written    — "New record stored in Kura"
oclaw.v1.minato.cmd.session.create    — "Start new session"
oclaw.v1.aegis.cmd.tool.approve       — "Human approval needed"
```

### NATS is Just the Doorbell

- **No persistence in NATS.** All state is in SQLite.
- **No message payloads larger than a pointer.** Send `{ledger_id, record_id, turn_id}`, not the actual data.
- **If NATS goes down:** Workers poll SQLite on reconnect. Nothing lost.
- **Embedded NATS server** — runs inside Mikado process. No separate server to manage.

```go
// Mikado starts embedded NATS
import "github.com/nats-io/nats-server/v2/server"

opts := &server.Options{
    Host: "127.0.0.1",
    Port: 4222,
    NoSig: true,
}
ns, _ := server.NewServer(opts)
go ns.Start()
```

---

## Go Structs (Kura SDK)

```go
package kura

import "time"

// Record is an immutable entry. Once created, never modified.
type Record struct {
    ID        string            `json:"id"`         // UUIDv7
    Hash      string            `json:"hash"`       // SHA-256(type + content + author_id)
    Type      string            `json:"type"`       // "task.instruction", "agent.thought", etc.
    Content   any               `json:"content"`    // TextContent | JSONContent | ToolCallContent
    AuthorID  string            `json:"author_id"`  // "user:oles", "agent:bmo"
    Metadata  map[string]string `json:"metadata,omitempty"`
    CreatedAt time.Time         `json:"created_at"`
}

// Ledger is an ordered, append-only collection of record references.
type Ledger struct {
    ID        string    `json:"id"`          // UUIDv7
    RootHash  string    `json:"root_hash"`   // Merkle root of record hashes
    Label     string    `json:"label"`       // human-readable
    ParentIDs []string  `json:"parent_ids"`  // lineage (like Git commit parents)
    RecordIDs []string  `json:"record_ids"`  // ordered references
    CreatedAt time.Time `json:"created_at"`
}

// Content types
type TextContent struct {
    Text string `json:"text"`
}

type ToolCallContent struct {
    ToolName  string         `json:"tool_name"`
    Arguments map[string]any `json:"arguments"`
    Status    string         `json:"status"` // "pending", "success", "error"
}

type ToolResultContent struct {
    Status string `json:"status"`
    Result any    `json:"result,omitempty"`
    Error  string `json:"error,omitempty"`
}

// Store interface (implemented by Kura's SQLite backend)
type Store interface {
    WriteRecord(record Record) error
    ReadRecord(id string) (Record, error)
    ExistsByHash(hash string) (bool, error)  // dedup check before writing
    WriteLedger(ledger Ledger) error
    ReadLedger(id string) (Ledger, error)
    Hydrate(ledgerID string) ([]Record, error) // resolve record_ids → Records
    Verify(ledgerID string) (bool, error)      // recompute root_hash, compare
    Query(recordType string, since time.Time, limit int) ([]Record, error)
}
```

---

## Implementation Phases

### Phase A: Record/Ledger in Kura (Week 1)
- Add `records` and `ledgers` tables to Kura's SQLite
- Implement `Store` interface in Go
- UUIDv7 generation
- Basic read/write + hydrate

### Phase B: Dual-Ledger in Hashi (Week 2)
- Hashi Worker reads context ledger from Kura
- Writes agent output as Records into output ledger
- Tool calls/results tracked as Records

### Phase C: NATS Doorbell in Mikado (Week 3)
- Embedded NATS server in Mikado
- Publish events on record/ledger writes
- Hashi subscribes to dispatch commands
- Replace cron polling with event-driven triggers

### Phase D: Memory Extraction (Week 4)
- Mikado watches output ledger events
- Extracts `memory.*` records from deliverables
- Feeds them into Kura's vector store + graph

---

## What This Buys Us

| Before | After |
|--------|-------|
| Sessions stored as opaque `.jsonl` blobs | Every agent artifact is a typed, queryable Record |
| No cross-session memory | Records persist across turns and sessions in Kura |
| Cron-based polling | Event-driven via NATS doorbell |
| Mutable shared state risk | Immutable records + dual-ledger isolation |
| Lost agent reasoning | `agent.thought` records preserved for debugging |
| No tool call audit trail | `tool.call` + `tool.result` records = full audit log |
| Memory extraction is manual | Mikado auto-extracts facts/decisions from output |

---

## Design Principles

1. **State and signal are separated** — SQLite = truth, NATS = doorbell
2. **Records are immutable** — No silent mutations, full audit trail
3. **Dual-ledger isolation** — Read-only context, write-only output per turn
4. **Local-first on edge hardware** — SQLite + embedded NATS, runs on Pi
5. **Zero cloud dependency** — Full data sovereignty
6. **Humans are agents** — Oles participates via the same Record protocol
7. **Pointers over payloads** — NATS carries IDs, not data
8. **Content-addressed like Git** — Records have hashes, ledgers have merkle roots, sessions are commit chains
