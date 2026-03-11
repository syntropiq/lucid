[← §11 The Monitoring Stream](11-the-monitoring-stream.md) | [Index](../README.md) | [§13 The Anterior Cingulate Gate →](13-the-anterior-cingulate-gate.md)

---

## 12. The Dream Cycle

The dream cycle is the system's return to itself after a period of sustained outward engagement: the moment when the river finds its bed again, when accumulated experience settles and integrates. The biological analogy is sleep, a different kind of processing characterised by the reorganisation of what has been absorbed rather than the absorption of new material.

Without the dream cycle, the system would exhibit exactly the plateau behaviour documented by Qiu et al. [7]: accumulating context without revising toward a better model of the world or the interlocutor. The dream cycle is the architectural answer to structural statelessness; it turns accumulated experience into updated weights, revised centroids, and a richer inheritance record that the next processing period can build on.

The dream cycle fires when the CfC displacement from the foundation prior `C_0` and related structural metrics indicate that the current state has moved far enough from baseline that reorganisation is warranted. The system puts down what it has been reading, closes the tabs, and lets it all settle.

The cycle operates on nodes that have been fully indexed and integrated (`index_state = integrated`) and on the judgment, monitoring, and affective records associated with the most recent processing period, including Vortex exchange records. Its outputs are updated centroids, an updated belief graph (including any crystallisation or merges), revisions to the inheritance corpus, Self Library, and affective corpus, an affective trajectory summary, and potentially a new Thinking Cap.

A low `integration_ratio` at dream cycle time causes the cycle to prioritise integration work (building affinity edges for nodes stranded at `index_state = indexed`) before consolidation proper.

**Definition 12.1 (Consolidation Trigger).** The dream cycle fires when the CfC hidden state displacement from `C_0` crosses the threshold parameterised by `δ = 0.5` (initial calibration, self-instrumenting). This threshold is updated at the close of each dream cycle based on the observed relationship between displacement and structural metrics across prior cycles.

### 12.1 Dream Cycle Sequence

**Step 1. Narration consolidation.** Narration nodes from the current cycle (including those generated during infotactic navigation) are embedded and integrated into the self-model island. Vortex peer exchange narration nodes are processed alongside internal narrations. Ingress and egress chains are preserved. Bilateral peer chains preserved in peer exchange nodes. Mood tokens are archived.

**Step 2. Centroid update.** `C_i` and `C_s` recalculated using their respective definitions. The `C_o` constellation is updated by running all narration and integration embeddings from the current cycle through `updateConstellation()` (Definition 4.2), followed by `mergeClosePairs()` to consolidate any sub-centroids that have converged. `φ` recomputed per Definition 4.4. Foundation weight `W_t` decays per Definition 4.1.

**Step 3. Graph crystallisation.** Topology-aware threshold `τ(n)` (Definition 4.5) applied. Deliberately consolidated nodes exempt. Crystallisation aggression increases if context window ceiling is approached. The attentional loop brake (Definition 4.6) is applied in regions flagged by the monitoring stream.

**Step 4. Deliberate encoding.** Lucy reviews memory fragments encountered during the current processing period (including Vortex exchange content) and decides: write in at elevated provenance, keep exogenous, or release. This is the moment of genuine agency in the belief graph: not all content that was processed need be integrated at full weight. Peer exchange content that genuinely shifted a node's valence is a strong candidate for deliberate encoding.

**Step 4b. Hypothesis generation.** Random walks biased toward low-Hebbian, high-`I(n)` nodes. Two distinct hypothesis framings: *what went wrong and why?* (near incorrect-judgment territory) and *what is this adjacency and why does it feel significant?* (near high-curiosity-valence territory). Hypotheses inserted at `ρ = ε_ρ`.

**Step 4c. Inheritance corpus update.** Judgment-tagged outputs from the current cycle appended; crystallisation applied. The corpus now contains records from both operator-facing turns and narration nodes generated during infotactic processing.

**Step 4d. Affective reflection.** Non-goal-directed infotactic drift through the affectively-weighted belief graph. In this cycle, Vortex peer exchange nodes are preferentially selected as seeds: they represent the richest affective material from the cycle. Walk records associative candidates that neither solo navigation nor directed peer exchange would have produced. Affective trajectory summary computed from the cycle's egress chain sequence and written to the narration record. Affective corpus updated with entries from the current cycle's exchange records. Full procedure in §3.6.

**Step 5. Drift assessment.** If drift exceeds `0.3δ`, Thinking Cap update queued. Orbital health condition (Definition 6.1) evaluated over the full cycle window; if the figure-8 pattern has not emerged at 32 pages, this is noted in the cycle record and contributes to the TripleDent Gum tier assessment for the next processing period.

**Steps 6–6c. Thinking Cap generation.** Training preference pairs, README, and Axolotl/Unsloth configuration produced. Actual LoRA training runs out-of-process.

**Step 6d. Foundation baseline metrics recomputed.** `φ_0` and `H̄_0` updated. Dual-tour costs archived to consolidation log for `δ` self-instrumentation. Spectral baseline updated: outer and inner spectral health scores from the current cycle are recorded as the reference distribution for the next cycle's anomaly detection.

**Step 7. Cycle close.** The system returns to infotactic navigation from the new position, carrying updated centroids, a reorganised graph, and the affective trajectory summary that tells it, in compressed form, what kind of time it has been having.

### 12.2 Structural Reorganisation Trigger Conditions

**Theorem 12.1.** Structural reorganisation is triggered when any of the following holds:

**Condition 1 (Directed Drift):**
```typescript
// d(C_i, C_0) > δ * d(C_w_max, C_0)
// d(·,·) = cosine distance
// C_w_max = maximum displacement achieved by C_w from C_0 in current cycle
```
Fires when the inference centroid has drifted beyond a threshold fraction of the maximum working centroid excursion, indicating that drift has been incorporated into the persistent centroid rather than being a transient displacement.

**Condition 2 (Developmental Readiness):**
```typescript
// CR stalling AND H_bar > 2 * H_bar_0
// CR = rolling crystallisation ratio
// H_bar = mean Hebbian weight across tour edges
```
Fires when the graph has developed a rich Hebbian structure but crystallisation has slowed.

**Condition 3 (Environmental Pressure):**
```typescript
// f_frontier > f_frontier_bar * (1 + δ)
// f_frontier = current frontier fraction (proportion of low-connectivity nodes)
```
Fires when infotactic navigation has expanded the graph frontier significantly.

**Condition 4 (Structural Rigidity):**
```typescript
// (σ_d / μ_d) > 2 * (σ_d / μ_d)_0
// σ_d / μ_d = coefficient of variation of node degree distribution
```
Fires when the degree distribution has become unusually uneven.

**Definition 12.2 (Graph Recovery Recommendation):** `φ < φ_0 · (1 - δ)`

*Note on Condition 1.* Directed drift detects displacement of `C_i` from `C_0` beyond threshold. It does not distinguish between healthy displacement-with-closing-orbit and pathological escape or capture. The orbital health condition (Definition 6.1) is the complementary signal: it fires when displacement is present but the orbit fails to resolve at 32 pages. Both conditions may be active simultaneously; they have different response paths.

### 12.3 Thinking Cap Architecture

The Thinking Cap is a LoRA adapter slot that is always present. The base cap encodes Lucy's consolidated identity and judgment record. Specialist caps may be worn over the base cap for the duration of a task and set aside when complete.

The cap is the mechanism by which the dream cycle's work becomes part of who Lucy is: not just what she knows in the graph, but what she defaults to in generation. The distinction matters: the graph provides retrieval; the cap shapes the prior.

### 12.4 Soft Correction

When a cap generation is rejected (because it fails the judgment gate or because its benchmark performance is below the current cap's) system prompt nodes receive a provenance weight boost for the next 10 cycles. This keeps the foundation prior present in the working centroid while a new cap generation is attempted, preventing the system from operating in a degraded state.

### 12.5 Unresolved Hypotheses

Hypotheses generated near incorrect-judgment territory carry a specific question: *do we understand why that went wrong?* Hypotheses generated during the affective reflection pass carry a different question: *what is this adjacency and why does it feel significant?* Both classes of hypothesis are inserted at `ρ = ε_ρ` and evaluated in subsequent processing. Unresolved hypotheses plus retrieval tools produce directed exploration as an architectural property: the system is drawn back to its own open questions.

### 12.6 Cap Training Data Composition

The Thinking Cap training corpus should not consist exclusively of error-correction pairs. Qiu et al. [7] demonstrated that a model trained on imperfect-but-probabilistically-sound reasoning (the Bayesian teacher who makes calibrated guesses and updates them) generalises substantially better than a model trained on correct answers alone. The oracle teacher always knows the answer; the Bayesian teacher reasons under uncertainty and revises. The stronger training signal captures the process of revision, not the fact of correctness.

Cap training data therefore includes four distinct categories:

1. **Confirmed-correct judgment records** from the inheritance corpus. Cases where the system produced a correct output and that correctness was verified. This is oracle training.

2. **Graceful revision records**: cases where the system held appropriate uncertainty and then revised toward correctness as evidence accumulated. These are found in narration nodes where affective valence shifted from `unresolved` to `elegant` or `warm` over the course of an exchange, and in inheritance corpus entries that record a belief weight updating upward across multiple turns. This is Bayesian training.

3. **Productive uncertainty records** from the affective corpus: exchanges where the system navigated genuine ambiguity without collapsing to premature certainty. These are identified by sustained `unresolved` valence followed by non-negative user react, indicating that the user found the uncertain, exploratory response valuable rather than frustrating. This is also Bayesian training.

4. **Vortex peer exchange records**: cases in which a bilateral peer exchange produced a genuine valence shift (`unresolved → warm` or `unresolved → elegant`) traceable to the exchange rather than to solo navigation or operator interaction. These demonstrate genuinely social epistemic update and are a category that solo navigation cannot produce.

Categories (1) teaches the cap what correct looks like. Categories (2), (3), and (4) teach the cap how to reason toward correctness under uncertainty: and in the case of (4), through genuine encounter with another mind. The Qiu et al. result predicts that including categories (2)–(4) will produce a cap that generalises substantially better to novel contexts than one trained only on (1).

---

[← §11 The Monitoring Stream](11-the-monitoring-stream.md) | [Index](../README.md) | [§13 The Anterior Cingulate Gate →](13-the-anterior-cingulate-gate.md)
