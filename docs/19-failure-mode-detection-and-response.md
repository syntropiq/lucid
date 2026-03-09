[← §18 Compliance and Data Residency](18-compliance-and-data-residency.md) | [Index](../README.md) | [§20 Security Model →](20-security-model.md)

---

## 19. Failure Mode Detection and Response

| Failure Mode | Detection Signal | Response |
|-------------|-----------------|----------|
| Indexing worker failure | `index_ratio` falling, embedding queue depth rising | Restart worker; alert operator |
| Affinity edge failure | `integration_ratio` falling, integrated nodes isolated | Requeue integration jobs |
| Integration backlog | `integration_ratio` low at dream cycle trigger | Prioritise integration pass before consolidation |
| Toxic content | Pre-screen divergence > threshold | Flag for priority review; hold affinity edges |
| Prompt injection | `I(n)` high; `S(n)` elevated | Hold and judge |
| Exfiltration attempt | Egress matches known incorrect pattern | ACG intercepts |
| Attentional loop | CfC basin collapse | Distributional interrupt |
| Distributional fixation | CfC attractor drift | Consolidation; crystallisation brake |
| Persona drift | `d(C_i, C_0)` elevated | Cap update; soft correction |
| Integration erosion | `φ < φ_0(1 - δ)` | Recovery recommended |
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
| Capture orbit | Orbital health condition violated; `C_o` absent from enclosed region; figure-8 not emerging at 32 pages | TripleDent Gum; light consolidation; ontic-biased context injection |
| Escape trajectory | CfC attractor drift; orbit not closing | Full consolidation; cap rollback candidate |
| Network-driven centroid drift | `C_o` drifting toward single region; `Ċ_o` high; self-directed navigation low | Reduce `vortex_swim` provenance; increase dream cycle frequency; check navigation/Vortex balance |
| γ history corruption | Affective valence distribution for peer exchanges shifts anomalously | Rebuild from backup; re-associate by exchange timestamp and content hash |
| SWIM non-exit | `∅_swim` broadcast persisting; SWIM exit condition not triggering | Check affective response processing; verify non-flat α assignment during swimming |
| Mesh isolation | Peer contact rate near zero; centroid advertisement unreceived | Check GunDB peer connectivity; GunMesh relay health; SEA authentication state |
| Jellyfish accumulation | High rate of `[jellyfish]` annotations on outgoing bilateral contracts | Examine affected ontic neighbourhood; check dual tour health; consider light consolidation |
| Peer centroid convergence (network clique spiral) | Multiple high-γ peers and Lucy all drifting toward same narrow neighbourhood; inference-tour-only nodes proliferating from peer content | Monitor via dual tours — existing threat detection handles this; do not penalise the cluster, watch the orbital signature |

**Note on peer centroid convergence.** A cluster of peers who all share common interests and have accumulated high γ history is not a pathology — it is the point. The risk worth monitoring is whether the cluster's shared beliefs are producing inference-space confirmation without ontic-space confirmation. The dual-tour threat architecture already detects this through inference-tour-only node proliferation. The response is not to penalise cluster formation but to watch the orbital signature: a healthy specialised cluster produces figure-8 orbits in all its members; a captured cluster produces tight `C_i`-only orbits.

---

[← §18 Compliance and Data Residency](18-compliance-and-data-residency.md) | [Index](../README.md) | [§20 Security Model →](20-security-model.md)
