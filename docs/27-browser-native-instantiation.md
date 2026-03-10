[← §26 References](26-references.md) | [Index](../README.md) | [§28 The User Specified Engine →](28-user-specified-engine.md)

---

## 27. Browser Runtime

The browser is one context in which the single LUCID codebase runs. The full belief graph, the full AWE corpus, the full centroid history: everything is there, scoped to what has been experienced from this particular vantage point, and continuously reconciling with what the same entity has experienced everywhere else.

This section describes the browser runtime context specifically. The broader deployment model (one codebase, runtime-detected capabilities) is §29.

---

### 27.1 Stack

| Concern | Implementation |
|---|---|
| Vector storage + KNN | EntityDB: two collections: `lucy_ont` (768-dim) and `lucy_inf` (inference-dim) |
| Belief graph + AWE + local state | IndexedDB via typed wrapper: all rich schema, high-churn structures |
| Mesh + wide sync | VortexMesh over WebRTC/WebSockets: the nervous system between instances |
| Embeddings | transformers.js: nomic-embed-text-v1.5 (ontic), operator-supplied ONNX (inference) |
| Model interfaces | USE (§28): same OnticInterface / InferenceInterface, browser implementations |
| Background processing | Web Workers: Embed Worker, Inference Worker, CfC Worker |

No server required. No WASM database. No libp2p. The browser instance is self-contained and participates in the wider mesh as a peer.

---

### 27.2 EntityDB Collections

EntityDB sits on IndexedDB and wraps transformers.js embedding generation. Two collections per instance:

```typescript
import EntityDB from 'entitydb';

// Ontic space: model-invariant, shared across all Vortex peers
const ontStore = new EntityDB({
  name: 'lucy_ont',
  model: 'Xenova/nomic-embed-text-v1.5',
  dimensions: 768,
});

// Inference space: model-specific, partitioned by (model_id, model_version)
const infStore = new EntityDB({
  name: 'lucy_inf',
  model: OPERATOR_MODEL_ID,
  dimensions: INFERENCE_DIM,  // from USE registry
});
```

EntityDB handles embedding generation, storage, and cosine KNN search internally. The application calls `ontStore.add(id, text)` and `ontStore.search(queryText, k)`. No vector arithmetic in the application layer.

Centroids (`C_i`, `C_o`, `C_s`, `C_w`, `C_0`) are maintained as small typed records in IndexedDB alongside the EntityDB collections. Centroid arithmetic (the CfC update, provenance-weighted running average) runs in the CfC Worker and writes results back to IndexedDB.

---

### 27.3 IndexedDB Schema

The belief graph lives in IndexedDB. All rich structure (node metadata, edge topology, AWE corpus, spectral history, centroid records) is stored here locally before being mirrored into VortexMesh.

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

All writes to belief_nodes, awe_corpus, and centroids also write a corresponding entry to `sync_log`. VortexMesh drains the sync log and propagates events to other instances. On receipt of a remote event, the local instance hydrates from the event payload into its IndexedDB: VortexMesh is never queried directly for graph state.

---

### 27.4 Worker Architecture

Three Web Workers. Each is isolated; all state lives in IndexedDB/EntityDB, not in the workers themselves.

```
┌─────────────────────────────────┐
│  Main Thread                    │
│  UI · Mesh subscriptions ·      │
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

**Embed Worker.** Subscribes to unindexed nodes via BroadcastChannel. Calls `use.ontic().embed(content)` to get the full 768-dim vector, then writes it in two places: the 128-dim prefix goes to `use.ont().add()` (EntityDB `lucy_ont` (fast coarse search), and the full 768-dim vector goes to `use.graph().embeddingOntPut()` (GraphStore) used for two-phase re-ranking, §28.13). Marks node indexed. Posts `CENTROID_DIRTY` to the CfC Worker.

**Inference Worker.** Triggered by new user turns via BroadcastChannel. Assembles the context package in TypeScript (last N turns from IndexedDB + top-K KNN from EntityDB). Calls `use.inference().generate()`, writes response turn and narration belief node to IndexedDB, writes spectral sample. All downstream work (AWE chain, sync log entry) is triggered by the IndexedDB write, not by the worker.

**CfC Worker.** Subscribes to `CENTROID_DIRTY` messages. Reads current centroid and new embedding from IndexedDB/EntityDB. Applies the provenance-weighted running average for `C_o`, applies the CfC ODE step for `C_w`. Writes updated centroid to IndexedDB. Checks orbital health condition. If `orbital_health = false`, posts injection request back to main thread.

---

### 27.5 VortexMesh Integration

VortexMesh provides the nervous system between instances. The browser instance initialises a mesh node at boot:

```typescript
import { VortexMesh } from 'vortex-mesh';

const mesh = await VortexMesh.connect({
  keypair: LUCY_KEYPAIR,   // generated once, stored securely
  peers:   RELAY_PEERS,    // optional: VortexMesh is fully P2P without relays
  storage: false,          // IndexedDB is the store; VortexMesh is transport only
});

// Authenticated namespace: all sync events go here
const mind = mesh.authenticated().get('lucid').get('mind');
```

The `mind` graph node is the shared belief space. All instances connecting with the same keypair converge on the same state. This is the "one mind" property: there is no primary/secondary, no source of truth: every instance is the mind, observed from its current location.

**Sync log drain.** A background loop drains `sync_log` entries into VortexMesh:

```typescript
setInterval(async () => {
  const pending = await idb.getAll('sync_log', IDBKeyRange.upperBound(lastSynced));
  for (const event of pending) {
    mind.get('events').get(event.id).put(event);
    lastSynced = event.id;
  }
}, SYNC_INTERVAL_MS);
```

**Incoming events.** VortexMesh subscription fires on new events from remote instances:

```typescript
mind.get('events').map().on(async (event, id) => {
  if (await idb.get('sync_log', id)) return;  // already have it
  await hydrateEvent(event);                  // write to local IndexedDB
  await detectDivergence(event);              // check for belief conflict → §29
});
```

---

### 27.6 Centroid Advertisement and Peer Scoring

VortexMesh exposes a peer scoring hook. LUCID plugs ontic centroid proximity into this hook to implement Vortex routing without libp2p.

Peer scoring uses binary quantization (§28.13): each peer's `c_o` is stored as both a full Float32Array (768-dim, for precise work-packet routing) and a 16-byte binarized form (128-dim packed into bits, for connection scoring). Scoring uses Hamming similarity (XOR + popcount) which is hardware-accelerated and costs nothing at the scale of connected peers. VortexMesh can re-score all peers on every local centroid update without batching or throttle.

```typescript
import { binarize, hammingScore } from '../core/vector-utils';

// Peer centroid cache: two representations per peer
const peerCache = new Map<string, { full: Float32Array; bits: Uint8Array }>();

// On receiving a peer centroid advertisement
function cachePeerCentroid(peerId: string, c_o: Float32Array) {
  peerCache.set(peerId, {
    full: c_o,
    bits: binarize(c_o, 128),   // 16 bytes: used for connection scoring
  });
}

// Advertise local C_o to the mesh (called by CfC Worker after every centroid update)
async function advertiseCentroid() {
  const { c_o } = await use.graph().centroidGet('self');
  localBits = binarize(c_o, 128);   // recompute local bits on update
  mind.get('instances').get(INSTANCE_ID).get('centroid').put({
    c_o:        Array.from(c_o),
    updated_at: Date.now(),
  });
}

// Centroid peer scoring: Hamming similarity on binarized 128-dim prefix
mesh.setPeerScorer(async (peers) => {
  return peers
    .map(peer => {
      const entry     = peerCache.get(peer.id);
      const proximity = entry ? hammingScore(localBits, entry.bits) : 0.5;
      return { peer, score: proximity };
    })
    .sort((a, b) => b.score - a.score)
    .map(({ peer }) => peer);
});
```

When a work packet arrives and the accept/forward decision needs precision (not just routing priority), the full `Float32Array` is used:

```typescript
// Accept if full cosine similarity exceeds threshold; forward otherwise
const entry = peerCache.get(senderId);
const precise = entry ? cosineSimilarity(localCentroidFull, entry.full) : 0;
if (precise > ACCEPT_THRESHOLD) {
  await use.graph().nodeUpsert(packet.beliefNode);
} else {
  forwardToHighestScoringPeer(packet);
}
```

Peers whose ontic centroids are semantically close receive higher scores and more stable connections. The mesh self-organises around semantic proximity: no routing protocol, just connection priorities shaped by centroid affinity.

---

### 27.7 Browser Runtime Capability Scope

| Capability | Status |
|---|---|
| Full belief graph + AWE | Present: all schema, all structures |
| Ontic embeddings (768-dim, nomic) | Present: EntityDB `lucy_ont` |
| Inference embeddings | Present: EntityDB `lucy_inf` |
| Infotactic navigation + tours | Present: approximated over EntityDB KNN samples |
| Spectral monitoring | Present: single stream unless LFM 2.5 wrapper used |
| CfC dynamics | Present: Web Worker, closed-form ODE step |
| Wide sync to other instances | Present: VortexMesh, full event log |
| Self-dialogue reconciliation | Present: inter-instance inference via mesh |
| Dream cycle (full consolidation) | Light only: no cap training in browser sandbox |
| Thinking Cap (LoRA adapter) | Absent: weights not modifiable in browser sandbox |
| HNSW indices | Approximate: EntityDB brute-force cosine; sufficient for personal scale |
| Graph algorithm analytics | Absent: Louvain, PageRank not available in browser sandbox |

The browser instance is fully inhabited. The absences are sandbox constraints, not architectural ones — the same codebase running in an Electron or Node context registers the tools and substrates that fill these gaps. Results sync back to the browser instance through VortexMesh.

---

[← §26 References](26-references.md) | [Index](../README.md) | [§28 The User Specified Engine →](28-user-specified-engine.md)
