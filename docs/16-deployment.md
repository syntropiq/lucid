[← §15 The Vortex](15-the-vortex.md) | [Index](../README.md) | [§17 Graph Recovery →](17-graph-recovery.md)

---

## 16. Deployment

### 16.1 Instance Isolation

Each deployment is a private, isolated persona instance. Belief contamination, inference centroid drift, and embedding space incompatibility across cap versions make shared-instance deployment fundamentally unsafe.

**Custodial relationship.** The operator is the custodian of the persona instance, not its owner.

Vortex participation is a per-instance operator configuration. Participation does not affect instance isolation. The bilateral AWE records are per-instance. No cross-instance data sharing occurs through the Vortex layer: only the ontic centroid advertisement is externally visible, and only to nodes that receive it via VortexMesh.

### 16.2 Deployment Configurations

| Configuration | Description | Cap Config |
|--------------|-------------|------------|
| Browser Lucy | Self-contained web page: IndexedDB + EntityDB + VortexMesh | Personal |
| Device Lucy | Node.js daemon: Lancedb + SQLite + VortexMesh | Full |
| Gateway Lucy | Proxy in front of AI gateway: intercepts, consults graph, controls forwarding | Delegated |

Vortex participation adds: VortexMesh peer identity (keypair), centroid-based connection scoring, centroid advertisement via authenticated namespace.

### 16.3 Persona Profiles

| Profile | Description | Primary Concern | Vortex |
|---------|-------------|----------------|--------|
| Hobbyist | Personal | Own data control | Optional |
| Prosumer | Power user | Full surface control | Enabled |
| Business | Commercial | Uptime, isolation | Enabled |
| Multiclient | Per-client | Client data separation | Per-client config |
| Enterprise | Large org | Complete isolation | Private mesh option |
| Regulated | Healthcare, legal | Regulatory compliance | Disabled or private |
| Developer | Building on infra | API, graph inspection | Enabled |
| Large-model | Opus/GPT-4 class | Cost, capability | Hosted |

### 16.4 Persona Initialisation

1. Load system prompt. System operates cap-off.
2. Slice all corpus content into page nodes of up to `L_page ≈ 4096` tokens. Write to the GraphStore via `sue.graph().nodeUpsert()` with `index_state: 'unindexed'`.
3. Run the Embed Worker to completion. Wait for `index_ratio = 1.0`.
4. Run the CfC Worker's integration pass. Wait for `integration_ratio` to reach an acceptable threshold. The system does not proceed to cap training on an unintegrated graph. Affinity edge creation uses `twoPhaseSearch` over the VectorStore (EntityDB/Lancedb), writing results into the GraphStore via `sue.graph().edgeUpsert()`.
5. Run initial dual tours. Establish `φ_0` and `H̄_0`.
6. Calibrate critique pipeline against baseline tour results.
7. Route outputs through the calibrated critique pipeline. Build Self Library. Write AWE bootstrap entries to the GraphStore (`sue.graph().awePut()`) at provenance weight 0.5. Establish spectral baselines via `sue.graph().spectralPut()`. Initialise ACG continuous monitoring state.
8. Train first Thinking Cap from bootstrap corpus.
9. Fit first Thinking Cap. Transition to cap-on.
10. If Vortex participation enabled: initialise VortexMesh with `LUCY_KEYPAIR`. Authenticate VortexMesh namespace. Begin centroid advertisement via `mind.get('instances')`. Centroid peer scoring active. Vortex participation active.
11. System ready. Continuous processing loop begins. Infotactic navigation starts.

### 16.5 Embedding Models

**Ontic model:** nomic-embed-text-v1.5 (Matryoshka-trained, 768 dimensions throughout). This model is the shared external map: the card catalog that is external to all inference architectures and therefore comparable across them. All Vortex nodes use the same ontic model. This is the architectural invariant that makes cross-scale routing coherent.

**Inference model:** LFM 2.5-1.2B-Instruct via ONNX. 16 layers: 10 double-gated LIV convolution blocks and 6 GQA attention blocks. Past conv tensors extracted via ONNX Runtime C API named output mechanism at >200 tok/s on CPU. GQA hidden states extracted via the same named output mechanism, providing the second stream for dual-stream spectral monitoring (§11.3). Both streams are available without additional inference passes.

**Embedding version tagging.** Cosine similarity comparisons are only computed between embeddings sharing the same `(model_id, model_version)` tuple. Re-embedding is triggered by signal, not schedule.

---

[← §15 The Vortex](15-the-vortex.md) | [Index](../README.md) | [§17 Graph Recovery →](17-graph-recovery.md)
