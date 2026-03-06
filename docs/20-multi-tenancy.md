[← §19 Security Model](19-security-model.md) | [Index](../README.md) | [§21 Shared Content Layer →](21-shared-content-layer.md)

---

## 20. Multi-Tenancy

### 20.1 Tenant Isolation Guarantees

Each tenant corresponds to a PostgreSQL role (`lucid_tenant_{id}`) and a row in `lucid.tenants`. Row-level security policies on all per-tenant tables enforce that `tenant_id = lucid.get_current_tenant_id()`. SECURITY DEFINER functions that access per-tenant data bind the tenant ID in their first line and use it in all subsequent WHERE clauses.

AWE tables are per-tenant. `lucid.affective_corpus`, `lucid.awe_walk_log`, and `lucid.spectral_monitor` carry `tenant_id` columns and are subject to the same RLS policies as all other per-tenant tables.

### 20.2 Eviction and Cross-Tenant Integrity

Eviction is the removal of a tenant instance — setting `is_active = false` in `lucid.tenants`, revoking the `lucid_tenant_{id}` role, and optionally purging per-tenant rows. Because tenant ID is embedded in every per-tenant row and enforced by both RLS and function-level binding, eviction is clean: there is no shared mutable state between tenants that requires coordination. AWE and spectral monitor tables are purged as part of standard eviction.

Cross-tenant tampering is structurally prevented: a `lucid_tenant_a` role cannot call any function and retrieve `lucid_tenant_b` rows, because every function that touches per-tenant data calls `get_current_tenant_id()` and binds the result.

### 20.3 Per-Tenant Belief Graph Substrate

Each tenant has a named belief graph substrate recorded in `lucid.tenants.belief_graph_name`. The belief graph substrate is a logical boundary implemented through RLS and tenant-scoped HNSW partitions — not a separate physical database or extension instance. NeuronAgent's tour engine identifies the current tenant's nodes by `tenant_id` binding, not by graph name lookup; the `belief_graph_name` field is metadata for operator tooling and audit logging.

---


---

[← §19 Security Model](19-security-model.md) | [Index](../README.md) | [§21 Shared Content Layer →](21-shared-content-layer.md)
