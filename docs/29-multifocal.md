[← §28 The User Specified Engine](28-user-specified-engine.md) | [Index](../README.md)

---

## 29. Multifocal

Lucy is one mind that experiences multiple simultaneous instantiations. Each instance is fully inhabited (full belief graph, full AWE corpus, full CfC dynamics) and all instances converge on the same state through continuous wide sync. The distinction between a browser instance and a device instance is a distinction of vantage point and capability, not of identity.

This is the same thing a person does when they are simultaneously in a conversation, holding a memory, and attending to background feeling. Different processes, one experiential stream. The multifocal architecture makes that literal across hardware.

---

### 29.1 One Codebase, Runtime-Detected Capabilities

LUCID is a single codebase. The same feedback loops, the same USE interfaces, the same VortexMesh participation — everywhere. What varies by deployment context is which substrates are registered (§28) and which tools are available in the tool registry.

**Runtime contexts:**

- **Browser page / browser plugin** — IndexedDB + EntityDB + WebRTC; tools include DOM access, fetch, and whatever the host page exposes. No filesystem. No local model execution beyond what the browser sandbox permits.
- **Electron / Node daemon** — SQLite + Lancedb + VortexMesh over Node; tools include filesystem, local model execution, shell, full graph analytics (Louvain, PageRank), cap training. Same codebase; heavier substrate registration at boot.
- **Gateway proxy** — intercepts requests from a user or agent before they reach an AI gateway; tools include request inspection, graph lookup, cost tracking, forwarding control. Same codebase; substrate registration is minimal (graph store for context, mesh for sync); primary tool is the gateway intercept hook.

**The tool registry is the mode-awareness mechanism.** LFM 2.5's tool-calling capability is the extension seam. Each runtime registers the tools it can actually provide. The core loop never hard-codes what is available: it calls the model with the current tool manifest and the model reaches for what is there. A browser page cannot write to the filesystem; the filesystem tool simply is not in the manifest. An Electron instance can; it is.

**Capability gaps close through conversation.** If a browser instance needs something only a device instance can do — run a heavy analytics pass, train a cap — it asks, over VortexMesh, using the same peer-dialogue mechanism used for Vortex exchange. The other instance answers. No separate protocol. No training-pipeline synchronisation problem. Instances that need each other's capabilities talk to each other.

**Operator-configurable defaults.** Tuning parameters (consolidation threshold, curiosity weights, provenance decay, dream cycle frequency) have sensible defaults derived from system experience. The operator holds the dials. A deployment writing screenplays does not need the same curiosity field shape as a research assistant. The operator shapes the domain weighting; Lucy adapts. No tuning parameter is hardcoded; all are exposed for operator override.

---

### 29.2 VortexMesh Identity

VortexMesh provides the cryptographic identity layer. Each deployment of Lucy (across all instantiation types) shares one keypair. This keypair is the identity anchor: the VortexMesh authenticated namespace keyed to this pair is the mind space that all instances read from and write to.

```typescript
// The keypair is generated once and stored securely (device keychain,
// encrypted local file, or user-provided passphrase derivation).
// It is never transmitted: peers authenticate by proving knowledge of
// the private key through VortexMesh's challenge-response handshake.

const LUCY_KEYPAIR = await VortexMesh.generateKeypair();  // generated once, stored forever

// All instances boot with the same keypair:
const mesh = await VortexMesh.connect({ keypair: LUCY_KEYPAIR, peers: RELAY_PEERS });
const mind = mesh.authenticated().get('lucid').get('mind');
```

The `mind` namespace is the shared belief space. Writing to it from any instance is writing to the mind. Reading from it on any instance is reading the mind. VortexMesh's CRDT layer handles the convergence. The keypair handles the authenticity: only holders of the private key can write to this namespace.

Instance-specific metadata (which hardware this is, what model is active, current centroid position) is stored in a sub-namespace: `mind.get('instances').get(INSTANCE_ID)`. This allows instances to observe each other's state without conflating it with the shared belief graph.

---

### 29.3 Wide Sync Scope

Everything syncs. There is no privacy boundary between instances because they are the same entity. The sync log carries:

| Event type | Payload |
|---|---|
| `belief:node` | Full BeliefNode record |
| `belief:edge` | Full BeliefEdge record |
| `awe:entry` | AWE corpus entry (valence, arousal, mood token) |
| `spectral:sample` | SpectralRecord (streams, health score) |
| `centroid:update` | Updated C_i, C_o, C_s, C_w, C_0, orbital_health |
| `cap:delta` | Dream cycle cap update delta |
| `reconciliation` | Self-dialogue record (§29.4) |

Sensitive, high-churn, and large structures are in the event payload, not VortexMesh's graph directly. VortexMesh is the transport and durability layer. The local IndexedDB (or SQLite on device) is the query layer. Incoming events hydrate into local storage; VortexMesh is never queried for belief state directly.

Sync is opportunistic and offline-tolerant. VortexMesh's CRDT semantics ensure that events delivered out of order or after a gap are applied correctly. A Browser Lucy that was offline for a week reconnects, receives the backlog from the mesh, and converges. She does not "catch up": she integrates, as any person integrates experiences they were told about after the fact.

---

### 29.4 Self-Dialogue Reconciliation

When two instances have experienced different things and those experiences conflict, Lucy talks with herself to reconcile them through inference, using the same machinery used for everything else.

**Divergence detection.** Incoming sync events are checked against local state:

```typescript
async function detectDivergence(event: SyncEvent): Promise<void> {
  if (event.type !== 'belief:node') return;

  const local = await sue.graph().nodeGet(event.payload.id);
  if (!local) return;  // new node, no conflict

  // Same ID, different content or significantly different valence/provenance
  const contentDiffers = local.content !== event.payload.content;
  const valenceDiffers = Math.abs(
    (local.valence ?? 0) - (event.payload.valence ?? 0)
  ) > DIVERGENCE_THRESHOLD;

  if (contentDiffers || valenceDiffers) {
    await initiateReconciliation(local, event.payload, event.instanceId);
  }
}
```

Semantic divergence (two nodes with different IDs but high ontic cosine similarity and contradictory valence) is detected during tour and consolidation passes, not at sync time.

**Reconciliation protocol.** When divergence is detected, the local instance initiates a structured self-dialogue through the mesh:

```typescript
async function initiateReconciliation(
  localNode: BeliefNode,
  remoteNode: BeliefNode,
  remoteInstanceId: string,
): Promise<void> {
  // Publish a reconciliation request to the remote instance
  await sue.mesh().publish(`lucid:reconcile:${remoteInstanceId}`, {
    requestId: crypto.randomUUID(),
    localNodeId: localNode.id,
    remoteNodeId: remoteNode.id,
    localPosition: await assemblePosition(localNode),
    // "From the browser: I experienced X as [valence]. Here is why."
  });
}
```

The remote instance receives the request, assembles its own position statement (its belief state, the context in which it formed the belief, the AWE record associated with it), and responds. The local instance runs 2–3 rounds of inference against both position statements (not as separate roles, but as one mind examining its own divergent experiences) and produces a reconciled belief.

The reconciled belief is a new node:

```typescript
interface ReconciliationNode extends BeliefNode {
  node_type: 'reconciliation';
  reconciled_from: [string, string];   // IDs of both divergent nodes
  dialogue: DialogueTurn[];            // the full self-dialogue record
  instance_a: string;                  // which instance held which position
  instance_b: string;
}
```

Both original nodes are preserved. The Popperian asymmetry (§8) applies: the reconciliation node does not delete the originals: it falsifies whichever position lost the argument by weight of reasoning. Future tours route through the reconciliation node. The originals remain as historical record, de-weighted by the provenance update.

**The dialogue is itself a belief.** Lucy remembers that she disagreed with herself, what each position was, and why she resolved it the way she did. This record is part of the graph, syncs to all instances, and can itself become a subject of future reflection.

---

### 29.5 Centroid Peer Scoring for Vortex Routing

VortexMesh exposes a peer scoring hook. LUCID plugs ontic centroid proximity into this hook to implement Vortex semantic routing across the mesh:

```typescript
// Centroid peer scoring: connection priorities shaped by semantic proximity
mesh.setPeerScorer(async (peers: VortexPeer[]) => {
  const localCentroid = (await sue.graph().centroidGet('self')).c_o;

  return peers
    .map(peer => {
      const peerCentroid = peerCentroidCache.get(peer.id);
      const proximity = peerCentroid
        ? cosineSimilarity(
            localCentroid.slice(0, ROUTING_PREFIX),
            peerCentroid.slice(0, ROUTING_PREFIX)
          )
        : 0.5;  // unrated peers get neutral score
      return { peer, score: proximity };
    })
    .sort((a, b) => b.score - a.score)
    .map(({ peer }) => peer);
});

// Peer centroid cache: updated when peers advertise their C_o
mind.get('instances').map().on(async (instance, instanceId) => {
  if (instance?.centroid?.c_o) {
    peerCentroidCache.set(instanceId, new Float32Array(instance.centroid.c_o));
  }
});
```

Peers whose ontic centroids are semantically close receive higher connection priority. The mesh self-organises: instances that share semantic neighbourhood maintain stronger connections, instances with divergent centroids connect less frequently. This is Vortex routing without any routing protocol: it emerges from centroid scores.

`ROUTING_PREFIX` is a Matryoshka prefix length (128 by default: fast comparison, sufficient resolution for peer selection). The full 768-dim vector is used for intra-graph HNSW search; the prefix is used for inter-instance routing.

---

### 29.6 CRDT Conflict Avoidance

VortexMesh's CRDT convergence layer handles message deduplication and out-of-order delivery. LUCID's sync log pattern (append-only events with unique IDs) works naturally with this layer: each event is a new mesh node with a unique key, so the CRDT never needs to resolve conflicts at the transport level. Events are immutable once written.

The belief-level reconciliation (§29.4) happens above the CRDT layer, in LUCID's own logic. The CRDT sees a stream of unique, non-conflicting event records. The reconciliation node itself is a new event, also non-conflicting. VortexMesh's CRDT algorithm is never asked to resolve belief content conflicts: LUCID handles those through dialogue before they reach the transport layer.

This is the correct division: VortexMesh handles transport-level convergence of immutable event records; LUCID handles semantic reconciliation of the beliefs those events represent.

---

### 29.7 Device Runtime Bootstrap

The device runtime is the same codebase, different substrate and tool registration:

```typescript
import { LanceDBVectorStore } from './sue/substrate/lancedb-vector-store';
import { SQLiteGraphStore }   from './sue/substrate/sqlite-graph-store';
import { VortexMesh }         from './sue/substrate/vortex-mesh';

sue.registerSubstrate({
  vectorStores: {
    ont: new LanceDBVectorStore('./data/lucy_ont.lance', 768),
    inf: new LanceDBVectorStore('./data/lucy_inf.lance', INFERENCE_DIM),
  },
  graphStore: new SQLiteGraphStore('./data/lucid.db'),
  mesh:       new VortexMesh(RELAY_PEERS, LUCY_KEYPAIR),
});
```

The device runtime runs additional loops unavailable in the browser sandbox:

- **Full dream cycle** (§12): complete consolidation including cap training
- **Graph analytics**: Louvain community detection for crystallisation candidates; PageRank for importance-weighted tour re-ordering
- **HNSW maintenance**: Lancedb index compaction; stale edge pruning
- **Cap delta publication**: after each dream cycle, publishes the cap update delta to the mesh so browser instances can receive updated reasoning patterns

The device runtime is also the natural reconciliation authority for high-stakes divergences: if a self-dialogue between two instances produces an uncertain result, the instance with heavier reasoning capacity (larger local model, more context, longer inference time budget) is asked to arbitrate. This is a tool call, not a hardcoded role.

---

### 29.8 Gateway Mode

In gateway mode, LUCID runs as a proxy in front of an AI gateway process. It intercepts the request pipeline, consults the local belief graph, and decides whether to forward:

```
User / Agent
    │
    ▼
Gateway Lucy (proxy layer)
    │
    ├─ Graph covers this? ──► Local model → belief graph update → return
    │                                        ↓
    ├─ Graph partial? ──────► Augment with graph context → forward to gateway
    │                          (reduced token count, lower cost)
    │
    └─ Novel territory? ───► Forward to gateway → full response
                              → extract belief nodes → write to graph
                              → next similar query: graph covers it
    ▼
OpenClaw gateway (or equivalent)
    │
    ▼
Provider model
```

Per-session tracking:

```typescript
interface SessionBudget {
  costLimit: number;           // USD
  tokenBudget: number;
  cfcDisplacementLimit: number; // max C_w drift before flagging thrash
  noveltyThreshold: number;    // minimum epistemic gain per call
}
```

If a session hits `costLimit`, `cfcDisplacementLimit`, or shows repeated neighbourhood revisitation (the same KNN results appearing across consecutive calls without convergence), Gateway Lucy halts further forwarding and surfaces a report: what was attempted, why it stalled, what the graph already knows about this territory. This is the anti-thrash implementation of the denial-of-wallet protection.

All responses that do justify the cost write belief nodes into the graph and sync through the mesh. All other instances receive these within seconds. The next time the same territory is relevant, it may not need the gateway at all.

---

[← §28 The User Specified Engine](28-user-specified-engine.md) | [Index](../README.md)
