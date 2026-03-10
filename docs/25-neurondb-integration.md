[← §24 Tour Engine Architecture](24-tour-engine-architecture.md) | [Index](../README.md) | [§26 References →](26-references.md)

---

## 25. USE Substrate Boundaries

This section documents the explicit boundary between what LUCID delegates to USE substrate implementations and what remains under LUCID's semantic ownership. The principle is the same regardless of which substrate is active: substrate implementations handle storage, retrieval, and transport mechanics; LUCID owns all semantic interpretation.

### 25.1 Delegation to USE Substrates

| Capability | Substrate implementation |
|---|---|
| Ontic embedding generation | `OnticInterface.embed()`: nomic-embed-text-v1.5 via transformers.js ONNX |
| Inference embedding / hidden state extraction | `InferenceInterface.generate()` + `spectralSample()`: LFM 2.5 or generic ONNX |
| Coarse KNN search (prefix) | `VectorStore.search()`: EntityDB (browser) or Lancedb (device) |
| Full-vector storage for rerank | `GraphStore.embeddingOntPut/Get()`, `embeddingInfPut/Get()` |
| Belief node and edge persistence | `GraphStore.nodeUpsert()`, `edgeUpsert()`, `edgesFor()` |
| Centroid persistence | `GraphStore.centroidPut/Get()` |
| AWE and spectral record persistence | `GraphStore.awePut()`, `spectralPut()` |
| Sync log durability | `GraphStore.syncLogAppend()`, `syncLogPending()` |
| Wide sync transport | `Mesh.publish()`, `subscribe()`: VortexMesh CRDT transport |
| Peer discovery and connection management | `Mesh.advertise()`, `observe()`, `peers()`: VortexMesh centroid routing |
| HNSW index maintenance (device) | Lancedb internal: compaction and rebuild handled automatically |
| Graph analytics (dream cycle) | `src/device/analytics.ts`: Louvain community detection, PageRank |

### 25.2 Retained in LUCID

The following capabilities remain under LUCID's semantic ownership regardless of which substrate is active:

| Capability | Why LUCID owns it |
|---|---|
| Belief node schema, edges, centroids, cycles | Core domain model |
| Provenance model and belief weight arithmetic | Semantic layer; substrates have no concept of provenance |
| Hebbian-Belief cost function (Definition 9.1) | LUCID-specific scoring |
| CfC attractor semantics | Pinned `C_0`, liquid time-constant response curve, consolidation trigger logic, orbital trajectory analysis, contact record interpretation: all LUCID-specific interpretations of vector arithmetic primitives |
| Dual tour protocol | Algorithm definition, overlap set `Ω`, injection signal, threat score decomposition |
| Dream cycle sequence | Steps 1–7 and their ordering logic, including Step 4d affective reflection |
| Thinking Cap lifecycle | LoRA adapter generation, soft correction, cap rollback |
| Inheritance corpus and Self Library | Judgment record and anomaly detection reference surface |
| AWE layer | Affective chain generation (ingress/egress), mood gauge, affective valence, user react semantics, curiosity navigation, emotional memory, affective corpus, circadian texture, TripleDent Gum: all LUCID-specific; substrates have no concept of affect |
| Spectral monitoring | Dual-stream FFT, cross-stream correlation, DC dominance detection, orbital health condition, TripleDent Gum tier assessment: all LUCID-specific; substrates have no concept of attentional state |
| ACG continuous monitoring | Ingress/egress chain generation, conflict detection, context integration, affective assessment: all LUCID-specific; substrates have no concept of continuous attentional monitoring |
| Vortex peer layer | Peer mesh participation, centroid advertisement, bilateral contracts, swimming trigger: all LUCID-specific; substrates handle transport only |
| Swimming mechanism | SWIM trigger condition, `∅_swim` broadcast, open receptivity processing, exit condition: LUCID-specific |
| Network Hebbian equivalence | Interpretation of `C_o` as network Hebbian weight: LUCID-specific semantics on top of substrate vector arithmetic |
| Vortex agenda | Set of nodes flagged for peer contemplation, maintained by ACG: LUCID-specific |
| Bilateral AWE contracts | Peer exchange chain storage, γ history, `[jellyfish]`/`[excellent]`/`[ack]` semantics: LUCID-specific |
| Self-dialogue reconciliation | Divergence detection, dialogue protocol, reconciliation node creation (§29.4): LUCID-specific |

### 25.3 Worker and Runtime Responsibilities

The substrate layer does not drive the operational loop: LUCID's workers do.

**Embed Worker:** Listens for `NEW_NODE` broadcasts. Calls `use.ontic().embed()` and `use.inference()` to generate embeddings. Writes prefix to `use.ont()` (VectorStore) and full vector to `use.graph()` (GraphStore). Upserts the node with updated `index_state`. Broadcasts `NODE_INDEXED`.

**Inference Worker:** Listens for `NEW_TURN` broadcasts. Assembles context from the GraphStore. Calls `use.inference().generate()` and `spectralSample()`. Writes narration node, AWE entry, and sync log event. Broadcasts `TURNS_UPDATED`.

**CfC Worker:** Listens for `NODE_INDEXED` and `CENTROID_DIRTY`. Runs the consolidation pass (affinity edges, integration promotion, centroid update, orbital health check). Broadcasts `HEALTH_UPDATED` or `INJECT_REQUEST` as needed.

**Device Lucy daemon:** Runs the full dream cycle (§12), graph analytics, and HNSW maintenance via Lancedb. Publishes cap deltas and consolidated state back into the mesh.

### 25.4 Analytics at Device Scale

The graph analytics capabilities that NeuronDB's `vgraph` provided are reproduced in `src/device/analytics.ts` for Device Lucy:

- **Louvain community detection** over the belief edge graph identifies mergeable belief node clusters for crystallisation.
- **PageRank** over the full graph identifies high-centrality nodes as merge survivors.
- **AWE walk log** (including Vortex peer exchange walks) provides seed material for community detection during affective reflection: nodes that appear as associative candidates across multiple infotactic walks form natural community seeds because they are nodes the system keeps finding itself drawn back to from different starting positions.

These analytics run only on Device Lucy, where Lancedb and SQLite scale to the full accumulated graph. Results are published to the mesh as cap deltas (§12) so Browser Lucy receives updated reasoning patterns without running the full analytics pass itself.

---

[← §24 Tour Engine Architecture](24-tour-engine-architecture.md) | [Index](../README.md) | [§26 References →](26-references.md)
