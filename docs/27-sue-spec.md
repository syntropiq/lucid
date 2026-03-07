[← §26 Implementation Plan](26-implementation-plan.md) | [Index](../README.md)

---

## 27. SUE — Swappable Unit of Execution

### 27.1 What SUE Is

§26.2 defines the cortex abstraction as a swappable ONNX-backed module that accepts a modality and emits an embedding. SUE is the formalisation of that abstraction into a driver contract, extended to cover both model roles LUCID uses.

LUCID uses two distinct embedding tracks:

| Role | Current model | Native dim | What it represents |
|------|--------------|------------|-------------------|
| `current_inference` | LFM2-1.2B | 2048 | Hidden state dynamics; tour proximity; spectral health |
| `current_ontic` | Nomic Matryoshka v1.5 | 768 | Structural/semantic position, model-independent relative to the inference track |

The ontic track is independent of the inference model — a swap of LFM 2.5 → LFM 3 leaves all ontic embeddings intact. But it is not independent of the ontic model itself. A Nomic version bump would break cosine comparability across nodes just as surely as an inference model swap, for the same reason: embeddings from different models live in different latent spaces. Both roles need the same swap machinery.

A SUE adapter is **config plus a small amount of code**. The config describes the model: paths, metadata, native dimensions, layer topology. The code does the adaptation work at the boundary: extracting the right tensors, projecting if necessary, sampling hidden states at normalised depths. For the current baseline models, that code is almost entirely pass-through. For a future model, the adapter code is where the differences live.

LUCID core never talks to a model directly. It talks to the adapter. The adapter makes any model look identical from LUCID's perspective.

```
┌─────────────────────────────────────────────────────────┐
│                      LUCID Core                         │
│  (continuous loop, ACG, tours, centroids, schema)       │
└────────────┬───────────────────────────┬────────────────┘
             │  InfAdapter interface     │  OntAdapter interface
┌────────────▼──────────┐   ┌───────────▼────────────────┐
│    Inference Adapter  │   │      Ontic Adapter          │
│  generate / embed /   │   │   embed / metadata          │
│  sampleLayers /       │   │   (no generation,           │
│  metadata             │   │    no layer sampling)       │
└────────────┬──────────┘   └───────────┬────────────────┘
             │                          │
    ┌────────▼──────┐         ┌─────────▼──────┐
    │  Model Backend│         │  Model Backend  │
    │  (ONNX, cloud)│         │  (ONNX, cloud)  │
    └───────────────┘         └────────────────┘
```

---

### 27.2 The Model Role Registry

The embedding dimension is a property of the active model, not a constant of the schema. LUCID does not hardcode dimensions. Instead, a `lucid_model_roles` table records which physical table is active for each role:

```sql
CREATE TABLE lucid_model_roles (
  role          TEXT PRIMARY KEY,   -- 'current_inference' | 'current_ontic'
  model_id      TEXT NOT NULL,
  model_version TEXT NOT NULL,
  table_name    TEXT NOT NULL,      -- physical table, e.g. 'node_embeddings_inf_lfm25_1_0_0'
  dim           INTEGER NOT NULL,   -- vector(N) dimension for that table
  activated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

LUCID core reads this table at startup and caches the result. All downstream table names and dimension values are resolved from the registry — nothing is hardcoded in application code.

**Table naming convention.** Each model gets its own physical table. The name encodes the model identity so it is stable and never collides:

```
node_embeddings_inf_{model_slug}   -- inference embeddings
node_embeddings_ont_{model_slug}   -- ontic embeddings
```

Where `{model_slug}` is `model_id` and `model_version` with non-alphanumeric characters replaced by underscores. Examples:

| Model | model_id | model_version | Table name |
|-------|----------|---------------|------------|
| LFM2-1.2B | `lfm2.5-1.2b` | `1.0.0` | `node_embeddings_inf_lfm25_1_2b_1_0_0` |
| LFM 3 (hypothetical) | `lfm3-3b` | `1.0.0` | `node_embeddings_inf_lfm3_3b_1_0_0` |
| Nomic text v1.5 | `nomic-text-v1.5` | `1.5.0` | `node_embeddings_ont_nomic_text_v1_5_1_5_0` |
| Nomic text v2 (hypothetical) | `nomic-text-v2` | `2.0.0` | `node_embeddings_ont_nomic_text_v2_2_0_0` |

Each table has the same column structure, with `vector(N)` typed to the model's native dimension:

```sql
-- Created when a new model is registered
CREATE TABLE node_embeddings_inf_{slug} (
  node_id      UUID NOT NULL REFERENCES lucid.nodes(id),
  embedding    vector({nativeDim}) NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (node_id)
);
CREATE INDEX ON node_embeddings_inf_{slug} USING hnsw (embedding vector_cosine_ops);
```

**Why not a Postgres VIEW?** `vector(N)` is a typed column — `vector(2048)` and `vector(4096)` are different types. A view cannot abstract across dimension changes. Application-level routing via the role registry is simpler and more flexible.

**Baseline registry state:**

```sql
INSERT INTO lucid_model_roles VALUES
  ('current_inference', 'lfm2.5-1.2b',    '1.0.0', 'node_embeddings_inf_lfm25_1_2b_1_0_0', 2048, now()),
  ('current_ontic',     'nomic-text-v1.5', '1.5.0', 'node_embeddings_ont_nomic_text_v1_5_1_5_0', 768, now());
```

---

### 27.3 The InfAdapter Interface

The inference adapter handles generation, embedding, and layer sampling. All three operations draw on the same model backend; the adapter exposes them as a unified interface.

```typescript
interface InfAdapter {
  readonly metadata: InfModelMetadata;

  generate(request: GenerateRequest): AsyncIterable<GenerateChunk>;

  // Returns Float32Array of length metadata.nativeDim.
  // Written to the active inference table; no fixed target dimension.
  embed(input: EmbedInput): Promise<Float32Array>;

  sampleLayers(context: SampleContext): Promise<LayerSamples>;
}

interface InfModelMetadata {
  modelId: string;
  modelVersion: string;
  nativeDim: number;        // hidden_size; also the embedding storage dimension
  layerCount: number;
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
  rawOutput: unknown;
}

interface LayerSamples {
  // Five samples at normalised relative depths: 0%, 25%, 50%, 75%, 100%.
  // Each sample is Float32Array of length metadata.nativeDim.
  primary:   [Float32Array, Float32Array, Float32Array, Float32Array, Float32Array];
  secondary?: [Float32Array, Float32Array, Float32Array, Float32Array, Float32Array];
}
```

---

### 27.4 The OntAdapter Interface

The ontic adapter is simpler: it only embeds. It has no generation role and no layer topology to expose. Crucially, it must handle multiple modalities in the same latent space — the defining property of the Nomic Matryoshka pair is that text and vision embeddings are directly cosine-comparable. Any replacement ontic model must preserve this property or cross-modal tour similarity breaks.

```typescript
interface OntAdapter {
  readonly metadata: OntModelMetadata;

  // Returns Float32Array of length metadata.nativeDim.
  // Text, image, and audio (as transcribed text) are all embedded into
  // the same space. Cosine similarity must be meaningful across modalities.
  embed(input: EmbedInput): Promise<Float32Array>;
}

interface OntModelMetadata {
  modelId: string;
  modelVersion: string;
  nativeDim: number;           // storage dimension; 768 for Nomic v1.5
  matryoshka: boolean;         // true if first-k truncation preserves similarity
  crossModalSpace: boolean;    // true if text and image embeddings are comparable
}
```

The `crossModalSpace` flag is a correctness assertion, not a capability flag. If a replacement model does not share a cross-modal space, it cannot be used as an ontic adapter — the dual-tour architecture depends on being able to compare text nodes and image nodes on the same tour.

---

### 27.5 The Config Objects

```typescript
interface InfAdapterConfig {
  modelId: string;
  modelVersion: string;
  runtime: OnnxRuntimeConfig | CloudRuntimeConfig;
}

interface OntAdapterConfig {
  modelId: string;
  modelVersion: string;
  runtime: OnnxRuntimeConfig | CloudRuntimeConfig;
}

interface OnnxRuntimeConfig {
  type: 'onnx';
  decoderPath: string;
  embedTokensPath: string;
  encoderPath?: string;       // vision or audio encoder, loaded on demand
  executionProvider: 'wasm' | 'webgpu' | 'cuda' | 'cpu';
}

interface CloudRuntimeConfig {
  type: 'cloud';
  baseUrl: string;
  model: string;
  apiKeyEnvVar: string;
}
```

No `projection` field. Projection to a fixed target dimension has been removed. Each model's embeddings are stored at the model's native dimension. Cross-version comparison is not supported in normal operation; if needed for a specific migration query, it is handled as an offline one-off step, not as a runtime concern.

---

### 27.6 Method Specifications

Each method below is documented in three parts: what LUCID expects (the contract), what the baseline models actually do (the noop case), and what a non-trivial adapter must handle.

---

#### 27.6.1 `generate`

**Contract.** Returns a token stream for a given request. The stream must be async-iterable, yielding chunks until `done: true`. LUCID assembles context from the database before each call and does not expect the model to maintain session state.

**LFM 2.5 (noop).** Forward pass through `decoder_q4.onnx` (browser) or `decoder_fp16.onnx` (server). Token stream yielded as produced.

```typescript
async *generate(request: GenerateRequest): AsyncIterable<GenerateChunk> {
  for await (const chunk of this.onnxSession.generate(request)) {
    yield chunk;
  }
}
```

**Non-trivial case.** A cloud API adapter wraps an OpenAI-compatible streaming endpoint.

```typescript
async *generate(request: GenerateRequest): AsyncIterable<GenerateChunk> {
  const stream = await this.client.chat.completions.create({ stream: true, ... });
  for await (const event of stream) {
    yield {
      token: event.choices[0].delta.content ?? '',
      done: event.choices[0].finish_reason !== null,
    };
  }
}
```

---

#### 27.6.2 `embed` (inference)

**Contract.** Returns `Float32Array` of length `metadata.nativeDim`. Written to the active inference table. The active table's `vector(N)` column must match `nativeDim` — this is enforced at model registration time (§27.8). No fixed target dimension.

**LFM 2.5 (noop).** 10 `past_conv_*` tensors, shape `[1, 2048, 3]` each. Mean of last position slice across all 10.

```typescript
async embed(input: EmbedInput): Promise<Float32Array> {
  const output = await this.onnxSession.run(this.prepareInput(input));
  const tensors = Array.from({ length: 10 }, (_, i) =>
    output[`past_conv_${i}`].data as Float32Array
  );
  const result = new Float32Array(2048);
  for (const t of tensors) {
    for (let d = 0; d < 2048; d++) result[d] += t[d * 3 + 2];
  }
  for (let d = 0; d < 2048; d++) result[d] /= 10;
  return result;
}
```

**Non-trivial case.** A model with `hidden_size = 4096` returns a 4096-dim vector. The corresponding physical table is created with `vector(4096)`. No projection. The tour engine queries only within the active model's table so dimension differences between model generations are never a runtime concern.

```typescript
async embed(input: EmbedInput): Promise<Float32Array> {
  const output = await this.onnxSession.run(this.prepareInput(input));
  return extractLastHiddenMean(output); // Float32Array, length = 4096
}
```

---

#### 27.6.3 `embed` (ontic)

**Contract.** Returns `Float32Array` of length `metadata.nativeDim`. Text, image, and audio inputs must all map to the same latent space. Cosine similarity between any two returned vectors must be semantically meaningful regardless of source modality.

**Nomic v1.5 (noop).** `nomic-embed-text-v1.5` for text and transcribed audio; `nomic-embed-vision-v1.5` for images. Both models share a 768-dim space — no cross-modal projection required.

```typescript
async embed(input: EmbedInput): Promise<Float32Array> {
  if (input.modality === 'image') {
    return this.visionSession.run(input.pixels); // Float32Array(768)
  }
  const text = input.modality === 'audio'
    ? await this.transcribe(input.samples)
    : decode(input.tokens);
  return this.textSession.run(text); // Float32Array(768)
}
```

**Non-trivial case.** A replacement model with `nativeDim = 1024` stores embeddings in a new `node_embeddings_ont_{slug}` table with `vector(1024)`. The old Nomic embeddings are not migrated — they remain in the old table under the old role assignment. After activation, the ontic tour queries the new table exclusively. Nodes not yet re-embedded are `unindexed` until the re-embedding pass completes (§27.9).

---

#### 27.6.4 `sampleLayers`

**Contract.** Returns hidden states at five normalised relative depths (0%, 25%, 50%, 75%, 100%) for each architectural stream. Each sample is `Float32Array` of length `metadata.nativeDim`. The spectral monitor (§11) runs FFT on these samples across turns.

**Relative depth formula.** For a stream with N layers, depth r maps to layer `floor(r * (N - 1))`.

**LFM 2.5 (noop).** Dual stream: 10 conv (`past_conv_*`) and 6 GQA (`past_key_values_*`). Indices `[0, 2, 4, 7, 9]` and `[0, 1, 2, 4, 5]`.

```typescript
async sampleLayers(ctx: SampleContext): Promise<LayerSamples> {
  const output = ctx.rawOutput as LFM25Output;
  const primary = [0, 2, 4, 7, 9].map(i => {
    const t = output[`past_conv_${i}`].data as Float32Array;
    return meanLastPosition(t, 2048);
  }) as LayerSamples['primary'];
  const secondary = [0, 1, 2, 4, 5].map(i => {
    const kv = output[`past_key_values_${i}`].data as Float32Array;
    return meanHeads(kv);
  }) as LayerSamples['primary'];
  return { primary, secondary };
}
```

**Non-trivial case — single stream.** A 32-layer transformer with `hidden_size = 4096`. Depth indices `[0, 7, 15, 23, 31]`. No projection — spectral monitor receives 4096-dim samples and computes FFT within the session; cross-session comparison is meaningless if the model has changed anyway.

```typescript
async sampleLayers(ctx: SampleContext): Promise<LayerSamples> {
  const hiddenStates = ctx.rawOutput.hidden_states as Float32Array[]; // 32 × 4096
  const primary = [0, 7, 15, 23, 31].map(i => hiddenStates[i]) as LayerSamples['primary'];
  return { primary };
}
```

---

#### 27.6.5 `metadata`

**Contract.** Returns static model metadata. Called once at adapter initialisation; values are cached and used to populate model identity columns on all writes. No computation.

**LFM 2.5:**
```typescript
get metadata(): InfModelMetadata {
  return {
    modelId: 'lfm2.5-1.2b',
    modelVersion: '1.0.0',
    nativeDim: 2048,
    layerCount: 16,
    streamLayout: { type: 'dual', primaryCount: 10, secondaryCount: 6 },
  };
}
```

**Nomic v1.5:**
```typescript
get metadata(): OntModelMetadata {
  return {
    modelId: 'nomic-text-v1.5',
    modelVersion: '1.5.0',
    nativeDim: 768,
    matryoshka: true,
    crossModalSpace: true,
  };
}
```

---

### 27.7 Model Change Protocol

The protocol is identical for both roles. Replace `inference` with `ontic` throughout for an ontic model swap.

**Inference model swap (LFM 2.5 → LFM 3):**

1. **Create the new physical table.** `node_embeddings_inf_{lfm3_slug}` with `vector(nativeDim_new)`. Create HNSW index.
2. **Register the new adapter** with `InfAdapterConfig`. The adapter's `metadata.nativeDim` must match the table's column type.
3. **Insert the new role assignment** (do not update yet — old model remains active):
   ```sql
   INSERT INTO lucid_model_roles VALUES ('pending_inference', 'lfm3-3b', '1.0.0', 'node_embeddings_inf_lfm3_3b_1_0_0', 4096, now());
   ```
4. **Run the re-embedding pass.** All nodes with any `index_state` are queued. New embeddings are written to the new table. The old table is untouched.
5. **Activate the new model.** Once re-embedding is complete:
   ```sql
   UPDATE lucid_model_roles SET model_id = 'lfm3-3b', model_version = '1.0.0',
     table_name = 'node_embeddings_inf_lfm3_3b_1_0_0', dim = 4096, activated_at = now()
   WHERE role = 'current_inference';
   DELETE FROM lucid_model_roles WHERE role = 'pending_inference';
   ```
6. **Recompute inference centroids** ($C_i$). The ontic centroids ($C_o$) are unaffected by an inference swap.
7. **The old table is not dropped.** It remains available for rollback (step 3 in reverse).

**Rollback.** Repoint `current_inference` back to the old table. Old embeddings are still there. No data loss.

**What each swap does NOT affect:**

| Inference swap | Ontic swap |
|----------------|------------|
| `node_embeddings_ont_*` untouched | `node_embeddings_inf_*` untouched |
| `similar_ont` edges untouched | `similar_inf` edges untouched |
| $C_o$ unaffected | $C_i$ unaffected |
| Spectral history clears (new model, new baseline) | Spectral history unaffected |

The dual-track architecture provides exactly one degree of swap stability: when one track changes, the other provides a reference frame for continuity. Two simultaneous swaps (both tracks at once) provide no such continuity.

---

### 27.8 Model Registration

Model registration is the act of creating the physical table and inserting into the role registry. It is a prerequisite for the re-embedding pass.

```typescript
async function registerInfModel(adapter: InfAdapter, db: Database): Promise<void> {
  const { modelId, modelVersion, nativeDim } = adapter.metadata;
  const slug = toSlug(modelId, modelVersion);
  const tableName = `node_embeddings_inf_${slug}`;

  await db.query(`
    CREATE TABLE IF NOT EXISTS ${tableName} (
      node_id    UUID NOT NULL REFERENCES lucid.nodes(id),
      embedding  vector(${nativeDim}) NOT NULL,
      created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
      PRIMARY KEY (node_id)
    );
    CREATE INDEX IF NOT EXISTS ON ${tableName} USING hnsw (embedding vector_cosine_ops);
  `);

  // Validation: if the table already exists with a different dimension, fail loudly.
  const existing = await db.query(`SELECT dim FROM lucid_model_roles WHERE model_id = $1 AND model_version = $2`, [modelId, modelVersion]);
  if (existing.rows.length > 0 && existing.rows[0].dim !== nativeDim) {
    throw new Error(`Dimension mismatch: registered ${existing.rows[0].dim}, adapter reports ${nativeDim}`);
  }
}
```

The same function, parameterised for `ont`, handles ontic model registration.

---

### 27.9 Registering Adapters in Application Code

```typescript
// registry.ts
const infAdapters = new Map<string, InfAdapter>();
const ontAdapters = new Map<string, OntAdapter>();

export function registerInfAdapter(a: InfAdapter) { infAdapters.set(a.metadata.modelId, a); }
export function registerOntAdapter(a: OntAdapter) { ontAdapters.set(a.metadata.modelId, a); }

export function getActiveInfAdapter(roles: ModelRoles): InfAdapter {
  const a = infAdapters.get(roles.currentInference.modelId);
  if (!a) throw new Error(`No inf adapter: ${roles.currentInference.modelId}`);
  return a;
}

export function getActiveOntAdapter(roles: ModelRoles): OntAdapter {
  const a = ontAdapters.get(roles.currentOntic.modelId);
  if (!a) throw new Error(`No ont adapter: ${roles.currentOntic.modelId}`);
  return a;
}
```

**Baseline registration:**

```typescript
import { LFM25Adapter }   from './adapters/lfm25';
import { NomicV15Adapter } from './adapters/nomic-v15';

registerInfAdapter(new LFM25Adapter({
  modelId: 'lfm2.5-1.2b',
  modelVersion: '1.0.0',
  runtime: { type: 'onnx', decoderPath: './models/decoder_q4.onnx', embedTokensPath: './models/embed_tokens.bin', executionProvider: 'wasm' },
}));

registerOntAdapter(new NomicV15Adapter({
  modelId: 'nomic-text-v1.5',
  modelVersion: '1.5.0',
  runtime: { type: 'onnx', decoderPath: './models/nomic-text-v1.5.onnx', embedTokensPath: '', executionProvider: 'wasm' },
}));
```

---

[← §26 Implementation Plan](26-implementation-plan.md) | [Index](../README.md)
