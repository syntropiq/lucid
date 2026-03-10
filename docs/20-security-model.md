[← §19 Failure Mode Detection and Response](19-failure-mode-detection-and-response.md) | [Index](../README.md) | [§21 Single-Identity Deployment →](21-multi-tenancy.md)

---

## 20. Security Model

The security model is built on two foundations: **SEA identity** for authentication and access control across instances, and **the threat architecture** (§9) for belief-level integrity. There is no server database to protect, no role hierarchy, and no tenant isolation machinery: the system is local-first and single-identity by design.

### 20.1 SEA Identity and Access Control

GunDB's SEA (Security, Encryption, Authorization) module provides the cryptographic identity layer. The SEA keypair is the only credential in the system.

```typescript
// The keypair grants write access to the Lucy user namespace.
// Holders of the private key can write to mind.get('lucid').get('mind').
// Non-holders can read (Gun is append-only and public by default)
// but cannot write authenticated data under this user identity.
const user = gun.user();
await user.auth(LUCY_SEA_PAIR);   // proves knowledge of private key
```

**What the keypair protects:**
- Write access to the shared belief namespace: only authenticated instances can write belief events
- Instance identity: Gun's SEA challenge-response prevents impersonation of a Lucy instance on the mesh
- Sync log integrity: events written under the authenticated user namespace are signed; unsigned events are rejected

**What the keypair does not protect:**
- Read access: the Gun namespace is readable without authentication. If confidentiality of belief content is required, field-level SEA encryption can be applied to individual node payloads before writing. This is an operator configuration choice.
- Local storage: IndexedDB and SQLite are protected by the OS and browser origin isolation, not by SEA.

### 20.2 Keypair Storage

The keypair is generated once at first boot and must be stored securely. Storage options in order of preference:

1. **Device Lucy**: OS keychain (macOS Keychain, Linux Secret Service, Windows Credential Manager)
2. **Browser Lucy**: Encrypted IndexedDB entry, key derived from a user-supplied passphrase using PBKDF2
3. **Operator-managed**: Provided via environment variable or secure secret injection at boot: never hardcoded, never committed to version control

The keypair is never logged, never transmitted in plaintext, and never committed to source control. Loss of the keypair means loss of write access to the shared mind namespace; other instances that held the keypair continue to function and will resync when the keypair is recovered or reissued.

### 20.3 No Direct Graph Access

The belief graph lives in local IndexedDB (browser) or SQLite (device). Core feedback loops interact with the graph exclusively through `sue.graph()` (the GraphStore interface) and `sue.ont()` / `sue.inf()` (the VectorStore interfaces). These are typed TypeScript interfaces: there is no SQL injection surface, no raw query API, and no way for a model output or injected prompt to issue arbitrary storage operations.

A prompt injection that achieves JavaScript code execution in the Worker context could in principle call `sue.graph().nodeUpsert()` directly. This is why the threat architecture (§9) treats high injection-signal nodes as holding actions rather than write actions, and why provenance weight limits the influence of any single write. The graph is protected by the mathematical structure of provenance, Hebbian weighting, and the dual tour.

### 20.4 Gateway Lucy Security

Gateway Lucy (§29.8) adds a cost-control and anti-thrash layer at the provider boundary:

- **Session budget enforcement**: hard stops on per-session token spend and CfC displacement
- **Thrash detection**: repeated KNN neighbourhood revisitation without epistemic gain halts forwarding
- **Belief graph as cache**: forwarded calls that return novel content write to the belief graph, reducing future forwarding; this is the primary cost control mechanism over time

Gateway Lucy does not authenticate end users. It authenticates as Lucy (via SEA keypair) on the Gun mesh and authenticates to the upstream provider via the provider's standard API key mechanism. End-user authentication, if required, is the responsibility of the layer in front of Gateway Lucy.

---

[← §19 Failure Mode Detection and Response](19-failure-mode-detection-and-response.md) | [Index](../README.md) | [§21 Single-Identity Deployment →](21-multi-tenancy.md)
