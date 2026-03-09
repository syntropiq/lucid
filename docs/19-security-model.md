[← §18 Failure Mode Detection and Response](18-failure-mode-detection-and-response.md) | [Index](../README.md) | [§20 Multi-Tenancy →](20-multi-tenancy.md)

---

## 19. Security Model

The security model uses a layered SECURITY DEFINER function API, row-level security, and a role hierarchy that prevents any Lucy instance from directly reading or writing database tables. AWE tables (`lucid.affective_corpus`, `lucid.awe_walk_log`, `lucid.spectral_monitor`) are subject to the same tenant isolation guarantees as all other per-tenant tables. No cross-tenant affective or spectral data is accessible.

### 19.1 Role Hierarchy

| Role | Purpose | Access |
|------|---------|--------|
| `lucid_definer` | Owns all `lucid.*` tables and functions. Never used directly. SECURITY DEFINER functions run as this role. | Full ownership |
| `lucid_operator` | Read access to monitoring views across all tenants. No write access. For hosted deployment administration. | Monitoring views only |
| `lucid_tenant_{id}` | One per tenant. EXECUTE on `lucid.*` functions only. No direct table access whatsoever. | Functions only |
| `lucid_shared_curator` | EXECUTE on shared layer ingest functions only. Used by the verification worker. | Shared layer functions only |

### 19.2 Tenant Identification

Tenant identification uses `current_user` rather than a session variable. Session variables can be set by any connection. `current_user` reflects the PostgreSQL role actually authenticated to the connection and cannot be spoofed within the session. The connection pooler's responsibility is to assign the correct `lucid_tenant_{id}` role before handing the connection to NeuronAgent. `lucid.get_current_tenant_id()` reads `current_user`, verifies it against `lucid.tenants`, and raises an exception if the role is not a registered active tenant. This call is the first substantive line of every SECURITY DEFINER function that touches per-tenant data.

### 19.3 Why Lucy Has No Direct Table Access

A compromised NeuronAgent process or a prompt injection that achieves code execution cannot directly query belief tables, read another tenant's data, or modify the inheritance corpus or affective corpus outside the function API. The function API enforces provenance, tenant isolation, and audit logging as invariants. Direct table access would bypass all three.

---


---

[← §18 Failure Mode Detection and Response](18-failure-mode-detection-and-response.md) | [Index](../README.md) | [§20 Multi-Tenancy →](20-multi-tenancy.md)
