# Kura Knowledge Base: Implementation Plan

**Date:** 2026-02-17
**Status:** Phase 1 COMPLETE, Phase 2 Planned

## Current State (Phase 1 - DONE)

Working minimal KB using FastEmbed (local ONNX embeddings, no API dependency):

- **Script:** `/home/oles/.openclaw/workspace/scripts/kb_embed.py`
- **Model:** `BAAI/bge-small-en-v1.5` (384-dim, runs locally on Pi via FastEmbed/ONNX)
- **Store:** JSON vector store at `data/kb_vectors.json` (56 notes, 109 chunks, 0.9 MB)
- **Features:** Incremental indexing (MD5 change detection), cosine similarity search, dedup by file
- **Cron:** `kb-reindex` fires every 6 hours
- **Zero API dependency** — fully offline capable

## Phase 2: Typed Memory + Graph Layer

Goals: structured memory with types, relationships, and decay scoring.

### 2a. Typed Memory Categories

Move from flat chunks to typed memories:

| Type | Description | Decay? |
|------|-------------|--------|
| **Fact** | Verified knowledge | Slow |
| **Preference** | User preferences / style | None |
| **Decision** | Architectural/design decisions made | Slow |
| **Identity** | Core identity info (who Oles is, who BMO is) | Never |
| **Event** | Things that happened (commits, deployments, meetings) | Fast |
| **Observation** | Agent observations about patterns | Medium |
| **Goal** | Active goals and objectives | Medium |
| **Todo** | Actionable items | Fast (done = archived) |

### 2b. Graph Connections

Add relationship edges between memories:

- `RelatedTo` — topical relationship
- `Updates` — newer memory supersedes older one
- `Contradicts` — conflicting information (flag for resolution)
- `CausedBy` — causal chain
- `PartOf` — hierarchical grouping

### 2c. Memory Decay / Importance Scoring

Score = f(frequency, recency, graph_centrality, type_weight)

- Each access reinforces a memory (frequency)
- Recent memories score higher (recency)
- Well-connected memories score higher (centrality)
- Identity memories never decay
- Events and Todos decay fastest

## Phase 3: Hybrid Search (RRF)

Combine vector similarity + keyword search using Reciprocal Rank Fusion:

1. **Vector search** — FastEmbed cosine similarity (current)
2. **Keyword search** — SQLite FTS5 (add to Kura core)
3. **RRF merge** — `score = Σ 1/(k + rank_i)` across both result lists

RRF is a proven, simple fusion method that needs no tuning.

## Phase 4: Auto-Capture

Implicit memory extraction after agent conversations:

1. Agent conversation ends
2. Extract key facts, decisions, observations
3. Classify into memory types
4. Embed and store with graph connections
5. No explicit tool call needed

## Phase 5: Migration to Kura Core (Go Binary)

Move from Python script to Kura's Go binary:

- SQLite + sqlite-vec for storage (replaces JSON file)
- MCP tools: `kura_search(query, mode="semantic")`
- FastEmbed via CGO or subprocess
- Dynamic collections / namespacing
- **Record/Ledger system** — immutable typed records with content-addressed hashing, dual-ledger turns, Git-like commit chains. See [RECORD_LEDGER.md](RECORD_LEDGER.md)

## Tech Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Embedding model | FastEmbed `bge-small-en-v1.5` | Local, fast, 384-dim, runs on Pi |
| Vector store (now) | JSON file | Simple, portable, good enough for <1000 chunks |
| Vector store (future) | SQLite + sqlite-vec | Single file, FTS5 for hybrid search, Go-native |
| Graph store (future) | SQLite adjacency table | No need for Neo4j at our scale |
| Search fusion | RRF | Simple, proven, no tuning needed |

## Design Principles

1. **Local-first on edge hardware** (Pi) — no cloud dependency
2. **Obsidian Vault integration** — direct access to user's knowledge base
3. **Zero recurring cost** — no API fees
4. **Full data sovereignty** — nothing leaves the device
5. **Open and hackable** — Python script you can read and modify
