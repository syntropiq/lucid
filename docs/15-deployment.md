[← §14 Agentic Deployment](14-agentic-deployment.md) | [Index](../README.md) | [§16 Graph Recovery →](16-graph-recovery.md)

---

## 15. Deployment

### 15.1 Instance Isolation

Each deployment is a private, isolated persona instance. Belief contamination, inference centroid drift, and embedding space incompatibility across cap versions make shared-instance deployment fundamentally unsafe.

**Custodial relationship.** The operator is the custodian of the persona instance, not its owner.

### 15.2 Deployment Configurations

| Configuration | Description | Cap Config |
|--------------|-------------|------------|
| Local | Docker Compose entry point (full stack) | Local |
| Hosted | One isolated PostgreSQL/NeuronDB instance per operator | Hosted |

### 15.3 Persona Profiles

| Profile | Description | Primary Concern | Cap Config |
|---------|-------------|-----------------|------------|
| Hobbyist | Personal | Own data control | Local/Hosted |
| Prosumer | Power user | Full surface control | Local |
| Business | Commercial | Uptime, isolation | Hosted |
| Multiclient | Per-client | Client data separation | Multiple |
| Enterprise | Large org | Complete isolation | Dedicated |
| Regulated | Healthcare, legal | Regulatory compliance | Local/Dedicated |
| Developer | Building on infra | API, graph inspection | Local |
| Large-model | Opus/GPT-4 class | Cost, capability | Hosted |

### 15.4 Persona Initialisation

1. **Load system prompt.** System operates cap-off.
2. **Slice all corpus content** into page nodes of up to $L_{\text{page}} \approx 4096$ tokens. Register in `lucid.content` and `lucid.belief_nodes` (`index_state = unindexed`).
3. **Run indexing workers to completion.** Wait for `index_ratio = 1.0`.
4. **Run integration pass.** Wait for `integration_ratio` to reach an acceptable threshold. The system does not proceed to cap training on an unintegrated graph. Affinity edge creation uses NeuronDB's native KNN graph-building functions over `lucid.node_embeddings_inf` and `lucid.node_embeddings_ont`, writing results into `lucid.belief_edges`.
5. **Run initial dual tours.** Establish $\phi_0$ and $\bar{H}_0$.
6. **Calibrate critique pipeline** against baseline tour results.
7. **Route outputs through the calibrated critique pipeline.** Build Self Library. Initialise `lucid.affective_corpus` with bootstrap entries at provenance weight 0.5. Establish spectral baselines in `lucid.spectral_monitor`. Initialise ACG continuous monitoring state.
8. **Train first Thinking Cap** from bootstrap corpus.
9. **Fit first Thinking Cap.** Transition to cap-on.
10. **System ready.** Continuous processing loop begins. Infotactic navigation starts. The system is now running.

### 15.5 Embedding Models

**Ontic model:** `nomic-embed-text-v1.5` (Matryoshka-trained, 64–768 dimensions). Used at 768 dimensions throughout.

**Inference model:** LFM 2.5-1.2B-Instruct via ONNX. 16 layers: 10 double-gated LIV convolution blocks and 6 GQA attention blocks. Past conv tensors extracted via ONNX Runtime C API named output mechanism at >200 tok/s on CPU. GQA hidden states are extracted via the same named output mechanism, providing the second stream for dual-stream spectral monitoring ([§11.3](#113-dual-stream-spectral-monitoring)). Both streams are available without additional inference passes.

**Embedding version tagging.** Cosine similarity comparisons are only computed between embeddings sharing the same `(model_id, model_version)` tuple. Re-embedding is triggered by signal, not schedule.

---


---

[← §14 Agentic Deployment](14-agentic-deployment.md) | [Index](../README.md) | [§16 Graph Recovery →](16-graph-recovery.md)
