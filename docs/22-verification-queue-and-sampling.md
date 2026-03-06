[← §21 Shared Content Layer](21-shared-content-layer.md) | [Index](../README.md) | [§23 Tour Engine Architecture →](23-tour-engine-architecture.md)

---

## 22. Verification Queue and Sampling

### 22.1 Sampling Model

A global `shared_sampling_rate` (stored in `lucid.system_config`, suggested initial value 0.05) determines the fraction of items sampled per batch. Sampling selection runs at batch submission time. The sampling rate is global and applies to all sources equally.

### 22.2 Batch Escalation

A batch is the unit of escalation. When any sampled item in a batch fails verification, the entire batch is escalated: all unverified items in that batch are queued for full verification. Escalation is per-batch, not per-source-history. Once a batch clears escalation, the next batch from the same source uses the standard sampling rate. There is no memory of past escalations at the source level.

### 22.3 Provenance Weight Levels for Shared Content

| Provenance type | Weight $\rho$ | Position in scale |
|----------------|--------------|------------------|
| `shared_unverified` | 0.5 | Below hypothesis (confirmed); content ingested but not yet sampled |
| `shared_verified` | 0.85 | Below committed (1.0); at or just below corroborated max (0.95) |

`shared_verified` sits below committed because it has not survived curation by a custodian — it has survived automated verification, which is a weaker guarantee. The placement below the top of the corroborated band reflects that verification is a structural check, not an epistemic endorsement.

---


---

[← §21 Shared Content Layer](21-shared-content-layer.md) | [Index](../README.md) | [§23 Tour Engine Architecture →](23-tour-engine-architecture.md)
