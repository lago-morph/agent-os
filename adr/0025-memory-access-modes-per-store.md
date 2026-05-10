# ADR 0025: Memory access modes declared per memory store

## Status
Accepted

## Context
Platform Agents need memory that ranges from strictly private (per-conversation scratch state) through tenant-shared (project context any agent in a namespace may read) to selectively shared across namespaces (curated knowledge fragments, cross-team agent memory). Letta (ADR 0005) provides the underlying memory service, with Postgres as the system of record and OpenSearch as a retrieval optimization (ADR 0014). Tenancy is namespaced (ADR 0016), and all access decisions compose under the RBAC-as-floor / OPA-as-restrictor rule (ADR 0018).

Without an explicit access model on the memory primitive itself, the platform would either have to (a) attach access rules per Agent, multiplying the surface every time an agent is added or moved, or (b) push every memory read through full OPA evaluation, paying per-decision policy cost on the hot path of agent reasoning. Neither matches how memory is actually used: the access shape almost always belongs to the store (this scratchpad is per-agent; that project memory is shared by the tenant; that curated knowledge is governed), not to the agent reading it.

Architecture overview §6.3 fixes the model — three modes, declared per `MemoryStore`, with Agent CRDs referencing memory stores by name and the platform deciding what is allowed from the store's declared mode. This ADR formalizes that model and the reasons for it.

## Decision
Each memory store declares exactly one of three access modes as part of the `MemoryStore` CRD spec:

1. **Private to the agent.** Only the Platform Agent that wrote the entry may read it. The default for transient or per-conversation state. Enforced at the Letta adapter by the writer's agent identity; no cross-agent read path exists for this mode.
2. **Shared by namespace.** Any Platform Agent in the same Kubernetes namespace may read entries. Used for tenant-shared context and common project state. The namespace boundary (ADR 0016) is the access boundary; no further policy is required on the read path.
3. **RBAC/OPA-controlled.** Visibility and read/write are governed by Kubernetes RBAC plus OPA. RBAC grants the underlying right to address the store; OPA may further restrict per request — by tenant pairing, time window, request shape, approval state, or capability binding (ADR 0018). Used for cross-namespace shared knowledge or selectively-shared agent memory.

Agent CRDs reference memory stores by name. The store's declared mode determines what the platform allows; the agent does not (and cannot) override it. The memory adapter (B11) and the Platform SDK's `memory.*` surface (B6) enforce the mode at the boundary, before reaching Letta. Mode is immutable after creation in the sense that changing modes is a deliberate, audit-emitting reconciliation — not a runtime flag.

`MemoryStore` provisioning is a Crossplane Composition (B4), so mode is declared in Git alongside the store, reviewed like any other config, and visible to every reviewer at the same place.

## Consequences
Positive:
- The common cases (private, namespace-shared) cost nothing on the read path beyond the namespace and identity check the platform already performs. Per-decision OPA evaluation is reserved for the cases that need it.
- The access shape lives on the store, where reviewers expect it. Adding an agent does not require touching memory access policy.
- The third mode keeps full RBAC + OPA composition available for the cases that genuinely need it, without forcing every store through that machinery.
- Mode is inspectable from the CRD alone; auditors do not have to cross-reference Agent specs to know who can read what.

Negative:
- Three modes is a coarse vocabulary. Some intermediate patterns (e.g., "shared with two named namespaces") fall into the RBAC/OPA-controlled bucket and require explicit policy authoring. This is intentional — the alternative is mode proliferation.
- Changing a store's mode is a meaningful operation and must be reconciled deliberately, not flipped at runtime. Tooling and audit must make this explicit.

## Alternatives considered
- **Per-agent ACLs only.** Rejected — the access surface scales with agent count rather than store count, and the same access shape gets re-encoded on every agent that touches the store. Per-store mode covers the overwhelming majority of needs with a fraction of the surface.
- **Fully OPA-driven for every memory access.** Rejected — pushes per-decision policy logic onto the hot path for the common case (private and namespace-shared memory) where the access rule is structural, not contextual. OPA is the right tool for the third mode, not the first two.
- **No declared mode; infer from labels or naming convention.** Rejected — implicit access policy is exactly what the architecture-backlog §6 invariants are meant to prevent.

## Related
- Architecture overview §6.3 (Memory and data architecture — Memory access modes).
- Architecture backlog §6 (invariant: memory access modes are per-store: private, namespace-shared, or RBAC/OPA-controlled), §7 (this ADR is candidate 25).
- ADR 0005 (Letta as the memory backend — the service whose access this ADR governs).
- ADR 0014 (Postgres primary, OpenSearch as retrieval — the storage layout `MemoryStore` provisions over).
- ADR 0016 (namespace tenancy — the boundary the namespace-shared mode uses).
- ADR 0018 (RBAC-as-floor / OPA-as-restrictor — the composition rule the third mode follows).
