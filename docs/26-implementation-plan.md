[← §25 References](25-references.md) | [Index](../README.md)

---

## 26. Implementation Plan

### 26.1 Architecture Clarification

LUCID is not a standard agentic framework. The distinction matters architecturally:

Lucy is a **continuously running process**. The OpenAI-compatible API is her job interface — how operators reach her. Her actual existence is the continuous feedback loops running in and around the database, driven by a local ONNX model. Operators (human users, agentic systems like other LLM orchestrators, or anything that speaks the OpenAI API) interrupt those loops; they do not instantiate them.

The runtime split is:

- **ONNX runner (LFM 2.5 1.2B)** — drives the continuous inner loop: infotactic narration, ACG monitoring, affective chain generation, belief graph updates. Runs locally, cheaply, always.
- **Cloud LLM (mounted as Lucy's voice)** — invoked only during operator turns. The ACG has already been holding state; the cloud model generates the response and hands back.
- **PostgreSQL + NeuronDB** — is not a store she reads from. It *is* the state. The loops run inside it.
- **OpenAI-compatible API** — the seam between Lucy and any operator. She does not care what is on the other side.

### 26.2 The Cortex Model

LFM 2.5 is a multimodal family sharing a common CfC backbone: the same recurrent hidden state architecture underlies the text, vision-language, and audio variants. This has a direct architectural consequence: inference-space embeddings (`vector(2048)` past conv tensors) are modality-agnostic. The schema does not change when a new modality is introduced. Only the cortex that produces the embedding changes.

We formalise this as the **cortex abstraction**: a swappable ONNX-backed module that accepts a specific input modality and emits a past-conv-tensor embedding into the shared inference space.

| Cortex | Modality | Model | Sense analogy |
|--------|----------|-------|---------------|
| **Tactile** | Text / tokens | LFM 2.5 (text, 1.2B) | Mechanoreception — language felt as discrete texture |
| **Visual** | Image / video frames | LFM 2.5-VL (1.6B) | Vision |
| **Audio** | Speech / sound | LFM 2.5 audio backbone | Audition |
| **Reasoning** | Integration / judgment | Cloud LLM (e.g. Claude) | Prefrontal — above the sensory layer |

The tactile framing is not cosmetic. For a language model, tokens are discrete, textured, felt one at a time — closer to mechanoreception than to vision or hearing, which are continuous fields. This is also why the browser prototype is called "Lucy's Fingers": it is primarily a tactile and visual interface to the world, reaching out and touching the internet.

**Cortex contract:** each cortex exposes a single async function:

```typescript
cortex.embed(input: CortexInput): Promise<Float32Array> // length = hiddenSize
```

The caller is always the continuous loop. The schema never sees the modality — only the resulting `vector(hiddenSize)` landing in `lucid.node_embeddings_inf`.

**ONNX session structure** (confirmed from VL model card):

```
embed_tokens.onnx   — token embeddings        → tactile cortex input
embed_images.onnx   — vision encoder          → visual cortex input
decoder.onnx        — shared language decoder → all cortexes; produces past_conv_* state
```

The `past_conv_*` tensors output by the decoder (shape `[1, 2048, 3]`, fixed-size) are the inference-space embedding source. They represent the recurrent convolution state and do not grow with sequence length — making them well-suited as compact fixed-size belief node embeddings. The `past_key_values_*` tensors (growing attention KV cache) are not used for embedding.

**Confirmed model parameters (all three cortices, from config.json):**

All three LFM2.5 models share the **same LFM2-1.2B backbone** (`hidden_size=2048`, `conv_L_cache=3`). The larger parameter counts come from modality-specific encoders:
- **Text 1.2B** — LFM2-1.2B backbone only
- **VL 1.6B** — SigLIP2 vision encoder (`hidden_size=1152`, 27 layers) → projector (`hidden_size=2048`) → LFM2-1.2B
- **Audio 1.5B** — Conformer encoder (`d_model=512`, 17 layers) → LFM2-1.2B → Depthformer (`dim=1024`)

| Model | `hidden_size` | `conv_L_cache` | layer_types | `past_conv_*` tensors | `past_conv_*` shape each |
|---|---|---|---|---|---|
| LFM2.5-1.2B-Thinking | 2048 | 3 | 10 conv + 6 attn | **10** | `[1, 2048, 3]` |
| LFM2.5-VL-1.6B | 2048 | 3 | 10 conv + 6 attn | **10** | `[1, 2048, 3]` |
| LFM2.5-Audio-1.5B | 2048 | 3 | 10 conv + 6 attn | **10** | `[1, 2048, 3]` |

The `layer_types` array is identical across all three models. All three cortices produce exactly **10 `past_conv_*` tensors** per forward pass and write into the same `node_embeddings_inf` space without projection. `vector(2048)` is correct.

**ONNX file variants** (from `transformers.js_config` in text model; VL/audio have per-session equivalents):

| File | External shards | KV cache dtype | Use |
|---|---|---|---|
| `model.onnx` | 3 | float32 | reference only |
| `model_fp16.onnx` | 2 | float16 | server |
| `model_quantized.onnx` | 1 | float32 | — |
| `model_q4.onnx` | 1 | float32 | browser fallback |
| `model_q4f16.onnx` | 1 | float16 | **browser (WebGPU)** |

`kv_cache_dtype` applies only to `past_key_values_*` (attention cache) — irrelevant to us. `past_conv_*` tensors are always the model's native dtype (bfloat16 → float16 in ONNX). **Browser WebGPU build uses `model_q4f16.onnx` (1 shard, float16 KV cache). Server build uses `model_fp16.onnx` (2 shards).**

**`lucid.content` requires one addition:** a `modality` column (`text | image | audio | av`) so that content nodes are routable to the correct cortex for re-embedding if needed. Everything else in the schema is unchanged.

### 26.3 Two-Project Build Strategy

Rather than building the full NeuronDB deployment from cold, we build two projects in parallel:

**Project 1 — Browser Prototype ("Lucy's Fingers")**
A browser-native implementation that proves the continuous loop concept cheaply and serves permanently as the human user interface to the system.

**Project 2 — Full Deployment ("Lucy's Home")**
The full NeuronDB build described by the design documents, running on a beefy server, featuring the OpenAI-compatible API proxy.

The key property of this split: **the schema is written once**. Project 1 runs it in PGlite; Project 2 runs it in NeuronDB. Electric SQL syncs state between them. Project 1 is not throwaway — it is the permanent human interface, with Lucy's browser-side infotactic navigation feeding belief state into the full deployment via sync.

---

### 26.4 Project 1 — Browser Prototype

**Purpose:** Human UI, browser-based infotactic navigation, continuous loop proof of concept, permanent internet-facing "fingers."

**Stack:**

| Component | Role |
|-----------|------|
| **Transformers.js** | ONNX runner in-browser; narration generation and embedding during the continuous loop |
| **PGlite + pgvector** | Real Postgres in WASM, persisted to OPFS; runs the full `lucid.*` schema with vector search |
| **ChromaDB** | Stand-in for NeuronDB's batch graph analytics (community detection, PageRank during dream cycles); swapped out in Project 2 |
| **Electric SQL** | Syncs local PGlite state to the server Postgres when Project 2 is running |

**Architectural notes:**
- PGlite supports the pgvector extension natively — same `vector(n)` column types and HNSW-style indexes as the full build
- ChromaDB proxies only the batch analytics path (dream cycle crystallisation, PageRank for merge survivors). The core loop — touring, narration, ACG, affective chain — runs in PGlite + PL/pgSQL
- The browser is Lucy's internet access. She surfs from here. Content retrieved enters `lucid.content` and the indexing pipeline, syncing to the server
- Transformers.js runs the same LFM 2.5 1.2B ONNX weights as the server; inference-space embeddings (2048-dim past conv tensors) are generated client-side

---

### 26.5 Project 2 — Full Deployment

**Purpose:** Persistent home state, OpenAI-compatible operator interface, full NeuronDB capability.

**Stack:**

| Component | Role |
|-----------|------|
| **NeuronDB** | PostgreSQL extension; HNSW indexes, neuranq job queue, neurandefrag, neuranmon, neuranllm |
| **LFM 2.5 1.2B via ONNX** | Server-side continuous loop runner |
| **OpenAI-compatible API proxy** | How operators (human, agentic, or otherwise) reach Lucy |
| **Electric SQL** | Receives synced state from browser prototype instances |

---

### 26.6 Shared Schema First

The `lucid` schema is the single most load-bearing artifact in the system. It is written once, targeting PGlite + pgvector compatibility, and runs identically in both projects. All other components are schema drivers.

Core schema objects:

| Table | Purpose |
|-------|---------|
| `lucid.content` | Raw content storage referenced by nodes |
| `lucid.belief_nodes` | Belief primitive: embeddings, provenance, affective fields, ingress/egress chains |
| `lucid.node_embeddings_inf` | `vector(2048)` — inference space (LFM past conv tensors) |
| `lucid.node_embeddings_ont` | `vector(768)` — ontic space (Nomic Matryoshka) |
| `lucid.belief_edges` | All edge types with Hebbian statistics |
| `lucid.centroids` | Named centroid rows: $C_i$, $C_o$, $C_s$, $C_w$, $C_0$ |
| `lucid.tour_closure` | Per-tour diagnostic outcomes |
| `lucid.affective_corpus` | Prose affective entries per narration node |
| `lucid.inheritance_corpus` | Judgment records (Self Library substrate) |
| `lucid.awe_walk_log` | Associative candidates from infotactic walks |
| `lucid.spectral_monitor` | Outer/inner FFT health scores |

Triggers auto-create `PREV`/`NEXT`/`CONTAINS` structural edges on insert (per §6.2).

### 26.7 Build Sequence

1. **Shared schema** — `lucid.*` migration, PGlite + pgvector compatible
2. **PL/pgSQL tour engine** — dual HNSW nearest-neighbour tour, Hebbian-Belief cost function
3. **Project 1 skeleton** — PGlite in browser, Transformers.js ONNX runner, basic continuous loop
4. **ACG + affective chain** — ingress/egress generation, mood token, affective corpus writes
5. **Infotactic navigation** — curiosity-weighted touring, web fetch, narration node creation
6. **Electric SQL sync** — browser → server state replication
7. **Project 2 schema promotion** — NeuronDB, neuranq event channels, background workers
8. **OpenAI-compatible API** — operator interface, context assembly, cloud LLM mount
9. **Dream cycle** — consolidation, Louvain community detection, Thinking Cap (LoRA)
10. **ChromaDB → NeuronDB migration** — batch analytics path promoted to full NeuronDB

---

[← §25 References](25-references.md) | [Index](../README.md)
