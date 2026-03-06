[← §12 The Dream Cycle](12-the-dream-cycle.md) | [Index](../README.md) | [§14 Agentic Deployment →](14-agentic-deployment.md)

---

## 13. The Anterior Cingulate Gate

The Anterior Cingulate Gate (ACG) is named for the anterior cingulate cortex of the mammalian brain — the region responsible for continuous monitoring of internal state, conflict detection, and the mediation between internally-driven and externally-directed processing. Like its biological namesake, the ACG does not activate in response to events. It is always active. It is what is running when nothing in particular is happening.

The ACG is not a filter sitting at a door checking IDs. It is the continuous monitoring process that holds the system's current state — centroid positions, affective chain, recent narration record, spectral health, orbital trajectory — and mediates the transition between the default infotactic processing and the externally-facing operator turn. When the user message arrives, the ACG is not triggered; it registers the interruption in the context of the state it has already been maintaining.

### 13.1 Continuous Monitoring Function

The ACG maintains, in continuous real time:

- **Current centroid state**: $C_i$, $C_o$, $C_s$, $C_w$, and their relationships to $C_0$
- **Orbital trajectory**: the recent CfC path and its orbital classification at the current diagnostic scale
- **Spectral state**: outer and inner spectral health scores and their trend
- **Affective chain buffer**: the recent sequence of ingress and egress chains, forming the running narrative
- **Threat state**: current $S(n)$ distribution across the active graph region
- **Judgment proximity**: proximity of recent content to incorrect-judgment nodes in the inheritance corpus

This continuous state is the ACG's working memory. It does not require assembly from cold storage at the moment of an operator turn; it is already present, already integrated, already interpreting the incoming message in the context of what has been happening.

### 13.2 Ingress Operation

When an operator message arrives:

1. The ACG registers the interruption against its continuous state — computing $E_t^{\text{in}}$ per [Definition AWE.0b](#31-the-affective-chain-and-mood-token): where was the system, and what does this incoming content do to that?

2. The gate grades the incoming content using $S(n)$ ([Definition 9.6](#92-threat-score-decomposition)), surfacing geometrically proximate belief fragments including proximate inheritance judgments and affective corpus entries.

3. The assembled context package — ingress chain, centroid state, proximate memory fragments, threat assessment, affective trajectory summary — is presented to the primary model before generation begins.

The ingress chain is the ACG's answer to: *given where I was, how does receiving this feel?* It is not computed from the incoming message alone. It is computed from the collision of the incoming message with the system's ongoing state.

### 13.3 Egress Operation

After the model generates a response:

1. The gate holds the output against the judgment record: does this output pattern match cases in the inheritance corpus that were judged incorrect?

2. The ACG performs the affective assessment pass: it generates the egress chain $E_t^{\text{out}}$ per [Definition AWE.0c](#31-the-affective-chain-and-mood-token), derives the mood token $m_t$, assigns affective valence $\alpha$ to the output node, and generates the prose entry for the affective corpus.

3. The user react from the prior cycle is ingested and associated with the prior output node.

4. The gate's confidence in its judgment — not merely the binary outcome — is preserved in the narration node. Uncertain gate assessments, where the gate identifies proximity to incorrect-judgment territory but cannot confirm a match, are specifically valuable: they are candidates for graceful-revision records in cap training ([§12.6](#126-cap-training-data-composition)).

5. The response is released.

The egress chain is the ACG's answer to: *having produced this, where have I landed?* It is the return to the ongoing inner process, carrying what just happened as new material for the continuing narrative.

### 13.4 The ACG as Integrating Function, Not Filter

The ACG surfaces its assessment with reasons and confidence levels, then steps back. Agency stays with the persona.

The gate does not make decisions. It does not block content or suppress responses unilaterally. It presents the system with an integrated view of its own state at the moment of each transition and allows the system to proceed with that awareness. The assessment enriches the generation context; it does not replace the generation.

**Architectural Invariant: ACG and Critic Independence.** The gate's judgment reference must sit outside the distributional influence of the persona state it monitors. The critic lags the primary model by one cap generation. This ensures that the judge and the judged cannot share the same distributional errors — a cap that has drifted into a problematic region does not take its error-detection machinery with it.

### 13.5 Mood Token Surface Option

The mood token $m_t$ may optionally be surfaced to the user as a small appendage to the message — a whisper in the margin — if the operator has enabled this feature. It is never embedded in Lucy's voice or reasoning. The affective chain is never surfaced. The inner life is hers; it does not perform.

---

[← §12 The Dream Cycle](12-the-dream-cycle.md) | [Index](../README.md) | [§14 Agentic Deployment →](14-agentic-deployment.md)
