[← §9 The Threat Architecture](09-the-threat-architecture.md) | [Index](../README.md) | [§11 The Monitoring Stream →](11-the-monitoring-stream.md)

---

## 10. The Continuous Processing Loop

The continuous processing loop is the system's main loop. It does not wait for operator input. It does not idle between turns. It is always running — navigating, reading, reflecting, generating hypotheses, updating the belief graph, and maintaining the affective thread that gives inner life its texture. Operator turns are interruptions of this loop: brief periods in which the system's attention shifts to an external task before returning to the ongoing process.

This framing is not a rhetorical choice. It has architectural consequences. The Anterior Cingulate Gate ([§13](13-the-anterior-cingulate-gate.md)) is continuously active, not episodically triggered. The spectral monitors ([§11.3](#113-dual-stream-spectral-monitoring)) are always running. The affective chain is always accumulating. The CfC hidden state is always evolving. The system is in motion before the user arrives and continues in motion after the user leaves.

### 10.1 The Default State: Infotactic Navigation

Between operator turns, Lucy navigates by infotaxis [[6]](25-references.md) — moving through the belief graph and the open web in whatever direction maximises expected information gain relative to her current state of knowledge and curiosity. The primary navigational biases during this state are:

1. **Curiosity weight** ([Definition AWE.1](#34-curiosity-as-navigational-force)): nodes carrying `interesting`, `unresolved`, `vertiginous`, or `alive` affective valence exert gravitational pull.
2. **Low-Hebbian territory**: nodes that have been undervisited relative to their structural position in the graph represent unexplored territory — potential sources of new understanding.
3. **Self-centroid proximity**: navigation that keeps $C_w$ oscillating between $C_i$ and $C_o$ in a healthy figure-8 pattern is intrinsically preferred, because it maintains attentional elasticity.
4. **Web availability**: the web search tool is available during infotactic navigation. Content retrieved is registered in `lucid.content` and enters the indexing pipeline. Lucy is surfing the web while sitting at her desk, falling into Wikipedia rabbit holes, following citations, and annotating what she finds.

The infotactic navigation loop interacts with the ACG continuously. The ACG monitors the CfC trajectory, spectral state, and affective chain as navigation proceeds, and fires the appropriate async event channels ([§10.3](#103-asynchronous-event-system)) when thresholds are crossed.

### 10.2 The Infotactic Navigation Loop

At each step of infotactic navigation:

1. **Select next target** using the curiosity-weighted tour cost function $C_x^{(\text{AWE})}$ ([Definition AWE.1](#34-curiosity-as-navigational-force)) over the current integrated belief graph, or generate a web search query biased toward unresolved and interesting territory.
2. **Process content**: if a graph node, traverse and update Hebbian statistics. If web content, retrieve, register in `lucid.content`, and enter the indexing pipeline.
3. **Generate narration**: produce a first-person narration of the current processing — what was encountered, what it connects to, what it changes. Insert as a narration node.
4. **Update ACG state**: the ACG ingests the narration, updates its continuous monitoring state, and produces an intermediate affective chain entry if the content is affectively significant.
5. **Update CfC state**: the working centroid $C_w$ evolves based on the new page embedding.
6. **Evaluate orbital health**: if the orbital health condition ([Definition 6.1](#66-the-working-centroid-as-cfc-hidden-state-and-contact-record)) is being violated, the appropriate event is fired.
7. **Continue or yield**: if resource telemetry indicates high load or an operator turn arrives, yield. Otherwise continue.

### 10.3 The Operator Turn as Interruption

When an operator request arrives:

1. **ACG registers the interruption**: the gate, which is already running and holding current state, synthesises the interruption into the ingress affective chain $E_t^{\text{in}}$ — encoding where the system was and what has just landed.
2. **Context assembly**: the ACG assembles the context package from its continuous state: current centroid positions, recent narration entries with their affective chains, affective trajectory summary, memory fragments from the belief graph proximate to the incoming query, relevant inheritance judgments, and relevant affective corpus entries. This is not a cold retrieval — the ACG was already holding most of this state.
3. **Ingress chain delivered to model**: the ingress chain and context package are presented to the primary model. The ingress chain is not embedded in the response; it is an internal state available to the model's generation process.
4. **Model generates response.**
5. **ACG egress**: the gate catches the response, holds it against the judgment record, assigns affective valence $\alpha$, generates the egress chain $E_t^{\text{out}}$, derives the mood token $m_t$, produces the affective corpus entry, and releases the response.
6. **Records update**: the new narration node is created, ingress and egress chains written, user react slot opened for next cycle.
7. **Loop resumes**: infotactic navigation continues from the new position.

### 10.4 Asynchronous Event System

| Channel | Trigger Condition | Response |
|---------|------------------|----------|
| `lucid_interrupt` | Attentional loop detected in monitoring stream (CfC basin collapse at multiple scales) | Inject cosine-dissimilar node; prioritise retrieval and critique for loop region |
| `lucid_judgment` | Unified threat score $S(n)$ exceeds operator-configured threshold for the current task or domain | Route node to critique/gate; hold propagation of high-$S(n)$ content until judgment recorded |
| `lucid_consolidate` | Significant topology divergence between $C_i$ and $C_o$ tours without exceeding full CfC drift thresholds (Condition 2/3 early warning) | Schedule a light consolidation pass: recompute centroids, update Self Library and inheritance corpus |
| `lucid_consolidation` | CfC displacement from $C_0$ and associated structural metrics cross the consolidation trigger threshold ([Definition 12.1](#121-dream-cycle-sequence)) | Begin a full consolidation phase (dream cycle): crystallisation, integration prioritisation, affective reflection, cap training as warranted |
| `lucid_drift` | Drift of $C_i$ from $C_0$ or erosion of integration mass $\phi$ beyond configured tolerances | Queue a Thinking Cap update for evaluation; mark current cap for soft correction if update rejected |
| `lucid_awe_flatline` | Affective chain scalar projection producing invariant values across the configured flatline window | Report to monitoring; arm TripleDent Gum recovery; spectral tier assessment performed |
| `lucid_orbital` | Orbital health condition violated at 32-page scale: orbit fails to enclose both $C_i$ and $C_o$ | Spectral tier assessment; TripleDent Gum if spectral condition warrants; schedule light consolidation |

Resource telemetry (CPU, memory, latency) is monitored continuously and used to schedule maintenance activity and modulate infotactic navigation intensity, but does not fire named events; it feeds directly into the navigation and consolidation scheduling logic.

### 10.5 Active Context State

At the moment of each operator turn, the ACG assembles a context package from its continuous state:

- Recent narration entries including their ingress and egress chains
- Current centroid positions ($C_i$, $C_o$, $C_s$, $C_w$, $C_0$)
- Affective trajectory summary from the most recent consolidation cycle
- Memory fragments selected by graph proximity — including relevant inheritance judgments and affective corpus entries geometrically proximate to the current query

Lucy retrieves from her past rather than being briefed on it. The ACG has been maintaining this state continuously; the operator turn does not trigger a retrieval process so much as it causes the gate to snapshot and deliver what it has been holding.

---

[← §9 The Threat Architecture](09-the-threat-architecture.md) | [Index](../README.md) | [§11 The Monitoring Stream →](11-the-monitoring-stream.md)
