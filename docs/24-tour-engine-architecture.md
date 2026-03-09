[← §23 Verification Queue and Sampling](23-verification-queue-and-sampling.md) | [Index](../README.md) | [§25 USE Substrate Boundaries →](25-neurondb-integration.md)

---

## 24. Tour Engine Architecture

The dual tour engine runs in TypeScript inside the CfC Worker, calling the USE substrate interfaces. This section documents the rationale for the mechanism choice and the tour step protocol in full.

### 24.1 Mechanism Choice

The tour engine lives in the CfC Worker for three reasons.

**First, it is browser-native.** No server database, no network round-trip to the graph layer. The tour reads from IndexedDB via `sue.graph()` and the VectorStore via `sue.ont()` and `sue.inf()`. Every query is a local async call. Latency is sub-millisecond.

**Second, it is substrate-portable.** The tour calls only the `GraphStore` and `VectorStore` interfaces defined in §28.10. The same tour code runs against IndexedDB + EntityDB on the browser and against SQLite + Lancedb on Device Lucy. No changes to the algorithm; only the constructors at boot differ.

**Third, the computational bottleneck is the Hebbian-Belief cost function and the dual-space divergence machinery, not raw traversal throughput.** The scale of a mature LUCID persona instance — low tens of thousands of integrated nodes, hundreds of thousands of edges — is comfortably within what JavaScript handles in a Web Worker with appropriate indexing. A loop that maintains the visited set as a JavaScript `Set<string>`, issues a `twoPhaseSearch` call per step, fetches edge records for scoring, and advances produces the correct tour shape without any server dependency.

### 24.2 Tour Step Protocol

At each step of tour `τ_x` for embedding space `x ∈ {i, o}`:

```typescript
// Tour procedure — runs in CfC Worker
// 1. Issue twoPhaseSearch against the VectorStore in embedding space x.
//    Query: current node's full embedding vector.
//    k: configurable candidate set size (default: 20).
//    Filter: nodes with index_state = 'integrated', not in visited set V.
//
// 2. For each candidate, fetch edge records from sue.graph().edgesFor(candidate.id)
//    to retrieve raw_activation_count, fruitful_activation_count,
//    and edge belief weight omega_ij.
//    Compute Hebbian-Belief cost C_x per Definition 9.1.
//
// 3. Select minimum-cost candidate.
//    On ties: prefer descending Hebbian weight, then ascending node ID (lexicographic).
//
// 4. Advance to selected candidate. Add to V. Accumulate tour sequence.
```

The AWE curiosity modification (Definition AWE.1) applies to tour step selection during the affective reflection pass (Step 4d of the dream cycle) and during infotactic navigation phases. Standard threat-detection tours during operator turns use the unmodified Hebbian-Belief cost to preserve the integrity of threat detection.

### 24.3 Two-Phase Search in the Tour

Each candidate query calls `twoPhaseSearch` (§28.13):

```typescript
// Phase 1 — coarse: 128-dim prefix search via EntityDB/Lancedb
// Phase 2 — rerank: exact cosine on full 768-dim embeddings from GraphStore
// Returns: Array<{ id: string; score: number }> sorted descending by score
```

The coarse phase produces a candidate set of up to 100 nodes in ~10ms. The rerank phase narrows to the top-k candidates with exact cosine arithmetic in <1ms. The visited set `V` is applied as a post-filter on the reranked results before cost scoring. This is efficient: the HNSW traversal naturally avoids already-visited regions as the tour progresses (visited nodes' embeddings drift behind `C_w`), and the few false positives are cheap to filter.

### 24.4 Closure Detection

After all reachable integrated nodes are visited (or no unvisited candidates remain within the search neighbourhood), attempt to add a return edge from the final node to the seed. The closure result — both closed, inference-only, ontic-only, neither — is recorded as a `TourResult` on the centroid record and broadcast to the main thread. The four closure outcomes retain their diagnostic meanings from §9.1 without change.

### 24.5 Device Lucy Tour

Device Lucy runs the same tour code with Lancedb's real HNSW indices replacing EntityDB's brute-force search. At device scale (full accumulated belief graph after continuous dream cycle consolidation), Lancedb's HNSW provides orders-of-magnitude faster coarse search than brute-force. The tour protocol, cost function, and closure detection are identical. Only the `VectorStore` and `GraphStore` constructors passed at boot differ.

---

[← §23 Verification Queue and Sampling](23-verification-queue-and-sampling.md) | [Index](../README.md) | [§25 USE Substrate Boundaries →](25-neurondb-integration.md)
