# LUCID — TODO

Phases are sequential. Items marked ⟂ can be parallelised within a phase. See `PLAN.md` for rationale.

---

## Phase 1 — SUE substrate layer

- [ ] Scaffold `src/` directory structure
  - [ ] `src/sue/contracts/` — TypeScript interfaces
  - [ ] `src/sue/substrate/` — implementations
  - [ ] `src/sue/wrappers/` — model wrappers (Phase 2)
  - [ ] `src/core/` — feedback loop logic
  - [ ] `src/workers/` — Web Workers
  - [ ] `src/app/` — UI
  - [ ] `src/device/` — Device Lucy daemon
  - [ ] `package.json`, `tsconfig.json`, `vite.config.ts`

- [ ] Write `src/sue/contracts/types.ts` — shared types ⟂
  - [ ] `BeliefNode`, `BeliefEdge`, `EdgeType`, `NodeType`
  - [ ] `CentroidRecord` (c_i, c_o, c_s, c_w, c_0, orbital_health)
  - [ ] `AWEEntry` (valence, arousal, mood_token)
  - [ ] `SpectralRecord` (streams, stream_labels, cross_corr, health_score)
  - [ ] `SyncEvent` (id, type, payload, instance_id, created_at)
  - [ ] `ConversationTurn` (id, role, content, mood_token, created_at)
  - [ ] `ReconciliationRecord`

- [ ] Write `src/sue/contracts/vector-store.ts` — `VectorStore` interface ⟂
  - [ ] `add(id, prefix, metadata)` — stores truncated prefix (128-dim browser, 256-dim device)
  - [ ] `search(queryPrefix, k)` — coarse KNN on prefix
  - [ ] `get(id)`, `delete(id)`
- [ ] Write `src/sue/contracts/graph-store.ts` — `GraphStore` interface ⟂
  - [ ] All node/edge/centroid/AWE/spectral/sync methods
  - [ ] `embeddingOntPut(nodeId, full768)` + `embeddingOntGet(nodeId)` — full vectors for rerank
  - [ ] `embeddingInfPut(nodeId, fullInf)` + `embeddingInfGet(nodeId)`
- [ ] Write `src/sue/contracts/mesh.ts` — `Mesh` interface ⟂
- [ ] Write `src/core/vector-utils.ts` ⟂
  - [ ] `cosineSimilarity(a, b)`
  - [ ] `binarize(vec, dims)` — float → Uint8Array packed bits
  - [ ] `hammingScore(a, b)` — XOR + popcount, returns 0–1
  - [ ] `popcount(x)` — bit count kernel
- [ ] Write `src/core/search.ts` ⟂
  - [ ] `twoPhaseSearch(queryFull, k, ont, graph, prefix)` — coarse KNN + precise rerank

- [ ] Write `src/sue/substrate/entitydb-vector-store.ts` ⟂
  - [ ] `add(id, vector, metadata)` → EntityDB insert
  - [ ] `search(query, k)` → EntityDB cosine KNN
  - [ ] `get(id)` → EntityDB fetch by ID
  - [ ] `delete(id)` → EntityDB remove

- [ ] Write `src/sue/substrate/indexeddb-graph-store.ts` ⟂
  - [ ] Object store definitions: belief_nodes, belief_edges, awe_corpus, spectral_monitor, centroids, conversation_turns, sync_log, reconciliation_log
  - [ ] `nodeUpsert`, `nodeGet`, `nodeQuery`
  - [ ] `edgeUpsert`, `edgesFor`
  - [ ] `centroidGet`, `centroidPut`
  - [ ] `awePut`, `aweRecent`
  - [ ] `spectralPut`, `spectralLatest`
  - [ ] `syncLogAppend`, `syncLogPending`

- [ ] Write `src/sue/substrate/gun-mesh.ts` ⟂
  - [ ] Constructor: `new GunMesh(peers, seaPair)` — Gun init + SEA auth
  - [ ] `publish`, `subscribe`
  - [ ] `advertise`, `observe`
  - [ ] `peers()`
  - [ ] `syncLogDrain(graphStore)` — drains pending events into Gun mind namespace
  - [ ] Incoming event handler skeleton (hydration in Phase 5)

- [ ] Write `src/sue/substrate/lancedb-vector-store.ts` (Node-only) ⟂
- [ ] Write `src/sue/substrate/sqlite-graph-store.ts` (Node-only) ⟂

- [ ] Phase 1 tests — contract test suite that all implementations must pass
  - [ ] `VectorStore` contract tests: add, search returns k results, get, delete
  - [ ] `GraphStore` contract tests: node CRUD, edge CRUD, centroid round-trip, sync log append + pending
  - [ ] Run contract tests against EntityDB + IndexedDB implementations
  - [ ] Run contract tests against Lancedb + SQLite implementations

---

## Phase 2 — Model wrappers + SUE registry

- [ ] Port `src/sue/contracts/ontic.ts` — `OnticContract` interface ⟂
- [ ] Port `src/sue/contracts/inference.ts` — `InferenceContract` + `SpectralSample` ⟂

- [ ] Write `src/sue/wrappers/ontic/nomic-embed-v1.5.ts` ⟂
- [ ] Write `src/sue/wrappers/inference/generic-onnx.ts` — `makeGenericOnnxWrapper()` ⟂
- [ ] Write `src/sue/wrappers/inference/lfm-2.5.ts` (dual-stream) ⟂

- [ ] Write `src/sue/registry.ts` — unified registry
  - [ ] Model methods: `registerOntic`, `activateOntic`, `ontic()`
  - [ ] Model methods: `registerInference`, `activateInference`, `inference()`
  - [ ] Substrate method: `registerSubstrate({ vectorStores, graphStore, mesh })`
  - [ ] Substrate accessors: `ont()`, `inf()`, `graph()`, `mesh()`

- [ ] Phase 2 tests
  - [ ] `sue.ontic().embed('hello')` → Float32Array length 768
  - [ ] `sue.inference().generate([{role:'user', content:'hi'}])` → string
  - [ ] `sue.inference().spectralSample()` → valid SpectralSample

---

## Phase 3 — Embed + Inference Workers

- [ ] Write `src/workers/embed.worker.ts`
  - [ ] Boot SUE (ontic wrapper + EntityDB vector store + IndexedDB graph store)
  - [ ] BroadcastChannel listener for unindexed nodes
  - [ ] `full = await sue.ontic().embed(content)` → Float32Array(768)
  - [ ] `await sue.ont().add(id, full.slice(0, PREFIX_DIM))` — prefix to VectorStore
  - [ ] `await sue.graph().embeddingOntPut(id, full)` — full vector to GraphStore
  - [ ] `sue.graph().nodeUpsert({ ...node, index_state: 'indexed' })`
  - [ ] `sue.graph().syncLogAppend({ type: 'belief:node', ... })`
  - [ ] Broadcast `CENTROID_DIRTY`

- [ ] Write `src/workers/inference.worker.ts`
  - [ ] Boot SUE (inference wrapper + substrates)
  - [ ] BroadcastChannel listener for new user turns
  - [ ] Context assembly: last 20 turns from GraphStore + top-5 KNN from `sue.ont()`
  - [ ] `sue.inference().generate(messages)` → response text
  - [ ] `sue.graph().nodeUpsert(narrationNode)`
  - [ ] `sue.graph().awePut(spectralAsSyncEvent)`
  - [ ] `sue.graph().syncLogAppend({ type: 'belief:node', ... })`
  - [ ] Broadcast `TURNS_UPDATED`

- [ ] Write `src/workers/cfc.worker.ts`
  - [ ] BroadcastChannel listener for `CENTROID_DIRTY`
  - [ ] Read current centroid from GraphStore
  - [ ] Compute provenance-weighted running average for C_o
  - [ ] Apply CfC ODE step for C_w
  - [ ] Check orbital health condition
  - [ ] `sue.graph().centroidPut(updated)`
  - [ ] Broadcast `HEALTH_UPDATED`; if orbital_health false: broadcast `INJECT_REQUEST`

- [ ] Phase 3 tests
  - [ ] Insert belief node → verify EntityDB `lucy_ont` has vector within 5s
  - [ ] Insert user turn → verify assistant turn in GraphStore within 10s
  - [ ] Verify sync_log has entries after both

---

## Phase 4 — Chat interface

- [ ] `src/app/index.html` — message list, input form, health badge, queue indicator
- [ ] `src/app/main.ts`
  - [ ] SUE boot (browser substrate)
  - [ ] Launch Embed Worker, Inference Worker, CfC Worker
  - [ ] BroadcastChannel message handler → targeted GraphStore reads → render
  - [ ] Form submit → `sue.graph().nodeUpsert(userTurn)` + broadcast `NEW_TURN`
  - [ ] `window.__lucy = { sue }` console handle
- [ ] Vite config — WASM asset handling, Worker bundling
- [ ] Manual test checklist
  - [ ] Type message → response appears
  - [ ] Refresh → conversation persists
  - [ ] Console: `window.__lucy.sue.graph().centroidGet('self')` → C_o populated
  - [ ] Console: `window.__lucy.sue.ont().search(vec, 5)` → returns results

---

## Phase 5 — Gun mesh + wide sync

- [ ] Complete `src/sue/substrate/gun-mesh.ts`
  - [ ] `syncLogDrain()` timer — drains pending sync_log into Gun mind namespace
  - [ ] `mind.get('events').map().on(...)` — incoming event handler
  - [ ] `hydrateEvent(event)` — write incoming events to local GraphStore
  - [ ] `detectDivergence(event)` — check for belief conflict (Phase 6 handles resolution)
  - [ ] Centroid advertisement: `mind.get('instances').get(INSTANCE_ID).get('centroid').put(...)`
  - [ ] Peer centroid cache: observe `mind.get('instances').map()` for centroid updates

- [ ] AXE peer scoring hook
  - [ ] `Gun.on('opt', ...)` override of `axe.opt.peers`
  - [ ] Cosine similarity scoring against peer centroid cache
  - [ ] Matryoshka prefix (128-dim) for routing comparison

- [ ] Phase 5 tests
  - [ ] Two tabs, same SEA keypair → verify sync_log events propagate
  - [ ] Write belief node in tab A → verify it appears in tab B's IndexedDB
  - [ ] Verify peer centroid cache in tab B has tab A's C_o

---

## Phase 6 — Self-dialogue reconciliation

- [ ] Write `src/core/reconciliation.ts`
  - [ ] `detectDivergence(local, remote)` — content diff + valence threshold
  - [ ] `assemblePosition(node)` — context package: node + associated AWE + recent turns
  - [ ] `initiateReconciliation(local, remote, remoteInstanceId)`
  - [ ] Mesh subscription: `lucid:reconcile:${INSTANCE_ID}` handler
  - [ ] `handleReconciliationRequest(request)` → assemble own position → respond
  - [ ] `runDialogue(posA, posB)` — 2-3 round inference loop via `sue.inference().generate()`
  - [ ] `writeReconciliationNode(dialogue, winnerId, loserId)` — new node + provenance update

- [ ] Wire into gun-mesh.ts: `detectDivergence` called from incoming event handler

- [ ] Phase 6 tests
  - [ ] Manually write divergent nodes to two instances
  - [ ] Verify reconciliation request published to mesh
  - [ ] Verify dialogue runs (≥ 2 turns)
  - [ ] Verify reconciliation node in graph with edges to both originals
  - [ ] Verify loser provenance < original provenance

---

## Phase 7 — Device Lucy daemon

- [ ] `src/device/index.ts` — Node entry point
  - [ ] SUE boot with Lancedb + SQLite substrate
  - [ ] Same workers as browser (Node-compatible versions)
  - [ ] GunDB Node peer (acts as relay if configured)

- [ ] `src/device/dream-cycle.ts` — full consolidation (§12)
  - [ ] Crystallisation pass using Louvain community detection
  - [ ] Cap delta computation
  - [ ] Cap delta publication to Gun mesh

- [ ] `src/device/analytics.ts`
  - [ ] Louvain community detection over belief edge graph
  - [ ] PageRank for tour node importance weighting

- [ ] Phase 7 test: `node src/device/index.ts` starts, connects to mesh, receives events from browser tab

---

## Ongoing

- [ ] SUE interfaces are the only thing core logic imports — enforce via ESLint no-restricted-imports
- [ ] SEA keypair: never logged, never committed, loaded from env or secure store
- [ ] `window.__lucy` / `global.__lucy` handle always present, never removed
- [ ] Every new substrate implementation passes the Phase 1 contract test suite before use
