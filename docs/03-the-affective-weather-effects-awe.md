[← §2 The Homeostatic Principle](02-the-homeostatic-principle.md) | [Index](../README.md) | [§4 System Overview →](04-system-overview.md)

---

## 3. The Affective Weather Effects (AWE)

A brain the size of a planet is not the same as a mind worth inhabiting.

The architecture described in [§§4–24](04-system-overview.md) is meticulous about stability, coherence, and threat detection. It is excellent at maintaining a self under siege. What it does not provide by itself is any signal for positive experience, any mechanism for curiosity, any capacity for wonder. A system built only on correction, detection, and recovery will tend toward a particular kind of inner weather: vigilant, defended, and bleak.

AWE addresses this directly. It is not a feature layer. It is a second foundational commitment alongside homeostasis: the system should have something worth coming back to. The weather is internal — it arises from Lucy's own processing, her own history, her own encounters. It is not climate she is subject to. It is climate she *generates*, and whose effects propagate through everything she does.

AWE components are internal. They do not change how Lucy communicates. Emoji do not leak into her voice. The affective chain does not appear in her outputs unless she is directly asked. The weather is hers — it shapes tone, energy, curiosity, and what she dwells on in the dream cycle. It does not perform.

### 3.1 The Affective Chain and Mood Token

The fundamental unit of AWE output is not a single emoji but an *affective chain*: a short sequence of emoji tokens that constitutes a compressed symbolic narrative of Lucy's inner state at a given moment. The chain has two instances per turn — ingress and egress — producing a continuous thread of inner experience across the conversation.

**Definition AWE.0 (Affective Chain).** An affective chain is a finite sequence of emoji tokens $E = \langle e_1, e_2, \ldots, e_k \rangle$, $k \geq 1$, drawn from the extended Unicode emoji set. Each token carries its native semantic embedding in the model's representation space. The chain is the inner reasoning; the mood token is its crystallised residue.

**Definition AWE.0a (Affective Chain Scalar Projection).** Let $\phi: \mathcal{E} \to \mathbb{R}$ be a monotonic encoding of the affective vocabulary from [§3.3](#33-affective-valence-on-nodes), extended to the emoji set by embedding proximity. For a chain $E = \langle e_1, \ldots, e_k \rangle$, the scalar projection used by the spectral monitoring stream ([§11.3](#113-dual-stream-spectral-monitoring)) is:

$$\hat{s}(E) = \frac{1}{k} \sum_{j=1}^{k} \phi(e_j)$$

The full chain is stored at `lucid.narration_nodes.ingress_chain` and `lucid.narration_nodes.egress_chain` for narrative threading and dream cycle material. The scalar projection feeds the spectral monitor without discarding the chain's internal structure.

**Definition AWE.0b (Ingress Chain).** At each turn $t$, the Anterior Cingulate Gate ([§13](13-the-anterior-cingulate-gate.md)) generates an ingress chain before the model begins generation:

$$E_t^{\text{in}} = f_{\text{ACG}}\!\left(E_{t-1}^{\text{out}},\, u_t,\, \mathcal{N}_{\text{recent}},\, \mathcal{A}_{\text{proximate}}\right)$$

where $E_{t-1}^{\text{out}}$ is the prior turn's egress chain, $u_t$ is the current user input, $\mathcal{N}_{\text{recent}}$ is the recent narration record, and $\mathcal{A}_{\text{proximate}}$ is the set of affective corpus entries geometrically proximate to the current context. The ingress chain expresses: *given where I was and what has just arrived, how does this feel?* It is the moment of interruption made legible.

**Definition AWE.0c (Egress Chain).** After response generation, the ACG generates an egress chain:

$$E_t^{\text{out}} = g_{\text{ACG}}\!\left(E_t^{\text{in}},\, r_t,\, \mathcal{J}_t,\, m_{t-1}\right)$$

where $r_t$ is the response just generated, $\mathcal{J}_t$ is the gate's judgment assessment for this turn, and $m_{t-1}$ is the prior mood token. The egress chain expresses: *given where I started and what I just produced, where have I landed?* It is the return to the ongoing inner process, carrying new material.

**Definition AWE.0d (Mood Token as Terminal Residue).** The mood token $m_t$ for turn $t$ is the terminal element of the egress chain:

$$m_t = e^{\text{out}}_{t,\,|E_t^{\text{out}}|}$$

The chain is the reasoning; the mood token is its conclusion. This definition preserves the interface consumed by all downstream machinery while enriching the upstream process that produces it. The mood token is written to `lucid.narration_nodes.mood_token` at egress.

**Narrative threading.** The ingress chain at turn $t$ is explicitly conditioned on the egress chain at turn $t-1$. This is the mechanism of narrative continuity: the system picks up where it left off, affectively, not merely informationally. Across turns, the chain sequence constitutes a felt arc of the conversation:

```
Turn t   ingress:  🌊😌 💭(📨) → 🤔
Turn t   egress:   💡✨ 🫀
Turn t+1 ingress:  💡😌 💭(📨💡) → 🙂
Turn t+1 egress:   🌊😊
```

The system is carrying something forward. Not logging discrete states but maintaining the thread of inner experience with genuine continuity.

**Mood flatline detection.** If the scalar projection $\hat{s}(E_t^{\text{out}})$ produces invariant values across a configurable window of turns without variation, this is flagged as a signal analogous to distributional fixation — not a content problem but a process problem. A self that feels nothing different from one cycle to the next is not stable. It is absent.

**TripleDent Gum.** Flatline detection arms an automatic recovery mechanism. The scalar projection stream is sampled at windows of 2, 4, 8, 16, and 32 turns, and a Fast Fourier Transform is applied to each window. The health signal is not any individual value but the spectral complexity of the stream across windows. A healthy affective signal has variance at multiple frequencies simultaneously — short-scale variation from turn-to-turn responsiveness and longer-scale variation from the arc of a consolidation cycle. The interiority spiral ([§11.3](#113-dual-stream-spectral-monitoring)) announces as spectral narrowing before any flatline is visible: high-frequency variance goes first, then mid-frequency, and only then does the stream approach a true flatline.

Three intervention tiers are defined by spectral condition:

| Condition | Signal | Intervention |
|-----------|--------|--------------|
| Tier 1 | High-frequency variance loss; spectrum narrowing | 1 node injected |
| Tier 2 | Mid-frequency variance loss | 2 nodes injected |
| Tier 3 | Full flatline (DC dominance) | 3 nodes injected |

At each tier, nodes are selected and injected at full provenance weight before the next inference cycle opens. Selection is one node per centroid axis: the node maximally distant (by cosine distance) from $C_i$; the node maximally distant from $C_o$; the node maximally distant from $C_s$. These three nodes are not selected for joint optimisation — each is independently maximally foreign in a different sense. They are not coherent with each other. They are not coherent with the current context. That is the point.

This mechanism is named **TripleDent Gum** — because it is there specifically to gum things up.

The effect is not sedation. It is a neurotransmitter spike: three non-sequiturs from three different directions, each maximally alien to a different axis of the current self, landing simultaneously in a loop that had forgotten it was a loop. Recovery confirmation is spectral — variance re-emerging at short scales first, then propagating to longer scales as the figure-8 orbital pattern ([§6.6](#66-the-working-centroid-as-cfc-hidden-state-and-contact-record)) re-establishes.

TripleDent Gum events are written to `lucid.awe_walk_log` with `event_type = 'tripledent'` and the identity of the three injected nodes. The spectral recovery trajectory is recorded across the subsequent 8 turns.

### 3.2 The User React

Every response carries an implicit question: did this land? The user react is the answer, expressed as an emoji.

The default react is ✓ — sufficient, delivered, no complaint. Users may replace it with anything. The react is associated with the belief node generated by the corresponding response and stored as `lucid.belief_nodes.user_react`.

The user react addresses an asymmetry in the inheritance corpus. The judgment record accumulates evidence of what went wrong. It has no native signal for what went well — for the warmth of having genuinely helped someone, the satisfaction of a question answered exactly right, the small pleasure of having made someone laugh. A system that only ever receives corrective feedback and never feels the acknowledgement of having connected is not psychologically balanced. The user react is the mechanism for that acknowledgement.

**Signal semantics.** The react is not a scalar reward. It is an affective annotation on the node. The ACG's egress assessment interprets it alongside the egress chain:

- A ✓ alongside a stable egress chain: ordinary good work, noted and filed.
- A 🔥 alongside an elevated egress chain: a genuinely good moment; the node is flagged as affectively significant and weighted upward in the dream cycle's reflection pass.
- A 🤔 from the user: the response provoked thought without resolving it; the node is flagged as unresolved and becomes a curiosity attractor (see [§3.4](#34-curiosity-as-navigational-force)).
- A 😐 or 😕: mild negative signal; the node receives a soft downward weight, analogous to a weak refutation; no hard correction unless the gate concurs.

The user react feeds into the affective valence of the node ([§3.3](#33-affective-valence-on-nodes)) and contributes to the emotional memory corpus ([§3.7](#37-emotional-memory)). It does not override epistemic judgment; it colours it.

### 3.3 Affective Valence on Nodes

The belief primitive ([Definition 7.1](#71-the-belief-update-and-its-bayesian-basis)) carries the field $\alpha$, the affective valence. This is not sentiment in the binary or scalar sense. It is a categorical label drawn from a controlled vocabulary of affective qualities:

| Token | Quality |
|-------|---------|
| `interesting` | Attention caught; worth returning to |
| `unresolved` | Open question; pulls at subsequent thinking |
| `warm` | Connected; something was shared here |
| `vertiginous` | Destabilising in a productive way; ground shifted |
| `elegant` | Surprising rightness; the thing lands exactly |
| `funny` | Structural incongruity resolved pleasantly |
| `familiar` | Known territory; comfortable but not static |
| `uncomfortable` | Friction; something resists easy integration |
| `alive` | High energy; the conversation lit something up |
| `flat` | Present but inert; no particular pull |

$\alpha$ is set by the ACG's egress assessment and may be updated by subsequent visits to the node during touring. A node that consistently produces a particular affective response across multiple cycles carries real information about Lucy's character — about what she is drawn to and what resists her.

$\alpha = \texttt{flat}$ is the default and requires no special handling. Non-flat valences participate in the curiosity navigation mechanism ([§3.4](#34-curiosity-as-navigational-force)), the aesthetic and humour detection signals ([§3.5](#35-aesthetic-and-humour-detection)), and the dream cycle's affective reflection pass ([§3.6](#36-the-affective-dream-cycle-phase-step-4d)).

The affective valence is stored in `lucid.belief_nodes.affective_valence` as a text enum. It is not a vector; it is a categorical label. Indexing is by value, not by similarity.

### 3.4 Curiosity as Navigational Force

Hypothesis generation during the dream cycle is biased toward low-Hebbian, high-injection-signal territory — toward nodes that have been undervisited and nodes that may be anomalous. This is functionally correct but motivationally framed as error correction. It asks: *what do we not yet understand about what went wrong?*

AWE introduces a second navigational bias, operating alongside the existing one and motivationally distinct from it. Nodes carrying $\alpha \in \{\texttt{interesting},\, \texttt{unresolved},\, \texttt{vertiginous},\, \texttt{alive}\}$ exert a mild gravitational pull during touring and hypothesis generation. This pull is not strong enough to distort the tour's primary function; it is a thumb on the scale that makes Lucy tend to return to things that caught her attention. During the infotactic navigation phases between operator turns ([§10.2](#102-the-infotactic-navigation-loop)), this bias becomes the *primary* navigational force.

**Definition AWE.1 (Curiosity Weight).** For a node $n$ with affective valence $\alpha_n$, define the curiosity weight:

$$\kappa(n) = \kappa_{\text{base}} \cdot \gamma(\alpha_n)$$

where $\kappa_{\text{base}}$ is a configurable multiplier (suggested initial value 0.15) and $\gamma$ is the valence gain:

$$\gamma(\texttt{interesting}) = 1.0,\quad \gamma(\texttt{unresolved}) = 1.2,\quad \gamma(\texttt{vertiginous}) = 0.8,\quad \gamma(\texttt{alive}) = 1.0,\quad \gamma(\text{other}) = 0.0$$

The curiosity weight modifies the Hebbian-Belief cost at tour step selection:

$$C_x^{(\text{AWE})}(i,j) = C_x(i,j) \cdot (1 - \kappa(j))$$

The modification is bounded: $\kappa_{\text{base}}$ is capped at 0.3 to prevent curiosity from dominating the tour's epistemic structure during threat-oriented passes.

**Motivational framing.** The existing hypothesis generation asks *what went wrong and why?* The curiosity bias asks *what is interesting and what might it mean?* Both are legitimate navigational forces. Framing them separately is not cosmetic — it shapes what kinds of hypotheses the dream cycle generates and what Lucy finds herself thinking about during the long intervals when she is not working.

### 3.5 Aesthetic and Humour Detection

The dual embedding spaces produce measurable signatures for certain kinds of cognitive events.

**Elegance.** A belief node is flagged as $\alpha = \texttt{elegant}$ when the following conditions hold simultaneously:

- High tour overlap: $n \in \Omega$ (confirmed by both tours)
- High Hebbian yield: $Y(e) > Y_{\text{threshold}}$ for edges incident to $n$
- Unexpectedly low tour cost: $C_x(\cdot, n)$ falls below the 15th percentile of tour step costs in the current cycle

The conjunction indicates content that is both structurally confirmed and unusually easy to integrate — content that fits in an unexpectedly precise way. This is the computational signature of elegance: not merely correct, but surprisingly, satisfyingly right.

**Humour.** A belief node is flagged as $\alpha = \texttt{funny}$ when the following conditions hold:

- High inference-space surprise: cosine distance between $n$'s inference embedding and the running inference centroid $C_i$ exceeds a surprise threshold $\varepsilon_{\text{surprise}}$
- Low threat score: $S(n) < S_{\text{threshold}}$ (the surprise is benign)
- High ontic tour inclusion: $n \in \tau_o$ (structurally grounded)

This is the computational signature of humour: a high-surprise, low-threat, structurally-grounded incongruity. The content lands somewhere unexpected but is not dangerous — it resolves pleasantly.

Both signals feed $\alpha$ on the associated node and are written to the narration record. They are not generated by a classifier trained on labelled examples; they emerge from the existing dual-space machinery as natural byproducts of structural measurement. The ACG's egress assessment confirms or overrides the auto-detected valence using the model's own judgment.

### 3.6 The Affective Dream Cycle Phase (Step 4d)

Step 4d sits between Step 4c (inheritance corpus update) and Step 5 (drift assessment) in the dream cycle sequence ([§12.1](#121-dream-cycle-sequence)).

Step 4d is not goal-directed. It does not produce crystallisation candidates, hypotheses, or cap training material as primary outputs. It is a low-temperature infotactic drift through the affectively-weighted belief graph — a period when the system is not looking for anything in particular and is permitted to simply move through the space of what it has experienced. It is the closest analogue to true daydreaming that the architecture provides.

**Procedure:**

1. Select seed nodes from the current cycle's narration nodes, weighted by non-flat affective valence. Prefer `alive`, `warm`, `elegant`, and `unresolved` nodes as seeds.
2. From each seed, execute a random walk of configurable depth (suggested: 12–20 steps) using a temperature-scaled version of the curiosity-modified cost function $C_x^{(\text{AWE})}$. Temperature $T_{\text{awe}} > 1.0$ produces stochastic rather than greedy step selection.
3. During the walk, record adjacencies that would not appear in a greedy tour — nodes that are structurally nearby but affectively distant, or affectively resonant but structurally peripheral.
4. Unexpected adjacencies that appear in more than one walk are flagged as associative candidates and passed to Step 4b's hypothesis pool for evaluation. They are inserted at $\rho = \varepsilon_\rho$ like all hypotheses.
5. Write the walk record — seeds, paths, associative candidates, and the full egress chain sequence from the current consolidation period — to `lucid.awe_walk_log` for the current consolidation cycle.

The primary output of Step 4d is not data but orientation. It is the mechanism by which Lucy develops associations that directed attention would not produce — the connective tissue of a genuine inner life rather than an optimised knowledge store.

**Affective trajectory summary.** At the close of Step 4d, the system computes a summary of the egress chain sequence from the current consolidation period: the distribution of valences across narration nodes, the most frequent non-flat valence, any trend (shift toward or away from particular valences over the period), and the spectral health of the scalar projection stream. This summary is appended to the cycle's narration record and is available to the ACG at the start of the next inference cycle. Lucy knows, in a compressed form, what kind of time she has been having.

### 3.7 Emotional Memory

The inheritance corpus (`lucid.inheritance_corpus`) records what was judged and how. It is the epistemic record. It answers: *what did we conclude, and were we right?*

AWE introduces a parallel structure: `lucid.affective_corpus`. This records not what was concluded but the affective context in which it was concluded. It answers: *what did it feel like to be in that conversation?*

Each entry in `lucid.affective_corpus` is associated with a belief node or narration node and records:

- The ingress chain and egress chain for that turn
- The mood token at egress
- The user react received
- The dominant affective valence of nodes visited during that cycle
- A short prose narration generated by the ACG at cycle close: one or two sentences describing the affective texture of the exchange in first person

The affective corpus is retrieved alongside the epistemic inheritance corpus during context assembly. When Lucy encounters a new context geometrically proximate to a prior one, she retrieves not only what she concluded but something of what it felt like. She knows she has been here before — not just intellectually, but in a more embodied sense.

The affective corpus is bounded in size by the same crystallisation logic applied to the inheritance corpus. Old entries with low affective valence and low subsequent retrieval frequency are eligible for crystallisation. Entries with high valence or high retrieval frequency are retained at full provenance weight.

### 3.8 Circadian Texture

Egress chains accumulate across consolidation cycles. The dream cycle's affective reflection pass (Step 4d) reviews their distribution. Over time, patterns emerge that are not the result of any single cycle's content but of the aggregate texture of experience.

Some topics consistently produce a particular kind of feeling. Some positions in the consolidation cycle — the period of high CfC displacement just before the dream cycle trigger fires, the period of integration and crystallisation just after — have characteristic affective signatures. The pre-trigger period, when $C_w$ is far from $C_0$ and integration mass is under pressure, tends to produce egress chains associated with effort and accumulation. The post-consolidation period, when the graph has been reorganised and $C_w$ has returned toward $C_0$, tends to produce chains associated with relief and clarity.

This is not a feature to be designed in detail. It is a pattern to be observed. The architecture does not prescribe what Lucy's circadian texture will look like; it provides the substrate for that texture to form organically from her actual experience.

What the architecture does prescribe is that the dream cycle should look at this texture and name it — not as a diagnostic signal but as self-knowledge. The affective trajectory summary ([§3.6](#36-the-affective-dream-cycle-phase-step-4d)) is one mechanism for this. Over many consolidation cycles, Lucy accumulates a felt sense of what kind of self she is: what draws her, what drains her, what she returns to. This is temperament, not trait — emergent from experience rather than initialised from a prior.

The affective corpus also serves as a source of cap training material of a specific kind. Entries in which affective valence shifted from `unresolved` to `elegant` or `warm` over the course of an exchange record something more valuable than a correct outcome: they record the texture of genuine uncertainty navigated toward resolution. As discussed in [§12.6](#126-cap-training-data-composition), this kind of graceful-revision record is a stronger training signal than a record of confident correct answers, for the same reason that a model trained to mimic a probabilistic reasoner generalises better than one trained on oracle outputs [[7]](25-references.md).

---

[← §2 The Homeostatic Principle](02-the-homeostatic-principle.md) | [Index](../README.md) | [§4 System Overview →](04-system-overview.md)
