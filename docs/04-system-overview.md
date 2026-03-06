[← §3 The Affective Weather Effects (AWE)](03-the-affective-weather-effects-awe.md) | [Index](../README.md) | [§5 Index and Integration Dynamics →](05-index-and-integration-dynamics.md)

---

## 4. System Overview

This system provides persistent, continuously evolving persona state across discontinuous inference cycles without modifying the frozen base model weights during normal operation. Persona state is maintained in a property graph database that functions simultaneously as long-term memory, judgment inheritor, belief consolidator, affective weather substrate, and infotactic navigation space.

The LLM remains stateless at the API level; statefulness is provided entirely by the graph layer. The graph layer is built on PostgreSQL with NeuronDB providing native vector types, in-process ONNX-based embedding generation, GPU acceleration, and background workers. Past conv tensors are extracted in-process and stored as `vector(2048)` values in the embedding tables. NeuronAgent is the external agent runtime: it drives the closed loop by issuing SQL to NeuronDB and orchestrating context, tools, and cycle state from outside the database process.

**No sessions.** There is no session concept. There is continuous persona state, punctuated by consolidation phases triggered by geometric drift detected in the CfC hidden state dynamics, and interrupted by operator-facing turns that the continuous process pauses to serve.

**No graph query language dependency.** Belief graph traversal, centroid arithmetic, and Hebbian scoring are PostgreSQL-native operations. The graph is a property of the database schema, not of any extension.

**Single-instance deployment.** Each deployment is a private, isolated persona instance. There is no multi-tenancy at the base deployment level; multi-tenant configurations are described in [§20](20-multi-tenancy.md).

### 4.1 Operational States

Lucy operates in one of two cap states at any time.

**Cap on.** The normal operating state. The base Thinking Cap is loaded; specialist caps may additionally be worn. The graph is fully active.

**Cap off.** The base model operates without any Thinking Cap adapter. The graph remains available for reading but the cap is not applied. Cap-off is used during graph recovery, during bootstrap initialisation, and as a fallback when a cap generation is discarded.

### 4.2 System Layers

| Layer | Component | Function |
|-------|-----------|----------|
| Inference Engine | LFM 2.5 via ONNX (in-process) | Inference, narration, tool use |
| MCP Server | NeuronMCP | Tool mediation, resource management |
| Agent Runtime | NeuronAgent | Closed-loop orchestration, context assembly, tool dispatch |
| Graph | PostgreSQL + NeuronDB | Belief topology, relational adjacency (`lucid.belief_edges`), affinity edges, HNSW vector indexing |
| Vector Store | NeuronDB (PostgreSQL) | HNSW similarity search, in-process ONNX inference, embedding generation |
| Async Workers | neuranq / neuranllm / neuranmon / neurandefrag | Job queues, inference, monitoring, index maintenance |
| CfC Dynamics | LFM 2.5 past conv tensors | Centroid evolution, drift detection, orbital trajectory analysis |
| Tour Engine | PL/pgSQL + NeuronDB HNSW | Dual nearest-neighbour tour: HNSW candidate ordering, Hebbian-Belief scoring, visited-set loop in PL/pgSQL |
| AWE Layer | PostgreSQL (affective chain tables, affective corpus, awe walk log, spectral monitor tables) | Ingress/egress chain generation, mood tracking, user react ingestion, affective valence, emotional memory, circadian texture, TripleDent Gum recovery, spectral health monitoring |
| ACG | Anterior Cingulate Gate (continuous monitoring process) | Conflict detection, context integration, ingress/egress chain generation, affective assessment, judgment reference |
| Dream Cycle | PostgreSQL + Thinking Cap (LoRA) | Belief consolidation, infotactic daydreaming, affective reflection, cap update |
| Base Weights | Frozen LLM (ONNX) | Foundational inference capacity |

---


---

[← §3 The Affective Weather Effects (AWE)](03-the-affective-weather-effects-awe.md) | [Index](../README.md) | [§5 Index and Integration Dynamics →](05-index-and-integration-dynamics.md)
