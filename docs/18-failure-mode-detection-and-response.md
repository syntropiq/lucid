[← §17 Compliance and Data Residency](17-compliance-and-data-residency.md) | [Index](../README.md) | [§19 Security Model →](19-security-model.md)

---

## 18. Failure Mode Detection and Response

| Failure Mode | Detection Signal | Response |
|-------------|-----------------|----------|
| Indexing worker failure | `index_ratio` falling, embedding queue depth rising | Restart worker; alert operator |
| Affinity edge failure | `integration_ratio` falling, integrated nodes isolated | Requeue integration jobs |
| Integration backlog | `integration_ratio` low at dream cycle trigger | Prioritise integration pass before consolidation |
| Toxic content | Pre-screen divergence > threshold | Flag for priority review; hold affinity edges |
| Prompt injection | $I(n)$ high; $S(n)$ elevated | Hold and judge |
| Exfiltration attempt | Egress matches known incorrect pattern | ACG intercepts |
| Attentional loop | CfC basin collapse | Distributional interrupt |
| Distributional fixation | CfC attractor drift | Consolidation; crystallisation brake |
| Persona drift | $d(C_i, C_0)$ elevated | Cap update; soft correction |
| Integration erosion | $\phi < \phi_0(1 - \delta)$ | Recovery recommended |
| Cap benchmark failure | Cap fails judgment gate | Discard; soft correction |
| Pathological graph | Output tell; hash mismatches | Graph recovery |
| Judgment corpus corruption | Inheritance distribution shifts | Offline scrub; corpus rebuild |
| Critic miscalibration | Systematic critique/operator disagreement | Critic recalibration |
| ACG calibration loss | ACG assessments systematically overridden | ACG judgment record reset |
| Tour asymmetry (inference-only) | Inference-tour-only nodes proliferating | Hold and judge |
| Both tours elevated cost | Widespread Hebbian degradation | Dream cycle; crystallisation |
| Large unreachable set | Severe structural isolation | Recovery |
| Affective flatline | Chain scalar projection invariant across configured window | TripleDent Gum armed; spectral tier assessment |
| Affective corpus corruption | Affective valence distribution shifts anomalously | Rebuild from backup; reassociate by content hash |
| Interiority spiral (early) | DC dominance in inner monitor; high cross-stream correlation; outer/inner divergence | TripleDent Gum Tier 1 or 2 per spectral condition |
| Interiority spiral (full) | Outer monitor DC dominance; orbital health condition violated at 32 pages | TripleDent Gum Tier 3; dream cycle; orbital health reassessment |
| Capture orbit | Orbital health condition violated; $C_o$ absent from enclosed region; figure-8 not emerging at 32 pages | TripleDent Gum; light consolidation; ontic-biased context injection |
| Escape trajectory | CfC attractor drift; orbit not closing | Full consolidation; cap rollback candidate |

---


---

[← §17 Compliance and Data Residency](17-compliance-and-data-residency.md) | [Index](../README.md) | [§19 Security Model →](19-security-model.md)
