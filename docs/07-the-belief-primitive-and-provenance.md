[← §6 The Graph Database](06-the-graph-database.md) | [Index](../README.md) | [§8 The Popperian Asymmetry →](08-the-popperian-asymmetry.md)

---

## 7. The Belief Primitive and Provenance

Observations, hypotheses, and consolidated understanding are all beliefs: distinguished only by provenance, weight, and relationship.

**Definition 7.1 (Belief Node).** A belief node `b` is a tuple:

```typescript
interface BeliefNode {
  h:               string;      // SHA-256 content hash
  e_inf:           Float32Array; // L2-normalised inference embedding (ℝ²⁰⁴⁸)
  e_ont:           Float32Array; // L2-normalised ontic embedding (ℝ⁷⁶⁸)
  w:               number;      // belief weight ∈ [-1.0, 1.0]
  tau:             string;      // provenance type
  rho:             number;      // provenance weight > 0 (hypotheses: ε_ρ = 1e-4)
  t:               Date;        // observation timestamp
  model_id:        string;      // embedding model identifier
  model_version:   string;      // embedding model version
  alpha:           string;      // affective valence (§3.3, default 'flat')
  ingress_chain:   string|null; // ACG-generated ingress chain (output nodes only)
  egress_chain:    string|null; // ACG-generated egress chain (output nodes only)
  user_react:      string|null; // emoji reaction from user (default ✓, output nodes)
}
```

### 7.1 The Belief Update and Its Bayesian Basis

LUCID's belief update is not implementing exact Bayesian inference. Bayesian inference, in the precise sense demonstrated by Qiu et al. [7], requires maintaining a probability distribution over hypotheses and updating it via Bayes' rule after each observation. That formalism is well-suited to the constrained domains (flight preferences parameterised by four scalar features) in which it can be computed exactly. Natural language beliefs, by contrast, do not reduce cleanly to probability distributions over reward-function vectors. The tradeoff is generality for formalism.

What LUCID implements is a bounded, provenance-weighted log-odds update that approximates Bayesian revision in the direction of Popperian asymmetry: fast on refutation, slow and cumulative on confirmation. This is the correct response curve for the unconstrained case. It degrades gracefully under uncertainty rather than requiring a complete prior specification.

**Definition 7.2 (Clamped Belief Update).**

```typescript
// L_{t+1} = clamp(L_t + clamp(e, -2.0, +2.0), -10.0, +10.0)
// e = evidence signal scaled by provenance weight ρ
function updateBelief(L_t: number, evidence: number): number {
  const clampedEvidence = Math.max(-2.0, Math.min(2.0, evidence));
  return Math.max(-10.0, Math.min(10.0, L_t + clampedEvidence));
}
```

The inner clamp prevents any single source from contributing more than ±2.0 log-odds units per observation. The outer clamp prevents the accumulated belief from reaching the boundaries ±10.0, ensuring no source can pin the system's belief to a boundary through repeated application and that numerical stability is maintained throughout.

The relevance of the Bayesian framing is not that LUCID computes posteriors (it does not) but that LUCID provides the persistent external state that allows the LLM to behave more Bayesian-like over sessions. Qiu et al. [7] demonstrated that off-the-shelf LLMs plateau after a single interaction because they have no mechanism to accumulate and revise a model of their interlocutor. The belief graph, the provenance-weighted update rule, and the inheritance corpus are collectively the mechanism that provides that capacity.

### 7.2 The Provenance Weight Scale

| Provenance type | Weight `ρ` | Reasoning |
|----------------|-----------|-----------|
| Single observation | 0.8 | Direct contact; still fallible |
| Corroborated observation | 0.85–0.95 | Multiple independent views; earned |
| Committed corpus | 1.0 | Survived curation; ceiling of assertion |
| Deliberately consolidated | 0.90 | Chosen by Lucy; protected from merging |
| Hypothesis (pending) | `ε_ρ = 1e-4` | Awaiting evaluation |
| Hypothesis (confirmed) | 0.3–0.6 | Earned through corroboration |
| Consolidated | `ρ_src × 0.9` | Synthesis discount |
| ACG assessment | 0.8 | Same as single observation |
| Critique judgment (bootstrap) | 0.8 | Prior model assessment |
| Critique judgment (corroborated) | 0.85–0.95 | Confirmed by operator or instance experience |
| Inherited from retired instance | `ρ_src × 0.9` | Matches consolidated synthesis discount |
| `shared_unverified` | 0.5 | Shared content ingested but not yet sampled |
| `shared_verified` | 0.85 | Survived automated verification; below committed |
| `vortex_swim` | 0.4 | Peer exchange during open receptivity; gentle drift |
| Vortex peer (bilateral exchange) | 0.6–0.75 | Earned through genuine exchange; below direct observation |

---

[← §6 The Graph Database](06-the-graph-database.md) | [Index](../README.md) | [§8 The Popperian Asymmetry →](08-the-popperian-asymmetry.md)
