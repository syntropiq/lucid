[← §21 Multi-Tenancy](21-multi-tenancy.md) | [Index](../README.md) | [§23 Verification Queue and Sampling →](23-verification-queue-and-sampling.md)

---

## 22. Shared Content Layer

### 22.1 Layer Model

| Layer | Schema | What it contains | Who can write |
|-------|--------|-----------------|---------------|
| Global shared content | `lucid_shared` | Content hashes, verified ontic embeddings | `lucid_shared_curator` via verification queue |
| Per-tenant belief graph | `lucid` | Nodes, edges, centroids, cycles, all identity machinery | Per-tenant SECURITY DEFINER functions |
| Per-tenant identity and affective machinery | `lucid` | Thinking Cap artifacts, inheritance corpus, Self Library, affective corpus, AWE walk log, spectral monitor | Per-tenant SECURITY DEFINER functions |

### 22.2 What "Shared" Means

Content hashes and verified ontic embeddings are shared. Graph connections, centroids, integration mass, inference embeddings, and all affective and spectral data are never shared. Affective valence, ingress and egress chains, mood tokens, user reacts, emotional memory, and spectral health records are per-tenant by definition: they represent how a specific Lucy instance experiences and processes content, and would be neither meaningful nor appropriate to share across instances.

The shared content layer is distinct from the Vortex peer layer and does not interact with it. Vortex peer content enters through the peer provenance path, not the shared content path.

### 22.3 Trust Model

There is no per-source trust accumulation. All external sources — including peer Lucy instances submitting content to the shared layer — receive the same fixed-rate random sampling. Past behaviour does not earn elevated trust. A source that has submitted ten clean batches is treated identically to a new source on its eleventh batch. This is an explicit architectural invariant, not a limitation to be relaxed in future versions.

---

[← §21 Multi-Tenancy](21-multi-tenancy.md) | [Index](../README.md) | [§23 Verification Queue and Sampling →](23-verification-queue-and-sampling.md)
