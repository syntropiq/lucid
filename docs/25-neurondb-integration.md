[← §24 Tour Engine Architecture](24-tour-engine-architecture.md) | [Index](../README.md) | [§26 References →](26-references.md)

---

## 25. NeuronDB Integration

This section documents the explicit boundary between NeuronDB substrate and LUCID-owned logic, and specifies how NeuronDB's background workers integrate with the LUCID operational model.

### 25.1 Delegation to NeuronDB

| Capability | NeuronDB mechanism |
|-----------|-------------------|
| Embedding generation | `embed_text(uri, model_name)` in-process ONNX |
| HNSW index maintenance | `neurandefrag` worker |
| Affinity edge candidate generation | HNSW kNN queries |
| KNN graph building (dream cycle) | Native KNN graph analytic |
| Centroid computation (clustering) | K-Means, centroid functions |
| Dream cycle crystallisation (community detection) | `vgraph_community_detection` (Louvain) |
| Crystallisation priority (merge survivors) | `vgraph_pagerank` |
| Async job queue | `neuranq` worker |
| Outlier/anomaly screening (pre-integration) | Z-score, IQR outlier detection over embedding neighbourhoods |
| Monitoring | `vector_stats`, `index_health`, `tenant_quota_usage` views |
| Multi-tenancy isolation | `tenant_quotas`, tenant-aware HNSW, RLS policies |

### 25.2 Retained in LUCID

The following capabilities remain under LUCID's semantic ownership regardless of NeuronDB substrate:

| Capability | Why LUCID owns it |
|-----------|------------------|
| Belief node schema, edges, centroids, cycles | Core domain model |
| Provenance model and belief weight arithmetic | Semantic layer; NeuronDB has no concept of provenance |
| Hebbian-Belief cost function (Definition 9.1) | LUCID-specific scoring; not a general-purpose metric |
| CfC attractor semantics | Pinned `C_0`, liquid time-constant response curve, consolidation trigger logic, orbital trajectory analysis, contact record interpretation — all LUCID-specific interpretations of vector arithmetic primitives |
| Dual tour protocol | Algorithm definition, overlap set `Ω`, injection signal, threat score decomposition |
| Dream cycle sequence | Steps 1–7 and their ordering logic, including Step 4d affective reflection |
| Thinking Cap lifecycle | LoRA adapter generation, soft correction, cap rollback |
| Inheritance corpus and Self Library | Judgment record and anomaly detection reference surface |
| AWE layer | Affective chain generation (ingress/egress), mood gauge, affective valence, user react semantics, curiosity navigation, emotional memory, affective corpus, circadian texture, TripleDent Gum — all LUCID-specific; NeuronDB has no concept of affect |
| Spectral monitoring | Dual-stream FFT, cross-stream correlation, DC dominance detection, orbital health condition, TripleDent Gum tier assessment — all LUCID-specific; NeuronDB has no concept of attentional state |
| ACG continuous monitoring | Ingress/egress chain generation, conflict detection, context integration, affective assessment — all LUCID-specific; NeuronDB has no concept of continuous attentional monitoring |
| Vortex peer layer | Peer mesh participation, centroid advertisement, bilateral contracts, swimming trigger — all LUCID-specific; NeuronDB has no concept of peer presence |
| Swimming mechanism | SWIM trigger condition, `∅_swim` broadcast, open receptivity processing, exit condition — LUCID-specific |
| Network Hebbian equivalence | Interpretation of `C_o` as network Hebbian weight — LUCID-specific semantics on top of NeuronDB vector arithmetic |
| Vortex agenda | Set of nodes flagged for peer contemplation, maintained by ACG — LUCID-specific |
| Bilateral AWE contracts | Peer exchange chain storage, γ history, `[jellyfish]`/`[excellent]`/`[ack]` semantics — LUCID-specific |

### 25.3 NeuronDB Background Worker Integration

The four NeuronDB workers operate alongside LUCID's operational states without modification:

**neuranq:** LUCID event channels (`lucid_interrupt`, `lucid_judgment`, `lucid_consolidate`, `lucid_consolidation`, `lucid_drift`, `lucid_awe_flatline`, `lucid_orbital`, `lucid_vortex_agenda`, `lucid_swim`, `lucid_swim_exit`) become job types in `neuranq`. LUCID-owned handlers process each job type.

**neuranmon:** Operates transparently. HNSW search parameter tuning and cache optimisation improve tour performance without LUCID awareness.

**neurandefrag:** Operates transparently. Index compaction and rebuild scheduling handled automatically.

**neuranllm:** The question of whether `neuranllm` can absorb NeuronAgent's LLM job dispatch, or whether NeuronAgent's orchestration requirements (context assembly, tool dispatch, cycle state management, ACG continuous monitoring) require it to remain external, is deferred pending instrumentation from a live deployment.

### 25.4 Note on vgraph

NeuronDB's `vgraph` type (adjacency list with BFS/DFS/PageRank/community detection) is not used for real-time touring. Node indices in `vgraph` are positional integers; LUCID nodes are identified by `bigint` row IDs. Edge weights are not stored in the `vgraph` structure, and the Hebbian-Belief cost function depends on per-edge runtime state that changes between cycles. `vgraph` is suitable for batch analytics during dream cycles:

- `vgraph_community_detection` (Louvain) identifies mergeable belief node clusters
- `vgraph_pagerank` identifies high-centrality nodes as merge survivors during crystallisation

The AWE walk log — now including Vortex peer exchange walks — provides additional seed material for `vgraph` community detection during affective reflection: nodes that appear as associative candidates across multiple infotactic walks, including walks seeded from peer exchange nodes, form natural community seeds because they are nodes the system keeps finding itself drawn back to from different starting positions.

---

[← §24 Tour Engine Architecture](24-tour-engine-architecture.md) | [Index](../README.md) | [§26 References →](26-references.md)
