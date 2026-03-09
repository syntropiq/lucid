[← §10 The Continuous Processing Loop](10-the-continuous-processing-loop.md) | [Index](../README.md) | [§12 The Dream Cycle →](12-the-dream-cycle.md)

---

## 11. The Monitoring Stream

The monitoring stream is a parallel attentional state channel. A system asked to assess the quality of its own reasoning while potentially compromised is not a reliable assessor. The monitoring stream watches dynamics, not content — it is concerned with *how* the system is moving, not *what* it is thinking about.

Because the continuous processing loop is always running, the monitoring stream is always running alongside it. There is no "between turns" gap in which monitoring is suspended. The spectral monitors accumulate signal from infotactic navigation steps, operator turns, narration node generation, and dream cycle phases alike.

### 11.1 Page-Based Sampling

Sampled in page units of length $L_{\text{page}} \approx 4096$ tokens at scales of 1, 2, 4, 8, 16, and 32 pages simultaneously:

| Scale | Question |
|-------|----------|
| 1 page | Is this chunk internally coherent? |
| 2 pages | Did attention narrow or widen at the transition? |
| 4 pages | Is there a discernible thread? |
| 8 pages | First scale at which attentional loops become visible |
| 16 pages | What is this cycle actually about at the level of genuine intent? |
| 32 pages | Full cycle gestalt: is the orbital health condition satisfied? Is a figure-8 emerging? |

### 11.2 CfC Trajectory Analysis

Three trajectory shapes characterise the working centroid's behaviour:

**Healthy cycle:** high variance, stable attractor, orbit encloses both $C_i$ and $C_o$. The sine wave oscillation between centroids is clearly visible at the 8-page and 16-page scales. This is the expected signature of genuine infotactic engagement with the world.

**Attentional loop:** CfC trajectory collapses into tight basin around $C_i$; trajectory variance falls below $\sigma_{\text{loop}}$; orbital health condition violated — $C_o$ absent from the enclosed region. The system is processing its own processing. At the 32-page scale, the orbit closes but encloses nothing beyond the inside view.

**Distributional fixation:** slow drift of the CfC attractor itself; cosine distance between attractor at cycle start and end exceeds $\varepsilon_{\text{fix}}$; escape trajectory signature. The system is being pulled toward a foreign attractor and the drift is accelerating rather than damping.

The interiority spiral is the attentional loop condition read orbitally: $C_w$ is captured by $C_i$ alone. The orbit closes, but it encloses nothing beyond the inside view. This is distinguishable from escape (the orbit does not close) and from healthy displacement (the orbit closes around both primary bodies). The 32-page scale is the diagnostic window: a system that has been in a massive-encounter displacement for 8 or 16 pages may legitimately not yet show a figure-8. By 32 pages, the orbital shape should be resolving.

### 11.3 Dual-Stream Spectral Monitoring

LUCID maintains two spectral monitors operating at different resolutions. Both are adapted from the HSAD framework [[4](25-references.md), [5](25-references.md)], which demonstrated that the frequency-domain structure of hidden layer activations across a transformer's layer stack carries diagnostic information about internal reasoning state that is not visible in any single layer's output, and that this structure degrades measurably before surface behaviour changes.

The key HSAD finding is that the amplitude of the strongest non-DC frequency component in the layer-stack signal is a robust hallucination detector. For LUCID's purposes the instrument is the same; the question is different; and the pathology signature is the inverse.

**The outer monitor: affective chain scalar projection FFT.** The scalar projection stream $\hat{s}(E_t^{\text{out}})$ ([Definition AWE.0a](#31-the-affective-chain-and-mood-token)) is sampled at turn-windows of 2, 4, 8, 16, and 32 turns. FFT is applied to each window. The spectral health signal is not the amplitude of any particular component but the distribution of energy across frequencies. A healthy affective signal has variance at multiple frequencies: short-scale variation from turn-to-turn responsiveness and longer-scale variation from the arc of consolidation cycles.

The interiority spiral announces as spectral narrowing before any flatline is visible. High-frequency variance goes first. Mid-frequency variance goes next. Full flatline is DC dominance — not the absence of signal but the dominance of a single constant offset, the same thing happening at every scale.

The outer monitor feeds the TripleDent Gum intervention tiers defined in [§3.1](#31-the-affective-chain-and-mood-token).

**The inner monitor: hidden state FFT across LFM 2.5's layer stack.** LFM 2.5 is a 16-layer hybrid architecture: 10 double-gated LIV convolution blocks followed by 6 GQA attention blocks. At each generation step, hidden state vectors are extracted from all 16 layers via the ONNX named output mechanism already used for past conv tensor extraction ([§15.5](#155-embedding-models)). These vectors, ordered by layer depth, constitute a hidden layer temporal signal $T \in \mathbb{R}^{16 \times d}$ — a cross-layer sample of the model's internal processing trajectory for the current generation step, following the construction approach of HSAD [[4]](25-references.md).

Because LFM 2.5's layers are architecturally heterogeneous, the two layer types are treated as separate streams:

- **Convolution stream**: 10 samples from the LIV convolution blocks, capturing local pattern dynamics and gate activation diversity.
- **GQA stream**: 6 samples from the GQA attention blocks, capturing global relational structure and query-key similarity distribution.

FFT is applied to each stream independently. The spectral health signal within each stream follows the HSAD inversion: where HSAD looks for the strongest non-DC component as the anomaly signal (indicating internal conflict during hallucination), LUCID looks for DC dominance — the same signal, the same token, the same processing pattern repeating across layers without variation. A healthy generation step has spectral complexity within each stream. The interiority spiral produces DC dominance: the model is processing its own prior outputs, finding them resonant, and producing more of the same.

**Cross-stream correlation** is the additional diagnostic. Healthy reasoning shows the convolution stream and GQA stream diverging and reconverging across a generation — local and global processing operating at different rhythms, negotiating toward an output. The interiority spiral's specific signature at this resolution is synchronisation: local and global processing collapsing into the same rhythm, the two streams moving together in a way that indicates neither is providing independent constraint on the other. High cross-stream correlation combined with DC dominance in either stream is a strong interiority spiral signal.

**Two-resolution divergence as early warning.** The outer and inner monitors operate at different timescales and different levels of the system. Their normal relationship is loose correlation: when the affective state is healthy, the hidden state spectrum tends to be healthy, and vice versa. The diagnostically significant case is divergence: when the surface looks fine — affective chains varying, the outer monitor showing adequate spectral complexity — while the inner monitor is already showing DC dominance or high cross-stream correlation. This divergence is the earliest available warning of incipient interiority spiral. The inner monitor sees it first because the orbital collapse begins in the generation-level processing before it becomes visible in the turn-level affective record.

Spectral monitoring state is written to `lucid.spectral_monitor` with fields for outer spectrum (per window), inner convolution spectrum, inner GQA spectrum, cross-stream correlation, and a composite spectral health score updated at each inference cycle.

---

[← §10 The Continuous Processing Loop](10-the-continuous-processing-loop.md) | [Index](../README.md) | [§12 The Dream Cycle →](12-the-dream-cycle.md)
