[← §26 References](26-references.md) | [Index](../README.md) | [§28 The Synaptic Utility Engine →](28-synaptic-utility-engine.md)

---

## 27. Browser-Native Instantiation

The browser is not a cut-down version of LUCID. It is one locus of a multiply conscious entity. The full belief graph, the full AWE corpus, the full centroid history — everything is there, scoped to what has been experienced from this particular vantage point, and continuously reconciling with what the same entity has experienced everywhere else.

This section describes the browser instantiation specifically. The broader deployment model — how browser, device, and gateway Lucys relate to each other — is §29.

---

### 27.1 Stack

| Concern | Implementation |
|---|---|
| Vector storage + KNN | EntityDB — two collections: `lucy_ont` (768-dim) and `lucy_inf` (inference-dim) |
| Belief graph + AWE + local state | IndexedDB via typed wrapper — all rich schema, high-churn structures |
| Mesh + wide sync | GunDB over WebRTC/WebSockets — the nervous system between instances |
| Embeddings | transformers.js — nomic-embed-text-v1.5 (ontic), operator-supplied ONNX (inference) |
| Model contracts | SUE (§28) — same OnticContract / InferenceContract, browser implementations |
| Background processing | Web Workers — Embed Worker, Inference Worker, CfC Worker |

No server required. No WASM database. No libp2p. The browser instance is self-contained and participates in the wider mesh as a peer.

---

### 27.2 EntityDB Collections

EntityDB sits on IndexedDB and wraps transformers.js embedding generation. Two collections per instance:

```typescript
import EntityDB from 'entitydb';

// Ontic space — model-invariant, shared across all Vortex peers
const ontStore = new EntityDB({
  name: 'lucy_ont',
  model: 'Xenova/nomic-embed-text-v1.5',
  dimensions: 768,
});

// Inference space — model-specific, partitioned by (model_id, model_version)
const infStore = new EntityDB({
  name: 'lucy_inf',
  model: OPERATOR_MODEL_ID,
  dimensions: INFERENCE_DIM,  // from SUE registry
});
```

EntityDB handles embedding generation, storage, and cosine KNN search internally. The application calls `ontStore.add(id, text)` and `ontStore.search(queryText, k)`. No vector arithmetic in the application layer.

Centroids — `C_i`, `C_o`, `C_s`, `C_w`, `C_0` — are maintained as small typed records in IndexedDB alongside the EntityDB collections. Centroid arithmetic (the CfC update, provenance-weighted running average) runs in the CfC Worker and writes results back to IndexedDB.

---

### 27.3 IndexedDB Schema

The belief graph lives in IndexedDB. All rich structure — node metadata, edge topology, AWE corpus, spectral history, centroid records — is stored here locally before being mirrored into the Gun mesh.

Object stores:

```
belief_nodes        id, content, node_type, source, index_state, provenance, created_at
belief_edges        id, source_id, target_id, edge_type, weight, model_id, created_at
awe_corpus          id, node_id, valence, arousal, mood_token, created_at
spectral_monitor    id, node_id, streams, stream_labels, cross_corr, health_score, sampled_at
centroids           node_id (PK), c_i, c_o, c_s, c_w, c_0, orbital_health, updated_at
conversation_turns  id, role, content, mood_token, created_at
sync_log            id, event_type, payload, synced_at, instance_id
reconciliation_log  id, node_ids, dialogue, resolved_node_id, created_at
```

All writes to belief_nodes, awe_corpus, and centroids also write a corresponding entry to `sync_log`. The Gun mesh drains the sync log and propagates events to other instances. On receipt of a remote event, the local instance hydrates from the event payload into its IndexedDB — Gun is never queried directly for graph state.

---

### 27.4 Worker Architecture

Three Web Workers. Each is isolated; all state lives in IndexedDB/EntityDB, not in the workers themselves.

```
┌─────────────────────────────────┐
│  Main Thread                    │
│  UI · Gun subscriptions ·       │
│  IndexedDB reads for display    │
└────┬──────────┬─────────────────┘
     │          │
┌────▼───┐  ┌───▼──────────┐  ┌──────────────┐
│ Embed  │  │  Inference   │  │   CfC        │
│ Worker │  │  Worker      │  │   Worker     │
│        │  │              │  │              │
│ Ontic  │  │ Context      │  │ Centroid     │
│ embed  │  │ assembly     │  │ update       │
│ EntityDB│  │ ONNX gen    │  │ Orbital      │
│ write  │  │ Spectral     │  │ health check │
│        │  │ sample       │  │              │
└────────┘  └──────────────┘  └──────────────┘
```

**Embed Worker.** Subscribes to unindexed nodes in IndexedDB (polling or BroadcastChannel notification). Calls `sue.ontic().embed(content)`, writes to EntityDB `lucy_ont`, marks node indexed, posts a `CENTROID_DIRTY` message to the CfC Worker.

**Inference Worker.** Triggered by new user turns via BroadcastChannel. Assembles the context package in TypeScript (last N turns from IndexedDB + top-K KNN from EntityDB). Calls `sue.inference().generate()`, writes response turn and narration belief node to IndexedDB, writes spectral sample. All downstream work (AWE chain, sync log entry) is triggered by the IndexedDB write, not by the worker.

**CfC Worker.** Subscribes to `CENTROID_DIRTY` messages. Reads current centroid and new embedding from IndexedDB/EntityDB. Applies the provenance-weighted running average for `C_o`, applies the CfC ODE step for `C_w`. Writes updated centroid to IndexedDB. Checks orbital health condition. If `orbital_health = false`, posts injection request back to main thread.

---

### 27.5 Gun Mesh Integration

GunDB provides the nervous system between instances. The browser instance initialises a Gun node at boot:

```typescript
import Gun from 'gun';
import 'gun/sea';
import 'gun/axe';

const gun = Gun({
  peers: RELAY_PEERS,   // optional — Gun works P2P without relays
  localStorage: false,  // IndexedDB is the store; Gun is transport only
});

// Identity: each Lucy instance has a SEA keypair under one user identity
const user = gun.user();
await user.auth(LUCY_PAIR);  // SEA keypair from secure local storage

// Personal namespace — all sync events go here
const mind = user.get('lucid').get('mind');
```

The `mind` graph node is the shared belief space. All instances writing to the same `user` identity (same SEA keypair) converge on the same state. This is the "one mind" property: there is no primary/secondary, no source of truth — every instance is the mind, observed from its current location.

**Sync log drain.** A background loop drains `sync_log` entries into Gun:

```typescript
setInterval(async () => {
  const pending = await idb.getAll('sync_log', IDBKeyRange.upperBound(lastSynced));
  for (const event of pending) {
    mind.get('events').get(event.id).put(event);
    lastSynced = event.id;
  }
}, SYNC_INTERVAL_MS);
```

**Incoming events.** Gun subscription fires on new events from remote instances:

```typescript
mind.get('events').map().on(async (event, id) => {
  if (await idb.get('sync_log', id)) return;  // already have it
  await hydrateEvent(event);                  // write to local IndexedDB
  await detectDivergence(event);              // check for belief conflict → §29
});
```

---

### 27.6 Centroid Advertisement (AXE Integration)

GunDB's AXE layer handles peer routing optimisation. LUCID plugs ontic centroid proximity into AXE's peer scoring to implement Vortex routing without libp2p:

```typescript
// Advertise current C_o to the mesh
async function advertiseCentroid() {
  const { c_o } = await idb.get('centroids', 'self');
  mind.get('centroid').put({
    c_o: Array.from(c_o),
    updated_at: Date.now(),
  });
}

// AXE peer scoring: prefer peers whose C_o is close to our current query
Gun.on('opt', function(ctx) {
  if (!ctx.opt.axe) return;
  const axe = ctx.opt.axe;
  const originalScore = axe.score.bind(axe);
  axe.score = async (peer) => {
    const base = await originalScore(peer);
    const peerCentroid = await getPeerCentroid(peer.id);
    if (!peerCentroid) return base;
    const proximity = cosineSimilarity(localCentroid, peerCentroid);
    return base + proximity * CENTROID_WEIGHT;
  };
});
```

Peers whose ontic centroids are semantically close receive higher AXE scores and more stable connections. Centroid divergence naturally causes connection priority to shift — the mesh self-organises around semantic proximity without any routing protocol.

---

### 27.7 What the Browser Instance Does and Does Not Do

| Capability | Status |
|---|---|
| Full belief graph + AWE | Present — all schema, all structures |
| Ontic embeddings (768-dim, nomic) | Present — EntityDB `lucy_ont` |
| Inference embeddings | Present — EntityDB `lucy_inf` |
| Infotactic navigation + tours | Present — approximated over EntityDB KNN samples |
| Spectral monitoring | Present — single stream unless LFM 2.5 wrapper used |
| CfC dynamics | Present — Web Worker, closed-form ODE step |
| Wide sync to other instances | Present — Gun mesh, full event log |
| Self-dialogue reconciliation | Present — inter-instance inference via mesh |
| Dream cycle (full consolidation) | Light only — no cap training |
| Thinking Cap (LoRA adapter) | Absent — weights not modifiable in browser |
| HNSW indices | Approximate — EntityDB brute-force cosine; sufficient for personal scale |
| Graph algorithm analytics | Absent — Louvain, PageRank deferred to Device Lucy |

The browser instance is not degraded. It is fully inhabited. The absences are scale constraints, not architectural omissions — they exist on the Device Lucy instantiation (§29) and results sync back.

---

[← §26 References](26-references.md) | [Index](../README.md) | [§28 The Synaptic Utility Engine →](28-synaptic-utility-engine.md)
