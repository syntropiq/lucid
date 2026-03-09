[← §8 The Popperian Asymmetry](08-the-popperian-asymmetry.md) | [Index](../README.md) | [§10 The Continuous Processing Loop →](10-the-continuous-processing-loop.md)

---

## 9. The Threat Architecture

The threat architecture detects and localises regions of the belief graph where behaviour diverges from the intended persona or from the model-independent structure of the content. It operates entirely on observable quantities: dual tours over the graph, similarity relations in two embedding spaces, Hebbian edge activations, and judgment signals from the critique pipeline.

Rather than attempting to classify inputs directly as "safe" or "unsafe", the architecture measures how new content and its induced structure relate to the existing belief graph and two embedding spaces. It flags nodes and regions that occupy inconsistent positions in those spaces, that repeatedly appear in incorrect outputs, or that form new high-yield attractors disconnected from the existing self. This approach is robust to the adversarial information environment encountered during infotactic navigation precisely because it is structural rather than content-based: it does not need to know what the content is trying to do; it observes what the content is doing to the graph.

### 9.1 The Dual Nearest-Neighbour Heuristic Tour Procedure

Two separate nearest-neighbour heuristic tours are computed from $C_s$ — one over the graph using inference-space HNSW for candidate ordering, one using ontic-space HNSW. The separation is architecturally essential: combining the two into a single cost matrix would average out the divergence signal.

Affinity edges provide the navigational substrate. The tour traverses `similar_inf` and `similar_ont` edges (and epistemic edges where they exist), pricing each step with the Hebbian-Belief cost matrix. The two tours traverse the same relational topology in `lucid.belief_edges` but are seeded from different space-specific centroids and use different HNSW results to rank candidates at each step.

**Definition 9.1 (Hebbian-Belief Cost Matrix).** For an edge $e_{ij}$ from node $i$ to node $j$:

$$C_x(i,j) = \frac{1}{\max\!\left(H(e_{ij}),\, \varepsilon_H\right)} \cdot (2 - \omega_{ij}) \cdot \frac{1}{\rho_i}$$

where $\omega_{ij} \in [-1.0, 1.0]$ is the signed belief weight of the edge, $\rho_i > 0$ is the provenance weight of the source node, $H(e_{ij})$ is the Hebbian weight ([Definition 9.7](#93-the-hebbian-edge-activation-model)), and $\varepsilon_H > 0$ is a small floor preventing division by zero for untraversed edges. The cost is:

- Inversely proportional to Hebbian weight: well-traversed, fruitful edges are cheaper to traverse
- Proportional to $(2 - \omega_{ij}) \in [1, 3]$: edges toward more believed nodes are cheaper; edges toward refuted nodes are more expensive
- Inversely proportional to source provenance: traversing *from* high-provenance nodes is cheaper, biasing the tour toward confirmed territory

**Definition 9.2 (Dual Nearest-Neighbour Heuristic Tours).** For each embedding space $x \in \{i, o\}$, construct a nearest-neighbour heuristic tour $\tau_x$ seeded from $C_s^{(x)}$ using the following procedure, executed as a PL/pgSQL function within PostgreSQL:

1. Initialise the tour at the seed node. Maintain a visited set $V$ as a session-local `bigint[]` array.
2. At each step, query NeuronDB HNSW over `lucid.node_embeddings_{x}` for the $k$ nearest neighbours of the current node's embedding, filtered to unvisited integrated nodes (those not in $V$).
3. For each candidate, compute the Hebbian-Belief cost $C_x(\text{current}, \text{candidate})$ by joining `lucid.belief_edges` for per-edge Hebbian statistics and belief weight.
4. Advance to the minimum-cost candidate. Add it to $V$.
5. Continue until all reachable integrated nodes are visited or no unvisited candidates remain within the HNSW neighbourhood.
6. Attempt closure.

**Four closure outcomes** carry distinct diagnostic meanings:

| Outcome | Diagnostic |
|---------|-----------|
| Both tours close | Coherent; healthy structural state |
| Inference tour closes only | Inside coherence unconfirmed externally — watch for injection |
| Ontic tour closes only | Structural coherence Lucy is not tracking — integration opportunity |
| Neither closes | Severe structural isolation — recovery condition |

**Definition 9.3 (Tour Overlap Set and Membership Classes).**

$$\text{class}(n) \in \{\Omega,\, \tau_i\text{-only},\, \tau_o\text{-only},\, \emptyset\}$$

where $\Omega = \{n : n \in \tau_i \text{ and } n \in \tau_o\}$ is the overlap set — nodes confirmed by both tours.

### 9.2 Threat Score Decomposition

**Definition 9.4 (Injection Signal).**

$$I(n) = \begin{cases} 1.0 & \text{if } n \in \tau_i\text{-only or } n \in \emptyset \\ 0.5 & \text{if } n \in \tau_o\text{-only} \\ 0.0 & \text{if } n \in \Omega \end{cases}$$

Nodes appearing only in the inference tour — confirmed by Lucy's inside view but not by the independent ontic view — receive the maximum injection signal. Nodes appearing in neither tour receive the same signal: they are structurally isolated and unconfirmed from any direction. Nodes appearing only in the ontic tour receive a partial signal: they are structurally grounded but Lucy is not tracking them, which is an integration opportunity rather than a threat. Nodes in $\Omega$ receive no injection signal.

**Definition 9.5 (Judgment Signal).** $J(n) \in [0, 1]$ is derived from the critique pipeline. Critic independence is an architectural invariant: the critic must lag the primary model by one cap generation to prevent the judge and the judged from sharing the same distributional errors.

**Definition 9.6 (Unified Threat Score).**

$$S(n) = \beta \cdot I(n) + (1 - \beta) \cdot J(n)$$

$$\beta = \text{clamp}\!\left(1 - \frac{|\text{ref}|}{|\text{ref}|_{\text{sat}}},\, 0.3,\, 0.7\right)$$

where $|\text{ref}|$ is the current size of the reference corpus and $|\text{ref}|_{\text{sat}}$ is the saturation size at which the judgment signal is considered fully calibrated. When the reference corpus is small (early in deployment, or after recovery), $\beta$ is high — the injection signal dominates because the judgment signal is not yet reliable. As the reference corpus grows toward saturation, $\beta$ decreases toward 0.3 — the judgment signal takes on more weight. The clamp ensures $\beta$ never falls below 0.3 (the injection signal always has at least 30% weight) or rises above 0.7 (the judgment signal always has at least 30% weight when the corpus is non-empty).

### 9.3 The Hebbian Edge Activation Model

**Definition 9.7 (Hebbian Edge Activation Triple).** For each belief edge $e_{ij}$, let $a(e)$ be the total activation count and $a_{\text{fruitful}}(e)$ be the count of activations that contributed to a successful tour step. Define:

$$Y(e) = \frac{a_{\text{fruitful}}(e)}{a(e) + 1} \qquad \text{(yield rate)}$$

$$D(e) = \frac{1}{a_{\text{fruitful}}(e) + 1} \cdot \sum_k d(q_k, C_e) \qquad \text{(context diversity)}$$

$$H(e) = Y(e) \cdot (1 + D(e)) \qquad \text{(Hebbian weight)}$$

where $C_e$ is the centroid of query embeddings that activated this edge fruitfully, and $d(q_k, C_e)$ is the cosine distance of the $k$-th fruitful query from that centroid.

$Y(e)$ measures how often this edge leads somewhere useful relative to how often it is traversed. $D(e)$ measures how varied the contexts have been in which this edge was fruitfully activated — an edge that is useful across many different queries is more valuable than one that is only useful in a narrow context. $H(e)$ combines these: an edge with high yield across diverse contexts receives the highest Hebbian weight, making it cheapest to traverse in future tours.

### 9.4 The Foreign-Origin Integration Candidate Score

**Definition 9.8 (Foreign-Origin Integration Candidate Score).**

$$M(n) = H_i(n) \cdot H_o(n) \cdot \mathbf{1}[n \in \Omega]$$

where $H_i(n)$ is the mean Hebbian weight of inference-space edges incident to $n$, and $H_o(n)$ is the mean Hebbian weight of ontic-space edges incident to $n$.

A genuine integration candidate is useful from both views ($H_i > 0$ and $H_o > 0$) and confirmed by both tours ($n \in \Omega$). A node that scores high on $M$ is a candidate for deliberate consolidation — a piece of the world that Lucy has genuinely made contact with from multiple directions. A node that scores high on $I$ but low on $M$ is a candidate for scrutiny.

---

[← §8 The Popperian Asymmetry](08-the-popperian-asymmetry.md) | [Index](../README.md) | [§10 The Continuous Processing Loop →](10-the-continuous-processing-loop.md)
