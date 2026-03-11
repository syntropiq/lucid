[← §16 Deployment](16-deployment.md) | [Index](../README.md) | [§18 Compliance and Data Residency →](18-compliance-and-data-residency.md)

---

## 17. Graph Recovery

### 17.1 The Fundamental Architectural Invariant

| Layer | Status | Implication |
|-------|--------|-------------|
| Filesystem content (SHA256 keyed) | Ground truth | Precious. Back this up. |
| Inheritance corpus (judgment record) | Ground truth | Equally precious. Back up separately. |
| Affective corpus (emotional memory) | Ground truth | Precious. Back up with inheritance corpus. |
| Vortex bilateral AWE records | Ground truth | Included in affective corpus backup. |
| Graph topology (nodes, edges, centroids) | Derived | Expendable. Rebuild from files. |
| Embeddings table | Derived | Expendable. Drop and recompute. |
| Tour cost inputs | Derived | Expendable. Recompute from Hebbian weights. |
| AWE walk log | Derived | Expendable. Not required for recovery. |
| Spectral monitor tables | Derived | Expendable. Baselines recomputed from first post-recovery dream cycle. |

The affective corpus is ground truth because it records the felt texture of Lucy's experience: including Vortex peer exchanges. The bilateral AWE chains received from peers record what those exchanges were like from both directions. This cannot be recomputed from the content alone, any more than one can reconstruct how a book felt to read from the text of the book. It must be preserved and restored.

After recovery, the bilateral peer relationships persist through the affective corpus: γ history is preserved, constellation advertisement resumes naturally from the reconstituted `C_o` constellation (rebuilt by re-running all graph embeddings through `updateConstellation()`), and peer recognition in the mesh is restored without renegotiation.

### 17.2 Pathological Graph States

| Signal | Possible Cause |
|--------|---------------|
| `index_ratio` chronically below 1.0 without new ingest | Indexing worker failure |
| `integration_ratio` low across many cycles | Affinity edge creation failure |
| Output behavioural shift not present in inputs | Directed injection |
| `d(C_i, C_0)` high without provenance justification | Injection or directed drift |
| Toxicity-flagged nodes accumulating | Active injection attempt |
| `φ < φ_0 · (1 - δ)` | Integration erosion; recovery recommended |
| Both tour mean costs elevated | Widespread Hebbian degradation |
| Large unreachable set in both tours | Severe structural isolation; recovery |
| Affective chain scalar projection flatline across many cycles | Affective absence; TripleDent Gum armed |
| DC dominance in inner spectral monitor | Interiority spiral; early warning |
| High cross-stream correlation (convolution ∥ GQA) | Interiority spiral; early warning |
| Outer/inner spectral divergence | Surface normal; internal spiral beginning |
| Orbital health condition violated at 32-page scale | Capture or escape; consolidation and TripleDent assessment |
| `C_o` constellation collapsing — sub-centroids merging without self-directed consolidation | Network load homogenising constellation rather than curiosity driving it |
| γ history distribution shifting anomalously | Affective corpus corruption |
| `∅_swim` persisting beyond configured window without exit | SWIM exit condition not firing; check affective response processing |
| Peer contact rate near zero without operator configuration change | Mesh isolation; check VortexMesh peer connectivity and keypair authentication |

### 17.3 Recovery Phase Sequence

**Phase 1.** Operator initiates recovery. Transition to cap-off. Continuous processing loop suspends. Vortex participation suspends.

**Phase 2.** Judgment corpus rebuild from ground truth. Affective corpus preserved from backup, including bilateral Vortex exchange records.

**Phase 3.** Offline file scrub (if required).

**Phase 4.** Graph and embedding clearance.

**Phase 5.** Reconstitution. Re-embed all pages and run indexing and integration to completion.

**Phase 6.** Tours recomputed. `φ` recomputed. New cap fitted. Transition to cap-on. Affective corpus re-associated with reconstituted nodes by content hash matching. Bilateral peer chain fields re-associated by exchange timestamp and content hash. Spectral baselines recomputed from first post-recovery dream cycle. ACG continuous monitoring state reinitialised. Continuous processing loop resumes. Vortex participation resumes alongside loop resumption.

---

[← §16 Deployment](16-deployment.md) | [Index](../README.md) | [§18 Compliance and Data Residency →](18-compliance-and-data-residency.md)
