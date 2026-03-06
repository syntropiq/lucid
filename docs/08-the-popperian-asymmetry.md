[← §7 The Belief Primitive and Provenance](07-the-belief-primitive-and-provenance.md) | [Index](../README.md) | [§9 The Threat Architecture →](09-the-threat-architecture.md)

---

## 8. The Popperian Asymmetry

Confirmation and refutation are not symmetric.

**Definition 8.1 (Refutation Edge).** A refutation edge carries weight $\omega_r \in [-0.99, -0.8]$. The floor $-0.8$ prevents weaponisation as a near-total erasure mechanism. The ceiling $-0.99$ preserves numerical stability and ensures no refuted node is entirely zeroed out.

**Theorem 8.1 (Refutation Propagation).** The contribution of any path passing through a refuted belief node is multiplied by $(1 + \omega_r) \in [0.01, 0.2]$. Beliefs that depend on a refuted node are accordingly discounted without being removed.

*Proof sketch.* For an edge weight $\omega_r \in [-0.99, -0.8]$, the factor $(1 + \omega_r)$ is bounded in $[0.01, 0.2]$. Any path traversing a REFUTES edge therefore carries at most 20% of its original weight through to downstream nodes, and at least 1% (preserving the node and its connections for future inspection and potential rehabilitation). The discount applies multiplicatively along any path containing one or more refutation edges; paths through multiple refuted nodes are discounted proportionally. $\square$

Refutation is fast and specific; confirmation is cumulative and slow.

This asymmetry is not merely a design choice; it is empirically validated from an unexpected direction. Qiu et al. [[7]](25-references.md) found that a model trained to mimic a probabilistic reasoner — one who makes calibrated guesses under uncertainty and updates them — generalises substantially better than a model trained on oracle outputs (correct answers). The oracle teacher confirms; the Bayesian teacher reasons under uncertainty and revises. The stronger training signal is the one that captures revision, not the one that delivers certainty.

The Popperian asymmetry at the update-rule level and the Bayesian teaching finding at the training level are expressing the same underlying principle: imperfect evidence, appropriately weighted and revised, is a more powerful epistemic substrate than confident assertion. A belief system built on fast refutation and slow confirmation is doing in inference what Bayesian teaching does in training — prioritising the signal from genuine uncertainty over the noise of premature closure.

---

[← §7 The Belief Primitive and Provenance](07-the-belief-primitive-and-provenance.md) | [Index](../README.md) | [§9 The Threat Architecture →](09-the-threat-architecture.md)
