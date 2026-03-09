# LUCID — Implementation Plan

> One mind, multiple instantiations. The substrate changes. The loops do not.

The build order moves from the smallest complete feedback loop outward. Each phase produces something runnable before the next begins.

---

## The Stack

| Concern | Browser Lucy | Device Lucy |
|---|---|---|
| Vector storage (ontic) | EntityDB `lucy_ont` | Lancedb |
| Vector storage (inference) | EntityDB `lucy_inf` | Lancedb |
| Graph / belief state | IndexedDB (typed wrapper) | SQLite (better-sqlite3) |
| P2P sync + mesh | GunDB (WebRTC/WSS) | GunDB (Node) |
| Embeddings | transformers.js (ONNX) | transformers.js or Ollama |
| Identity | GunDB SEA keypair | Same keypair |
| Peer routing | AXE + centroid scoring | AXE + centroid scoring |

No PostgreSQL. No libp2p. No WASM database. The same LUCID feedback loop logic runs on both instantiation types — the substrate is what changes.

---

## The Core Bet

**USE is the seam.** All feedback loops call `use.ont()`, `use.inf()`, `use.graph()`, `use.mesh()`. They never import EntityDB, IndexedDB, Lancedb, or GunDB directly. Swapping the substrate means changing the constructor passed to `use.registerSubstrate()` at boot. Nothing else changes.

---

## Phase 1 — USE substrate layer

**Goal:** The three substrate interfaces (`VectorStore`, `GraphStore`, `Mesh`) implemented and tested against real backing stores. No feedback loops yet. Just the plumbing that everything else will call.

### Deliverables

`src/sue/contracts/` — TypeScript interfaces: `VectorStore`, `GraphStore`, `Mesh`, `BeliefNode`, `BeliefEdge`, `CentroidRecord`, `AWEEntry`, `SpectralRecord`, `SyncEvent`

`src/sue/substrate/entitydb-vector-store.ts` — EntityDB implementation of `VectorStore`

`src/sue/substrate/indexeddb-graph-store.ts` — IndexedDB implementation of `GraphStore`
- Object stores: `belief_nodes`, `belief_edges`, `awe_corpus`, `spectral_monitor`, `centroids`, `conversation_turns`, `sync_log`, `reconciliation_log`
- Typed wrappers for all reads and writes — no raw IDBRequest anywhere in application code

`src/sue/substrate/gun-mesh.ts` — GunDB implementation of `Mesh`
- SEA auth at construction time
- `advertise` / `observe` map to `user.get('lucid').get('instances').get(instanceId)`
- `publish` / `subscribe` map to Gun topic nodes under the shared mind namespace

`src/sue/substrate/lancedb-vector-store.ts` — Lancedb implementation (Device Lucy, Node-only)

`src/sue/substrate/sqlite-graph-store.ts` — SQLite implementation (Device Lucy, Node-only)

### Tests

All four implementations tested against the same test suite via the interface. An implementation is correct if it passes the contract tests — the tests do not know which backing store they are running against.

---

## Phase 2 — Model wrappers + USE registry (from §28)

**Goal:** `use.ontic().embed()` and `use.inference().generate()` work. USE registry activates wrappers. The model layer is done and never touched again for substrate reasons.

### Deliverables

`src/sue/wrappers/ontic/nomic-embed-v1.5.ts`

`src/sue/wrappers/inference/generic-onnx.ts` — `makeGenericOnnxWrapper(config)`

`src/sue/wrappers/inference/lfm-2.5.ts` — full dual-stream implementation

`src/sue/registry.ts` — unified registry: model + substrate, `use.ontic()`, `use.inference()`, `use.ont()`, `use.inf()`, `use.graph()`, `use.mesh()`

### Tests

- `use.ontic().embed('hello world')` returns a `Float32Array` of length 768
- `use.inference().generate(messages)` returns a string
- `use.inference().spectralSample()` returns a valid `SpectralSample`
- Swap substrate implementations mid-test, verify same results

---

## Phase 3 — Embed + Inference Workers

**Goal:** Two Web Workers wired to the USE substrate. Inserting a belief node produces an ontic embedding. Inserting a user turn produces an assistant response. The feedback loop exists, unverified by UI.

### Embed Worker flow

```
belief_node written to GraphStore (index_state: 'unindexed')
    │ BroadcastChannel notification
    ▼
Embed Worker: use.ont().add(node.id, embedding)
    │
    ▼
GraphStore: nodeUpsert({ ...node, index_state: 'indexed' })
    │
    ▼
BroadcastChannel: 'CENTROID_DIRTY'
```

### Inference Worker flow

```
conversation_turn written (role: 'user')
    │ BroadcastChannel notification
    ▼
Inference Worker: assemble context (last 20 turns + top-5 KNN from ont store)
    │
    ▼
use.inference().generate(messages)
    │
    ▼
GraphStore: nodeUpsert (narration belief node)
GraphStore: awePut (spectral sample)
GraphStore: syncLogAppend (belief:node event)
    │
    ▼
BroadcastChannel: 'CENTROID_DIRTY'
```

### Tests

- Insert belief node → verify `lucy_ont` EntityDB has a vector for that ID within timeout
- Insert user turn → verify assistant turn appears in GraphStore within timeout
- Verify `sync_log` has entries after both operations

---

## Phase 4 — Chat interface

**Goal:** Open browser, type, get response. Everything under the hood verified by opening the browser's IndexedDB inspector.

### Deliverables

`src/app/main.ts` — boots USE (browser substrate), starts workers, wires BroadcastChannel, attaches UI

`src/app/index.html` — minimal: message list + input + health badge

`window.__lucy = { sue }` — surgical instrument for console debugging

### Reactive pattern

Without pglite live queries, reactivity is BroadcastChannel + targeted IndexedDB reads:

```typescript
const bc = new BroadcastChannel('lucid');
bc.onmessage = async (evt) => {
  if (evt.data.type === 'TURNS_UPDATED') {
    const turns = await use.graph().nodeQuery({ /* recent conversation */ });
    renderConversation(turns);
  }
  if (evt.data.type === 'HEALTH_UPDATED') {
    const spectral = await use.graph().spectralLatest();
    renderHealthBadge(spectral?.health_score);
  }
};
```

Workers broadcast after every write. The UI reads on broadcast. Simple, no polling.

### Tests (manual)

- Type message → response appears
- Refresh page → conversation persists from IndexedDB
- `window.__lucy.use.graph().centroidGet('self')` in console → C_o has moved

---

## Phase 5 — Gun mesh + wide sync

**Goal:** Two browser tabs (or browser + Node process) share the same belief graph. A belief node written in tab A appears in tab B within seconds.

### Deliverables

`src/sue/substrate/gun-mesh.ts` — complete implementation:
- SEA auth
- `syncLogDrain()` — drains pending sync_log entries into Gun mind namespace
- Incoming event handler → `hydrateEvent()` → `detectDivergence()` (§29.4)
- Centroid advertisement → `mind.get('instances').get(INSTANCE_ID).get('centroid').put(...)`

AXE peer scoring hook (§29.5):
- Override `axe.opt.peers` with centroid proximity sorting
- Peer centroid cache updated from Gun `instances` observations

### Tests

- Open two tabs with same SEA keypair
- Write belief node in tab A → verify it appears in tab B's IndexedDB within timeout
- Verify peer centroid cache in tab B has tab A's C_o

---

## Phase 6 — Self-dialogue reconciliation

**Goal:** When two instances diverge on a belief, they negotiate a reconciliation through inference. The reconciliation record appears in the graph. Both originals are preserved.

### Deliverables

`src/core/reconciliation.ts`:
- `detectDivergence(localNode, remoteNode)` — content diff + valence threshold
- `initiateReconciliation(local, remote, remoteInstanceId)` — publishes position statement to mesh
- `handleReconciliationRequest(request)` — assembles position, responds
- `runDialogue(positionA, positionB)` — 2-3 round inference loop
- `writeReconciliationNode(dialogue, winner, loser)` — new belief node, provenance update on loser

### Tests

- Manually write divergent belief nodes to two instances
- Verify reconciliation request published to mesh
- Verify dialogue runs (3 turns)
- Verify reconciliation node appears in graph with edges to both originals
- Verify loser's provenance reduced

---

## Phase 7 — Device Lucy daemon

**Goal:** `node src/device/index.ts` starts a Device Lucy with Lancedb + SQLite + GunDB, connects to the same mind namespace, and participates in the mesh as a heavier peer.

### Deliverables

`src/device/index.ts` — boot sequence with Lancedb + SQLite substrate

`src/device/dream-cycle.ts` — full consolidation loop (§12)

`src/device/analytics.ts` — Louvain community detection (for crystallisation), PageRank (for tour re-ordering)

Cap delta publication: after dream cycle, publish delta to mesh for Browser Lucy

### Not in scope for this phase

Gateway Lucy — that is a separate deployment with its own integration surface (OpenClaw or equivalent gateway API). Architecture is defined in §29.8; implementation depends on the gateway in use.

---

## Cross-cutting

- USE interfaces are the only thing the feedback loops import. No backing store leaks into core logic.
- SEA keypair is generated once, stored securely, never transmitted or logged.
- The `window.__lucy` (browser) / `global.__lucy` (device) handle is always present — never removed.
- Every phase has contract tests against the substrate interface, not the implementation.
