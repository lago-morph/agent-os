# SPEC V6-12 — CRD inventory [PROPOSED]

> kind: VIEW · workstream: — · tier: T0
> upstream: [] · downstream: [] · adrs: [0030, 0041, 0013, 0025, 0034, 0037, 0035, 0017] · views: [6.12]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

This view is the **integration contract** for the platform's full CRD/XRD surface: the **master
inventory** of every Custom Resource Definition and Crossplane composite/claim in the architecture,
who reconciles it, its scope, and its key attributes (§6.12). It is the single authoritative list
that every component spec, every CRD owner, and the interface-contract registry must agree with.

This SPEC builds nothing. As a T0 view its job is to (a) pin the inventory so the ~30 parallel
component specs share one CRD namespace, (b) assert the cross-cutting invariants that hold over the
*whole* set (all namespaced; per-component reconciler ownership; the X-prefix/claim naming rule), and
(c) **cross-check the inventory against `_meta/interface-contract.md` §1 and flag any divergence**.
The detailed schema of each CRD lives with its owning component and with the realizing views
(V6-08 capability CRDs, V6-09 tenancy, V6-03 data, etc.); this view owns the *list and its
invariants*, not the per-CRD field design.

## 2. Scope

### 2.1 In scope
- The master inventory table (§4.1) reproduced from §6.12, grouped by reconciler.
- The cross-cutting invariants over the whole set: **all platform CRDs are namespaced**; **no
  cluster-scoped platform CRDs in v1.0**; **per-component reconciler ownership** of each CRD's
  versioning lifecycle (ADR 0030); the **X-prefix XR ↔ unprefixed claim** naming rule (ADR 0041).
- The reconciler-ownership map (ARK / agent-sandbox / kopf-B13 / Argo+B19 / per-component /
  Crossplane-B4).
- The cross-check against interface-contract.md §1 and the divergence log (§10).

### 2.2 Out of scope (and where it lives instead)
- Per-CRD **field schema design** — the owning component spec + the realizing view (capability CRDs →
  V6-08, tenancy → V6-09, data/substrate → V6-03, observability → V6-05, approval → §7.5/B19).
- **Versioning policy mechanics** (conversion webhooks, deprecation windows) — **V6-13 / ADR 0030**.
- Substrate Composition behaviour + connection-secret shape — **ADR 0041 / V6-03**.
- The CloudEvent taxonomy (not CRDs) — **V6-07 / ADR 0031**.
- `Tool` and `Query` field sets — **defer to ARK** (component A5).

## 3. Context & Dependencies

Realizing components (the reconciler owners whose specs must match this inventory):
- **A5** (ARK install) — owns `Agent`, `AgentRun`, `Team`, `Tool`, `Memory`, `Evaluation`, `Query`.
- **A6** (agent-sandbox install) — owns `Sandbox`, `SandboxTemplate`.
- **B13** (kopf operator) — owns `MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`,
  `CapabilitySet`, `VirtualKey`, `BudgetPolicy`.
- **B19** (approval system) — owns `Approval` (Argo Workflow + B19).
- **B4** (Crossplane v2 Compositions) — owns all XRs/XRDs: `MemoryStore`, `AgentEnvironment`,
  `SyntheticMCPServer`, `GrafanaDashboard`, `AuditLog`, `TenantOnboarding`, `XAgentDatabase`,
  `XPostgres`, `XSearchIndex`, `XObjectStore`, `XMongoDocStore`.
- per-component — `LogLevel` (in-process or rolling-restart, ADR 0035).

ADR decisions honored: ADR 0030 (versioning + per-component ownership), ADR 0041 (XRD naming +
substrate), ADR 0013 (capability CRDs), ADR 0025 (memory access mode on `MemoryStore`), ADR 0034
(audit pipeline XRD), ADR 0037 (`TenantOnboarding`), ADR 0035 (`LogLevel`), ADR 0017 (`Approval`).

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (the master inventory — Canon, version per ADR 0030)

**ARK-reconciled (owner A5; all namespaced):** `Agent`, `AgentRun`, `Team`, `Tool`, `Memory`,
`Evaluation`, `Query`.

**agent-sandbox-reconciled (owner A6; all namespaced):** `Sandbox`, `SandboxTemplate`.

**kopf-operator-reconciled (owner B13; all namespaced):** `MCPServer`, `A2APeer`, `RAGStore`,
`EgressTarget`, `Skill`, `CapabilitySet`, `VirtualKey`, `BudgetPolicy`.

**Other reconcilers (namespaced):** `Approval` (Argo Workflow + B19); `LogLevel` (per-component).

**Crossplane XRs / XRDs (owner B4; all namespaced):** `MemoryStore` (XR), `AgentEnvironment` (XR),
`SyntheticMCPServer` (XR), `GrafanaDashboard` (XR; XR form `XGrafanaDashboard`), `AuditLog` (XRD; XR
form `XAuditLog`), `TenantOnboarding` (XRD), `XAgentDatabase` (XRD; claim `AgentDatabase`),
`XPostgres` (XRD; claim `Postgres`), `XSearchIndex` (XRD; claim `SearchIndex`), `XObjectStore`
(XRD; claim `ObjectStore`), `XMongoDocStore` (XRD; claim `MongoDocStore`).

Key attributes per CRD are the source-stated field sets in `_meta/interface-contract.md` §1.2–1.6
and §6.12; this view does not re-design them. The X-prefix/claim naming rule (ADR 0041): the
cluster-scoped composite is `X`-prefixed; the user-facing namespaced **claim drops the prefix**; both
names refer to the same XRD.

### 4.2 APIs / SDK surfaces
N/A — VIEW; the inventory is the surface. CRD APIs are versioned per V6-13 / ADR 0030.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
N/A — this view enumerates CRDs, not events. CRD lifecycle changes surface under
`platform.lifecycle.*` / `platform.capability.*` / `platform.tenant.*` per the emitting component;
the taxonomy is owned by V6-07.

### 4.4 Data schemas / connection-secret contracts
The substrate XRDs (`XPostgres`, `XSearchIndex`, `XObjectStore`, `XMongoDocStore`) write the uniform
connection-secret shape (`host`, `port`, `user`, `password`, `dbname` or per-primitive equivalent)
and the substrate-agnostic status (`ready`, `endpoint`, `version`) per ADR 0041 / interface-contract
§4. This view references that contract; V6-03 owns its design.

## 5. OSS-vs-Custom Decision
N/A — VIEW. Reconcilers carry their own decisions (ARK A5, agent-sandbox A6, custom kopf B13,
Crossplane B4, Argo+B19).

## 6. Functional Requirements

- **REQ-V6-12-01** (invariant): The platform CRD/XRD set MUST be exactly the master inventory in §4.1;
  any CRD a component spec defines MUST appear here, and any CRD here MUST have exactly one owning
  reconciler from §3.
- **REQ-V6-12-02** (invariant): **All platform CRDs MUST be namespaced.** There MUST be no
  cluster-scoped platform CRD in v1.0. (Crossplane XRs are cluster-scoped composites, but the
  user-facing platform resource is the namespaced claim; no *platform* CRD a user manages is
  cluster-scoped.)
- **REQ-V6-12-03** (constraint): Each CRD's versioning lifecycle MUST be owned by the component that
  owns its reconciler (ADR 0030); ownership MUST match the §3 map.
- **REQ-V6-12-04** (constraint): For every substrate-asymmetric XRD, the `X`-prefixed name MUST be the
  cluster-scoped composite and the namespaced claim MUST drop the prefix (ADR 0041); specs MUST NOT
  coin variant names.
- **REQ-V6-12-05** (constraint): This inventory MUST stay consistent with
  `_meta/interface-contract.md` §1; any divergence MUST be logged in §10 and reconciled before Canon
  freeze, not silently resolved.
- **REQ-V6-12-06** (constraint): `Tool` and `Query` field sets defer to ARK; `MemoryStore` carries
  the memory `accessMode` (not the `Memory` binding, ADR 0025) — specs MUST honor these placements.

## 7. Non-Functional Requirements
- **Multi-tenancy (§6.9):** the namespaced-everything invariant (REQ-02) is the structural basis of
  the tenancy boundary (V6-09).
- **Security:** a complete, single inventory is a security property — no shadow CRD escapes
  governance, admission, or audit.
- **Observability (§6.5):** `LogLevel` and `GrafanaDashboard`/`AuditLog` XRDs are the
  observability-facing entries; detailed in V6-05.
- **Versioning (ADR 0030):** the whole inventory is governed by V6-13's policy.

## 8. Cross-Cutting Deliverable Checklist
N/A — VIEW. CRD manifests, conversion webhooks, and tests belong to the owning components
(A5/A6/B13/B19/B4 + per-component for `LogLevel`), verified end-to-end in PLAN §5.

## 9. Acceptance Criteria
The view holds when:
- **AC-V6-12-01** (→REQ-01): The union of CRDs across all component specs equals the §4.1 inventory;
  no spec introduces an unlisted CRD and no listed CRD is orphaned.
- **AC-V6-12-02** (→REQ-02): Every installed platform CRD is namespaced; a cluster-scoped *platform*
  CRD (other than Crossplane composite XRs) fails the inventory check.
- **AC-V6-12-03** (→REQ-03): Each CRD's `CustomResourceDefinition` / conversion-webhook ownership
  traces to the reconciler component in §3.
- **AC-V6-12-04** (→REQ-04): For each substrate XRD, both the `X`-prefixed XR and the unprefixed claim
  resolve to one XRD; no variant name is used in any spec.
- **AC-V6-12-05** (→REQ-05): A diff of this inventory against interface-contract.md §1 yields only the
  items in the §10 divergence log (ideally empty after reconciliation).

## 10. Risks & Open Questions — interface-contract.md §1 cross-check (divergence log)

Cross-checked the §4.1 inventory against `_meta/interface-contract.md` §1.2–1.6. **No missing or
extra CRDs** were found: every ARK / agent-sandbox / kopf / Approval / LogLevel / Crossplane entry
matches one-to-one. The following are **naming / annotation nuances**, not membership divergences —
flagged for Canon-review awareness, none requiring an inventory change:

- **(low) `GrafanaDashboard` XR-vs-claim form.** §6.12 names the resource `GrafanaDashboard (XR)`;
  interface-contract §1.6 + glossary add the XR form `XGrafanaDashboard`. Both refer to the same XRD
  per the ADR 0041 naming rule. No divergence; noted so specs do not treat them as two resources.
- **(low) `LogLevel` owner.** §6.12 lists the reconciler as "per-component (in-process or
  rolling-restart)"; interface-contract §1.5 agrees. There is **no single owning component** — flagged
  because the ADR-0030 "owner column" pattern assumes one owner; `LogLevel`'s owner is intentionally
  distributed (ADR 0035). Specs MUST treat per-component reconcile as correct, not a gap.
- **(low) Claim names for `XSearchIndex` / `XObjectStore` / `XMongoDocStore`.** ADR 0041's rule
  implies claims `SearchIndex` / `ObjectStore` / `MongoDocStore`; the glossary spells out only
  `Postgres`, `AuditLog`, `AgentDatabase`, `GrafanaDashboard` explicitly. The §4.1 derivations
  (`SearchIndex` / `ObjectStore` / `MongoDocStore`) follow the rule deterministically but are
  **`[PROPOSED]`** as exact claim spellings pending explicit Canon confirmation — they are not
  invented variants, just the prefix-drop applied. Routed to ADR 0041 owners.
- **(low) `AuditLog` / `XAuditLog` scope.** §6.12 marks `AuditLog (XRD)` namespaced; interface-contract
  §1.6 agrees and notes the XR form `XAuditLog`. The XRD composes cluster-touching resources (S3,
  CronJob) but the **claim is namespaced** — consistent; flagged only to preempt a "but it touches
  cluster resources" objection.
- **[PROPOSED]** This SPEC is `[PROPOSED]` pending Canon review. It reproduces, and does not coin,
  CRD names; the only proposed strings are the three derived claim spellings above.
- **(med, open question)** Whether `LogLevel` should carry a nominal "lead owner" for the ADR-0030
  versioning column despite distributed reconcile — routed to V6-13 / ADR 0030 owners.

## 11. References
- architecture-overview.md §6.12 (CRD inventory table, ~L942–979); §6.13 (versioning, ~L981–1001);
  glossary "CRDs and XRDs" + XR↔claim naming note; interface-contract.md §1.2–1.6 + §4.
- ADR 0030 (versioning/ownership), ADR 0041 (XRD naming/substrate), ADR 0013 (capability CRDs),
  ADR 0025 (memory access mode), ADR 0034 (audit XRD), ADR 0037 (TenantOnboarding), ADR 0035
  (LogLevel), ADR 0017 (Approval).
- Realizing components: A5, A6, B13, B19, B4 (+ per-component LogLevel). Related views: V6-13
  (versioning), V6-08 (capability CRDs), V6-09 (tenancy), V6-03 (data/substrate), V6-05 (observability).
