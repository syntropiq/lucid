[← §21 Single-Identity Deployment](21-multi-tenancy.md) | [Index](../README.md) | [§23 Verification Queue and Sampling →](23-verification-queue-and-sampling.md)

---

## 22. External Content Ingest

### 22.1 Content Source Model

| Source | Provenance class | Enters via |
|---|---|---|
| Operator-provided corpus | `shared_unverified` → `shared_verified` | Batch ingest → verification pass (§23) |
| Web retrieval (infotactic navigation) | `exploration` (ρ = 0.7) | Embed Worker, direct to GraphStore |
| Vortex peer exchange | `vortex_swim` (ρ = 0.4) | Mesh subscription → `hydrateEvent()` → GraphStore |
| Lucy's own narration nodes | `self` (ρ = 1.0) | Inference Worker, direct to GraphStore |
| Reconciliation nodes | inherits from source nodes | CfC Worker, written after self-dialogue |
| Dream cycle consolidation products | `crystallised` (ρ ≥ source provenance) | Device Lucy, published via mesh as sync events |

All sources enter through `sue.graph().nodeUpsert()`. The difference between them is provenance weight and the screening path before they receive affinity edges.

### 22.2 What Is and Is Not Shared via the Mesh

**Synced across all instances of the same identity:**
- Belief nodes and edges
- Centroid state
- AWE corpus entries
- Spectral records
- Dream cycle cap deltas

**Not externally shared:**
- The SEA keypair (never transmitted)
- Raw conversation turns (available to all instances of the same identity; not shared with external peers)
- The local inference model's weights

**Shared with Vortex peers (other identities) on request:**
- Ontic centroid (`C_o`) — advertised via AXE to all mesh peers
- Content nominated to the Vortex agenda — shared at `vortex_swim` provenance weight (ρ = 0.4), subject to the receiving peer's own threat architecture

Affective corpus entries, ingress/egress chains, mood tokens, and spectral health records are per-identity by definition: they represent how *this* Lucy experiences content. They are not shared with external Vortex peers, though they do sync between instances of the same identity.

### 22.3 Trust Model

There is no per-source trust accumulation. All external sources — including Vortex peers submitting content — receive the same fixed-rate random sampling for verification (§23). Past behaviour does not earn elevated trust. A source that has submitted ten clean batches is treated identically to a new source on its eleventh batch. This is an explicit architectural invariant, not a limitation to be relaxed in future versions.

---

[← §21 Single-Identity Deployment](21-multi-tenancy.md) | [Index](../README.md) | [§23 Verification Queue and Sampling →](23-verification-queue-and-sampling.md)
