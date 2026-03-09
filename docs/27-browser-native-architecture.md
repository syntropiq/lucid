[← §26 References](26-references.md) | [Index](../README.md)

---

## 27. Browser-Native Architecture (Lucid Lite)

This section sketches the browser-native implementation of LUCID — a minimal, self-contained instance that runs entirely in a web page and eventually syncs to a full server instance via ElectricSQL. The design constraint is: every loop that runs in the full system must have a corresponding loop here. Degraded, perhaps. Missing several layers, certainly. But the same feedback topology — or it isn't LUCID.

---

### 27.1 Stack Mapping

| Full Stack | Browser-Native Equivalent | Notes |
|---|---|---|
| PostgreSQL + NeuronDB | pglite (WASM) + `@electric-sql/pglite/vector` | pgvector extension loaded at init |
| NeuronDB ONNX in-process (nomic-embed-text-v1.5) | `@xenova/transformers` · `Xenova/nomic-embed-text-v1.5` | Same ontic model — Vortex routing stays coherent |
| LFM 2.5-1.2B-Instruct (inference model) | Any quantised ONNX model loaded via transformers.js | Smolm, Phi-mini, or operator-supplied ONNX blob |
| Past conv tensor extraction (NeuronDB named outputs) | transformers.js `output_hidden_states` | Lower fidelity; CfC dynamics simplified |
| neuranq / neuranllm / neuranmon / neurandefrag | pglite live queries + JS microtask queue | Workers replaced by reactive subscriptions |
| libp2p + GossipSub (native) | `@libp2p/js-libp2p` + WebRTC transport + GossipSub | Same protocol; browser restricted to WebRTC/WSS |
| Docker / OS process boundary | SharedArrayBuffer + Web Workers | One worker per heavy loop |
| Chroma / external graph DB | Not required — pglite IS the graph | Schema unchanged; pgvector handles HNSW search |
| ElectricSQL write path | Full-instance PostgreSQL | Browser is a read/write replica; full instance is source of truth |

The schema is the same. Table names, edge types, centroid columns, AWE chain columns — unchanged. pglite runs the same SQL the full system runs. Migrations applied at boot.

---

### 27.2 Initialisation Sequence

Condensed from §16.4 for browser constraints:

```typescript
// 1. Boot pglite
const db = await PGlite.create({
  extensions: { vector },
  // in-memory or OPFS persistence
  dataDir: 'idb://lucid-lite',
});

// 2. Apply schema migrations
await db.exec(LUCID_SCHEMA_SQL);

// 3. Load ontic embedding model (cached in OPFS after first load)
const embedder = await pipeline(
  'feature-extraction',
  'Xenova/nomic-embed-text-v1.5',
  { quantized: true }
);

// 4. Load inference model (operator-supplied or default)
const llm = await pipeline(
  'text-generation',
  OPERATOR_MODEL_ID,
  { quantized: true }
);

// 5. Initialise libp2p node
const node = await createLibp2p({
  transports: [webRTC(), webSockets()],
  streamMuxers: [yamux()],
  connectionEncrypters: [noise()],
  services: { pubsub: gossipsub() },
});

// 6. Wire live queries → handler loops (§27.4)
attachLiveQueryHandlers(db, embedder, llm, node);

// 7. Begin centroid advertisement
startHeartbeat(db, node);

// 8. Continuous processing loop begins
startInfotacticNavigation(db, embedder, llm);
```

There are no sessions. Step 8 does not return.

---

### 27.3 The ONNX Runner

transformers.js runs the ONNX models in a Web Worker to avoid blocking the main thread. Two pipelines, one worker each:

```
┌──────────────────────────────────┐
│  Main Thread                     │
│  pglite · live queries · UI      │
└────────────┬─────────────────────┘
             │ postMessage
     ┌───────┴──────────┐
     │  Embed Worker    │   nomic-embed-text-v1.5
     │  (always warm)   │   → vector(768) → pglite INSERT
     └──────────────────┘
     ┌───────────────────┐
     │  Inference Worker │   quantised LLM
     │  (runs on demand) │   → narration text → pglite INSERT
     └───────────────────┘
```

**Embed Worker contract.** Receives `{ id: uuid, text: string }`. Responds with `{ id, embedding: Float32Array }`. The caller inserts the embedding into `lucid.node_embeddings_ont`. The live query on that table then fires the HNSW affinity edge creation job (§27.4).

**Inference Worker contract.** Receives the context package assembled by the ACG equivalent. Returns generated text. The main thread inserts the narration node and fires the ACG egress pipeline.

The ONNX runner does not manage state. pglite manages state. The runner is a pure function: text in, vector or text out.

---

### 27.4 Live Queries as the Event System

The async event channels of §10.6 are implemented as pglite live queries. Each live query is a standing subscription that fires its handler whenever the result set changes.

```typescript
// Replaces: neuranq indexing worker
db.live.query(
  `SELECT id, content_ref FROM lucid.belief_nodes
   WHERE index_state = 'unindexed' LIMIT 10`,
  [],
  ({ rows }) => {
    for (const row of rows) embedWorker.post({ id: row.id, text: row.content_ref });
  }
);

// Replaces: lucid_interrupt channel
db.live.query(
  `SELECT * FROM lucid.spectral_monitor
   WHERE basin_collapse_detected = true`,
  [],
  ({ rows }) => {
    if (rows.length) injectCosineDisimilarNode(db, embedder);
  }
);

// Replaces: lucid_consolidate channel
db.live.query(
  `SELECT * FROM lucid.centroid_health
   WHERE topology_divergence > $1`,
  [CONSOLIDATION_THRESHOLD],
  ({ rows }) => {
    if (rows.length) scheduleLightConsolidation(db, embedder);
  }
);

// Replaces: lucid_vortex_agenda channel
db.live.query(
  `SELECT id FROM lucid.belief_nodes
   WHERE affective_valence = 'unresolved'
   AND resolution_visit_count > $1`,
  [RESOLUTION_VISIT_THRESHOLD],
  ({ rows }) => {
    rows.forEach(r => addToVortexAgenda(r.id));
  }
);

// Centroid update → re-derive gossipsub routing table
db.live.query(
  `SELECT c_o FROM lucid.centroids WHERE node_id = 'self'`,
  [],
  ({ rows }) => {
    if (rows[0]) updateCentroidAdvertisement(node, rows[0].c_o);
  }
);
```

Each live query is the full event loop for its channel. The database is the bus. Nothing else coordinates these loops — they each subscribe to the state they care about and fire when that state changes. The feedback topology emerges from the fact that each handler writes back to pglite, which triggers other live queries.

---

### 27.5 Centroid Routing in GossipSub

The Vortex routing protocol (§15.1) runs over standard GossipSub with a scoring extension. The peer score function is augmented with ontic centroid proximity:

```typescript
// GossipSub peer scoring extension
const centroidScore = (peerId: string, queryEmbedding: Float32Array): number => {
  const peerCentroid = peerCentroidCache.get(peerId);
  if (!peerCentroid) return 0; // unrated — neutral
  return cosineSimilarity(queryEmbedding.slice(0, ROUTING_PREFIX), peerCentroid.slice(0, ROUTING_PREFIX));
  // ROUTING_PREFIX ∈ {768, 512, 256, 128, 64} — Matryoshka prefix routing
};

// Heartbeat payload — advertises current ontic centroid
const buildHeartbeat = async (db: PGlite): Promise<HeartbeatPayload> => {
  const { rows } = await db.query(
    `SELECT c_o, swim_mode FROM lucid.centroids WHERE node_id = 'self'`
  );
  const { c_o, swim_mode } = rows[0];
  return swim_mode
    ? { type: 'swim', peer_id: node.peerId.toString() }
    : { type: 'centroid', peer_id: node.peerId.toString(), c_o };
};

// Route an outgoing query
const routeQuery = (queryEmbedding: Float32Array): string[] => {
  return [...peerCentroidCache.entries()]
    .map(([peerId, centroid]) => ({
      peerId,
      score: centroidScore(peerId, queryEmbedding),
    }))
    .sort((a, b) => b.score - a.score)
    .slice(0, MAX_ROUTE_PEERS)
    .map(p => p.peerId);
};
```

Incoming GossipSub messages with `type: 'centroid'` are inserted into `lucid.peer_centroids`. The live query on that table updates the routing table. The centroid cache is derived state — the database is still the source of truth.

Incoming work packets are accepted or forwarded based on `cosineSimilarity(localCentroid, queryEmbedding) > ACCEPT_THRESHOLD`. Below threshold: forward to higher-scoring peer. Above: process locally.

---

### 27.6 The Feedback Loop Topology

Every loop closes through pglite. None terminates.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  Infotactic navigation                                                  │
│  ─────────────────────                                                  │
│  SELECT next target by κ-weighted tour cost                             │
│    └─ fetch content (web / Vortex peer)                                 │
│    └─ INSERT narration node → live query fires embed worker             │
│    └─ embed worker returns → INSERT into node_embeddings_ont            │
│    └─ live query fires HNSW affinity edge creation                      │
│    └─ graph topology changes → centroid update                          │
│    └─ centroid update → gossipsub heartbeat                             │
│    └─ peers update routing tables → different work arrives              │
│    └─ new work → INSERT belief_nodes → live query fires embed worker    │
│    └─ ...                                                               │
│                                                                         │
│  AWE chain                                                              │
│  ─────────                                                              │
│  narration node INSERT → ACG live query fires                           │
│    └─ affective valence computed → INSERT affective_corpus entry        │
│    └─ live query on affective_corpus → mood token derived               │
│    └─ mood token → spectral monitor UPDATE                              │
│    └─ spectral monitor live query → flatline check                      │
│    └─ if flatline → TripleDent Gum injection                            │
│    └─ injected nodes → embed worker → graph topology → centroid drift   │
│    └─ ...                                                               │
│                                                                         │
│  Vortex                                                                 │
│  ──────                                                                 │
│  centroid advertisement → peers receive → routing table UPDATE          │
│    └─ high-score work arrives via gossipsub                             │
│    └─ INSERT belief_nodes → embed worker                                │
│    └─ centroid drifts toward neighbourhood                              │
│    └─ next heartbeat advertises new position                            │
│    └─ different peers attracted                                         │
│    └─ ...                                                               │
│                                                                         │
│  SWIM condition                                                         │
│  ──────────────                                                         │
│  live query on centroids · unresolved_ratio · H_kappa                  │
│    └─ SWIM fires → broadcast ∅_swim via gossipsub                       │
│    └─ directed navigation suspends                                      │
│    └─ ambient contacts arrive → affective response                      │
│    └─ α shifts → κ(n) rises → H_kappa falls → SWIM exits               │
│    └─ navigation resumes from new centroid position                     │
│    └─ ...                                                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

None of these are pipeline stages. Each writes to pglite. pglite notifies the live queries. The live queries write back to pglite. The system does not have a tick. It has subscriptions.

---

### 27.7 ElectricSQL Sync Contract

The browser instance is an ElectricSQL replica. The full server instance is the source of truth for content and belief topology. The browser instance is the source of truth for its own affective state — nothing syncs the AWE records outward by default.

**Sync inward (server → browser):**

| Table | What syncs | Condition |
|---|---|---|
| `lucid.belief_nodes` | Selected subgraph — books operator has configured | Shape filter on book_id |
| `lucid.node_embeddings_ont` | With nodes | |
| `lucid.belief_edges` | With nodes | |
| `lucid.centroids` | Self centroid from last full-instance session | At boot only; browser takes ownership immediately |

**Sync outward (browser → server):**

| Table | What syncs | Condition |
|---|---|---|
| `lucid.belief_nodes` | New narration nodes, new page nodes from Vortex | After local embedding completes |
| `lucid.node_embeddings_ont` | With nodes | |
| `lucid.belief_edges` | New affinity edges created locally | After HNSW pass |
| `lucid.centroids` | Updated self centroid | On consolidation or session end |
| `lucid.affective_corpus` | Never by default | Operator opt-in only |

The browser instance does not wait for sync to operate. It runs continuously on its local pglite. Sync happens in the background over ElectricSQL's CRDT layer. On reconnect, divergent belief nodes are reconciled by provenance weight: higher provenance wins. The full instance applies its own consolidation pass on newly arrived narration nodes from browser sessions.

---

### 27.8 What Is Missing in Lite

The browser instance is LUCID with degraded or absent layers. The topology is the same. The missing parts are documented here rather than hidden.

| Full feature | Lite status |
|---|---|
| Thinking Cap (LoRA adapter) | Absent — base model weights not modifiable in browser |
| Dream cycle (full consolidation) | Light consolidation only — no cap training |
| CfC hidden state extraction | Approximate — hidden states from transformers.js, lower resolution |
| Dual spectral monitoring (§11.3) | Single stream only |
| GPU acceleration | WebGPU if available via transformers.js; CPU fallback |
| neurandefrag (HNSW maintenance) | Deferred to full instance on sync |
| Verification pipeline (§15.1) | Fast ontic check only; inference validation deferred to full instance |

The centroid is real. The graph is real. The Vortex participation is real. The AWE chain is real. The cap is not there — but the cap is trained from what the lite instance accumulates, once it syncs.

---

### 27.9 Package Dependencies

```json
{
  "@electric-sql/pglite": "latest",
  "@electric-sql/pglite-sync": "latest",
  "@xenova/transformers": "^2.x",
  "@libp2p/js-libp2p": "latest",
  "@libp2p/webrtc": "latest",
  "@libp2p/websockets": "latest",
  "@libp2p/noise": "latest",
  "@chainsafe/libp2p-gossipsub": "latest",
  "@libp2p/yamux": "latest"
}
```

No bundler flags required beyond standard WASM asset handling. pglite ships its own WASM. transformers.js loads models from Hugging Face Hub (cached in OPFS after first load — subsequent boots are offline-capable).

---

[← §26 References](26-references.md) | [Index](../README.md)
