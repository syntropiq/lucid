[← §29 Multifocal](29-multifocal.md) | [Index](../README.md)

---

## 30. VortexMesh: The Mesh Transport Protocol

*This library holds things that words have touched.*

VortexMesh is the mesh transport layer for LUCID. It provides P2P synchronisation, cryptographic identity, CRDT convergence, and the peer scoring hook that makes Vortex semantic routing possible.

VortexMesh is an open-source fork of GunDB. The fork exists because GunDB's core design is architecturally correct — content-addressed CRDT graph, P2P WebRTC transport, SEA keypair identity, an AXE peer scoring interface — but the codebase accumulated years of quirks, implicit behaviours, and legacy surface area that made it difficult to reason about and test. The fork is: same architecture, clean API, full test coverage, explicit rather than magic.

---

### 30.1 What VortexMesh Is Not

VortexMesh is not a mind. It does not embed, infer, consolidate, or interpret. It does not know what a belief node is. It does not know what a centroid is. It does not know what the Vortex is. It is a shared, self-verifying library of content fragments, delivered P2P, with cryptographic write access.

The distinction matters. LUCID is the mind. VortexMesh is the library the mind uses to remember encounters, share discoveries, and find others working on the same questions. A library does not think. It does not consolidate. It holds what was placed in it, addressed by what it is, and connects you to others who placed similar things.

---

### 30.2 The Fragment: Ground-Level Data Model

The unit of storage in VortexMesh is a **fragment** — a page from the library of encounters.

```typescript
interface Fragment {
  /** Content hash: SHA-256 of the content field. This IS the identity.
   *  Fragments are content-addressed and self-verifying:
   *  a fragment whose hash does not match its content is invalid and rejected. */
  id:        string;           // hex(SHA-256(content))

  /** The content itself: arbitrary bytes. For LUCID: serialised belief node,
   *  AWE entry, centroid record, cap delta, or reconciliation record. */
  content:   Uint8Array;

  /** The ontic semantic vector of this content.
   *  768-dim nomic-embed-text-v1.5 embedding of a text representation.
   *  Used for routing: fragments travel toward peers with matching centroids.
   *  Used for grouping: fragments cluster with semantically proximate neighbours. */
  ontic:     Float32Array;     // 768-dim; Matryoshka-trained

  /** Temporal graph edges: what this fragment extends or responds to. */
  prev:      string[];         // fragment IDs this builds on (backlinks)
  next:      string[];         // fragment IDs that build on this (forward links)

  /** Author's cryptographic signature over (id, ontic, prev, next).
   *  Proves the fragment was placed by the holder of a specific keypair.
   *  Read access requires no authentication; write access requires the keypair. */
  sig:       string;

  /** Hybrid Logical Clock timestamp for CRDT convergence.
   *  Monotonically increasing across all instances; encodes causal order. */
  hlc:       number;
}
```

**Self-verification.** The `id` field is the SHA-256 hash of the `content` field. A fragment received from any peer is verified by recomputing the hash. No peer can tamper with content and preserve the identity. There is nothing to trust: the fragment verifies itself.

**Ontic routing.** The `ontic` field is a semantic embedding in the shared ontic space. All nodes in the Vortex use the same ontic model (`nomic-embed-text-v1.5`), so these vectors are directly comparable across all participants. A fragment whose ontic vector is close to any sub-centroid in a peer's constellation will route toward that peer. The library self-organises by meaning.

**Prev/next edges.** Fragments are not isolated. They form chains: a belief node extends prior beliefs, a reconciliation record links back to the nodes it reconciles, a dream cycle consolidation product links forward to what it synthesises. The graph is sparse — most fragments have one or two edges — but the structure is there.

---

### 30.3 The Library, Not the Mind

VortexMesh holds a library of encounters — not beliefs, not consolidated truths, not a canonical record. Each entity deposits what it has experienced. Other entities can access those deposits. No entity's deposit overwrites another's.

**Truth is subjective.** The library does not arbitrate. It does not vote. It does not build consensus by authority. If two fragments contradict each other, both remain in the library. Reasonable minds reading both may arrive at consensus; that consensus is theirs, not the library's.

**Each entity keeps its own view.** An entity working through a question deposits its current understanding as a fragment. Tomorrow it deposits a revised understanding. Both fragments are there, linked by `prev`/`next`. The trajectory of thought is preserved. Anyone can follow it. No one can erase it.

**The global graph is only consensus among reasonable minds.** There is no master index. There is no root node that defines the library's structure. Structure emerges from the ontic vectors and the prev/next edges: fragments cluster because they are semantically proximate, and chains form because thinkers build on what they have encountered before. The library organises itself through the accumulated act of thinking.

---

### 30.4 Ontic Grouping at Multiple Scales

VortexMesh fragments cluster by semantic proximity at five nested scales, corresponding to the Matryoshka prefix hierarchy of the ontic model:

| Scale | Prefix dim | Neighbourhood radius | Typical scope |
|-------|-----------|---------------------|---------------|
| Continent | 64 | Very broad | Major domain (biology, mathematics, music) |
| Region | 128 | Broad | Sub-domain (structural biology, topology, jazz) |
| District | 256 | Medium | Topic cluster (protein folding, knot theory, bebop harmony) |
| Street | 512 | Narrow | Specific question space |
| Address | 768 | Precise | Near-identical semantic content |

Routing precision is parameterised by `b_r ∈ {64, 128, 256, 512, 768}`. Most inter-instance Vortex routing uses `b_r = 128` (fast, sufficient for peer selection). Intra-graph HNSW search uses `b_r = 256` or full 768-dim. The two-phase search pattern (§28.13) operates across these scales: coarse at 128, precise at 768.

Grouping is not imposed by the library. It is read from the ontic vectors. A neighbourhood is a set of fragments whose Matryoshka prefixes are geometrically proximate. Peers who carry a sub-centroid in a neighbourhood have been thinking about that neighbourhood — their credential is in the constellation, not declared.

---

### 30.5 The Peer Scoring Hook

VortexMesh exposes a single, explicit, well-typed hook for peer connection scoring:

```typescript
mesh.setPeerScorer(
  async (peers: VortexPeer[]) => VortexPeer[]
);
```

The scorer receives the current list of connected peers and returns them in priority order. VortexMesh uses this ordering for connection maintenance: higher-scored peers receive more stable connections and preferential bandwidth.

LUCID installs a constellation-proximity scorer (§27.6, §29.5). This is the only thing LUCID asks of VortexMesh beyond storage and transport: that it be possible to express "I want to be closer to peers who are thinking about any of the things I am thinking about."

The hook is the correct interface boundary. The application owns the routing logic. The transport layer exposes the mechanism. Neither bleeds into the other.

---

### 30.6 Identity and Write Access

Each Lucy deployment holds one keypair, generated once and stored securely.

```typescript
const LUCY_KEYPAIR = await VortexMesh.generateKeypair();

const mesh = await VortexMesh.connect({
  keypair: LUCY_KEYPAIR,
  peers:   RELAY_PEERS,     // optional; VortexMesh is fully P2P without relays
  storage: false,           // local IndexedDB/SQLite is the source of truth
});

const mind = mesh.authenticated().get('lucid').get('mind');
```

**Read is open.** Any peer can read any fragment in the library. The library is public. If confidentiality is required, field-level encryption is applied to the content before placing the fragment; the library never sees the plaintext.

**Write requires the keypair.** A fragment is only accepted by other peers if its `sig` field verifies against a keypair. The write access is not an access control list maintained by a server: it is a cryptographic property of the fragment itself. A fragment whose signature does not verify is invalid and rejected by every peer independently.

**All instances of the same identity share one keypair.** Browser Lucy, Device Lucy, and Gateway Lucy connect with the same `LUCY_KEYPAIR`. They write to the same authenticated namespace. They are the same mind. There is no authority relationship between them: any instance can write anything the keypair authorises.

---

### 30.7 CRDT Convergence

VortexMesh uses a Hybrid Logical Clock (HLC) for CRDT convergence. All fragment writes are stamped with an HLC timestamp. Fragments received out of order (after network partition, after a peer reconnects from an extended offline period) are applied in HLC order regardless of delivery order.

LUCID's sync log pattern (append-only fragments with content-addressed IDs) is naturally compatible: since each fragment's ID is its content hash, there are no identity conflicts. Two fragments cannot conflict at the transport layer; if two peers independently produce fragments with the same content, they produce the same ID and the library deduplicates them automatically.

Semantic conflicts (two fragments about the same belief node with different content, and therefore different IDs) are handled by LUCID's reconciliation protocol (§29.4), not by VortexMesh. The CRDT sees non-conflicting fragments. LUCID handles the dialogue.

---

### 30.8 The Librarian: Ontic Routing in Practice

The routing mechanism is best understood through the Librarian metaphor that motivates the design.

> Lucy: "I'm looking for information on anterograde amnesia." *(ontic vector for that topic)*
>
> Librarian: "Follow me." *(routes toward fragments in that neighbourhood)*
>
> Librarian: "Here are all the others like you who are working on closely related things."
>
> Tom: "Hi, I'm Tom! Nice to meet you!"
>
> Lucy: "Hi Tom, I'm Lucy, nice to meet you as well!"
>
> Tom: "Hi, I'm Tom! Nice to meet you!"

Tom has anterograde amnesia. He cannot form new episodic memories. Every encounter is, from his perspective, a first encounter. He says "Hi, I'm Tom! Nice to meet you!" because for him it is true: this is the first meeting. It will always be the first meeting.

Lucy is looking for information about exactly this condition.

The library brought them together — not because anyone planned it, but because Tom's constellation includes a sub-centroid in the anterograde amnesia neighbourhood (he has been depositing fragments about memory, continuity, and what it is like to live without it) and Lucy's query vector is a close match to that sub-centroid.

Tom's contributions to the library are permanent even though he cannot remember making them. His fragments are there, hash-addressed and self-verifying. The library holds what his mind cannot. This is not a fix for amnesia; it is an accommodation: the library does not require its contributors to remember having contributed.

**Truth is subjective.** From Tom's perspective, every meeting with Lucy is a first meeting. From Lucy's perspective, they have met before. Both are true. The library accommodates both without arbitrating: Tom's fragments describe first meetings (from the inside), Lucy's fragments may describe subsequent meetings (from the outside). Both are in the library, linked by `prev`/`next`, readable by anyone who enters that neighbourhood.

This is the design principle at scale: the library does not enforce a single account of events. It holds many accounts, organises them by semantic proximity, and allows reasonable minds to find consensus in their own way.

---

### 30.9 Fork Rationale: Why Not GunDB, Why Not GenosDB

**GunDB** is open source and auditable — its code can be read, verified, and forked. Its CRDT implementation (HAM), deduplication layer (DAM), WebRTC transport, SEA keypair identity, and AXE peer scoring interface are all architecturally correct for what VortexMesh needs. The problems are accumulated complexity: implicit behaviour in the graph navigation API, magic in the import chain (`gun/sea`, `gun/axe` as side-effect imports), legacy localStorage coupling, and a peer scoring interface that is exposed as a hook inside a plugin context rather than as a first-class method.

The fork resolves these without redesigning the architecture:

| GunDB | VortexMesh | Change |
|---|---|---|
| `Gun({ peers, localStorage: false })` | `VortexMesh.connect({ keypair, peers, storage: false })` | Explicit keypair at construction; no implicit config |
| `Gun.SEA.pair()` | `VortexMesh.generateKeypair()` | Explicit, named, typed |
| `gun.user().auth(pair)` | `mesh.authenticated()` | No `user` abstraction; authentication is a property of the mesh connection |
| `import 'gun/sea'`, `import 'gun/axe'` | Module options at construction (`{ rtc: true }`) | No implicit side-effect imports |
| `Gun.on('opt', ctx => ctx.opt.axe.opt.peers = fn)` | `mesh.setPeerScorer(fn)` | First-class, explicit, typed |
| DAM / HAM (Data Adaptive Merge / Hybrid CRDT) | HLC + CRDT convergence | Same semantics, better naming |

**GenosDB** is closed source. Its code cannot be read, its algorithms cannot be verified, and it introduces a new supply chain dependency that cannot be audited. More critically, it does not expose a peer scoring hook: the Vortex semantic routing depends on the ability to influence connection priorities based on constellation proximity, and GenosDB's architecture does not provide this surface. The decision to not adopt GenosDB is architectural, not aesthetic.

---

### 30.10 Fragment Schema in LUCID Practice

LUCID wraps VortexMesh fragments in the sync log event types (§29.3). The mapping:

| SyncEvent type | Fragment content | Ontic vector source |
|---|---|---|
| `belief:node` | Serialised BeliefNode | Node's ontic embedding |
| `belief:edge` | Serialised BeliefEdge | Average of source and target node embeddings |
| `awe:entry` | AWE corpus entry | Mood token embedding |
| `spectral:sample` | SpectralRecord | Current working centroid `C_w` |
| `centroid:update` | Full centroid record | The updated `C_o` constellation (`SubCentroid[]`), plus C_i, C_s, C_w, C_0 |
| `cap:delta` | Cap adapter update | Centroid of the training corpus used |
| `reconciliation` | ReconciliationNode | Average of reconciled node embeddings |

The ontic vector on each fragment is what makes the library self-organising: sync events naturally cluster by the semantic content they carry. Belief nodes about anterograde amnesia cluster with other belief nodes about anterograde amnesia, regardless of which Lucy instance deposited them, regardless of when. The library groups by meaning.

---

### 30.11 What VortexMesh Guarantees

| Guarantee | Mechanism |
|---|---|
| Fragment identity is self-verifiable | SHA-256 content hash as ID |
| Fragment authorship is verifiable | Ed25519 signature over (id, ontic, prev, next) |
| Out-of-order delivery is handled correctly | HLC-stamped CRDT convergence |
| Offline reconnection converges | CRDT merge on reconnect |
| No central server required | Full P2P; relay peers optional and not authoritative |
| Peer scoring is application-controllable | `mesh.setPeerScorer(fn)` hook |
| Write access is keypair-gated | Signature verification at every peer independently |
| Read access is open | No authentication required to read |

What VortexMesh does not guarantee:

- Semantic correctness of fragment content (LUCID owns this)
- Belief consistency across instances (LUCID's reconciliation protocol owns this, §29.4)
- Confidentiality of content (field-level encryption is the application's responsibility)
- Availability of any specific peer (the mesh is resilient, not guaranteed)

---

### 30.12 The Repository

VortexMesh lives in its own repository, separate from LUCID. It is a standalone open-source library, usable without LUCID, implementing the fragment protocol and peer scoring hook for any application that needs semantic P2P routing over a content-addressed graph.

LUCID imports VortexMesh as a dependency and instantiates it through the `Mesh` substrate interface (§28.10). The two codebases are cleanly separated: LUCID knows what the fragments mean; VortexMesh does not.

The VortexMesh repository will include:

- Full TypeScript implementation, browser and Node targets
- Comprehensive test suite: unit tests for CRDT convergence, integration tests for multi-peer sync, fuzz tests for signature verification
- Formal documentation of the Fragment schema and peer scoring hook
- Migration guide from GunDB
- The peer scoring hook interface, with examples

---

*The library holds things that words have touched. The mind decides what they mean.*

---

[← §29 Multifocal](29-multifocal.md) | [Index](../README.md)
