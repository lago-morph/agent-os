# SPEC B20 — Persistent volume access for Platform Agents

> kind: COMPONENT · workstream: B · tier: T1
> upstream: [A6, A7, A5] · downstream: [] · adrs: [0016, 0018, 0002, 0003, 0030, 0025] · views: [6.3, 6.9, 6.2]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

Some Platform Agents need read access to large pre-existing datasets — reference corpora, shared
document sets, fixture data — that are impractical to deliver through the RAG / memory call path
and are better surfaced as a mounted filesystem. §6.3 already contemplates a filesystem-mount
access pattern for RAG-style structured grep ("structured grep via mount"), and §6.4 notes the
SDK-API-vs-mount choice is per-agent and use-pattern-driven. B20 supplies the missing piece: a
**declarative way to map pre-defined Kubernetes PersistentVolumes into Platform Agents**, with the
namespace, RBAC, and OPA controls that the platform requires of every capability.

The problem B20 solves is that ad-hoc PV mounting bypasses the platform's tenancy and policy
perimeter. A Platform Agent runs in a `Sandbox` (agent-sandbox, A6) with a deliberately small
attack surface; mounting host or cluster storage into it is exactly the kind of access that must
be governed by the §6.9 namespace boundary, the RBAC-as-floor / OPA-as-restrictor model (ADR
0018), and Gatekeeper admission (ADR 0002). B20 makes PV access a first-class, declarative,
policy-gated capability rather than an out-of-band sandbox edit.

## 2. Scope

### 2.1 In scope
- A **declarative mapping** from pre-defined Kubernetes PVs (reference datasets, shared corpora)
  into Platform Agents' Sandboxes (§14.2 B20).
- Namespace-scoped binding: a PV mapping is valid only within an approved namespace and respects
  the §6.9 tenancy boundary.
- RBAC + OPA-driven access control over which Platform Agents in which namespaces may mount which
  pre-defined volumes, following RBAC-as-floor / OPA-as-restrictor (ADR 0018) and Gatekeeper
  admission (ADR 0002).
- Read-oriented mounts by default (reference data is read-mostly); any writable mapping is
  explicitly flagged and policy-gated.
- Integration with the `Sandbox` / `SandboxTemplate` (A6) so the mount is realized in the agent
  pod, and with the `Agent` CRD (A5) so the mapping is referenced declaratively.
- GitOps packaging so mappings reconcile from Git like every other platform declaration.

### 2.2 Out of scope
- Provisioning the underlying storage / PVs themselves — substrate XRDs (`XObjectStore`,
  `XPostgres`, etc.) are owned by **B4**; B20 maps **pre-defined** PVs, it does not create them.
- The RAG / memory call-path access to data — **B6** (Platform SDK `rag.*`/`memory.*`),
  **A10/B11** (memory), `RAGStore` (B13/A17). B20 is the filesystem-mount alternative, not RAG.
- Memory-store access modes (`private` / `namespace-shared` / RBAC-OPA) on `MemoryStore` —
  **ADR 0025 / B4**; B20 governs PV mounts, a distinct mechanism.
- The Sandbox runtime, warm pool, and Envoy egress proxy themselves — **A6**.
- OPA policy framework + initial Rego content — **B3** / **B16**; B20 authors the PV-specific
  policy targets but contributes its Rego into that library.
- Cross-tenant publication mechanics for shared corpora beyond namespace scope — governed by the
  §6.9 OPA-checked publication model; B20 honors it, does not redefine it.

## 3. Context & Dependencies

Upstream consumed:
- **A6** (agent-sandbox + Envoy egress proxy) — the `Sandbox` / `SandboxTemplate` CRDs and pod
  runtime into which a PV is mounted. What is consumed: the sandbox volume/mount surface.
- **A7** (OPA / Gatekeeper) — admission + runtime policy engine that gates which agent may mount
  which PV. What is consumed: Gatekeeper admission hooks and the OPA decision point.
- **A5** (ARK) — the `Agent` CRD that references a PV mapping; ARK reconciles the agent that the
  Sandbox runs. What is consumed: the `Agent` CRD reference surface and reconcile ordering.

ADR decisions honored:
- **ADR 0016** — multi-tenancy via Kubernetes namespaces with RBAC + OPA + network enforcement;
  PV mappings are namespaced and tenancy-bounded.
- **ADR 0018** — RBAC-as-floor / OPA-as-restrictor; RBAC grants the mount-permission floor, OPA
  may further restrict per decision.
- **ADR 0002** — OPA + Gatekeeper as policy engine; PV-mapping admission is a Gatekeeper policy.
- **ADR 0003** — Envoy egress proxy is the egress control; B20 governs storage mounts, which are
  orthogonal to egress but share the defense-in-depth posture.
- **ADR 0030** — CRD/API versioning; any CRD/field B20 introduces is versioned per the owning component.
- **ADR 0025** — memory access modes are a separate mechanism on `MemoryStore`; B20 deliberately
  does not overload them for PV mounts.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
The Canon interface contract does **not** define a PV-mapping CRD or PV-mapping fields on `Agent`
or `Sandbox`. B20's declarative-mapping mechanism is therefore `[PROPOSED — not in source]`.
Two candidate shapes, both flagged, decision deferred to design (see §10 R1):
- **Option A (preferred):** a new namespaced **`AgentVolumeMapping` CRD `[PROPOSED — not in
  source]`** reconciled into the `Sandbox`/`SandboxTemplate` volume surface, with fields
  (all `[PROPOSED — not in source]`): `pvRef`/`pvcRef`, `agentRef` or `agentSelector`,
  `mountPath`, `readOnly`, `namespace`. Versioned per ADR 0030, owner = B20.
- **Option B:** carry the mapping inside existing Canon fields — reference the volume via the
  `Agent`/`Sandbox` pod spec surface owned by A5/A6 — adding **no new top-level field name** but
  relying on A6's `SandboxTemplate` (`resourceLimits`, `runtime`, … — §1.3) to expose a volume
  hook. Whether A6 exposes such a hook is `[PROPOSED — not in source]`.
B20 introduces **no new capability primitive** outside `CapabilitySet`'s §1.4 field set; PV access
is governed as its own mechanism, not by adding a field to `CapabilitySet`.

### 4.2 APIs / SDK surfaces
N/A — B20 is declarative + policy; it exposes no new API/SDK surface. A mounted volume is read by
the agent via the normal pod filesystem; the Platform SDK (B6) is not extended by B20.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- Admission/deny of a PV mapping is an OPA decision → `platform.policy.*` (owned by the OPA
  decision point / A7, surfaced by B20's policy contribution).
- A mapping add/update/delete reconciling is a configuration lifecycle change → candidate
  `platform.lifecycle.*` (Sandbox lifecycle) `[PROPOSED — not in source]` for the specific
  event-type name (per-event names are deferred to B12's registry, §2). B20 mints no new
  top-level namespace.

### 4.4 Data schemas / connection-secret contracts
N/A — B20 maps **pre-defined PVs**; it provisions no backend and writes no connection secret. The
volume's own backing store (if Crossplane-composed) follows the ADR 0041 connection-secret
contract owned by B4 — out of scope here.

## 5. OSS-vs-Custom Decision
**Wrap + build-thin.** B20 wraps native Kubernetes PV/PVC + the A6 sandbox volume surface and the
A7 OPA/Gatekeeper engine; it builds a thin declarative-mapping layer (the `[PROPOSED]`
`AgentVolumeMapping` reconcile, or the A6/A5 field-reuse path) plus the PV-specific Rego. No new
storage system is built — pre-defined PVs are assumed to exist. Rationale: PV/PVC and OPA already
provide the primitives; the gap is a governed, declarative binding into the agent sandbox.

## 6. Functional Requirements
- REQ-B20-01: B20 SHALL allow a pre-defined Kubernetes PV (reference dataset / shared corpus) to
  be mapped into a Platform Agent's Sandbox **declaratively** from Git, with no manual sandbox edit (§14.2 B20).
- REQ-B20-02: A PV mapping SHALL be namespace-scoped and SHALL only bind volumes and agents within
  an approved namespace, honoring the §6.9 tenancy boundary (ADR 0016).
- REQ-B20-03: Whether a given Platform Agent may mount a given pre-defined volume SHALL be
  controlled by RBAC as the permission floor and OPA as the per-decision restrictor (ADR 0018).
- REQ-B20-04: PV-mapping creation SHALL be admitted by a Gatekeeper policy that rejects mappings
  violating namespace/tenancy or referencing a non-approved volume (ADR 0002).
- REQ-B20-05: Mappings SHALL default to **read-only**; any read-write mapping SHALL be explicitly
  declared and subject to an additional OPA check.
- REQ-B20-06: The mapping SHALL be realized in the agent pod via the A6 `Sandbox`/`SandboxTemplate`
  volume surface, without B20 re-implementing the sandbox runtime.
- REQ-B20-07: A denied or admitted PV-mapping decision SHALL be observable as a `platform.policy.*`
  CloudEvent and SHALL be audited via the platform audit adapter (ADR 0034).
- REQ-B20-08: Any CRD/field B20 introduces SHALL be versioned per ADR 0030 with B20 named as the
  versioning owner, and SHALL be tagged `[PROPOSED — not in source]` until adopted into Canon.
- REQ-B20-09: B20 SHALL NOT overload `MemoryStore` access modes (ADR 0025) or `CapabilitySet`
  fields (§1.4) to express PV access; PV mapping is a distinct, separately-governed mechanism.

## 7. Non-Functional Requirements
- Security / multi-tenancy (§6.9, ADR 0016/0018): defense-in-depth — admission (Gatekeeper) +
  runtime OPA + namespace boundary. A mapping never crosses a tenancy boundary except via the §6.9
  explicit OPA-checked publication path. Mounting a volume confers no egress; Envoy (ADR 0003)
  remains the egress control.
- Observability (§6.5): mount decisions and reconcile outcomes are traceable via `platform.policy.*`
  / `platform.lifecycle.*` and audit emission (ADR 0034).
- Scale: reference corpora may be large; B20 governs the **mount**, not throughput — read-only,
  many-agent fan-out of a shared PV is supported subject to the underlying storage's access mode
  (e.g. ReadOnlyMany) — `[PROPOSED — not in source]` as to specific access-mode enforcement.
- Versioning (ADR 0030): the mapping mechanism is versioned by B20; additive policy targets are backward-compatible.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests: **Applicable** — the mapping reconcile/manifests + example PV mappings in Git.
- Per-product docs (10.5): **Applicable** — how PV mapping works, the access-control model, examples.
- Runbook (10.7): **Applicable** — diagnosing a failed/denied mount, RBAC/OPA troubleshooting.
- Alerts: **Applicable (thin)** — alert on repeated mapping-admission denials / mount failures.
- Grafana dashboard (Crossplane XR): **N/A — no dedicated metrics surface in v1.0; policy-decision metrics ride A7/audit dashboards.**
- Headlamp plugin: **N/A — no dedicated plugin; mappings visible via standard CRD views / the capability inspector (B5) where applicable.**
- OPA/Rego integration: **Applicable** — admission + runtime Rego for PV-mapping authorization, contributed to B16/B3.
- Audit emission (ADR 0034): **Applicable** — mount-grant/deny audited via the adapter.
- Knative trigger flow: **N/A — no originating trigger flow; policy events ride the existing `platform.policy.*` namespace.**
- HolmesGPT toolset: **N/A — no toolset contribution in v1.0.**
- 3-layer tests (Chainsaw/Playwright/PyTest): **Applicable** — Chainsaw for mapping admission/reconcile + mount realization; PyTest for OPA decision logic; Playwright N/A (no dedicated UI).
- Tutorials & how-tos: **Applicable** — "mount a reference dataset into an agent" how-to (content in C3, seeded here).

## 9. Acceptance Criteria
- AC-B20-01 (REQ-B20-01): Applying a PV-mapping manifest from Git results in the referenced
  pre-defined PV being mounted in the target agent's Sandbox with no manual sandbox edit.
- AC-B20-02 (REQ-B20-02): A mapping referencing a volume or agent outside the approved namespace is rejected.
- AC-B20-03 (REQ-B20-03): With RBAC granting mount but OPA denying, the mount does not occur; with both allowing, it does (RBAC-as-floor / OPA-as-restrictor).
- AC-B20-04 (REQ-B20-04): A Gatekeeper admission test rejects a mapping that references a non-approved volume or violates tenancy.
- AC-B20-05 (REQ-B20-05): A mapping with no explicit `readOnly:false` mounts read-only; a read-write mapping requires and passes the extra OPA check before mounting.
- AC-B20-06 (REQ-B20-06): The mount is realized through the A6 `Sandbox`/`SandboxTemplate` surface (verified by inspecting the agent pod), not via a bespoke runtime.
- AC-B20-07 (REQ-B20-07): An admit and a deny each produce a `platform.policy.*` event and an audit record via the adapter.
- AC-B20-08 (REQ-B20-08): Any new CRD/field is versioned (apiVersion present) and carries a `[PROPOSED — not in source]` marker in its docs until Canon adoption.
- AC-B20-09 (REQ-B20-09): A lint/review check confirms no PV access is expressed through `MemoryStore` access modes or `CapabilitySet` fields.

## 10. Risks & Open Questions
- R1 (high): The declarative-mapping **mechanism is not in Canon**. `[PROPOSED — not in source]`
  for the `AgentVolumeMapping` CRD (Option A) and for any volume hook on `SandboxTemplate` (Option
  B). Open question: new CRD vs. extending A6's `SandboxTemplate`. Reconciliation note: requires an
  A6 design agreement and likely a Canon/ADR revision before build; spec commits to the **governed
  outcome**, not the syntax.
- R2 (med): Whether A6's `Sandbox`/`SandboxTemplate` exposes a volume-mount hook today is unknown
  (interface-contract §1.3 lists `runtime`, `warmPoolSize`, `hibernationEnabled`, `resourceLimits`
  only). Open question for A6 owners; blocks REQ-B20-06 realization path.
- R3 (med): Shared-corpus mounts that cross namespaces interact with the §6.9 cross-tenant
  publication model. Open question: is a published shared PV a tenant-publish event
  (`platform.tenant.*`) or purely policy? Default: namespace-private, cross-namespace only via §6.9 OPA-checked publication.
- R4 (low): Read-write / concurrent-writer semantics (access modes like ReadWriteMany) and their
  policy implications are `[PROPOSED]`; v1.0 default is read-only reference data.
- R5 (low): Interaction with sandbox runtime isolation (gVisor/Kata) for arbitrary volume types is
  unverified; some volume types may not be mountable under a hardened runtime. Open question for A6.

## 11. References
- architecture-overview.md §6.3 Memory and data architecture (~L294–347; filesystem-mount vs SDK pattern L368 / §6.4).
- architecture-overview.md §6.4 Knowledge Base — SDK-API vs filesystem mount choice (~L368).
- architecture-overview.md §6.9 Multi-tenancy and namespacing (~L720–761; three-layer enforcement L754–758; cross-tenant publication L760).
- architecture-overview.md §6.2 Agent runtime architecture — Sandbox/skill volumes (~L247–280).
- architecture-overview.md §14.2 Workstream B, B20 row (~L1714); A6 row (~L1672); A7 row (~L1673).
- ADR 0016 (namespace multi-tenancy + RBAC/OPA/network), ADR 0018 (RBAC-as-floor / OPA-as-restrictor),
  ADR 0002 (OPA + Gatekeeper), ADR 0003 (Envoy egress), ADR 0025 (memory access modes — distinct), ADR 0030 (versioning).
- _meta/interface-contract.md §1.2 (`Agent`), §1.3 (`Sandbox`/`SandboxTemplate`), §1.4 (`CapabilitySet`), §2 (`platform.policy.*` / `platform.lifecycle.*`), §5 (audit adapter).
- _meta/glossary.md (Platform Agent, Sandbox, Approved namespace, Tenant, RBAC-as-floor / OPA-as-restrictor).
