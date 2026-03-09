[← §22 Verification Queue and Sampling](22-verification-queue-and-sampling.md) | [Index](../README.md) | [§24 NeuronDB Integration →](24-neurondb-integration.md)

---

## 23. Tour Engine Architecture

The dual tour engine runs in PL/pgSQL, not in any graph extension or application layer. This section documents the rationale for the mechanism choice and the tour step protocol in full.

### 23.1 Mechanism Choice

Apache AGE was evaluated and rejected due to operational fragility across the multi-year lifecycle of a deployed persona instance: it broke on PostgreSQL 17.1 due to an ABI struct change, explicitly does not support `pg_upgrade`, and had open PostgreSQL 18 support as of this writing.

pgRouting was investigated as an alternative and rejected: it requires PostGIS as a hard dependency, rebuilds the in-memory graph from a full edge-table scan on every call, and its algorithm set (Dijkstra, A*, TSP via simulated annealing) does not match the LUCID tour shape.

The scale of a mature LUCID persona instance — low tens of thousands of integrated nodes, hundreds of thousands of edges — is comfortably within what PostgreSQL handles natively with appropriate indexing. The computational bottleneck is the Hebbian-Belief cost function and the dual-space divergence machinery, not raw traversal throughput. A PL/pgSQL loop that maintains the visited set as a session-local `bigint[]` array, issues a single HNSW kNN query per step, joins `belief_edges` for scoring, and advances produces the correct tour shape in a single database round-trip from NeuronAgent.

### 23.2 Tour Step Protocol

At each step of tour $\tau_x$ for embedding space $x$:

1. Issue an HNSW kNN query against `lucid.node_embeddings_{x}` for the current node's embedding. Retrieve the $k$ nearest neighbours, partitioned by `(model_id, model_version)`, restricted to `index_state = integrated` nodes not in the visited set $V$.
2. Join each candidate against `lucid.belief_edges` to retrieve `raw_activation_count`, `fruitful_activation_count`, and edge belief weight $\omega_{ij}$. Compute the Hebbian-Belief cost $C_x$ per [Definition 9.1](#91-the-dual-nearest-neighbour-heuristic-tour-procedure).
3. Select the minimum-cost candidate. On ties, prefer descending Hebbian weight, then ascending node ID.
4. Advance to the selected candidate. Add it to $V$.

The AWE curiosity modification ([Definition AWE.1](#34-curiosity-as-navigational-force)) applies to tour step selection during the affective reflection pass (Step 4d of the dream cycle) and during infotactic navigation phases. Standard threat-detection tours during operator turns use the unmodified Hebbian-Belief cost to preserve the integrity of threat detection.

### 23.3 Closure Detection

After all reachable integrated nodes are visited, attempt to add a return edge from the final node to the seed. The closure result (both closed, inference-only, ontic-only, neither) is recorded in `lucid.tour_closure`. The four closure outcomes retain their diagnostic meanings from [§9.1](#91-the-dual-nearest-neighbour-heuristic-tour-procedure) without change.

### 23.4 Future Consideration: OneSparse/GraphBLAS

The SuiteSparse:GraphBLAS PostgreSQL extension (OneSparse) is a strong candidate for a future iteration. It would allow efficient in-database sparse matrix operations over the belief graph, potentially replacing the PL/pgSQL loop with significantly faster bulk traversal. It requires PG18 features and is explicitly early-stage as of this writing. The tour protocol defined in [§9.1](#91-the-dual-nearest-neighbour-heuristic-tour-procedure) and [Definition 9.2](#91-the-dual-nearest-neighbour-heuristic-tour-procedure) would be preserved if OneSparse is adopted; only the execution mechanism would change, with no schema changes required.

---


---

[← §22 Verification Queue and Sampling](22-verification-queue-and-sampling.md) | [Index](../README.md) | [§24 NeuronDB Integration →](24-neurondb-integration.md)
