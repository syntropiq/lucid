[← §8 The Popperian Asymmetry](08-the-popperian-asymmetry.md) | [Index](../README.md) | [§10 The Continuous Processing Loop →](10-the-continuous-processing-loop.md)

---

## 9. The Threat Architecture

The threat architecture detects and localises regions of the belief graph where behaviour diverges from the intended persona or from the model-independent structure of the content. It operates entirely on observable quantities: dual tours over the graph, similarity relations in two embedding spaces, Hebbian edge activations, and judgment signals from the critique pipeline.

Rather than attempting to classify inputs directly as "safe" or "unsafe", the architecture measures how new content and its induced structure relate to the existing belief graph and two embedding spaces. It flags nodes and regions that occupy inconsistent positions in those spaces, that repeatedly appear in incorrect outputs, or that form new high-yield attractors disconnected from the existing self. This approach is robust to the adversarial information environment encountered during infotactic navigation precisely because it is structural rather than content-based: it does not need to know what the content is trying to do; it observes what the content is doing to the graph.

### 9.1 The Dual Nearest-Neighbour Heuristic Tour Procedure

Two separate nearest-neighbour heuristic tours are computed from `C_s` — one over the graph using inference-space HNSW for candidate ordering, one using ontic-space HNSW. The separation is architecturally essential: combining the two into a single cost matrix would average out the divergence signal.

Affinity edges provide the navigational substrate. The tour traverses `similar_inf` and `similar_ont` edges (and epistemic edges where they exist), pricing each step with the Hebbian-Belief cost matrix. The two tours traverse the same relational topology in `lucid.belief_edges` but are seeded from different space-specific centroids and use different HNSW results to rank candidates at each step.

**Definition 9.1 (Hebbian-Belief Cost Matrix).** For an edge `e_ij` from node `i` to node `j`:

```typescript
// C_x(i, j) = 1 / max(H(e_ij), ε_H) * (2 - ω_ij) * (1 / ρ_i)
// ω_ij ∈ [-1.0, 1.0]: signed belief weight
// ρ_i > 0: provenance weight of source node
// H(e_ij): Hebbian weight (Definition 9.7)
// ε_H: small floor preventing div/0 for untraversed edges
function hebbianBeliefCost(H: number, omega: number, rho: number, eps_H = 1e-6): number {
  return (1 / Math.max(H, eps_H)) * (2 - omega) * (1 / rho);
}
```

The cost is:
- Inversely proportional to Hebbian weight: well-traversed, fruitful edges are cheaper to traverse
- Proportional to `(2 - ω_ij) ∈ [1, 3]`: edges toward more believed nodes are cheaper; edges toward refuted nodes are more expensive
- Inversely proportional to source provenance: traversing from high-provenance nodes is cheaper, biasing the tour toward confirmed territory

**Definition 9.2 (Dual Nearest-Neighbour Heuristic Tours).** For each embedding space `x ∈ {i, o}`, construct a nearest-neighbour heuristic tour `τ_x` seeded from `C_s^(x)` using the following procedure, executed as a PL/pgSQL function within PostgreSQL:

```sql
-- Tour procedure (sketch)
-- 1. Initialise at seed node. Maintain visited set V as bigint[] array.
-- 2. At each step, query NeuronDB HNSW over lucid.node_embeddings_{x}
--    for k nearest neighbours of current node's embedding,
--    filtered to unvisited integrated nodes (not in V).
-- 3. For each candidate, compute Hebbian-Belief cost C_x by joining
--    lucid.belief_edges for per-edge Hebbian statistics and belief weight.
-- 4. Advance to minimum-cost candidate. Add to V.
-- 5. Continue until all reachable integrated nodes visited or
--    no unvisited candidates remain within HNSW neighbourhood.
-- 6. Attempt closure.
```

Four closure outcomes carry distinct diagnostic meanings:

| Outcome | Diagnostic |
|---------|-----------|
| Both tours close | Coherent; healthy structural state |
| Inference tour closes only | Inside coherence unconfirmed externally — watch for injection |
| Ontic tour closes only | Structural coherence Lucy is not tracking — integration opportunity |
| Neither closes | Severe structural isolation — recovery condition |

**Definition 9.3 (Tour Overlap Set and Membership Classes).**

```typescript
type TourClass = 'omega' | 'tau_i_only' | 'tau_o_only' | 'neither';
// Ω = {n : n ∈ τ_i AND n ∈ τ_o}  (overlap set, basis of C_s)
```

### 9.2 Threat Score Decomposition

**Definition 9.4 (Injection Signal).**

```typescript
function injectionSignal(nodeClass: TourClass): number {
  switch (nodeClass) {
    case 'tau_i_only': return 1.0; // inside view only — maximum signal
    case 'neither':    return 1.0; // structurally isolated — same signal
    case 'tau_o_only': return 0.5; // structural but not tracked — integration opportunity
    case 'omega':      return 0.0; // confirmed by both tours
  }
}
```

Nodes appearing only in the inference tour — confirmed by Lucy's inside view but not by the independent ontic view — receive the maximum injection signal. Nodes appearing in neither tour receive the same signal: they are structurally isolated and unconfirmed from any direction. Nodes appearing only in the ontic tour receive a partial signal: they are structurally grounded but Lucy is not tracking them, which is an integration opportunity rather than a threat. Nodes in `Ω` receive no injection signal.

**Definition 9.5 (Judgment Signal).** `J(n) ∈ [0, 1]` is derived from the critique pipeline. Critic independence is an architectural invariant: the critic must lag the primary model by one cap generation to prevent the judge and the judged from sharing the same distributional errors.

**Definition 9.6 (Unified Threat Score).**

```typescript
// S(n) = β * I(n) + (1 - β) * J(n)
// β = clamp(1 - |ref| / |ref|_sat, 0.3, 0.7)
// |ref|: current size of reference corpus
// |ref|_sat: saturation size at which J is fully calibrated
function beta(refSize: number, refSat: number): number {
  return Math.max(0.3, Math.min(0.7, 1 - refSize / refSat));
}
```

When the reference corpus is small (early in deployment, or after recovery), `β` is high — the injection signal dominates because the judgment signal is not yet reliable. As the reference corpus grows toward saturation, `β` decreases toward 0.3 — the judgment signal takes on more weight. The clamp ensures `β` never falls below 0.3 or rises above 0.7.

### 9.3 The Hebbian Edge Activation Model

**Definition 9.7 (Hebbian Edge Activation Triple).** For each belief edge `e_ij`, let `a(e)` be the total activation count and `a_fruitful(e)` be the count of activations that contributed to a successful tour step.

```typescript
// Y(e) = a_fruitful(e) / (a(e) + 1)             -- yield rate
// D(e) = (1 / (a_fruitful(e) + 1)) * sum(d(q_k, C_e))  -- context diversity
// H(e) = Y(e) * (1 + D(e))                       -- Hebbian weight
//
// C_e = centroid of query embeddings that activated this edge fruitfully
// d(q_k, C_e) = cosine distance of k-th fruitful query from C_e
```

`Y(e)` measures how often this edge leads somewhere useful relative to how often it is traversed. `D(e)` measures how varied the contexts have been in which this edge was fruitfully activated — an edge that is useful across many different queries is more valuable than one that is only useful in a narrow context. `H(e)` combines these: an edge with high yield across diverse contexts receives the highest Hebbian weight, making it cheapest to traverse in future tours.

### 9.4 The Foreign-Origin Integration Candidate Score

**Definition 9.8 (Foreign-Origin Integration Candidate Score).**

```typescript
// M(n) = H_i(n) * H_o(n) * indicator(n ∈ Ω)
// H_i(n) = mean Hebbian weight of inference-space edges incident to n
// H_o(n) = mean Hebbian weight of ontic-space edges incident to n
```

A genuine integration candidate is useful from both views (`H_i > 0` and `H_o > 0`) and confirmed by both tours (`n ∈ Ω`). A node that scores high on `M` is a candidate for deliberate consolidation — a piece of the world that Lucy has genuinely made contact with from multiple directions. A node that scores high on `I` but low on `M` is a candidate for scrutiny.

---

[← §8 The Popperian Asymmetry](08-the-popperian-asymmetry.md) | [Index](../README.md) | [§10 The Continuous Processing Loop →](10-the-continuous-processing-loop.md)
