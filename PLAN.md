# LUCID Lite — Implementation Plan

> The fastest path from nothing to a working feedback loop is: database first, workers second, UI third, network last.

The build order is chosen so that each phase produces something runnable and testable before the next phase begins. No phase depends on a later phase being complete. Each phase extends what came before without rewriting it.

---

## The Core Architectural Bet

**The database is the event bus.**

`pglite.live.query()` subscriptions replace message queues, channels, and worker orchestration. Every handler writes to pglite. pglite notifies the subscriptions. The subscriptions write back. No tick. No polling. No coordinator.

This means:
- Replacing a worker is replacing a live query handler
- Adding a network peer is adding an INSERT path into the same tables the chat UI uses
- The Vortex is the same as the chat interface, differing only in who is doing the INSERT

---

## Phase 1 — DDL + pglite shell

**Goal:** A running pglite instance with the SUE registry, the minimal LUCID schema, and enough queue tables that the live query pattern can be demonstrated. No ONNX yet. No UI yet. Just `node index.ts` or `vitest` against a real in-process pglite.

### Tables in scope

```
sue.model_registry          -- active model config (from §28)
sue.activation_log          -- model activation history

lucid.conversation_turns    -- user/assistant messages (the chat entry point)
lucid.belief_nodes          -- belief primitive (§7)
lucid.belief_edges          -- structural + affinity + epistemic edges (§6.2)

lucid.embedding_queue       -- pending ontic embedding jobs
lucid.inference_queue       -- pending LLM generation jobs

lucid.node_embeddings_ont   -- vector(768) — ontic space
lucid.node_embeddings_inf   -- vector(dim) — inference space, dim from SUE registry

lucid.centroids             -- C_i, C_o, C_s, C_w, C_0 (§6.3)
lucid.spectral_monitor      -- inner/outer monitor state (§11.3)
lucid.affective_corpus      -- AWE chain entries (§3)
```

### Queue pattern

Every queue table has the same shape:

```sql
CREATE TABLE lucid.<name>_queue (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  payload     JSONB NOT NULL,
  status      TEXT NOT NULL DEFAULT 'pending'
                CHECK (status IN ('pending','processing','complete','error')),
  error_msg   TEXT,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ON lucid.<name>_queue (status, created_at)
  WHERE status = 'pending';
```

The live query subscription is always:

```typescript
db.live.query(
  `SELECT id, payload FROM lucid.<name>_queue
   WHERE status = 'pending'
   ORDER BY created_at
   LIMIT 1`,
  [],
  async ({ rows }) => { /* process rows[0] if present */ }
);
```

When the handler marks the row `processing` → `complete`, the result set changes, which fires the handler again for the next pending item. The queue drains itself without polling.

### Triggers in scope

```sql
-- New conversation turn → INSERT inference job
CREATE OR REPLACE FUNCTION lucid.enqueue_inference()
RETURNS trigger AS $$
BEGIN
  IF NEW.role = 'user' THEN
    INSERT INTO lucid.inference_queue (payload)
    VALUES (jsonb_build_object('turn_id', NEW.id));
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_enqueue_inference
  AFTER INSERT ON lucid.conversation_turns
  FOR EACH ROW EXECUTE FUNCTION lucid.enqueue_inference();

-- New belief node → INSERT embedding job
CREATE OR REPLACE FUNCTION lucid.enqueue_embedding()
RETURNS trigger AS $$
BEGIN
  INSERT INTO lucid.embedding_queue (payload)
  VALUES (jsonb_build_object('node_id', NEW.id, 'content', NEW.content));
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_enqueue_embedding
  AFTER INSERT ON lucid.belief_nodes
  FOR EACH ROW EXECUTE FUNCTION lucid.enqueue_embedding();
```

### Deliverable

`src/db/schema.sql` — single file, idempotent (`CREATE TABLE IF NOT EXISTS`, `CREATE OR REPLACE FUNCTION`), applies cleanly on a fresh pglite instance and re-applies without error on an existing one.

`src/db/seed.ts` — boots pglite with `vector` + `live` extensions, applies schema, activates SUE default wrappers (nomic-embed-v1.5, LFM 2.5 or generic fallback), runs a smoke-test INSERT and verifies the queue rows appear.

---

## Phase 2 — ONNX workers as live subscribers

**Goal:** Two Web Workers (or Node.js worker_threads), each subscribing to one queue. By the end of this phase, inserting a row into `lucid.belief_nodes` produces a populated `lucid.node_embeddings_ont` row without any manual intervention.

### Embed Worker

```
embedding_queue (status='pending')
        │ live.query fires
        ▼
embedder.embed(content)              ← sue.ontic().embed()
        │
        ▼
INSERT node_embeddings_ont            ← vector(768)
UPDATE embedding_queue status='complete'
        │
        ▼
centroid_update_trigger fires         ← §6.5 provenance-weighted update
        │
        ▼
lucid.centroids UPDATE c_o            ← live query fires centroid advertisement
```

The embed worker does not know about inference. It knows about `embedding_queue` and `node_embeddings_ont`. Nothing else.

### Inference Worker

```
inference_queue (status='pending')
        │ live.query fires
        ▼
assemble context package              ← recent turns + relevant belief nodes by HNSW
        │
        ▼
sue.inference().generate(prompt)      ← ONNX LLM call
        │
        ▼
INSERT conversation_turns (role='assistant')
INSERT belief_nodes (narration node)  ← triggers embedding_queue INSERT
UPDATE inference_queue status='complete'
        │
        ▼
sue.inference().spectralSample()      ← same forward pass, cached tensors
        │
        ▼
lucid.write_spectral_sample()         ← §28.7 SQL function
```

The context assembly step is deliberately minimal in this phase: last N turns + top-K HNSW neighbours of the user message embedding. The full ACG (§13) comes later.

### Deliverable

`src/workers/embed.worker.ts` — subscribes to embedding queue, calls SUE ontic wrapper, writes result.

`src/workers/inference.worker.ts` — subscribes to inference queue, assembles minimal context, calls SUE inference wrapper, writes narration node + spectral sample.

`src/db/centroid.sql` — `lucid.update_centroid_ont()` and `lucid.update_centroid_inf()` PL/pgSQL functions triggered after embedding inserts.

---

## Phase 3 — Chat interface (closes the loop)

**Goal:** A minimal web page. User types a message. Response appears. Under the hood: INSERT → live query → ONNX worker → INSERT → live query → UI update. The full feedback loop running end-to-end, verifiable by opening DevTools and watching the pglite tables.

### Data flow

```
User input
    │ HTML form submit
    ▼
INSERT lucid.conversation_turns (role='user', content='...')
    │ trg_enqueue_inference fires
    ▼
INSERT lucid.inference_queue (status='pending')
    │ inference worker live.query fires
    ▼
ONNX generation + INSERT response turn
    │ trg_enqueue_embedding fires (for new narration belief node)
    ▼
INSERT lucid.embedding_queue
    │ embed worker live.query fires
    ▼
embed + INSERT node_embeddings_ont
    │ centroid trigger fires
    ▼
UPDATE lucid.centroids
    │
    ▼
UI live.query on conversation_turns fires
    │
    ▼
Response rendered
```

### UI live queries

```typescript
// Conversation display
db.live.query(
  `SELECT role, content, created_at
   FROM lucid.conversation_turns
   ORDER BY created_at DESC
   LIMIT 50`,
  [],
  ({ rows }) => renderConversation(rows.reverse())
);

// Queue depth indicator (dev/debug)
db.live.query(
  `SELECT
     (SELECT count(*) FROM lucid.inference_queue WHERE status='pending') AS inf_pending,
     (SELECT count(*) FROM lucid.embedding_queue  WHERE status='pending') AS emb_pending`,
  [],
  ({ rows }) => renderQueueStatus(rows[0])
);

// Spectral health badge
db.live.query(
  `SELECT composite_health_score FROM lucid.spectral_monitor ORDER BY sampled_at DESC LIMIT 1`,
  [],
  ({ rows }) => renderHealthBadge(rows[0]?.composite_health_score)
);
```

### Deliverable

`src/app/` — Vite + vanilla TypeScript. No framework dependency for this phase. Single `index.html`, single `main.ts`. The objective is to verify the feedback loop, not build a product UI.

The app should expose a console handle: `window.__lucid = { db, sue }` so the loop can be poked directly from DevTools without a UI form.

---

## Phase 4 — Feedback loops in SQL

**Goal:** The loops that currently only exist as design (§10, §11, §12) run automatically as consequences of Phase 3 inserts. No new workers. New SQL only.

### Loops to activate

**AWE chain (§3):** After each narration node INSERT, a trigger computes affective valence (placeholder: sentiment heuristic on content), writes to `lucid.affective_corpus`, derives a mood token, UPDATEs `lucid.conversation_turns` with the `mood_token` field.

**Spectral health composite (§11.3):** After `lucid.write_spectral_sample()` writes a row, a trigger computes the composite health score from the stream data. A live query on `composite_health_score < THRESHOLD` fires the TripleDent Gum injection (inserts a cosine-dissimilar belief node to break DC dominance).

**Working centroid CfC update (§6.6):** After each centroid UPDATE, a PL/pgSQL function applies the closed-form CfC approximation and writes `c_w` back to `lucid.centroids`.

**Orbital health check (§6.6):** A live query on `lucid.centroids` computes whether the 32-page window encloses both `c_i` and `c_o`. Writes a boolean `orbital_health` flag. A live query on `orbital_health = false` fires the intervention tier.

### Note on complexity

Phase 4 activates loops that are self-sustaining once running. Each INSERT cascades through multiple triggers and live queries. This is the intended behaviour — it is also where things first become hard to debug. The `window.__lucid.db` console handle from Phase 3 is essential here. Instrument before activating.

### Deliverable

`src/db/loops.sql` — all Phase 4 triggers and functions. Applied as a separate migration after `schema.sql`. Can be applied/rolled back without touching the queue infrastructure.

---

## Phase 5 — Vortex / libp2p

**Goal:** A second browser tab (or a second process) participates in the belief graph. Peer centroid advertisement works. Incoming work packets from peers are accepted or forwarded based on centroid proximity.

### The key insight

The network hook attaches exactly like the chat interface. A peer message arrives via GossipSub → parsed → `INSERT lucid.belief_nodes`. That INSERT fires `trg_enqueue_embedding`. The embed worker fires. The centroid updates. The next heartbeat advertises the new position. Nothing in Phases 1–4 changes.

### libp2p initialisation

```typescript
const node = await createLibp2p({
  transports: [webRTC(), webSockets()],
  streamMuxers: [yamux()],
  connectionEncrypters: [noise()],
  services: { pubsub: gossipsub() },
});
await node.start();

// Centroid advertisement heartbeat
setInterval(async () => {
  const heartbeat = await buildHeartbeat(db);
  node.services.pubsub.publish('lucid:centroid', encode(heartbeat));
}, HEARTBEAT_INTERVAL_MS);

// Incoming message handler
node.services.pubsub.subscribe('lucid:work');
node.services.pubsub.addEventListener('message', async (evt) => {
  const packet = decode(evt.detail.data);
  if (cosineSimilarity(localCentroid, packet.queryEmbedding) > ACCEPT_THRESHOLD) {
    await db.exec(
      `INSERT INTO lucid.belief_nodes (content, source) VALUES ($1, $2)`,
      [packet.content, 'vortex:' + evt.detail.from]
    );
  } else {
    forwardToHigherScoringPeer(packet);
  }
});
```

### Deliverable

`src/network/vortex.ts` — libp2p initialisation, heartbeat, incoming message handler, forward logic.

`src/db/peer_centroids.sql` — `lucid.peer_centroids` table + update function.

Live query that watches `lucid.peer_centroids` and rebuilds the routing table in memory.

---

## Cross-cutting concerns

### SUE activation at every boot

Every phase boots via the same sequence:
```typescript
import { sue } from './sue/registry';
sue.registerOntic(nomicEmbedV15);
sue.registerInference(lfm25 /* or makeGenericOnnxWrapper(...) */);
await sue.activateOntic(db, 'nomic-embed-text-v1.5', '1.5.0');
await sue.activateInference(db, INFERENCE_MODEL_ID, INFERENCE_MODEL_VERSION);
```

The schema SQL is applied before SUE activation. SUE activation runs `setupSQL()` inside the already-migrated schema.

### Migration discipline

Each phase adds a new migration file. No phase modifies a previous phase's migration file. Migrations are applied in order at boot:

```
src/db/
  01-schema.sql        Phase 1
  02-centroid.sql      Phase 2
  03-loops.sql         Phase 4
  04-peer-centroids.sql Phase 5
```

Phase 3 adds no new migrations — the UI hooks into the schema established in Phase 1.

### Testing

Each phase has a corresponding test file that inserts synthetic data and verifies the expected live query callbacks fire. Tests run against a real in-process pglite instance (not a mock). The test environment is Node.js or Bun; the same pglite WASM binary runs in both.

---

## What this is not

This plan does not cover:
- The Thinking Cap (LoRA adapter training) — requires server-side full instance
- neurandefrag (HNSW index compaction) — deferred to full instance on sync
- ElectricSQL sync contract — follow-on after Phase 5
- ACG full implementation (§13) — the minimal context assembly in Phase 2 is a placeholder
- Dream cycle consolidation (§12) — requires cap training infrastructure
- Multi-tenancy (§21) — single-tenant throughout these phases

These are all additive. None of them require modifying what is built here.
