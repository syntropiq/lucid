# LUCID Lite — TODO

Phases are sequential. Within a phase, items can be parallelised where marked ⟂. See `PLAN.md` for rationale and detail behind each item.

---

## Phase 1 — DDL + pglite shell

- [ ] Scaffold repo structure
  - [ ] `src/db/` — SQL migrations
  - [ ] `src/sue/` — SUE registry + wrappers (from §28)
  - [ ] `src/workers/` — ONNX worker shells (empty stubs for now)
  - [ ] `src/app/` — UI shell (empty index.html for now)
  - [ ] `src/network/` — libp2p shell (empty stub for now)
  - [ ] `package.json` with deps: pglite, @xenova/transformers, vite, vitest
  - [ ] `tsconfig.json`

- [ ] Write `src/db/01-schema.sql`
  - [ ] `sue` schema + `sue.model_registry` + `sue.activation_log` (from §28.2)
  - [ ] `lucid.conversation_turns`
  - [ ] `lucid.belief_nodes`
  - [ ] `lucid.belief_edges`
  - [ ] `lucid.embedding_queue` (status enum, index on pending)
  - [ ] `lucid.inference_queue` (status enum, index on pending)
  - [ ] `lucid.node_embeddings_ont` — `vector(768)` ⟂
  - [ ] `lucid.node_embeddings_inf` — `vector(dim)` dim from SUE registry ⟂
  - [ ] `lucid.centroids` — columns: `c_i`, `c_o`, `c_s`, `c_w`, `c_0`, `foundation_weight`, `orbital_health`
  - [ ] `lucid.spectral_monitor` — columns: `inner_streams jsonb`, `stream_labels text[]`, `cross_stream_correlation float`, `dual_stream_available bool`, `composite_health_score float`, `sampled_at timestamptz`
  - [ ] `lucid.affective_corpus` — mood token + valence fields
  - [ ] Trigger: `trg_enqueue_inference` — new user turn → inference_queue INSERT
  - [ ] Trigger: `trg_enqueue_embedding` — new belief_node → embedding_queue INSERT
  - [ ] Confirm schema is idempotent (safe to re-apply)

- [ ] Port SUE code from §28 into `src/sue/`
  - [ ] `src/sue/contracts/ontic.ts` — `OnticContract` interface
  - [ ] `src/sue/contracts/inference.ts` — `InferenceContract` + `SpectralSample` interfaces
  - [ ] `src/sue/wrappers/ontic/nomic-embed-v1.5.ts`
  - [ ] `src/sue/wrappers/inference/lfm-2.5.ts`
  - [ ] `src/sue/wrappers/inference/generic-onnx.ts` — `makeGenericOnnxWrapper()`
  - [ ] `src/sue/registry.ts` — `sue.registerOntic`, `sue.activateOntic`, `sue.ontic()`, etc.

- [ ] Write `src/db/seed.ts`
  - [ ] Boot pglite with `vector` + `live` extensions
  - [ ] Apply `01-schema.sql`
  - [ ] Activate SUE (register + activate wrappers)
  - [ ] Smoke test: INSERT belief_node → verify embedding_queue row appears
  - [ ] Smoke test: INSERT user conversation_turn → verify inference_queue row appears

- [ ] Phase 1 test: `vitest` against in-process pglite, no ONNX, queue rows only

---

## Phase 2 — ONNX workers as live subscribers

- [ ] Write `src/db/02-centroid.sql`
  - [ ] `lucid.update_centroid_ont()` — provenance-weighted centroid update (§6.5)
  - [ ] `lucid.update_centroid_inf()` — same for inference space
  - [ ] Trigger: `trg_centroid_update_ont` — after INSERT on `node_embeddings_ont`
  - [ ] `lucid.write_spectral_sample()` function (from §28.7)

- [ ] Write `src/workers/embed.worker.ts` ⟂
  - [ ] Boot SUE (register ontic wrapper only)
  - [ ] `db.live.query` on `embedding_queue WHERE status='pending' LIMIT 1`
  - [ ] Handler: mark `processing` → `sue.ontic().embed(content)` → INSERT `node_embeddings_ont` → mark `complete`
  - [ ] Error handling: mark `error`, log `error_msg`

- [ ] Write `src/workers/inference.worker.ts` ⟂
  - [ ] Boot SUE (register inference wrapper only)
  - [ ] `db.live.query` on `inference_queue WHERE status='pending' LIMIT 1`
  - [ ] Context assembly: last 10 turns + top-5 HNSW neighbours of user message embedding
  - [ ] `sue.inference().generate(prompt)` → INSERT `conversation_turns` (assistant) + `belief_nodes` (narration)
  - [ ] `sue.inference().spectralSample()` → `lucid.write_spectral_sample()`
  - [ ] Mark queue item `complete`

- [ ] Wire workers to pglite instance (SharedArrayBuffer approach for browser; worker_threads for Node.js)

- [ ] Phase 2 test
  - [ ] INSERT belief_node with content → verify `node_embeddings_ont` populated within timeout
  - [ ] INSERT user turn → verify assistant turn appears within timeout
  - [ ] Verify `lucid.centroids` `c_o` updated after embedding
  - [ ] Verify `lucid.spectral_monitor` row written after inference

---

## Phase 3 — Chat interface (closes the loop)

- [ ] Vite project scaffold in `src/app/`
  - [ ] `index.html` — minimal: message list + input form + queue status badge + health badge
  - [ ] `main.ts` — boots pglite, applies schema, activates SUE, launches workers, attaches live queries

- [ ] Live queries for UI
  - [ ] Conversation display: `SELECT role, content, created_at FROM conversation_turns ORDER BY created_at DESC LIMIT 50`
  - [ ] Queue depth: pending counts for inference + embedding queues
  - [ ] Spectral health badge: `composite_health_score` from latest spectral_monitor row
  - [ ] Centroid position (debug): current `c_o`, `c_w` distance from `c_0`

- [ ] Form submit handler
  - [ ] `INSERT lucid.conversation_turns (role='user', content=$1)`
  - [ ] That is all — the rest is live queries

- [ ] `window.__lucid = { db, sue }` console handle

- [ ] OPFS persistence: `PGlite.create({ dataDir: 'idb://lucid-lite' })`

- [ ] Phase 3 manual test
  - [ ] Open browser, type message, receive response
  - [ ] Open DevTools → Application → IndexedDB → verify belief_nodes accumulating
  - [ ] Refresh page → verify conversation persists from OPFS
  - [ ] Open `window.__lucid.db` in console → `SELECT * FROM lucid.centroids` → verify c_o has moved

---

## Phase 4 — Feedback loops in SQL

- [ ] Write `src/db/03-loops.sql`

  **AWE chain**
  - [ ] `lucid.compute_affective_valence(content text)` — placeholder: simple sentiment heuristic returning `{valence: float, arousal: float}`
  - [ ] Trigger: `trg_awe_chain` — after INSERT on `lucid.belief_nodes` WHERE `node_type = 'narration'` → INSERT `affective_corpus`, derive `mood_token`, UPDATE `conversation_turns`
  - [ ] `lucid.affective_corpus` LIMIT + rolling window management

  **Spectral health composite**
  - [ ] `lucid.compute_spectral_composite()` — reads latest `spectral_monitor` row, computes composite score from stream FFT data
  - [ ] Trigger: `trg_spectral_composite` — after INSERT/UPDATE on `spectral_monitor`
  - [ ] Live query hook in inference worker: `composite_health_score < 0.3` → call `injectCosineDisimilarNode()`

  **Working centroid CfC**
  - [ ] `lucid.update_working_centroid()` — closed-form CfC step (§6.6)
  - [ ] Trigger: `trg_cfc_step` — after UPDATE on `lucid.centroids` WHERE `c_o` changed

  **Orbital health**
  - [ ] `lucid.check_orbital_health()` — 32-page window check: does trajectory enclose both `c_i` and `c_o`?
  - [ ] Updates `orbital_health` boolean on `lucid.centroids`
  - [ ] Live query: `orbital_health = false` → log intervention tier, INSERT TripleDent Gum node

- [ ] Phase 4 test
  - [ ] Verify `affective_corpus` grows after conversation
  - [ ] Verify `mood_token` appears on conversation turns
  - [ ] Verify `composite_health_score` is computed and written
  - [ ] Manually trigger low-health condition (INSERT 20 identical turns) → verify injection fires
  - [ ] Verify `c_w` changes after centroid update

---

## Phase 5 — Vortex / libp2p

- [ ] Write `src/db/04-peer-centroids.sql`
  - [ ] `lucid.peer_centroids` table: `peer_id text PK`, `c_o vector(768)`, `last_seen timestamptz`
  - [ ] `lucid.upsert_peer_centroid(peer_id, c_o)` function

- [ ] Write `src/network/vortex.ts`
  - [ ] `initVortex(db, sue)` — creates libp2p node with WebRTC + WebSockets + Noise + Yamux + GossipSub
  - [ ] Heartbeat: `setInterval` → read `lucid.centroids c_o` → publish to `lucid:centroid` topic
  - [ ] Incoming centroid: `lucid:centroid` subscription → `lucid.upsert_peer_centroid()`
  - [ ] Incoming work: `lucid:work` subscription → cosine similarity check → accept (INSERT `belief_nodes`) or forward
  - [ ] `buildHeartbeat(db)` — reads swim mode, returns appropriate payload (§27.5)
  - [ ] `forwardToHigherScoringPeer(packet)` — publish to highest-scoring peer by centroid proximity

- [ ] Live query: `lucid.peer_centroids` UPDATE → rebuild in-memory routing table

- [ ] Wire `vortex.ts` into `src/app/main.ts` boot sequence

- [ ] Phase 5 test
  - [ ] Open two browser tabs → verify each receives the other's heartbeat
  - [ ] Verify `lucid.peer_centroids` has a row for the remote peer
  - [ ] Send a work packet from tab A → verify tab B INSERTs to belief_nodes
  - [ ] Verify tab B centroid moves after receiving work

---

## Ongoing / cross-phase

- [ ] Keep `src/sue/` in sync with §28 as the design evolves
- [ ] Keep `src/db/01-schema.sql` the single source of truth — no schema defined outside SQL files
- [ ] Every new live query gets a corresponding test INSERT that verifies the callback fires
- [ ] `window.__lucid` console handle remains available in all phases — do not remove
