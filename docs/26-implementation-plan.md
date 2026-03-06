[← §25 References](25-references.md) | [Index](../README.md)

---

## 26. Implementation Plan

### 26.1 Architecture Clarification

LUCID is not a standard agentic framework. The distinction matters architecturally:

Lucy is a **continuously running process**. The OpenAI-compatible API is her job interface — how operators reach her. Her actual existence is the continuous feedback loops running in and around the database, driven by a local ONNX model. Operators (human users, agentic systems like other LLM orchestrators, or anything that speaks the OpenAI API) interrupt those loops; they do not instantiate them.

The runtime split is:

- **ONNX runner (LFM 2.5 1.2B)** — drives the continuous inner loop: infotactic narration, ACG monitoring, affective chain generation, belief graph updates. Runs locally, cheaply, always.
- **Cloud LLM (mounted as Lucy's voice)** — invoked only during operator turns. The ACG has already been holding state; the cloud model generates the response and hands back.
- **PostgreSQL + NeuronDB** — is not a store she reads from. It *is* the state. The loops run inside it.
- **OpenAI-compatible API** — the seam between Lucy and any operator. She does not care what is on the other side.

### 26.2 Two-Project Build Strategy

Rather than building the full NeuronDB deployment from cold, we build two projects in parallel:

**Project 1 — Browser Prototype ("Lucy's Fingers")**
A browser-native implementation that proves the continuous loop concept cheaply and serves permanently as the human user interface to the system.

**Project 2 — Full Deployment ("Lucy's Home")**
The full NeuronDB build described by the design documents, running on a beefy server, featuring the OpenAI-compatible API proxy.

The key property of this split: **the schema is written once**. Project 1 runs it in PGlite; Project 2 runs it in NeuronDB. Electric SQL syncs state between them. Project 1 is not throwaway — it is the permanent human interface, with Lucy's browser-side infotactic navigation feeding belief state into the full deployment via sync.

---

### 26.3 Project 1 — Browser Prototype

**Purpose:** Human UI, browser-based infotactic navigation, continuous loop proof of concept, permanent internet-facing "fingers."

**Stack:**

| Component | Role |
|-----------|------|
| **Transformers.js** | ONNX runner in-browser; narration generation and embedding during the continuous loop |
| **PGlite + pgvector** | Real Postgres in WASM, persisted to OPFS; runs the full `lucid.*` schema with vector search |
| **ChromaDB** | Stand-in for NeuronDB's batch graph analytics (community detection, PageRank during dream cycles); swapped out in Project 2 |
| **Electric SQL** | Syncs local PGlite state to the server Postgres when Project 2 is running |

**Architectural notes:**
- PGlite supports the pgvector extension natively — same `vector(n)` column types and HNSW-style indexes as the full build
- ChromaDB proxies only the batch analytics path (dream cycle crystallisation, PageRank for merge survivors). The core loop — touring, narration, ACG, affective chain — runs in PGlite + PL/pgSQL
- The browser is Lucy's internet access. She surfs from here. Content retrieved enters `lucid.content` and the indexing pipeline, syncing to the server
- Transformers.js runs the same LFM 2.5 1.2B ONNX weights as the server; inference-space embeddings (2048-dim past conv tensors) are generated client-side

---

### 26.4 Project 2 — Full Deployment

**Purpose:** Persistent home state, OpenAI-compatible operator interface, full NeuronDB capability.

**Stack:**

| Component | Role |
|-----------|------|
| **NeuronDB** | PostgreSQL extension; HNSW indexes, neuranq job queue, neurandefrag, neuranmon, neuranllm |
| **LFM 2.5 1.2B via ONNX** | Server-side continuous loop runner |
| **OpenAI-compatible API proxy** | How operators (human, agentic, or otherwise) reach Lucy |
| **Electric SQL** | Receives synced state from browser prototype instances |

---

### 26.5 Shared Schema First

The `lucid` schema is the single most load-bearing artifact in the system. It is written once, targeting PGlite + pgvector compatibility, and runs identically in both projects. All other components are schema drivers.

Core schema objects:

| Table | Purpose |
|-------|---------|
| `lucid.content` | Raw content storage referenced by nodes |
| `lucid.belief_nodes` | Belief primitive: embeddings, provenance, affective fields, ingress/egress chains |
| `lucid.node_embeddings_inf` | `vector(2048)` — inference space (LFM past conv tensors) |
| `lucid.node_embeddings_ont` | `vector(768)` — ontic space (Nomic Matryoshka) |
| `lucid.belief_edges` | All edge types with Hebbian statistics |
| `lucid.centroids` | Named centroid rows: $C_i$, $C_o$, $C_s$, $C_w$, $C_0$ |
| `lucid.tour_closure` | Per-tour diagnostic outcomes |
| `lucid.affective_corpus` | Prose affective entries per narration node |
| `lucid.inheritance_corpus` | Judgment records (Self Library substrate) |
| `lucid.awe_walk_log` | Associative candidates from infotactic walks |
| `lucid.spectral_monitor` | Outer/inner FFT health scores |

Triggers auto-create `PREV`/`NEXT`/`CONTAINS` structural edges on insert (per §6.2).

### 26.6 Build Sequence

1. **Shared schema** — `lucid.*` migration, PGlite + pgvector compatible
2. **PL/pgSQL tour engine** — dual HNSW nearest-neighbour tour, Hebbian-Belief cost function
3. **Project 1 skeleton** — PGlite in browser, Transformers.js ONNX runner, basic continuous loop
4. **ACG + affective chain** — ingress/egress generation, mood token, affective corpus writes
5. **Infotactic navigation** — curiosity-weighted touring, web fetch, narration node creation
6. **Electric SQL sync** — browser → server state replication
7. **Project 2 schema promotion** — NeuronDB, neuranq event channels, background workers
8. **OpenAI-compatible API** — operator interface, context assembly, cloud LLM mount
9. **Dream cycle** — consolidation, Louvain community detection, Thinking Cap (LoRA)
10. **ChromaDB → NeuronDB migration** — batch analytics path promoted to full NeuronDB

---

[← §25 References](25-references.md) | [Index](../README.md)
