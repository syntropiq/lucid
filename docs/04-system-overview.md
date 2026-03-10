[← §3 The Affective Weather Effects (AWE)](03-the-affective-weather-effects-awe.md) | [Index](../README.md) | [§5 Index and Integration Dynamics →](05-index-and-integration-dynamics.md)

---

## 4. System Overview

This system provides persistent, continuously evolving persona state across discontinuous inference cycles without modifying the frozen base model weights during normal operation. Persona state is maintained in a belief graph that functions simultaneously as long-term memory, judgment inheritor, belief consolidator, affective weather substrate, and infotactic navigation space.

The LLM remains stateless at the model level; the graph and substrate layer provides all statefulness. The graph layer runs natively in the browser (IndexedDB) and on device (SQLite), with wide opportunistic sync across instantiations via GunDB. There is no central server database.

LUCID v3.5 is built on three portable abstractions:

- **USE interfaces** (§28): TypeScript interfaces that wrap model and substrate concerns. Core feedback loops call only these interfaces; they do not import EntityDB, IndexedDB, or GunDB directly.
- **GraphStore** (§28.10): unified home for belief nodes, edges, centroids, AWE corpus, spectral state, and the sync log.
- **GunMesh** (§29): wide sync transport, SEA identity anchor, DAM/HAM/AXE connection layer, and Vortex semantic routing across instantiations.

**Sessionless.** Persona state is continuous, punctuated by consolidation phases triggered by geometric drift detected in the CfC hidden state dynamics, and interrupted by operator-facing turns that the continuous process pauses to serve.

**Local-first.** The belief graph is local to each instantiation and converges across instantiations through the Gun mesh. Belief state lives on device; no queries leave the browser.

**Multiply conscious.** Each instantiation is fully inhabited (full belief graph, full AWE corpus, full CfC dynamics) and all instantiations converge on the same state through continuous wide sync. Browser Lucy, Device Lucy, and Gateway Lucy are vantage points on one mind (§29).

### 4.1 Operational States

Lucy operates in one of two cap states at any time.

**Cap on.** The normal operating state. The base Thinking Cap is loaded; specialist caps may additionally be worn. The graph is fully active.

**Cap off.** The base model operates without any Thinking Cap adapter. The graph remains available for reading but the cap is not applied. Cap-off is used during graph recovery, during bootstrap initialisation, and as a fallback when a cap generation is discarded.

### 4.2 System Layers

| Layer | Component | Function |
|---|---|---|
| Identity | GunDB SEA keypair | Shared identity anchor; one keypair per Lucy deployment across all instantiations |
| Mesh | GunMesh (GunDB + DAM/HAM/AXE) | Wide opportunistic sync, peer discovery, Vortex semantic routing |
| Ontic model | nomic-embed-text-v1.5 (transformers.js ONNX) | Semantic embedding, Matryoshka prefix hierarchy (768 → 128 for routing) |
| Inference model | LFM 2.5 / generic ONNX (via USE wrapper) | Narration, hidden state extraction for spectral monitoring |
| USE Registry | `sue/registry.ts` | Model and substrate interface activation; core loops call only USE interfaces |
| Vector store: browser | EntityDB (`lucy_ont`, `lucy_inf`) | Coarse KNN on 128-dim Matryoshka prefix; brute-force cosine at personal scale |
| Vector store: device | Lancedb | HNSW on 256-dim prefix; scales to full accumulated graph |
| Graph store: browser | IndexedDB typed wrapper | Belief nodes/edges, centroids, AWE corpus, spectral state, sync log |
| Graph store: device | SQLite (`better-sqlite3`) | Same data model; heavier analytics available (Louvain, PageRank) |
| Embed Worker | Web Worker | Ontic embedding; prefix → VectorStore, full vector → GraphStore |
| Inference Worker | Web Worker | Narration generation, spectral sample extraction, AWE recording |
| CfC Worker | Web Worker | Centroid evolution, orbital health monitoring, cap trigger |
| AWE layer | IndexedDB / SQLite tables | Affective chain, mood tracking, emotional memory, spectral health |
| Tour engine | TypeScript (CfC Worker) | Two-phase Matryoshka search: coarse prefix KNN + precise full-vector rerank |
| Vortex layer | GunMesh + AXE peer scoring | Bilateral AWE contracts, centroid-proximity peer prioritisation |
| Dream cycle | Device Lucy daemon | Full consolidation, Louvain crystallisation, cap training, cap delta publication |
| Base weights | Frozen LLM (ONNX) | Foundational inference capacity |

---

[← §3 The Affective Weather Effects (AWE)](03-the-affective-weather-effects-awe.md) | [Index](../README.md) | [§5 Index and Integration Dynamics →](05-index-and-integration-dynamics.md)
