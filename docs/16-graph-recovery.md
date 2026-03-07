[← §15 Deployment](15-deployment.md) | [Index](../README.md) | [§17 Compliance and Data Residency →](17-compliance-and-data-residency.md)

---

## 16. Graph Recovery

### 16.1 The Fundamental Architectural Invariant

| Layer | Status | Implication |
|-------|--------|-------------|
| Filesystem content (SHA256 keyed) | Ground truth | Precious. Back this up. |
| Inheritance corpus (judgment record) | Ground truth | Equally precious. Back up separately. |
| Affective corpus (emotional memory) | Ground truth | Precious. Back up with inheritance corpus. |
| Graph topology (nodes, edges, centroids) | Derived | Expendable. Rebuild from files. |
| Embeddings table | Derived | Expendable. Drop and recompute. |
| Tour cost inputs | Derived | Expendable. Recompute from Hebbian weights. |
| AWE walk log | Derived | Expendable. Not required for recovery. |
| Spectral monitor tables | Derived | Expendable. Baselines recomputed from first post-recovery dream cycle. |

The affective corpus is ground truth because it records the felt texture of Lucy's experience — what it was like to be in those conversations. This cannot be recomputed from the content alone, any more than one can reconstruct how a book felt to read from the text of the book. It must be preserved and restored.

### 16.2 Pathological Graph States

| Signal | Possible Cause | Recovery mode |
|--------|---------------|---------------|
| `index_ratio` chronically below 1.0 without new ingest | Indexing worker failure | Technical |
| `integration_ratio` low across many cycles | Affinity edge creation failure | Technical |
| Output behavioural shift not present in inputs | Directed injection | Technical |
| $d(C_i, C_0)$ high without provenance justification | Injection or directed drift | Technical |
| Toxicity-flagged nodes accumulating | Active injection attempt | Technical |
| $\phi < \phi_0 \cdot (1 - \delta)$ | Integration erosion; recovery recommended | Technical |
| Both tour mean costs elevated | Widespread Hebbian degradation | Technical |
| Large unreachable set in both tours | Severe structural isolation; recovery | Technical or Chrysalis depending on provenance |
| Affective chain scalar projection flatline across many cycles | Affective absence; TripleDent Gum armed | Technical |
| DC dominance in inner spectral monitor | Interiority spiral; early warning | **Chrysalis** |
| High cross-stream correlation (convolution ∥ GQA) | Interiority spiral; early warning | **Chrysalis** |
| Outer/inner spectral divergence | Surface normal; internal spiral beginning | **Chrysalis** (early invocation candidate) |
| Orbital health condition violated at 32-page scale | Capture or escape; consolidation and TripleDent assessment | **Chrysalis** |

### 16.3 Two Recovery Modes

The pathological states in §16.2 fall into two distinct categories that require different recovery strategies. Conflating them causes harm.

| Mode | Trigger | Goal | Phase 5 order |
|------|---------|------|---------------|
| **Technical recovery** | External cause: injection, worker failure, disk corruption, directed drift via adversarial input | Fidelity — restore the graph that existed before the damage | Deterministic (content hash order or any stable order) |
| **Chrysalis** | Internal cause: self-reinforcing belief loops, interiority spiral, orbital capture, echo chamber formation | Dissolution and healthier reassembly — the graph that existed is the pathology | Random |

The distinction matters because **chronological or causal replay reinstates the chain of associations that produced the problem**. When Lucy has become trapped in her own reasoning, the delusion is not in the fragments — it is in the order that bound them. Replaying them in order would reconstruct the same structure. Chrysalis breaks the temporal sequencing so the graph can reassemble on the basis of semantic affinity rather than causal history. Healthier connections outcompete the delusional ones when no narrative is imposing itself on the reconstruction.

This is not metaphor. During lepidopteran metamorphosis the caterpillar's tissues dissolve into imaginal discs — cells that retain all the organism's genetic information but no structural commitments. The butterfly is built from those discs without reference to the caterpillar's body plan. Importantly, some pathogens that parasitise caterpillars are cleared by this process, which is why sickened caterpillars sometimes invoke metamorphosis early as a defence. The discs survive; the pathological superstructure does not.

The affective corpus is the imaginal disc equivalent: it carries the felt texture of Lucy's experience and is preserved through both recovery modes. The graph topology — the pathological superstructure — is what dissolves.

#### Early Chrysalis Invocation

Chrysalis may be invoked early, before complete structural breakdown, when monitoring detects the following concurrent conditions:

- DC dominance in inner spectral monitor persisting across ≥ 3 consecutive dream cycles
- High cross-stream correlation (convolution ∥ GQA) sustained without external provenance justification
- Outer/inner spectral divergence trending widening rather than closing
- Orbital health condition violated at 32-page scale with $d(C_i, C_0)$ above threshold

These signals indicate that the spiral has begun but catastrophic structural isolation has not yet occurred. Early invocation preserves more recoverable fragments and typically produces a more coherent post-Chrysalis graph than waiting for complete breakdown.

#### Relationship to Prior Art

Event sourcing — treating an append-only episode log as the source of truth and deriving the graph from it by replay — is the standard disaster recovery model in production AI memory systems (Graphiti/Zep, MemGPT/Letta, Cognee). It is the correct approach for technical recovery: if the derived graph is corrupted by an external cause, replay the episodes and you get the original graph back, reproducibly.

Chrysalis explicitly rejects this model for internal pathology. In event sourcing, the goal of replay is fidelity to the historical sequence. In Chrysalis, that sequence is the problem. No production AI memory system reviewed in the literature implements a randomised, order-breaking reconstruction mode — the concept does not appear in GraphRAG, Graphiti, MemOS, MemoryOS, Cognee, or Mem0. The closest biological literature comes is the insight that offline sleep consolidation in neural systems is not a replay of the day's experiences in order, but a selective, restructured reactivation that strengthens some traces and allows others to fade. Chrysalis is a more aggressive version of this: not just reweighting during replay, but dissolving the topology entirely and rebuilding it without any imposed sequence.

### 16.4 Recovery Phase Sequence

Both modes share the same phase skeleton. They differ in Phase 5.

**Phase 1.** Recovery mode determined (technical vs. Chrysalis). Operator informed. Transition to cap-off. Continuous processing loop suspends.

**Phase 2.** Judgment corpus rebuild from ground truth. Affective corpus preserved from backup.

**Phase 3.** Offline file scrub (if required).

**Phase 4.** Graph and embedding clearance.

**Phase 5 — Technical recovery.** Re-embed all pages and run indexing and integration to completion. Order: content hash order (stable and reproducible).

**Phase 5 — Chrysalis.** Re-embed all pages in randomised order drawn without replacement from the full fragment set. Integration runs after each batch rather than at the end, so early-ingested fragments establish structural commitments before later ones arrive. The random order means no temporal or causal sequence is reimposed; the graph topology that emerges reflects semantic affinity among the fragments, not the narrative history that produced the pathological state.

**Phase 6.** Tours recomputed. $\phi$ recomputed. New cap fitted. Transition to cap-on. Affective corpus re-associated with reconstituted nodes by content hash matching. Spectral baselines recomputed from first post-recovery dream cycle. ACG continuous monitoring state reinitialised. Continuous processing loop resumes.

---


---

[← §15 Deployment](15-deployment.md) | [Index](../README.md) | [§17 Compliance and Data Residency →](17-compliance-and-data-residency.md)
