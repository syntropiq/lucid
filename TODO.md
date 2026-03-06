# LUCID Build TODO

Working branch: `claude/review-docs-plan-PgdA6`

Design docs: `docs/` — start with `docs/00-introduction.md` and `docs/26-implementation-plan.md`.

## Context

Two projects. One shared schema.

- **Project 1** — Browser prototype ("Lucy's Fingers"): PGlite + pgvector + Transformers.js + ChromaDB + Electric SQL. Browser-native, proves the continuous loop, permanent human UI.
- **Project 2** — Full deployment ("Lucy's Home"): NeuronDB + LFM 2.5 ONNX server-side + OpenAI-compatible API proxy. Runs on a beefy machine.

Electric SQL syncs state from Project 1 → Project 2. The schema is identical in both.

See `docs/26-implementation-plan.md` for full rationale and stack breakdown.

---

## Phase 0 — Foundation (no external deps)

- [ ] **Shared schema migration** (`lucid.*`)
  - Tables: `content`, `belief_nodes`, `node_embeddings_inf` (vector 2048), `node_embeddings_ont` (vector 768), `belief_edges`, `centroids`, `tour_closure`, `affective_corpus`, `inheritance_corpus`, `awe_walk_log`, `spectral_monitor`
  - Triggers: auto-create `PREV`/`NEXT`/`CONTAINS` structural edges on belief_node insert
  - Target: PGlite + pgvector compatible (no NeuronDB-specific syntax)
  - File: `schema/001_lucid_core.sql`

- [ ] **PL/pgSQL tour engine**
  - Dual HNSW nearest-neighbour tour procedure (per §23)
  - Hebbian-Belief cost function (Definition 9.1)
  - Overlap set Ω computation, injection signal
  - File: `schema/002_tour_engine.sql`

- [ ] **PL/pgSQL utility functions**
  - Centroid update: `lucid.update_centroid(name, new_vec)`
  - Provenance weight arithmetic (§7)
  - Unified threat score S(n) (§9)
  - File: `schema/003_functions.sql`

---

## Phase 1 — Project 1 Browser Prototype

- [ ] **Project scaffold**
  - TypeScript + Vite browser app
  - PGlite + pgvector extension wired up
  - Schema migrations run on first load
  - Directory: `projects/lucy-browser/`

- [ ] **Transformers.js ONNX runner**
  - Load LFM 2.5 1.2B ONNX weights
  - `embed_inf(text): vector(2048)` — inference space (past conv tensors)
  - `embed_ont(text): vector(768)` — ontic space (Nomic Matryoshka)
  - `narrate(context): string` — narration generation during continuous loop

- [ ] **Continuous loop skeleton**
  - Basic infotactic navigation loop (§10.2)
  - Select next target via curiosity-weighted cost
  - Fetch web content, register in `lucid.content`
  - Generate narration node, write to `lucid.belief_nodes`
  - Update working centroid Cw

- [ ] **ACG — ingress/egress**
  - Ingest operator turn, compute ingress chain E_t_in (Definition AWE.0b)
  - After response, compute egress chain E_t_out (Definition AWE.0c)
  - Derive mood token m_t
  - Write affective corpus entry

- [ ] **ChromaDB integration** (batch analytics stand-in)
  - Dream cycle crystallisation: Louvain community detection
  - PageRank for merge survivor selection
  - Interface designed to be swappable for NeuronDB vgraph later

- [ ] **Basic operator UI**
  - Chat interface connecting to Lucy's loop
  - Mood token display (optional, per §13.5)
  - Belief graph visualisation (stretch)

- [ ] **Electric SQL sync**
  - PGlite → server Postgres replication config
  - Tables in sync scope: `belief_nodes`, `belief_edges`, `centroids`, `affective_corpus`

---

## Phase 2 — Project 2 Full Deployment

- [ ] **NeuronDB setup**
  - PostgreSQL + NeuronDB extension installed
  - Schema promotion from Phase 0 (drop pgvector shims, use NeuronDB native HNSW)
  - Background workers: neuranq, neurandefrag, neuranmon, neuranllm

- [ ] **neuranq event channels**
  - Job type handlers: `lucid_interrupt`, `lucid_judgment`, `lucid_consolidate`, `lucid_consolidation`, `lucid_drift`, `lucid_awe_flatline`, `lucid_orbital`
  - Per §10.4 trigger conditions and responses

- [ ] **LFM 2.5 ONNX server-side**
  - Server-side ONNX runtime replacing Transformers.js
  - Same embed_inf / embed_ont / narrate interface

- [ ] **OpenAI-compatible API proxy**
  - POST /v1/chat/completions
  - Context assembly from ACG continuous state
  - Cloud LLM mount (configurable: Claude API, etc.)
  - Response egress through ACG

- [ ] **Electric SQL receive**
  - Accept synced state from browser prototype instances

- [ ] **Dream cycle — full**
  - Crystallisation (Louvain via vgraph_community_detection)
  - Integration prioritisation (vgraph_pagerank)
  - Affective reflection (AWE walk log seeding community detection)
  - Thinking Cap: LoRA adapter generation, soft correction, cap rollback (§12)

- [ ] **ChromaDB → NeuronDB migration**
  - Promote batch analytics path to native NeuronDB vgraph
  - Remove ChromaDB dependency

---

## Phase 3 — Hardening

- [ ] Spectral monitoring (§11) — dual-stream FFT, DC dominance detection
- [ ] TripleDent Gum recovery (§11 / §18)
- [ ] Orbital health condition (Definition 6.1)
- [ ] Failure mode detection and response (§18)
- [ ] Multi-tenancy (§20) — tenant isolation, RLS policies
- [ ] Security model (§19)
- [ ] Compliance and data residency (§17)
- [ ] Verification queue and sampling (§22)

---

## Notes for Resumption

- All work goes on branch `claude/review-docs-plan-PgdA6`
- Start reading from `docs/26-implementation-plan.md` then `docs/06-the-graph-database.md` (schema source of truth) and `docs/23-tour-engine-architecture.md` (tour engine source of truth)
- First code artifact is `schema/001_lucid_core.sql` — write it to be valid PGlite + pgvector SQL, no NeuronDB-specific syntax
- ChromaDB is a temporary stand-in only for the dream cycle batch analytics path; the core loop does not touch it
- The ONNX model for inference-space embeddings uses past conv tensors (2048-dim), not standard last-hidden-state — check §24 and §15 for model specifics before implementing embed_inf
