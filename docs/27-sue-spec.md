[← §26 Implementation Plan](26-implementation-plan.md) | [Index](../README.md)

---

## 27. SUE — Swappable Unit of Execution

### 27.1 What SUE Is

§26.2 defines the cortex abstraction as a swappable ONNX-backed module that accepts a modality and emits a `vector(2048)` embedding. SUE is the formalisation of that abstraction into a driver contract.

A SUE adapter is **config plus a small amount of code**. The config describes the model: paths, metadata, native dimensions, layer topology. The code does the adaptation work at the boundary: extracting the right tensors, projecting if necessary, sampling hidden states at normalised depths. For LFM 2.5, that code is entirely pass-through — the model was the reference implementation the schema was built around, so no adaptation is needed. For a future model, the adapter code is where the differences live.

LUCID core never talks to a model directly. It talks to the adapter. The adapter makes any model look identical from LUCID's perspective.

```
┌─────────────────────────────────────────────────────┐
│                    LUCID Core                       │
│  (continuous loop, ACG, tours, centroids, schema)   │
└────────────────────────┬────────────────────────────┘
                         │  SUEAdapter interface
┌────────────────────────▼────────────────────────────┐
│                  SUE Adapter                        │
│   config (paths, metadata)                          │
│   + adapter code (extraction, projection, sampling) │
└────────────────────────┬────────────────────────────┘
                         │
              ┌──────────▼──────────┐
              │    Model Backend    │
              │  (ONNX, cloud API,  │
              │   future runtime)   │
              └─────────────────────┘
```

### 27.2 The SUEAdapter Interface

```typescript
interface SUEAdapter {
  readonly metadata: ModelMetadata;

  generate(request: GenerateRequest): AsyncIterable<GenerateChunk>;

  embed(input: EmbedInput): Promise<Float32Array>; // length = CANON_DIM (2048)

  sampleLayers(context: SampleContext): Promise<LayerSamples>;
}

interface ModelMetadata {
  modelId: string;        // stored in node_embeddings_inf.model_id
  modelVersion: string;   // stored in node_embeddings_inf.model_version
  nativeDim: number;      // hidden_size of the model's backbone
  layerCount: number;     // total number of backbone layers
  streamLayout: StreamLayout;
}

type StreamLayout =
  | { type: 'dual'; primaryCount: number; secondaryCount: number }
  | { type: 'single'; layerCount: number };

interface GenerateRequest {
  messages: Message[];
  systemPrompt?: string;
  maxTokens?: number;
  temperature?: number;
}

interface GenerateChunk {
  token: string;
  logprob?: number;
  done: boolean;
}

type EmbedInput =
  | { modality: 'text';  tokens: Uint32Array }
  | { modality: 'image'; pixels: ImageData }
  | { modality: 'audio'; samples: Float32Array };

interface SampleContext {
  // The hidden states produced during a forward pass.
  // The adapter receives whatever the model backend natively exposes;
  // it is the adapter's job to extract the five normalised samples from it.
  rawOutput: unknown;
}

interface LayerSamples {
  // Five samples at normalised relative depths: 0%, 25%, 50%, 75%, 100%.
  // Each sample is a flat Float32Array of length CANON_DIM (2048).
  // Two streams if the architecture supports it; one stream otherwise.
  primary: [Float32Array, Float32Array, Float32Array, Float32Array, Float32Array];
  secondary?: [Float32Array, Float32Array, Float32Array, Float32Array, Float32Array];
}
```

**CANON_DIM = 2048.** This is not a configuration value. It is the embedding dimension of the `lucid.node_embeddings_inf` table. Adapters for models with a different native dimension must project to 2048 before returning. Adapters for models with a 2048 native dimension return tensors directly.

---

### 27.3 The Config Object

Each adapter is instantiated from a config object. This is the passive, declarative part — no code runs here.

```typescript
interface SUEConfig {
  modelId: string;
  modelVersion: string;

  runtime: OnnxRuntimeConfig | CloudRuntimeConfig;

  // Optional: projection config for non-2048 models.
  // Omit for LFM 2.5 (native dim = 2048, no projection needed).
  projection?: ProjectionConfig;
}

interface OnnxRuntimeConfig {
  type: 'onnx';
  decoderPath: string;        // path to decoder_q4.onnx or decoder_fp16.onnx
  embedTokensPath: string;    // path to embed_tokens.bin
  encoderPath?: string;       // vision or audio encoder, loaded on demand
  executionProvider: 'wasm' | 'webgpu' | 'cuda' | 'cpu';
}

interface CloudRuntimeConfig {
  type: 'cloud';
  baseUrl: string;
  model: string;
  apiKeyEnvVar: string;
}

interface ProjectionConfig {
  method: 'pca' | 'random';   // PCA for quality; random projection (JL) for speed
  matrixPath: string;          // path to serialised projection matrix
}
```

---

### 27.4 Method Specifications

Each method below is documented in three parts: what LUCID expects (the contract), what LFM 2.5 actually returns (the noop case), and what a non-trivial adapter must do differently.

---

#### 27.4.1 `generate`

**Contract.** Returns a token stream for a given request. LUCID uses this during operator turns (cloud LLM mounted as Lucy's voice) and during continuous loop narration (local ONNX model). The stream must be async-iterable, yielding chunks until `done: true`. LUCID does not care about the model's internal state — it assembles context from the database before each call.

**LFM 2.5 (noop).** Forward pass through `decoder_q4.onnx` (browser) or `decoder_fp16.onnx` (server) using Transformers.js or a direct ONNX Runtime session. Token stream yielded as produced. No transformation required.

```typescript
// LFM 2.5 adapter — generate
async *generate(request: GenerateRequest): AsyncIterable<GenerateChunk> {
  for await (const chunk of this.onnxSession.generate(request)) {
    yield chunk; // pass through
  }
}
```

**Non-trivial case.** A cloud API adapter wraps an OpenAI-compatible streaming endpoint. The format transformation (SSE → `GenerateChunk`) is the entire job of the adapter. No projection involved.

```typescript
// Cloud adapter — generate
async *generate(request: GenerateRequest): AsyncIterable<GenerateChunk> {
  const stream = await this.client.chat.completions.create({ stream: true, ... });
  for await (const event of stream) {
    yield { token: event.choices[0].delta.content ?? '', done: event.choices[0].finish_reason !== null };
  }
}
```

---

#### 27.4.2 `embed`

**Contract.** Returns a `Float32Array` of exactly length 2048 (CANON_DIM) representing the inference-space embedding for the given input. This value is written to `lucid.node_embeddings_inf.embedding` as `vector(2048)`. The `model_id` and `model_version` columns on that table are populated from `adapter.metadata` — cosine similarity comparisons in the tour engine are only performed within matching `(model_id, model_version)` partitions.

**LFM 2.5 (noop).** The LFM2-1.2B backbone produces 10 `past_conv_*` tensors per forward pass, each of shape `[1, 2048, 3]`. The embedding is the mean of the last position slice across all 10 tensors, producing `Float32Array` of length 2048. No projection.

```typescript
// LFM 2.5 adapter — embed
async embed(input: EmbedInput): Promise<Float32Array> {
  const output = await this.onnxSession.run(this.prepareInput(input));

  // 10 past_conv_* tensors, shape [1, 2048, 3] each.
  // Take last position (index 2) from each, mean across the 10.
  const tensors = Array.from({ length: 10 }, (_, i) => output[`past_conv_${i}`].data as Float32Array);
  const result = new Float32Array(2048);
  for (const t of tensors) {
    for (let d = 0; d < 2048; d++) result[d] += t[d * 3 + 2]; // last of 3 positions
  }
  for (let d = 0; d < 2048; d++) result[d] /= 10;
  return result; // length = 2048, no projection needed
}
```

**Non-trivial case.** A model with `hidden_size = 4096` (e.g. a larger LFM generation or a third-party transformer) produces embeddings that cannot be written directly to `vector(2048)`. The adapter applies a pre-computed projection matrix to reduce to CANON_DIM:

```typescript
// Non-2048 adapter — embed (projection required)
async embed(input: EmbedInput): Promise<Float32Array> {
  const native = await this.extractNativeEmbedding(input); // Float32Array, length = 4096
  return this.projectionMatrix.multiply(native);            // → Float32Array, length = 2048
}
```

The projection matrix is a `(2048 × nativeDim)` matrix, either:
- **PCA**: computed offline on a representative corpus. Preserves maximum variance. `cos(e_canon^i, e_canon^j) ≈ cos(e_native^i, e_native^j)` for well-separated vectors.
- **Random (JL)**: random Gaussian entries, scaled by `1/sqrt(2048)`. Dimension-reduction guarantee without a corpus. Cheaper; slightly more distortion.

The projection matrix is stored at `config.projection.matrixPath` and loaded once at adapter initialisation.

---

#### 27.4.3 `sampleLayers`

**Contract.** Returns hidden states sampled at five normalised relative depths — 0%, 25%, 50%, 75%, 100% — for each of the model's streams. This feeds the inner spectral monitor (§11): FFT is run on these five samples across turns to detect DC dominance (repeating signal across layers), narrowing spectral variance, and loss of cross-stream independence.

The samples are flat `Float32Array` of length CANON_DIM (2048). If the model has two architecturally distinct layer types, `secondary` is populated; otherwise only `primary` is returned and the spectral monitor operates on a single stream.

**Relative depth formula.** For a stream with N layers, the sample at relative depth r is taken from layer `floor(r * (N - 1))`.

**LFM 2.5 (noop).** LFM2-1.2B has two natural streams: 10 CfC/LIV convolution blocks (primary) and 6 GQA attention blocks (secondary). Hidden states are available as `past_conv_*` and `past_key_values_*` tensors respectively. The 5-point samples are extracted at indices `[0, 2, 4, 7, 9]` (conv) and `[0, 1, 2, 4, 5]` (GQA). No projection.

```typescript
// LFM 2.5 adapter — sampleLayers
async sampleLayers(ctx: SampleContext): Promise<LayerSamples> {
  const output = ctx.rawOutput as LFM25Output;

  // Primary: 10 conv layers. Relative depths → layer indices [0, 2, 4, 7, 9].
  const convIndices = [0, 2, 4, 7, 9];
  const primary = convIndices.map(i => {
    const t = output[`past_conv_${i}`].data as Float32Array;
    return meanLastPosition(t, 2048); // last of conv_L_cache=3 positions
  }) as LayerSamples['primary'];

  // Secondary: 6 GQA layers. Relative depths → layer indices [0, 1, 2, 4, 5].
  const gqaIndices = [0, 1, 2, 4, 5];
  const secondary = gqaIndices.map(i => {
    const kv = output[`past_key_values_${i}`].data as Float32Array;
    return meanHeads(kv); // mean across 8 heads → Float32Array(2048)
  }) as LayerSamples['primary'];

  return { primary, secondary }; // no projection, native dim = 2048
}
```

**Non-trivial case — single stream model.** A standard transformer with 32 layers and `hidden_size = 4096` has one stream. Sample at relative depths `floor(r * 31)` for r ∈ {0, 0.25, 0.5, 0.75, 1.0} → indices `[0, 7, 15, 23, 31]`. Project each sample from 4096 to 2048. Return as `primary` only; `secondary` is absent. The spectral monitor operates in single-stream mode and cannot compute cross-stream correlation.

```typescript
// Single-stream non-2048 adapter — sampleLayers
async sampleLayers(ctx: SampleContext): Promise<LayerSamples> {
  const hiddenStates = ctx.rawOutput.hidden_states as Float32Array[]; // 32 × 4096
  const indices = [0, 7, 15, 23, 31];
  const primary = indices.map(i =>
    this.projectionMatrix.multiply(hiddenStates[i]) // → Float32Array(2048)
  ) as LayerSamples['primary'];
  return { primary }; // no secondary
}
```

---

#### 27.4.4 `metadata`

**Contract.** Returns static model metadata. Called once at adapter initialisation; values are cached by LUCID core and used to populate `model_id`/`model_version` on all writes to `node_embeddings_inf`. No computation occurs here.

**LFM 2.5 (noop).** Returns constants derived directly from the model's `config.json`.

```typescript
// LFM 2.5 adapter — metadata
get metadata(): ModelMetadata {
  return {
    modelId: 'lfm2.5-1.2b',
    modelVersion: '1.0.0',
    nativeDim: 2048,
    layerCount: 16, // 10 conv + 6 attn
    streamLayout: { type: 'dual', primaryCount: 10, secondaryCount: 6 },
  };
}
```

**Non-trivial case.** Any values consistent with the model. `streamLayout: { type: 'single', layerCount: 32 }` for a standard transformer. The tour engine uses `modelId`/`modelVersion` to partition KNN queries; it does not inspect the layout. The spectral monitor uses `streamLayout` to decide whether cross-stream correlation is computable.

---

### 27.5 Model Change Protocol

When the running model changes (LFM 2.5 → LFM 3, or to any other model):

1. **Register the new adapter** with its config. New `modelId`/`modelVersion` values.
2. **Re-embed pending nodes.** All nodes with `index_state = 'unindexed'` or `'ont_indexed'` are re-embedded by the new adapter. New rows are inserted into `node_embeddings_inf` with the new `(model_id, model_version)`.
3. **Existing nodes are not re-projected.** The old embeddings remain under the old `(model_id, model_version)` partition. The tour engine queries only within the active partition. Nodes with no embedding in the new partition are `unindexed` until re-embedded.
4. **Centroids are recomputed** after the re-embedding pass. $C_o$ (ontic, Nomic Matryoshka 768-dim) is unaffected — it has no dependency on LFM 2.5.
5. **The old adapter is not removed.** It remains registered and can be reactivated via config flag, enabling rollback without data loss.

The ontic track ($C_o$, `node_embeddings_ont`, `similar_ont` edges) is model-independent by design and survives any inference model change intact. The dual-tour architecture's structural coherence is preserved through model transitions because the ontic tour provides a stable reference frame.

---

### 27.6 Registering an Adapter

The adapter registry is a simple map from `modelId` to `SUEAdapter` instance. LUCID core resolves the active adapter from the registry on startup.

```typescript
// registry.ts
const registry = new Map<string, SUEAdapter>();

export function registerAdapter(adapter: SUEAdapter): void {
  registry.set(adapter.metadata.modelId, adapter);
}

export function getActiveAdapter(): SUEAdapter {
  const id = process.env.LUCID_MODEL_ID ?? 'lfm2.5-1.2b';
  const adapter = registry.get(id);
  if (!adapter) throw new Error(`No adapter registered for model: ${id}`);
  return adapter;
}
```

The v2.5 baseline registration:

```typescript
import { LFM25Adapter } from './adapters/lfm25';

registerAdapter(new LFM25Adapter({
  modelId: 'lfm2.5-1.2b',
  modelVersion: '1.0.0',
  runtime: {
    type: 'onnx',
    decoderPath: './models/decoder_q4.onnx',
    embedTokensPath: './models/embed_tokens.bin',
    executionProvider: 'wasm',
  },
  // No projection config — native dim is CANON_DIM.
}));
```

---

[← §26 Implementation Plan](26-implementation-plan.md) | [Index](../README.md)
