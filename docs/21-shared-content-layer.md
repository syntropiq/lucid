[← §20 Multi-Tenancy](20-multi-tenancy.md) | [Index](../README.md) | [§22 Verification Queue and Sampling →](22-verification-queue-and-sampling.md)

---

## 21. Shared Content Layer

### 21.1 Layer Model

| Layer | Schema | What it contains | Who can write |
|-------|--------|-----------------|---------------|
| Global shared content | `lucid_shared` | Content hashes, verified ontic embeddings | `lucid_shared_curator` via verification queue |
| Per-tenant belief graph | `lucid` | Nodes, edges, centroids, cycles, all identity machinery | Per-tenant SECURITY DEFINER functions |
| Per-tenant identity and affective machinery | `lucid` | Thinking Cap artifacts, inheritance corpus, Self Library, affective corpus, AWE walk log, spectral monitor | Per-tenant SECURITY DEFINER functions |

### 21.2 What "Shared" Means

Content hashes and verified ontic embeddings are shared. Graph connections, centroids, integration mass, inference embeddings, and all affective and spectral data are never shared. Affective valence, ingress and egress chains, mood tokens, user reacts, emotional memory, and spectral health records are per-tenant by definition: they represent how a specific Lucy instance experiences and processes content, and would be neither meaningful nor appropriate to share across instances.

### 21.3 Trust Model

There is no per-source trust accumulation. All external sources — including peer Lucy instances submitting content — receive the same fixed-rate random sampling. Past behaviour does not earn elevated trust. A source that has submitted ten clean batches is treated identically to a new source on its eleventh batch. This is an explicit architectural invariant, not a limitation to be relaxed in future versions.

---


---

[← §20 Multi-Tenancy](20-multi-tenancy.md) | [Index](../README.md) | [§22 Verification Queue and Sampling →](22-verification-queue-and-sampling.md)
