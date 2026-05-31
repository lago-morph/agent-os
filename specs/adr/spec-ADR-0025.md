# SPEC ADR-0025 — Memory access modes per memory store [PROPOSED]

> kind: ADR · workstream: — · tier: T1
> upstream: [A10, A7, B4] · downstream: [B11, A18, A5] · adrs: [0025] · views: [6.3]
> canon-glossary: cf2d1a754a58 · canon-interface: 45ee7b798c47

## 1. Purpose & Problem Statement

ADR 0025 is a settled decision: each `MemoryStore` declares exactly one of three access modes in its CRD spec — **private to the agent**, **shared by namespace**, or **RBAC/OPA-controlled** — and the referenced store's declared mode determines what the platform allows. There is no platform-wide override that changes a store's mode after declaration. This SPEC states what honoring that contract obliges: the `accessMode` field is required and constrained to the three values, Gatekeeper admission rejects missing/invalid modes, private and namespace-shared modes are enforced structurally (no per-request policy), the RBAC/OPA mode follows ADR 0018, and audit attributes reads/writes to agent identity plus declared mode. It does not re-argue why access mode is a per-store property.

The problem the decision solves: agents need memory spanning strictly-private per-conversation state, namespace-shared context, and selectively-shared cross-namespace knowledge under policy. A single platform-wide setting cannot serve all three; making access mode a property of the store (not platform, not agent) keeps authorization local to where the data lives and aligns with the namespace-as-tenant boundary (ADR 0016).

## 2. Scope

### 2.1 In scope
- The obligation that every `MemoryStore` carries exactly one `accessMode` of the three permitted values.
- The Gatekeeper admission obligation rejecting `MemoryStore` resources with missing or out-of-range `accessMode`.
- The structural-enforcement obligation: private (writer identity) and namespace-shared (namespace membership) are checked without per-request OPA evaluation.
- The RBAC/OPA-controlled mode obligation: follows ADR 0018 (RBAC floor, OPA restricts per decision, OPA never grants beyond RBAC).
- The obligation that backend choice (Letta-backed service or Crossplane-composed `MemoryStore` XR) does not change access semantics.
- The audit obligation: memory reads/writes attribute to agent identity and the store's declared mode.

### 2.2 Out of scope (and where it lives instead)
- Letta memory backend install — component **A10** SPEC.
- Memory backend adapter — component **B11** SPEC.
- `MemoryStore` XR composition — component **B4** (Crossplane) SPEC.
- Gatekeeper admission policy authoring — component **A7** / OPA policy library (**B3**/**B16**).
- Memory namespace/sharing semantics — naming, lifecycle, cross-store joins, eviction — deferred (backlog §4).
- The platform authorization model itself — **ADR 0018**; namespace tenancy — **ADR 0016**.

## 3. Context & Dependencies

Upstream consumed: **A10** (Letta) and **B4** (Crossplane-composed `MemoryStore` XR) are the two backends that must honor the declared mode; **A7** (OPA/Gatekeeper) provides admission rejection and the RBAC/OPA-mode decision point.
Downstream consumers: **B11** (memory backend adapter) surfaces the mode to agents; **A18** (audit) records reads/writes with mode attribution; **A5** (ARK) reconciles `Agent` CRDs that reference stores by name via `memoryRefs[]`/`Memory`.

ADR decisions honored:
- **ADR 0025** (this) — three access modes per store; no post-declaration override; Gatekeeper enforces validity.
- **ADR 0005** — the Letta-backed memory service honors the declared mode.
- **ADR 0016** — namespace isolation is the default posture; cross-namespace sharing is opt-in via the RBAC/OPA mode.
- **ADR 0018** — RBAC-floor / OPA-restrictor governs the RBAC/OPA-controlled mode.
- **ADR 0002** — Gatekeeper admission rejects invalid `accessMode`.
- **ADR 0027** — audit attribution by agent identity + mode enables later detection of unintended access excess (the v1.0 threat-model focus).
- **ADR 0030** — the `MemoryStore` XR/CRD `accessMode` field versions per the versioning policy.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs
- `MemoryStore` (XR, Crossplane B4, namespaced) — source-stated fields `accessMode` (private / namespace-shared / RBAC-OPA), `backendType`. The `accessMode` enum is the load-bearing contract; exactly one of the three values is required.
- `Memory` (ARK A5, namespaced) — source-stated field `memoryStoreRef`. Access mode lives on `MemoryStore`, NOT on `Memory` (interface-contract §1.2). Agent CRDs reference stores by name; the referenced store's declared mode governs.

### 4.2 APIs / SDK surfaces
- `memory.*` (Platform SDK, owner B6) — the surface agents use to read/write memory; the platform enforces the referenced store's mode behind it. Method signatures beyond the named surface group are not specified in source — `[PROPOSED — not in source]` if detailed.

### 4.3 CloudEvents emitted / consumed
- Memory-store lifecycle (created/.../deleted) rides `platform.lifecycle.*` (ADR 0031, §2). Memory read/write audit rides `platform.audit.*` via the audit adapter (interface-contract §5). Per-event-type names deferred to B12; `[PROPOSED — not in source]` any concrete name.

### 4.4 Data schemas / connection-secret contracts
- The `MemoryStore` XR composes a substrate backend; any composed Postgres/store writes the uniform connection-secret shape (`host`, `port`, `user`, `password`, `dbname`) per ADR 0044. The connection secret carries no access-mode field — mode is enforced at the platform/SDK plane, not the backend wire.

## 5. OSS-vs-Custom Decision
N/A — ADR. (Enforcement note: backends are upstream **Letta** (A10) and Crossplane-composed XRs (B4); the access-mode contract is enforced by the platform/SDK plane plus a Gatekeeper admission policy — config/wrap, no fork.)

## 6. Functional Requirements
- REQ-ADR-0025-01: Every `MemoryStore` MUST declare exactly one `accessMode` ∈ {private, namespace-shared, RBAC/OPA-controlled}.
- REQ-ADR-0025-02: Gatekeeper admission MUST reject any `MemoryStore` whose `accessMode` is missing or outside the three permitted values.
- REQ-ADR-0025-03: There MUST be no platform-wide override that changes a store's `accessMode` after declaration.
- REQ-ADR-0025-04: Private mode MUST permit reads only by the writing Platform Agent; namespace-shared mode MUST permit reads by any Platform Agent in the same namespace — both enforced structurally without per-request OPA evaluation.
- REQ-ADR-0025-05: RBAC/OPA-controlled mode MUST follow ADR 0018 — RBAC grants the floor, OPA may restrict per decision, OPA MUST NOT grant access RBAC did not already permit.
- REQ-ADR-0025-06: The Letta-backed service AND the Crossplane-composed `MemoryStore` XR MUST both honor the declared mode; backend choice MUST NOT change the access semantics surfaced to agents.
- REQ-ADR-0025-07: Agent CRDs MUST reference memory stores by name (`memoryRefs[]`/`Memory.memoryStoreRef`); the referenced store's declared mode MUST determine what the platform allows.
- REQ-ADR-0025-08: Memory reads and writes MUST emit audit events attributing the event to the agent identity and the store's declared mode.

## 7. Non-Functional Requirements
- Security/multi-tenancy (§6.9): namespace isolation is the default (ADR 0016); cross-namespace sharing requires the RBAC/OPA mode, not a separate flag.
- Performance: private and namespace-shared modes are cheap structural checks (no policy round-trip).
- Observability (§6.5): mode-attributed audit enables later detection of unintended access excess (ADR 0027 threat focus).
- Versioning (ADR 0030): the `MemoryStore` `accessMode` field versions through conversion webhooks on any breaking change.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR (verification map lives in the PLAN). The §14.1 set applies to enforcing components (A10, B11, B4, A7, A18), not to this decision record.

## 9. Acceptance Criteria
- AC-ADR-0025-01: Honored when a `MemoryStore` with a valid `accessMode` admits and one with missing/invalid `accessMode` is rejected by Gatekeeper. (REQ-01/02)
- AC-ADR-0025-02: Honored when no API path or controller can mutate an existing store's `accessMode` via a platform-wide override. (REQ-03)
- AC-ADR-0025-03: Honored when a private store denies reads from a second agent and a namespace-shared store permits reads from a same-namespace agent, with no OPA call in either path. (REQ-04)
- AC-ADR-0025-04: Honored when an RBAC/OPA store grants a read only where RBAC permits AND OPA does not deny, and OPA cannot grant beyond RBAC. (REQ-05)
- AC-ADR-0025-05: Honored when the same access-mode test suite passes against both the Letta-backed store and the Crossplane-composed XR. (REQ-06)
- AC-ADR-0025-06: Honored when an `Agent` referencing a store by name is allowed/denied per the referenced store's declared mode. (REQ-07)
- AC-ADR-0025-07: Honored when a memory read and a memory write each produce an audit record naming the agent identity and the store's declared mode. (REQ-08)

## 10. Risks & Open Questions
- OQ-1 (med): Memory namespace/sharing semantics (naming, lifecycle, cross-store joins, eviction) are deferred (backlog §4); ACs depending on those are partial until that design lands. `[PROPOSED]`
- R-1 (low): The RBAC/OPA mode's correctness depends on ADR 0018 enforcement being intact; if OPA is misconfigured to grant beyond RBAC, the invariant breaks — mitigated by ADR 0018's own "OPA never grants" guarantee being separately tested.
- R-2 (low): Structural enforcement of namespace-shared mode assumes namespace = tenant boundary (ADR 0016); a future cross-namespace use case would force the RBAC/OPA mode, not a relaxation of namespace-shared.

## 11. References
- ADR 0025 (`adr/0025-memory-access-modes-per-store.md`) — the decision enforced here.
- architecture-overview.md §6.3 (memory and data architecture).
- architecture-backlog.md §4 (memory namespace and sharing model details), §6.
- Enforcing/related components: A10 (Letta), B11 (memory adapter), B4 (Crossplane `MemoryStore` XR), A7 (OPA/Gatekeeper), A18 (audit), A5 (ARK).
- ADR 0005 (Letta), 0016 (multi-tenancy), 0018 (RBAC-floor/OPA-restrictor), 0002 (Gatekeeper), 0027 (threat model), 0030 (versioning), 0044 (connection secret).
