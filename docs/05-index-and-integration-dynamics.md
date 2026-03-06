[← §4 System Overview](04-system-overview.md) | [Index](../README.md) | [§6 The Graph Database →](06-the-graph-database.md)

---

## 5. Index and Integration Dynamics

The index and integration dynamics are the foundation of LUCID. They determine how raw content becomes embedded structure in the belief graph, and when that structure is made available to tours, centroids, and consolidation. The graph cannot operate on content that has not been indexed; the consolidation and threat mechanisms cannot act on nodes that have not been integrated into the similarity structure of the graph. This is true whether the content originates from an operator turn or from Lucy's own infotactic reading.

### 5.1 Node Indexing States

Each belief node progresses through a defined sequence of indexing states recorded in `lucid.index_state`:

```
unindexed → inf_indexed / ont_indexed → indexed → integrated
```

- **`unindexed`**: A stub row exists in `lucid.belief_nodes`, with topology (e.g. PREV/NEXT) but no embeddings.
- **`inf_indexed`**: An inference embedding has been stored in `lucid.node_embeddings_inf` for this node.
- **`ont_indexed`**: An ontic embedding has been stored in `lucid.node_embeddings_ont` for this node. Order of `inf_indexed` and `ont_indexed` is not guaranteed.
- **`indexed`**: Both embeddings are present; the node is ready for similarity-based operations but has not yet been connected via affinity edges.
- **`integrated`**: Affinity edges have been created; the node participates fully in tours, centroids, all identity machinery, and affective valence assignment.

Embedding generation is performed by background workers invoking NeuronDB's in-process ML functions over content registered in `lucid.content`. Each successful INSERT into the embedding tables triggers `lucid._embedding_inserted_trigger`, which promotes the node's `index_state` and records the time at which it first became indexed.

### 5.2 Index Progress Ratio

Indexing progress is summarised by a ratio reported in `lucid.index_progress`:

$$\text{index\_ratio} = \frac{\text{indexed\_nodes} + \text{integrated\_nodes}}{\text{total\_nodes}}$$

This is computed from counts over `index_state` and does not require a full table scan. It serves as a coarse measure of how much of the currently known content has reached a state where embedding-based retrieval and integration are possible. It does not, by itself, gate ingest; it is an observable used by operators and higher-level control logic.

### 5.3 Integration Ratio

Integration is measured by a second ratio, also exposed by `lucid.index_progress`:

$$\text{integration\_ratio} = \frac{\text{integrated\_nodes}}{\text{indexed\_nodes} + \text{integrated\_nodes}}$$

This expresses how much of the indexed population has been connected via affinity edges and promoted to `index_state = integrated`. A low `integration_ratio` indicates that embeddings have been generated but the similarity structure of the graph has not yet been populated for a substantial fraction of nodes; consolidation should prioritise integration work in that regime.

### 5.4 Affinity Edges

When a node reaches `index_state = indexed`, it becomes eligible for integration. During consolidation, the integration process runs the following steps for each such node:

1. Query HNSW in inference space: $k$ nearest neighbours from `lucid.node_embeddings_inf` within the same `(model_id, model_version)` partition. These are nodes that the assistant model processes in a similar way.
2. Query HNSW in ontic space: $k$ nearest neighbours from `lucid.node_embeddings_ont`. These are nodes that are structurally related in a model-independent embedding space.
3. Create `similar_inf` edges in `lucid.belief_edges` for inference neighbours, annotating them with cosine similarity and Hebbian statistics.
4. Create `similar_ont` edges analogously for ontic neighbours.
5. Promote the node to `index_state = integrated`.

Affinity edges are the structural substrate of the graph. They make it navigable before any explicit epistemic judgments have been attached. Their initial weight is cosine similarity. They are created during consolidation and updated by the Hebbian activation model as traversal accumulates.

### 5.5 Pre-Integration Screening

Before affinity edges are committed, a fast screening step detects structurally anomalous nodes:

1. Compute the centroid of the $k$ inference neighbours and the centroid of the $k$ ontic neighbours in their respective spaces.
2. Measure divergence between these centroids (cosine distance in each space).
3. If divergence exceeds a configurable threshold, flag the node for priority assessment and optionally hold its affinity edges pending review.

This is a structural anomaly detector operating directly on the dual embedding spaces. It does not replace the full threat architecture ([§9](09-the-threat-architecture.md)); it provides a low-cost early signal that a node occupies inconsistent positions in the two similarity structures.

### 5.6 Operational Loop

These mechanisms implement a continuous indexing and integration loop:

1. Content is registered in `lucid.content` and `lucid.belief_nodes` (`index_state = unindexed`). Sources include operator turns, Lucy's own narration nodes, and content encountered during infotactic navigation.
2. Background workers generate embeddings inside PostgreSQL via NeuronDB and advance nodes through `inf_indexed` and `ont_indexed` to `indexed`.
3. Consolidation phases select `indexed` nodes, create affinity edges, and promote them to `integrated`, raising `integration_ratio`.
4. Index and integration ratios, together with CfC drift and tour metrics, inform when additional consolidation is warranted and when the system is ready to serve operator-facing cycles with a stable internal structure.

---

[← §4 System Overview](04-system-overview.md) | [Index](../README.md) | [§6 The Graph Database →](06-the-graph-database.md)
