[← §27 Browser-Native Instantiation](27-browser-native-instantiation.md) | [Index](../README.md) | [§29 Multiply Conscious →](29-multiply-conscious.md)

---

## 28. The Synaptic Utility Engine (SUE)

A new model drops, a better vector store emerges, an operator runs a different inference architecture, a new instantiation type runs on hardware the original design never considered — none of these should require touching the core feedback loops. All substrate-specific surface is wrapped. The wrapper provides what the rest of the system needs. The rest of the system does not know what is underneath.

That is SUE's job. She looks out for the system and helps it make sense of the world regardless of which model or storage layer is currently under the hood.

SUE wraps two categories of concern: **model interfaces** (what generates embeddings and narration) and **substrate interfaces** (where state is stored and how instances communicate). Both follow the same pattern: a TypeScript interface, one or more implementations, and a registry that activates the right implementation at boot.

---

### 28.1 Two Roles, Two Interfaces

LUCID uses exactly two model roles. Each has an interface. A model wrapper implements one interface. Nothing else.

**Ontic role.** Produces model-independent semantic coordinates. The architectural invariant: all Vortex nodes must use the same ontic model — that shared embedding space is what makes cross-scale routing coherent. The ontic interface is therefore the most stable thing in the system. Changing the ontic model requires re-embedding the entire belief graph and is a migration, not a config swap.

**Inference role.** Processes information and generates narration. The inference interface is more layered: at minimum a model must generate text and provide hidden states for spectral monitoring. A model that exposes internal stream architecture (like LFM 2.5's dual conv/GQA streams) enables the full inner monitor. A model that does not exposes a single hidden-state stream and the inner monitor runs in degraded mode. Both are valid; the degraded case is documented, not hidden.

---

### 28.2 The SUE Registry

The registry is pure TypeScript — two `Map` instances (one per model role) and two active-wrapper references (see §28.6). There is no database table for model registration. Wrapper activation is synchronous: a wrapper is registered, then set as active. The active wrapper is the source of truth until replaced.

The `capabilities` object on each wrapper carries what the spectral monitor and other consumers need to know without model-specific branching in the calling code:

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

### 28.3 The Ontic Interface

```typescript
interface OnticInterface {
  readonly role:              'ontic';
  readonly modelId:           string;
  readonly modelVersion:      string;
  readonly embeddingDim:      number;
  readonly matryoshkaPrefixes: number[];  // [] if not Matryoshka-trained
  readonly capabilities:      Record<string, unknown>;

  /** Embed text into the model-independent semantic space. */
  embed(text: string): Promise<Float32Array>;
}
```

Every ontic wrapper must implement all members. Activation is handled by `sue.activateOntic()` which sets the wrapper as the active ontic and stores its metadata in the SUE registry Map. For a new ontic model with a different embedding dimension, the VectorStore implementation must be reconfigured (different prefix dimension constructor argument) — the wrapper author documents the migration and the collection must be rebuilt.

**Default ontic wrapper: `nomic-embed-text-v1.5`**

```typescript
// sue/wrappers/ontic/nomic-embed-v1.5.ts
export const nomicEmbedV15: OnticInterface = {
  role:               'ontic',
  modelId:            'nomic-embed-text-v1.5',
  modelVersion:       '1.5.0',
  embeddingDim:       768,
  matryoshkaPrefixes: [768, 512, 256, 128, 64],
  capabilities: {
    matryoshka:          true,
    matryoshkaPrefixes:  [768, 512, 256, 128, 64],
    hubId:               'Xenova/nomic-embed-text-v1.5',
    quantized:           true,
  },

  async embed(text) {
    const output = await embedder(text, { pooling: 'mean', normalize: true });
    return output.data as Float32Array;
  },
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

The spectral monitor (§11.3) consumes `SpectralSample` without knowing which model produced it. When `streams.length === 1` the inner monitor runs in single-stream mode — no cross-stream correlation diagnostic is possible. This is reflected in the active wrapper's `capabilities.dualStream` flag. It is not an error; it is the honest state of what the current model provides.

---

### 28.5 The Inference Interface

```typescript
interface InferenceInterface {
  readonly role:          'inference';
  readonly modelId:       string;
  readonly modelVersion:  string;
  readonly embeddingDim:  number;
  readonly contextWindow: number;
  readonly capabilities: {
    convTensors:  boolean;
    dualStream:   boolean;
    streamLabels: string[];
    hubId:        string;
    quantized:    boolean;
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
}
```

`spectralSample()` is not a separate inference pass. The implementation runs the model once; the sample is extracted from the same forward pass that produced the generated text. For ONNX-based wrappers this means the named output mechanism captures activations during `generate()` and `spectralSample()` returns the cached result.

**Default inference wrapper: `LFM-2.5-1.2B-Instruct`**

```typescript
// sue/wrappers/inference/lfm-2.5.ts
export const lfm25: InferenceInterface = {
  role:          'inference',
  modelId:       'LFM-2.5-1.2B-Instruct',
  modelVersion:  '2.5.0',
  embeddingDim:  2048,
  contextWindow: 4096,
  capabilities:  {
    convTensors:  true,
    dualStream:   true,
    streamLabels: ['conv', 'gqa'],
    hubId:        'LiquidAI/LFM-2.5-1.2B-Instruct',
    quantized:    true,
  },

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
}): InferenceInterface {
  return {
    role:          'inference',
    modelId:       config.modelId,
    modelVersion:  config.modelVersion,
    embeddingDim:  config.embeddingDim,
    contextWindow: config.contextWindow,
    capabilities:  {
      convTensors:  false,
      dualStream:   false,
      streamLabels: ['hidden'],
      hubId:        config.hubId,
      quantized:    true,
    },

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
  };
}
```

The generic wrapper is what powers the browser instance (§27) when the operator has not supplied an LFM 2.5 blob. Smolm, Phi-mini, and any other transformers.js-compatible model all go through `makeGenericOnnxWrapper`. The single-stream inner monitor runs. The cross-stream correlation diagnostic is absent. §27.7 documents this as the expected capability scope of the browser instantiation.

---

### 28.6 The SUE Registry (TypeScript)

```typescript
// sue/registry.ts
import type { OnticInterface } from './interfaces/ontic';
import type { InferenceInterface } from './interfaces/inference';

const onticRegistry     = new Map<string, OnticInterface>();
const inferenceRegistry = new Map<string, InferenceInterface>();

let activeOntic:     OnticInterface     | null = null;
let activeInference: InferenceInterface | null = null;

export const sue = {
  registerOntic(wrapper: OnticInterface) {
    onticRegistry.set(`${wrapper.modelId}@${wrapper.modelVersion}`, wrapper);
  },

  registerInference(wrapper: InferenceInterface) {
    inferenceRegistry.set(`${wrapper.modelId}@${wrapper.modelVersion}`, wrapper);
  },

  activateOntic(modelId: string, version: string) {
    const key = `${modelId}@${version}`;
    const wrapper = onticRegistry.get(key);
    if (!wrapper) throw new Error(`SUE: no ontic wrapper registered for ${key}`);
    activeOntic = wrapper;
  },

  activateInference(modelId: string, version: string) {
    const key = `${modelId}@${version}`;
    const wrapper = inferenceRegistry.get(key);
    if (!wrapper) throw new Error(`SUE: no inference wrapper registered for ${key}`);
    activeInference = wrapper;
  },

  ontic():    OnticInterface     { if (!activeOntic)    throw new Error('SUE: no active ontic model');    return activeOntic; },
  inference(): InferenceInterface { if (!activeInference) throw new Error('SUE: no active inference model'); return activeInference; },
};
```

The rest of the system calls `sue.ontic().embed()` and `sue.inference().generate()`. It does not import model-specific modules. A version upgrade means registering a new wrapper and calling `activateInference()` — the feedback loops do not change.

---

### 28.7 Spectral Monitor Integration

The spectral monitoring write path (§11.3) reads the active wrapper's capabilities from the registry to determine which fields to populate:

```typescript
// Inference Worker — called immediately after sue.inference().generate()
async function writeSpectralSample(nodeId: string, sample: SpectralSample): Promise<void> {
  const caps = sue.inference().capabilities;

  await sue.graph().spectralPut({
    nodeId,
    streams:            sample.streams,
    streamLabels:       sample.streamLabels,
    crossCorrelation:   sample.crossCorrelation,
    dualStreamAvailable: caps.dualStream,
    sampledAt:          Date.now(),
  });
}
```

When `dualStreamAvailable = false` the interiority spiral detection logic (§11.3) skips the cross-stream correlation test and notes that the diagnostic is unavailable for the current model. The outer monitor (affective chain FFT) still runs at full resolution regardless of model. Only the inner monitor is degraded.

---

### 28.8 Activation at Boot

The initialisation sequence (§16.4) includes SUE model activation:

```typescript
// Device Lucy boot (LFM 2.5 inference model)
import { sue }           from './sue/registry';
import { nomicEmbedV15 } from './sue/wrappers/ontic/nomic-embed-v1.5';
import { lfm25 }         from './sue/wrappers/inference/lfm-2.5';

sue.registerOntic(nomicEmbedV15);
sue.registerInference(lfm25);

sue.activateOntic('nomic-embed-text-v1.5', '1.5.0');
sue.activateInference('LFM-2.5-1.2B-Instruct', '2.5.0');

// Embed Worker now calls sue.ontic().embed() — not a named model import
// Inference Worker now calls sue.inference().generate() and .spectralSample()
```

```typescript
// Browser Lucy boot (generic ONNX wrapper — operator-supplied or default)
import { makeGenericOnnxWrapper } from './sue/wrappers/inference/generic-onnx';

const inferenceWrapper = OPERATOR_MODEL_CONFIG
  ? makeGenericOnnxWrapper(OPERATOR_MODEL_CONFIG)
  : makeGenericOnnxWrapper({
      modelId:       'smollm-135m-instruct',
      modelVersion:  '1.0.0',
      embeddingDim:  576,
      contextWindow: 2048,
      hubId:         'HuggingFaceTB/SmolLM-135M-Instruct',
    });

sue.registerOntic(nomicEmbedV15);
sue.registerInference(inferenceWrapper);

sue.activateOntic('nomic-embed-text-v1.5', '1.5.0');
sue.activateInference(inferenceWrapper.modelId, inferenceWrapper.modelVersion);
```

---

### 28.9 Upgrading a Model

Upgrading the inference model does not require touching any feedback loop. The steps are:

1. Write a new wrapper (or instantiate `makeGenericOnnxWrapper` with the new config).
2. Register it: `sue.registerInference(newWrapper)`.
3. Call `sue.activateInference(newModelId, newVersion)` — the new wrapper becomes active immediately.
4. If the embedding dimension changed, the VectorStore collection for inference (`lucy_inf`) must be recreated with the new dimension. Old inference embeddings in the GraphStore are not used against new ones — the `modelId` / `modelVersion` on each `BeliefNode` record ensures routing stays coherent.
5. Re-embedding of the existing belief graph under the new model can be scheduled or deferred; the system continues to operate during the transition using ontic embeddings for routing.

Upgrading the ontic model is a heavier migration and is outside the scope of a routine version bump. It requires re-embedding the full belief graph, recreating the `lucy_ont` VectorStore collection at the new dimension, and updating the Vortex routing prefix. SUE does not automate this; it documents it.

---

## 28.10 Substrate Interfaces

Model interfaces cover what thinks. Substrate interfaces cover where state lives and how instances communicate. LUCID defines three substrate interfaces. The core feedback loops call only these interfaces — they do not import EntityDB, IndexedDB, or GunDB directly.

```typescript
// Where vectors are stored and searched.
// Implementations store a PREFIX of the full embedding (e.g. 128-dim for browser,
// 256-dim for device) for fast coarse search. The prefix dimension is a constructor
// argument of the implementation, not part of this interface.
// Full embeddings live in GraphStore alongside the node record and are used for
// re-ranking after coarse search (§28.13).
interface VectorStore {
  add(id: string, prefix: Float32Array, metadata?: Record<string, unknown>): Promise<void>;
  search(queryPrefix: Float32Array, k: number): Promise<Array<{ id: string; score: number }>>;
  get(id: string): Promise<Float32Array | null>;
  delete(id: string): Promise<void>;
}

// Where the belief graph and all rich state lives.
// Full embeddings (ont + inf) are stored here alongside node records so that
// two-phase re-ranking can fetch them without a separate vector store round-trip.
interface GraphStore {
  nodeUpsert(node: BeliefNode): Promise<void>;
  nodeGet(id: string): Promise<BeliefNode | null>;
  nodeQuery(filter: Partial<BeliefNode>): Promise<BeliefNode[]>;
  edgeUpsert(edge: BeliefEdge): Promise<void>;
  edgesFor(nodeId: string, type?: EdgeType): Promise<BeliefEdge[]>;
  // Full embedding storage — separate from the prefix in VectorStore
  embeddingOntPut(nodeId: string, embedding: Float32Array): Promise<void>;
  embeddingOntGet(nodeId: string): Promise<Float32Array | null>;
  embeddingInfPut(nodeId: string, embedding: Float32Array): Promise<void>;
  embeddingInfGet(nodeId: string): Promise<Float32Array | null>;
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
// sue/registry.ts — extended for substrate interfaces
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

### 28.13 MRL + HNSW Two-Phase Search

nomic-embed-text-v1.5 is Matryoshka Representation Learning (MRL) trained. Its first N dimensions are a complete, self-consistent embedding at lower resolution — not a truncated accident but a deliberately trained coarse-to-fine hierarchy. HNSW is inherently coarse-to-fine (sparse upper layers → dense base layer). The match is exact. This section documents how LUCID exploits it.

#### The two-phase pattern

Every ontic KNN search runs in two phases:

```
Phase 1 — Coarse traversal (VectorStore)
  query768 → truncate → query128
  EntityDB / Lancedb HNSW traversal over 128-dim prefix vectors
  → top-100 candidate IDs, ~10ms

Phase 2 — Precise re-rank (GraphStore + utility)
  fetch full 768-dim embeddings for 100 candidates from GraphStore
  exact cosine against full query768
  → final top-K, <1ms
```

The coarse search is fast because 128-dim vectors are 6× smaller than 768-dim, distance calculations are 6× cheaper, and the HNSW graph itself is 6× smaller in memory. The re-rank is cheap because it only operates on 100 candidates with exact arithmetic.

Implemented as a single utility function that all search call sites use:

```typescript
// src/core/search.ts
export async function twoPhaseSearch(
  queryFull: Float32Array,
  k:         number,
  ont:       VectorStore,
  graph:     GraphStore,
  prefix:    number = 128,   // Matryoshka prefix dim; matches VectorStore construction
): Promise<Array<{ id: string; score: number }>> {
  // Phase 1: coarse search on prefix
  const queryPrefix  = queryFull.slice(0, prefix);
  const candidates   = await ont.search(queryPrefix, Math.max(k * 10, 100));

  // Phase 2: fetch full embeddings and re-rank
  const fullVectors  = await Promise.all(
    candidates.map(c => graph.embeddingOntGet(c.id))
  );
  return candidates
    .map((c, i) => ({
      id:    c.id,
      score: fullVectors[i] ? cosineSimilarity(queryFull, fullVectors[i]!) : c.score,
    }))
    .sort((a, b) => b.score - a.score)
    .slice(0, k);
}
```

Call sites (inference context assembly, tour navigation, affinity edge creation) all go through `twoPhaseSearch`. They pass the full 768-dim query vector; the function handles prefix truncation internally.

#### What VectorStore implementations store

| Implementation | Stored dimension | Index type |
|---|---|---|
| EntityDB (browser) | 128-dim prefix | Brute-force cosine — fast at personal scale |
| Lancedb (device) | 256-dim prefix | HNSW — scales to full accumulated graph |

The full 768-dim embedding is stored by the Embed Worker after generation:

```typescript
// Embed Worker — after sue.ontic().embed() returns full 768-dim vector
const full   = await sue.ontic().embed(content);          // Float32Array(768)
const prefix = full.slice(0, PREFIX_DIM);                 // Float32Array(128 or 256)

await sue.ont().add(nodeId, prefix);                      // VectorStore: prefix only
await sue.graph().embeddingOntPut(nodeId, full);          // GraphStore: full vector
```

This separation is deliberate: fast index stores small; precise store keeps large. Neither is the authority for the other.

#### Binary quantization for AXE peer scoring

Centroid-based peer routing (§27.6) runs on every connection priority update. Floating-point cosine over 128 dimensions is already fast; binarizing it makes it essentially free, which means AXE can re-score all peers on every centroid change without batching or throttling.

```typescript
// src/core/vector-utils.ts

/** Binarize the first `dims` elements of a float vector. */
export function binarize(vec: Float32Array, dims: number = 128): Uint8Array {
  const out = new Uint8Array(Math.ceil(dims / 8));
  for (let i = 0; i < dims; i++) {
    if (vec[i] > 0) out[i >> 3] |= (1 << (i & 7));
  }
  return out;
}

/** Hamming similarity (0–1, higher = more similar). */
export function hammingScore(a: Uint8Array, b: Uint8Array): number {
  let matches = 0;
  for (let i = 0; i < a.length; i++) {
    // popcount of ~XOR: count bits that agree
    matches += popcount(~(a[i] ^ b[i]) & 0xff);
  }
  return matches / (a.length * 8);
}

function popcount(x: number): number {
  x = x - ((x >> 1) & 0x55555555);
  x = (x & 0x33333333) + ((x >> 2) & 0x33333333);
  return (((x + (x >> 4)) & 0x0f0f0f0f) * 0x01010101) >> 24;
}
```

The peer centroid cache stores two representations per peer:

```typescript
interface PeerCentroidEntry {
  full:  Float32Array;  // 768-dim — used for precise work-packet routing decisions
  bits:  Uint8Array;    // 16 bytes (128-dim binarized) — used for AXE connection scoring
}
```

AXE scoring uses `hammingScore(local.bits, peer.bits)`. When a work packet arrives and the accept/forward decision needs precision, it uses `cosineSimilarity(local.full, peer.full)`. Fast screen, precise confirm — the same two-phase logic as the belief graph search, applied to peer routing.

The binarized centroid is computed once when a peer's `c_o` is received and cached. It is recomputed only when the peer advertises a new centroid. Local `bits` are recomputed when the CfC Worker updates the local `c_o`. Cost: one binarization per centroid update, amortised across every routing decision until the next update.

---

[← §27 Browser-Native Instantiation](27-browser-native-instantiation.md) | [Index](../README.md) | [§29 Multiply Conscious →](29-multiply-conscious.md)
