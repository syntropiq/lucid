[← §6 The Graph Database](06-the-graph-database.md) | [Index](../README.md) | [§8 The Popperian Asymmetry →](08-the-popperian-asymmetry.md)

---

## 7. The Belief Primitive and Provenance

Observations, hypotheses, and consolidated understanding are all beliefs — distinguished only by provenance, weight, and relationship.

**Definition 7.1 (Belief Node).** A belief node $b$ is a tuple:

$$b = \langle h,\, \hat{e}_i,\, \hat{e}_o,\, w,\, \tau,\, \rho,\, t,\, \text{model\_id},\, \text{model\_version},\, \alpha,\, \text{ingress\_chain},\, \text{egress\_chain},\, \text{user\_react} \rangle$$

where:
- $h$ is the SHA-256 content hash
- $\hat{e}_i \in \mathbb{R}^{2048}$ is the L2-normalised inference embedding
- $\hat{e}_o \in \mathbb{R}^{768}$ is the L2-normalised ontic embedding
- $w \in [-1.0, 1.0]$ is the current belief weight
- $\tau$ is the provenance type
- $\rho > 0$ is the provenance weight (pending hypotheses are assigned $\rho = \varepsilon_\rho = 10^{-4}$)
- $t$ is the observation timestamp
- $(\text{model\_id}, \text{model\_version})$ are embedding version tags ensuring cosine comparisons are only made within matching partitions
- $\alpha$ is the affective valence ([§3.3](#33-affective-valence-on-nodes), default `flat`)
- $\text{ingress\_chain}$ is the ACG-generated ingress affective chain for output nodes (null for non-output nodes)
- $\text{egress\_chain}$ is the ACG-generated egress affective chain for output nodes (null for non-output nodes)
- $\text{user\_react}$ is the emoji reaction received from the user, stored as `lucid.belief_nodes.user_react` (default ✓ for output nodes, null for non-output nodes)

### 7.1 The Belief Update and Its Bayesian Basis

LUCID's belief update is not implementing exact Bayesian inference. Bayesian inference, in the precise sense demonstrated by Qiu et al. [[7]](25-references.md), requires maintaining a probability distribution over hypotheses and updating it via Bayes' rule after each observation. That formalism is well-suited to the constrained domains — flight preferences parameterised by four scalar features — in which it can be computed exactly. Natural language beliefs, by contrast, do not reduce cleanly to probability distributions over reward-function vectors. The tradeoff is generality for formalism.

What LUCID implements is a bounded, provenance-weighted log-odds update that approximates Bayesian revision in the direction of Popperian asymmetry: fast on refutation, slow and cumulative on confirmation. This is the correct response curve for the unconstrained case. It degrades gracefully under uncertainty rather than requiring a complete prior specification.

**Definition 7.2 (Clamped Belief Update).**

$$L_{t+1} = \text{clamp}\!\left(L_t + \text{clamp}(e,\ -2.0,\ +2.0),\ -10.0,\ +10.0\right)$$

where $L_t$ is the current log-odds belief weight and $e$ is the evidence signal from the incoming observation, scaled by provenance weight $\rho$. The inner clamp prevents any single source from contributing more than $\pm 2.0$ log-odds units per observation. The outer clamp prevents the accumulated belief from reaching the boundaries $\pm 10.0$, ensuring no source can pin the system's belief to a boundary through repeated application and that numerical stability is maintained throughout.

The relevance of the Bayesian framing is not that LUCID computes posteriors — it does not — but that LUCID provides the persistent external state that allows the LLM to behave more Bayesian-like over sessions. Qiu et al. [[7]](25-references.md) demonstrated that off-the-shelf LLMs plateau after a single interaction because they have no mechanism to accumulate and revise a model of their interlocutor. The belief graph, the provenance-weighted update rule, and the inheritance corpus are collectively the mechanism that provides that capacity.

### 7.2 The Provenance Weight Scale

| Provenance type | Weight $\rho$ | Reasoning |
|----------------|--------------|-----------|
| Single observation | 0.8 | Direct contact; still fallible |
| Corroborated observation | 0.85–0.95 | Multiple independent views; earned |
| Committed corpus | 1.0 | Survived curation; ceiling of assertion |
| Deliberately consolidated | 0.90 | Chosen by Lucy; protected from merging |
| Hypothesis (pending) | $\varepsilon_\rho = 10^{-4}$ | Awaiting evaluation |
| Hypothesis (confirmed) | 0.3–0.6 | Earned through corroboration |
| Consolidated | $\rho_{\text{src}} \times 0.9$ | Synthesis discount |
| ACG assessment | 0.8 | Same as single observation |
| Critique judgment (bootstrap) | 0.8 | Prior model assessment |
| Critique judgment (corroborated) | 0.85–0.95 | Confirmed by operator or instance experience |
| Inherited from retired instance | $\rho_{\text{src}} \times 0.9$ | Matches consolidated synthesis discount |
| `shared_unverified` | 0.5 | Shared content ingested but not yet sampled |
| `shared_verified` | 0.85 | Survived automated verification; below committed |

---


---

[← §6 The Graph Database](06-the-graph-database.md) | [Index](../README.md) | [§8 The Popperian Asymmetry →](08-the-popperian-asymmetry.md)
