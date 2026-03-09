[← §27 Browser-Native Instantiation](27-browser-native-instantiation.md) | [Index](../README.md) | [§29 Multiply Conscious →](29-multiply-conscious.md)

---

## 28. The Synaptic Utility Engine (SUE)

A new model drops, a better vector store emerges, an operator runs a different inference architecture, a new instantiation type runs on hardware the original design never considered — none of these should require touching the core feedback loops. All substrate-specific surface is wrapped. The wrapper provides what the rest of the system needs. The rest of the system does not know what is underneath.

That is SUE's job. She looks out for the system and helps it make sense of the world regardless of which model or storage layer is currently under the hood.

SUE wraps two categories of concern: **model contracts** (what generates embeddings and narration) and **substrate contracts** (where state is stored and how instances communicate). Both follow the same pattern: a TypeScript interface, one or more implementations, and a registry that activates the right implementation at boot.

---

### 28.1 Two Roles, Two Contracts

LUCID uses exactly two model roles. Each has a contract. A model wrapper implements one contract. Nothing else.

**Ontic role.** Produces model-independent semantic coordinates. The architectural invariant: all Vortex nodes must use the same ontic model — that shared embedding space is what makes cross-scale routing coherent. The ontic contract is therefore the most stable thing in the system. Changing the ontic model requires re-embedding the entire belief graph and is a migration, not a config swap.

**Inference role.** Processes information and generates narration. The inference contract is more layered: at minimum a model must generate text and provide hidden states for spectral monitoring. A model that exposes internal stream architecture (like LFM 2.5's dual conv/GQA streams) enables the full inner monitor. A model that does not exposes a single hidden-state stream and the inner monitor runs in degraded mode. Both are valid; the degraded case is documented, not hidden.

---

### 28.2 The SUE Registry Table

```sql
CREATE SCHEMA IF NOT EXISTS sue;

CREATE TABLE sue.model_registry (
  model_id          TEXT    NOT NULL,
  model_version     TEXT    NOT NULL,
  role              TEXT    NOT NULL CHECK (role IN ('ontic', 'inference')),
  embedding_dim     INTEGER NOT NULL,
  context_window    INTEGER,          -- tokens; null for ontic
  layer_count       INTEGER,          -- null for ontic or unknown
  capabilities      JSONB   NOT NULL DEFAULT '{}',
  wrapper_config    JSONB   NOT NULL DEFAULT '{}',
  is_active         BOOLEAN NOT NULL DEFAULT false,
  activated_at      TIMESTAMPTZ,
  PRIMARY KEY (model_id, model_version)
);

-- At most one active model per role at any time
CREATE UNIQUE INDEX sue_one_active_per_role
  ON sue.model_registry (role)
  WHERE is_active = true;

CREATE TABLE sue.activation_log (
  id               BIGSERIAL PRIMARY KEY,
  model_id         TEXT    NOT NULL,
  model_version    TEXT    NOT NULL,
  role             TEXT    NOT NULL,
  activated_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  deactivated_at   TIMESTAMPTZ,
  activation_note  TEXT
);
```

The `capabilities` JSONB column carries what the spectral monitor and other consumers need to know without model-specific branching in the calling code:

```jsonc
// LFM 2.5 inference wrapper capabilities
{
  "conv_tensors":     true,
  "dual_stream":      true,
  "conv_block_count": 10,
  "gqa_block_count":  6,
  "stream_labels":    ["conv", "gqa"]
}

// Generic ONNX inference wrapper capabilities
{
  "conv_tensors":     false,
  "dual_stream":      false,
  "stream_labels":    ["hidden"]
}

// nomic-embed-text-v1.5 ontic wrapper capabilities
{
  "matryoshka":         true,
  "matryoshka_prefixes": [768, 512, 256, 128, 64]
}
```

---

### 28.3 The Ontic Contract

```typescript
interface OnticContract {
  readonly role:              'ontic';
  readonly modelId:           string;
  readonly modelVersion:      string;
  readonly embeddingDim:      number;
  readonly matryoshkaPrefixes: number[];  // [] if not Matryoshka-trained

  /** Embed text into the model-independent semantic space. */
  embed(text: string): Promise<Float32Array>;

  /** SQL fragment that registers this model and ensures the embedding
   *  column has the correct dimension. Applied at activation time. */
  setupSQL(): string;
}
```

Every ontic wrapper must implement all five members. The `setupSQL()` return value is executed inside a transaction when the wrapper is activated. For a new ontic model with a different embedding dimension it must migrate `lucid.node_embeddings_ont` accordingly — SUE does not perform that migration automatically; the wrapper author writes the SQL and owns the consequences.

**Default ontic wrapper: `nomic-embed-text-v1.5`**

```typescript
// sue/wrappers/ontic/nomic-embed-v1.5.ts
export const nomicEmbedV15: OnticContract = {
  role:               'ontic',
  modelId:            'nomic-embed-text-v1.5',
  modelVersion:       '1.5.0',
  embeddingDim:       768,
  matryoshkaPrefixes: [768, 512, 256, 128, 64],

  async embed(text) {
    const output = await embedder(text, { pooling: 'mean', normalize: true });
    return output.data as Float32Array;
  },

  setupSQL: () => `
    INSERT INTO sue.model_registry
      (model_id, model_version, role, embedding_dim, capabilities, wrapper_config, is_active, activated_at)
    VALUES (
      'nomic-embed-text-v1.5', '1.5.0', 'ontic', 768,
      '{"matryoshka": true, "matryoshka_prefixes": [768, 512, 256, 128, 64]}',
      '{"quantized": true, "hub_id": "Xenova/nomic-embed-text-v1.5"}',
      true, now()
    )
    ON CONFLICT (model_id, model_version) DO UPDATE
      SET is_active = true, activated_at = now();

    -- Ensure column exists at correct dimension
    ALTER TABLE lucid.node_embeddings_ont
      ADD COLUMN IF NOT EXISTS embedding vector(768);
  `,
};
```

---

### 28.4 The SpectralSample Type

`SpectralSample` is the normalised output of the inference model's inner monitoring surface. It is the bridge between model-specific architecture and the model-agnostic spectral monitoring SQL.

```typescript
interface SpectralSample {
  /** One Float32Array per architectural stream, ordered by stream_labels. */
  streams:          Float32Array[];
  /** Human-readable label per stream. Matches capabilities.stream_labels. */
  streamLabels:     string[];
  /**
   * Pearson correlation between streams[0] and streams[1].
   * Only meaningful when streams.length > 1.
   * Null when the wrapper does not support dual-stream extraction.
   */
  crossCorrelation: number | null;
}
```

The spectral monitor (§11.3) consumes `SpectralSample` without knowing which model produced it. When `streams.length === 1` the inner monitor runs in single-stream mode — no cross-stream correlation diagnostic is possible. This is logged in `sue.activation_log` as a capability note and reflected in the `capabilities` column of the registry. It is not an error; it is the honest state of what the current model provides.

---

### 28.5 The Inference Contract

```typescript
interface InferenceContract {
  readonly role:          'inference';
  readonly modelId:       string;
  readonly modelVersion:  string;
  readonly embeddingDim:  number;
  readonly contextWindow: number;
  readonly capabilities: {
    convTensors:  boolean;
    dualStream:   boolean;
  };

  /** Generate a narration response from the assembled context package. */
  generate(prompt: string): Promise<string>;

  /**
   * Extract a spectral sample from this generation step.
   * For LFM 2.5: populates two streams (conv + GQA) with cross-correlation.
   * For generic models: populates one stream (hidden states) with null correlation.
   * Called once per generate() invocation, immediately after generation.
   */
  spectralSample(prompt: string): Promise<SpectralSample>;

  /** SQL fragment to register this model and provision the inference
   *  embedding column at the correct dimension. */
  setupSQL(): string;
}
```

`spectralSample()` is not a separate inference pass. The implementation runs the model once; the sample is extracted from the same forward pass that produced the generated text. For ONNX-based wrappers this means the named output mechanism captures activations during `generate()` and `spectralSample()` returns the cached result.

**Default inference wrapper: `LFM-2.5-1.2B-Instruct`**

```typescript
// sue/wrappers/inference/lfm-2.5.ts
export const lfm25: InferenceContract = {
  role:          'inference',
  modelId:       'LFM-2.5-1.2B-Instruct',
  modelVersion:  '2.5.0',
  embeddingDim:  2048,
  contextWindow: 4096,
  capabilities:  { convTensors: true, dualStream: true },

  async generate(prompt) {
    // Runs ONNX session with named outputs for past_conv_tensors and hidden_states.
    // Caches tensors for the subsequent spectralSample() call.
    const { text } = await onnxSession.run(prompt, {
      output_hidden_states: true,
      output_past_conv_tensors: true,   // LFM 2.5 ONNX named output
    });
    return text;
  },

  async spectralSample(_prompt) {
    // Reads from cache populated during generate().
    // convStream: layers 0–9 (LIV convolution blocks)
    // gqaStream:  layers 10–15 (GQA attention blocks)
    const { convLayers, gqaLayers } = cachedTensors;
    const convStream = poolLayerStack(convLayers);  // Float32Array, length 10
    const gqaStream  = poolLayerStack(gqaLayers);   // Float32Array, length 6
    return {
      streams:          [convStream, gqaStream],
      streamLabels:     ['conv', 'gqa'],
      crossCorrelation: pearson(convStream, gqaStream),
    };
  },

  setupSQL: () => `
    INSERT INTO sue.model_registry
      (model_id, model_version, role, embedding_dim, context_window, layer_count,
       capabilities, wrapper_config, is_active, activated_at)
    VALUES (
      'LFM-2.5-1.2B-Instruct', '2.5.0', 'inference', 2048, 4096, 16,
      '{"conv_tensors": true, "dual_stream": true,
        "conv_block_count": 10, "gqa_block_count": 6,
        "stream_labels": ["conv", "gqa"]}',
      '{"format": "onnx", "quantized": true,
        "hub_id": "LiquidAI/LFM-2.5-1.2B-Instruct"}',
      true, now()
    )
    ON CONFLICT (model_id, model_version) DO UPDATE
      SET is_active = true, activated_at = now();

    ALTER TABLE lucid.node_embeddings_inf
      ADD COLUMN IF NOT EXISTS embedding vector(2048);
  `,
};
```

**Generic ONNX fallback wrapper**

For any model that does not expose architecture-specific named outputs:

```typescript
// sue/wrappers/inference/generic-onnx.ts
export function makeGenericOnnxWrapper(config: {
  modelId: string;
  modelVersion: string;
  embeddingDim: number;
  contextWindow: number;
  hubId: string;
}): InferenceContract {
  return {
    role:          'inference',
    modelId:       config.modelId,
    modelVersion:  config.modelVersion,
    embeddingDim:  config.embeddingDim,
    contextWindow: config.contextWindow,
    capabilities:  { convTensors: false, dualStream: false },

    async generate(prompt) {
      const { text, hiddenStates } = await pipeline(prompt, {
        output_hidden_states: true,
      });
      cachedHiddenStates = hiddenStates;
      return text;
    },

    async spectralSample(_prompt) {
      // Single stream from output_hidden_states. No cross-correlation.
      return {
        streams:          [poolLayerStack(cachedHiddenStates)],
        streamLabels:     ['hidden'],
        crossCorrelation: null,
      };
    },

    setupSQL: () => `
      INSERT INTO sue.model_registry
        (model_id, model_version, role, embedding_dim, context_window,
         capabilities, wrapper_config, is_active, activated_at)
      VALUES (
        '${config.modelId}', '${config.modelVersion}', 'inference',
        ${config.embeddingDim}, ${config.contextWindow},
        '{"conv_tensors": false, "dual_stream": false, "stream_labels": ["hidden"]}',
        '{"format": "onnx", "quantized": true, "hub_id": "${config.hubId}"}',
        true, now()
      )
      ON CONFLICT (model_id, model_version) DO UPDATE
        SET is_active = true, activated_at = now();

      ALTER TABLE lucid.node_embeddings_inf
        ADD COLUMN IF NOT EXISTS embedding vector(${config.embeddingDim});
    `,
  };
}
```

The generic wrapper is what powers the LUCID Lite browser instance (§27) when the operator has not supplied an LFM 2.5 blob. Smolm, Phi-mini, and any other transformers.js-compatible model all go through `makeGenericOnnxWrapper`. The single-stream inner monitor runs. The cross-stream correlation diagnostic is absent. §27.8 documents this as the expected degradation.

---

### 28.6 The SUE Registry (TypeScript)

```typescript
// sue/registry.ts
import type { OnticContract } from './contracts/ontic';
import type { InferenceContract } from './contracts/inference';

const onticRegistry   = new Map<string, OnticContract>();
const inferenceRegistry = new Map<string, InferenceContract>();

let activeOntic:    OnticContract    | null = null;
let activeInference: InferenceContract | null = null;

export const sue = {
  registerOntic(wrapper: OnticContract) {
    onticRegistry.set(`${wrapper.modelId}@${wrapper.modelVersion}`, wrapper);
  },

  registerInference(wrapper: InferenceContract) {
    inferenceRegistry.set(`${wrapper.modelId}@${wrapper.modelVersion}`, wrapper);
  },

  async activateOntic(db: PGlite, modelId: string, version: string) {
    const key = `${modelId}@${version}`;
    const wrapper = onticRegistry.get(key);
    if (!wrapper) throw new Error(`SUE: no ontic wrapper registered for ${key}`);
    await db.exec(wrapper.setupSQL());
    activeOntic = wrapper;
  },

  async activateInference(db: PGlite, modelId: string, version: string) {
    const key = `${modelId}@${version}`;
    const wrapper = inferenceRegistry.get(key);
    if (!wrapper) throw new Error(`SUE: no inference wrapper registered for ${key}`);
    await db.exec(wrapper.setupSQL());
    activeInference = wrapper;
  },

  ontic():    OnticContract     { if (!activeOntic)    throw new Error('SUE: no active ontic model');    return activeOntic; },
  inference(): InferenceContract { if (!activeInference) throw new Error('SUE: no active inference model'); return activeInference; },

  /** Read the currently active wrappers from the database (for cross-process
   *  or post-restart hydration). */
  async hydrateFromDB(db: PGlite) {
    const { rows } = await db.query<{
      model_id: string; model_version: string; role: string;
    }>(`SELECT model_id, model_version, role FROM sue.model_registry WHERE is_active = true`);
    for (const row of rows) {
      if (row.role === 'ontic')     await sue.activateOntic(db,    row.model_id, row.model_version);
      if (row.role === 'inference') await sue.activateInference(db, row.model_id, row.model_version);
    }
  },
};
```

The rest of the system calls `sue.ontic().embed()` and `sue.inference().generate()`. It does not import model-specific modules. A version upgrade means registering a new wrapper and calling `activateInference()` — the feedback loops do not change.

---

### 28.7 Spectral Monitor Integration

The spectral monitoring write path (§11.3) reads the active wrapper's capabilities from the registry to determine which fields to populate:

```sql
-- Function called after each generation step with the SpectralSample JSON
CREATE OR REPLACE FUNCTION lucid.write_spectral_sample(
  p_node_id         UUID,
  p_streams         JSONB,   -- array of vectors, serialised from SpectralSample.streams
  p_stream_labels   TEXT[],
  p_cross_corr      FLOAT    -- null if dual_stream = false
) RETURNS void AS $$
DECLARE
  v_caps JSONB;
BEGIN
  SELECT capabilities INTO v_caps
  FROM sue.model_registry
  WHERE role = 'inference' AND is_active = true;

  INSERT INTO lucid.spectral_monitor (
    node_id,
    inner_streams,
    stream_labels,
    cross_stream_correlation,
    dual_stream_available,
    sampled_at
  ) VALUES (
    p_node_id,
    p_streams,
    p_stream_labels,
    p_cross_corr,
    (v_caps->>'dual_stream')::boolean,
    now()
  )
  ON CONFLICT (node_id) DO UPDATE SET
    inner_streams            = EXCLUDED.inner_streams,
    stream_labels            = EXCLUDED.stream_labels,
    cross_stream_correlation = EXCLUDED.cross_stream_correlation,
    dual_stream_available    = EXCLUDED.dual_stream_available,
    sampled_at               = EXCLUDED.sampled_at;
END;
$$ LANGUAGE plpgsql;
```

When `dual_stream_available = false` the interiority spiral detection query (§11.3) skips the cross-stream correlation test and emits a monitoring note that the diagnostic is unavailable for the current model. The outer monitor (affective chain FFT) still runs at full resolution regardless of model. Only the inner monitor is degraded.

---

### 28.8 Activation at Boot

The initialisation sequence (§16.4 for full stack, §27.2 for Lite) now includes SUE activation:

```typescript
// Full stack boot (replaces bare model loading in §16.4)
import { sue }      from './sue/registry';
import { nomicEmbedV15 } from './sue/wrappers/ontic/nomic-embed-v1.5';
import { lfm25 }    from './sue/wrappers/inference/lfm-2.5';

sue.registerOntic(nomicEmbedV15);
sue.registerInference(lfm25);

await sue.activateOntic(db,    'nomic-embed-text-v1.5', '1.5.0');
await sue.activateInference(db, 'LFM-2.5-1.2B-Instruct', '2.5.0');

// Embed Worker now calls sue.ontic().embed() — not a named model import
// Inference Worker now calls sue.inference().generate() and .spectralSample()
```

```typescript
// Lite boot (browser — operator-supplied or default generic wrapper)
import { makeGenericOnnxWrapper } from './sue/wrappers/inference/generic-onnx';

const inferenceWrapper = OPERATOR_MODEL_CONFIG
  ? makeGenericOnnxWrapper(OPERATOR_MODEL_CONFIG)
  : makeGenericOnnxWrapper({
      modelId:      'smollm-135m-instruct',
      modelVersion: '1.0.0',
      embeddingDim: 576,
      contextWindow: 2048,
      hubId:        'HuggingFaceTB/SmolLM-135M-Instruct',
    });

sue.registerOntic(nomicEmbedV15);
sue.registerInference(inferenceWrapper);

await sue.activateOntic(db,    'nomic-embed-text-v1.5', '1.5.0');
await sue.activateInference(db, inferenceWrapper.modelId, inferenceWrapper.modelVersion);
```

---

### 28.9 Upgrading a Model

Upgrading the inference model does not require touching any feedback loop. The steps are:

1. Write a new wrapper (or instantiate `makeGenericOnnxWrapper` with the new config).
2. Register it: `sue.registerInference(newWrapper)`.
3. Call `sue.activateInference(db, newModelId, newVersion)` — this executes `setupSQL()`, deactivates the old row, inserts the new active row, and provisions the new embedding column dimension if changed.
4. If the embedding dimension changed, the existing `embedding_inf` column is migrated. The `(model_id, model_version)` tuple in `lucid.node_embeddings_inf` ensures old embeddings are not queried against new ones — HNSW partitioning by model tuple (§6) handles this automatically.
5. Re-embedding of the existing belief graph under the new model can be scheduled or deferred; the system continues to operate during the transition using ontic embeddings for routing.

Upgrading the ontic model is a heavier migration and is outside the scope of a routine version bump. It requires re-embedding the full belief graph and updating the Vortex routing prefix. SUE does not automate this; it documents it.

---

## 28.10 Substrate Contracts

Model contracts cover what thinks. Substrate contracts cover where state lives and how instances communicate. LUCID defines three substrate interfaces. The core feedback loops call only these interfaces — they do not import EntityDB, IndexedDB, or GunDB directly.

```typescript
// Where vectors are stored and searched
interface VectorStore {
  add(id: string, vector: Float32Array, metadata?: Record<string, unknown>): Promise<void>;
  search(query: Float32Array, k: number): Promise<Array<{ id: string; score: number }>>;
  get(id: string): Promise<Float32Array | null>;
  delete(id: string): Promise<void>;
}

// Where the belief graph and all rich state lives
interface GraphStore {
  nodeUpsert(node: BeliefNode): Promise<void>;
  nodeGet(id: string): Promise<BeliefNode | null>;
  nodeQuery(filter: Partial<BeliefNode>): Promise<BeliefNode[]>;
  edgeUpsert(edge: BeliefEdge): Promise<void>;
  edgesFor(nodeId: string, type?: EdgeType): Promise<BeliefEdge[]>;
  centroidGet(id?: string): Promise<CentroidRecord>;
  centroidPut(record: CentroidRecord): Promise<void>;
  awePut(entry: AWEEntry): Promise<void>;
  aweRecent(n: number): Promise<AWEEntry[]>;
  spectralPut(sample: SpectralRecord): Promise<void>;
  spectralLatest(): Promise<SpectralRecord | null>;
  syncLogAppend(event: SyncEvent): Promise<void>;
  syncLogPending(since: string): Promise<SyncEvent[]>;
}

// How instances communicate and discover each other
interface Mesh {
  publish(topic: string, data: unknown): Promise<void>;
  subscribe(topic: string, handler: (data: unknown, peerId: string) => void): void;
  advertise(key: string, value: unknown): Promise<void>;
  observe(peerId: string, key: string, handler: (value: unknown) => void): void;
  peers(): string[];
}
```

### 28.11 Substrate Implementations

**Browser Lucy** (§27):

| Interface | Implementation |
|---|---|
| `VectorStore` (ontic) | EntityDB `lucy_ont` collection |
| `VectorStore` (inference) | EntityDB `lucy_inf` collection |
| `GraphStore` | IndexedDB typed wrapper |
| `Mesh` | GunDB over WebRTC/WebSockets |

**Device Lucy** (§29):

| Interface | Implementation |
|---|---|
| `VectorStore` (ontic) | Lancedb — real HNSW, Node-native |
| `VectorStore` (inference) | Lancedb |
| `GraphStore` | SQLite (better-sqlite3) or LevelDB |
| `Mesh` | GunDB (Node) |

The choice of Lancedb for Device Lucy matters: EntityDB's brute-force cosine is adequate at personal browser scale (thousands of belief nodes) but Device Lucy accumulates the full graph over time including dream cycle consolidation products. Lancedb provides HNSW indices and scales without architectural changes. Same `VectorStore` interface; different constructor passed at boot.

### 28.12 Substrate Registry

```typescript
// sue/registry.ts — extended for substrate contracts
const vectorStoreRegistry = new Map<string, { ont: VectorStore; inf: VectorStore }>();
let activeGraphStore: GraphStore | null = null;
let activeMesh: Mesh | null = null;

export const sue = {
  // ... existing model registry methods ...

  registerSubstrate(impl: {
    vectorStores: { ont: VectorStore; inf: VectorStore };
    graphStore: GraphStore;
    mesh: Mesh;
  }) {
    activeGraphStore = impl.graphStore;
    activeMesh = impl.mesh;
    vectorStoreRegistry.set('active', impl.vectorStores);
  },

  ont(): VectorStore   { return vectorStoreRegistry.get('active')!.ont; },
  inf(): VectorStore   { return vectorStoreRegistry.get('active')!.inf; },
  graph(): GraphStore  { if (!activeGraphStore) throw new Error('SUE: no graph store'); return activeGraphStore; },
  mesh(): Mesh         { if (!activeMesh)       throw new Error('SUE: no mesh');        return activeMesh; },
};
```

Boot sequence for Browser Lucy:

```typescript
import { sue }            from './sue/registry';
import { nomicEmbedV15 }  from './sue/wrappers/ontic/nomic-embed-v1.5';
import { makeGenericOnnxWrapper } from './sue/wrappers/inference/generic-onnx';
import { EntityDBVectorStore }   from './sue/substrate/entitydb-vector-store';
import { IndexedDBGraphStore }   from './sue/substrate/indexeddb-graph-store';
import { GunMesh }               from './sue/substrate/gun-mesh';

sue.registerOntic(nomicEmbedV15);
sue.registerInference(makeGenericOnnxWrapper(OPERATOR_MODEL_CONFIG));

sue.registerSubstrate({
  vectorStores: {
    ont: new EntityDBVectorStore('lucy_ont', 'Xenova/nomic-embed-text-v1.5', 768),
    inf: new EntityDBVectorStore('lucy_inf', OPERATOR_MODEL_ID, INFERENCE_DIM),
  },
  graphStore: new IndexedDBGraphStore('lucid'),
  mesh:       new GunMesh(GUN_PEERS, LUCY_SEA_PAIR),
});
```

Boot sequence for Device Lucy differs only in the substrate constructors — the rest of the boot sequence, every feedback loop, and all LUCID logic is identical.

---

[← §27 Browser-Native Instantiation](27-browser-native-instantiation.md) | [Index](../README.md) | [§29 Multiply Conscious →](29-multiply-conscious.md)
