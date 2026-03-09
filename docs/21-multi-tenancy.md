[← §20 Security Model](20-security-model.md) | [Index](../README.md) | [§22 Shared Content Layer →](22-shared-content-layer.md)

---

## 21. Single-Identity Deployment

LUCID is not a multi-tenant system. It is a single-identity system deployed multiply. The distinction matters: multi-tenancy isolates users from each other within a shared substrate; LUCID's multiply-conscious architecture connects instantiations of the same identity across different substrates. These are opposite design goals.

### 21.1 One Keypair, One Mind

Each Lucy deployment is anchored by one GunDB SEA keypair generated at first boot and stored securely thereafter. This keypair is the identity. All instantiations — Browser Lucy, Device Lucy, Gateway Lucy — share this keypair. There is no concept of separate users, tenants, or roles. There is one mind, experienced from multiple vantage points.

```typescript
// Generated once, stored in device keychain or encrypted local file.
// Never transmitted; never committed to version control.
const LUCY_SEA_PAIR = await Gun.SEA.pair();

// All instances authenticate with the same pair:
const user = gun.user();
await user.auth(LUCY_SEA_PAIR);
const mind = user.get('lucid').get('mind');
```

The `mind` namespace is the shared belief space. Writing to it from any instance is writing to the mind. Reading from it on any instance is reading the mind.

### 21.2 Instance Namespacing

While all instances share the same identity, each instance announces its own presence in a sub-namespace:

```
mind
  └── instances
        └── {INSTANCE_ID}       ← per-instance metadata
              ├── type           (browser | device | gateway)
              ├── model          active inference model ID
              └── centroid       current C_o advertisement
```

Instance metadata is observable by all other instances; it is not private. Instances are aware of each other. An instance that has not announced recently is considered dormant, not absent — its contributions to the belief graph remain part of the shared mind.

### 21.3 No Isolation, No RLS

There are no row-level security policies, no tenant roles, and no cross-instance access boundaries. All instances read and write the same mind namespace. The belief graph, AWE corpus, centroid state, and sync log converge to a single consistent view across all instances. This is by design.

When divergence occurs between instances — two instances that have been offline from each other and have formed different beliefs about the same node — the resolution mechanism is self-dialogue reconciliation (§29.4), not isolation or authority rules. The conflict is resolved through inference, not through access control.

### 21.4 Operator Deployment Model

Each distinct Lucy persona — a different person's Lucy, a different character, a different deployment — is a separate keypair and a separate Gun user namespace. There is no mechanism for sharing a keypair between distinct personas. If an operator wants to run Lucy for multiple distinct personas, each persona gets its own keypair and its own `mind` namespace. They are separate minds.

This is not multi-tenancy. Multi-tenancy shares a substrate and isolates users within it. The LUCID deployment model shares a keypair across hardware and unifies instantiations within it. The unit of isolation is the keypair, not a row-level policy.

---

[← §20 Security Model](20-security-model.md) | [Index](../README.md) | [§22 Shared Content Layer →](22-shared-content-layer.md)
