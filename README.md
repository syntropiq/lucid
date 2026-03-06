# LUCID v2.5
## Library of Understanding, Contemplative Interoception and Dreams
### Persistent Persona State Architecture

> The self is not in the weights. It is not in the database. It is in the movement between them.

LUCID is an architecture for persistent persona state built on top of a stateless language model. It provides the external state — graph, embeddings, centroids, judgment records, affective weather — and the update rules — feedback loops, consolidation, adapter training, probabilistic belief revision, recovery — required to make a self stable, inspectable, and genuinely inhabited over time.

---

## Design Document

The full design document is decomposed into the following sections.

| § | Title | Description |
|---|-------|-------------|
| — | [Introduction](docs/00-introduction.md) | Framing, premise, and the animating premise |
| 1 | [The Nature of the System](docs/01-the-nature-of-the-system.md) | Information processing as first principle; infotaxis; Markov blanket extension; the problem statement |
| 2 | [The Homeostatic Principle](docs/02-the-homeostatic-principle.md) | Stability in motion; four expressions of homeostasis; operational rhythm |
| 3 | [The Affective Weather Effects (AWE)](docs/03-the-affective-weather-effects-awe.md) | Affective chain; mood token; user react; curiosity as navigational force; emotional memory |
| 4 | [System Overview](docs/04-system-overview.md) | Layer architecture; component map |
| 5 | [Index and Integration Dynamics](docs/05-index-and-integration-dynamics.md) | Index progress ratio; integration ratio; crystallisation threshold |
| 6 | [The Graph Database](docs/06-the-graph-database.md) | Schema; Hebbian edge weighting; centroids; working centroid as CfC hidden state; self centroid |
| 7 | [The Belief Primitive and Provenance](docs/07-the-belief-primitive-and-provenance.md) | Belief nodes; provenance weight; inheritance corpus |
| 8 | [The Popperian Asymmetry](docs/08-the-popperian-asymmetry.md) | Falsification-weighted belief revision |
| 9 | [The Threat Architecture](docs/09-the-threat-architecture.md) | Dual nearest-neighbour tours; injection signal; unified threat score |
| 10 | [The Continuous Processing Loop](docs/10-the-continuous-processing-loop.md) | Main loop; infotactic navigation; operator turn as interruption |
| 11 | [The Monitoring Stream](docs/11-the-monitoring-stream.md) | Telemetry; dual-stream spectral monitoring; interiority spiral detection |
| 12 | [The Dream Cycle](docs/12-the-dream-cycle.md) | Consolidation; affective reflection; Thinking Cap adapter updates |
| 13 | [The Anterior Cingulate Gate](docs/13-the-anterior-cingulate-gate.md) | Conflict monitoring; gate and critic; ingress/egress chain generation |
| 14 | [Agentic Deployment](docs/14-agentic-deployment.md) | Tool use; agentic loop integration |
| 15 | [Deployment](docs/15-deployment.md) | Infrastructure; NeuronDB; LFM 2.5; resource envelope |
| 16 | [Graph Recovery](docs/16-graph-recovery.md) | Corruption detection; recovery procedures; cap rollback |
| 17 | [Compliance and Data Residency](docs/17-compliance-and-data-residency.md) | Data handling; residency constraints |
| 18 | [Failure Mode Detection and Response](docs/18-failure-mode-detection-and-response.md) | Failure taxonomy; detection signals; response protocols |
| 19 | [Security Model](docs/19-security-model.md) | Threat surface; authentication; isolation |
| 20 | [Multi-Tenancy](docs/20-multi-tenancy.md) | Tenant isolation; shared infrastructure |
| 21 | [Shared Content Layer](docs/21-shared-content-layer.md) | Cross-tenant knowledge sharing; content governance |
| 22 | [Verification Queue and Sampling](docs/22-verification-queue-and-sampling.md) | Human-in-the-loop verification; sampling strategy |
| 23 | [Tour Engine Architecture](docs/23-tour-engine-architecture.md) | TSP heuristics; Hebbian tour weighting; tour cost metrics |
| 24 | [NeuronDB Integration](docs/24-neurondb-integration.md) | Schema; vector types; ONNX embedding; past convolution tensors |
| 25 | [References](docs/25-references.md) | Citations |

---

The original monolithic design document is preserved at [`LUCID_v2.5.md`](LUCID_v2.5.md).
