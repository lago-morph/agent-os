# SPEC ADR-0032 — CapabilitySet overlay semantics `[PROPOSED]`

> kind: ADR · workstream: — · tier: T1
> upstream: [A1; B13] · downstream: [B13; A7; A22; A20; A18] · adrs: [0032] · views: [6.8]
> canon-glossary: FROZEN · canon-interface: FROZEN

## 1. Purpose & Problem Statement

ADR 0032 is a settled decision: `CapabilitySet` resolution is a **field-level overlay** with
**add-if-not-there / replace-if-there** semantics, referenced sets processed one-at-a-time in
declared order, and per-Agent `overrides` applied last. This SPEC states what honoring that
decision requires of every component that reads or computes the resolved capability set — it
does not re-argue the merge rule.

The problem the decision solves: multiple consumers (the kopf operator, OPA, Headlamp, the
audit path) read the resolved set, and a non-deterministic or additive merge would make
security review and audit unsound. Determinism is a security property: the same Agent spec plus
the same referenced `CapabilitySet`s MUST yield the same effective set every time, on every
consumer.

## 2. Scope

### 2.1 In scope
- The overlay resolution rule as a conformance contract on every resolved-set consumer.
- Field-level replace-not-merge for list fields (`mcpServers[]`, `a2aPeers[]`, `ragStores[]`,
  `egressTargets[]`, `skills[]`, `llmProviders[]`, `opaPolicyRefs[]`) — an overlay restates the
  full list it wants.
- Ordered processing of referenced sets; per-Agent `overrides` applied last.
- Determinism guarantee: identical inputs ⇒ identical resolved set across all consumers.
- The amendment rule: any change to the overlay rule is a breaking change under ADR 0030.

### 2.2 Out of scope (and where it lives instead)
- Whether a `CapabilitySet` may include other `CapabilitySet`s, recursion depth, missing/deleted
  reference validation, namespace-boundary rules on cross-references, and whether `overrides` may
  grant capabilities absent from referenced sets or must be a subset — all **deferred to
  design-time** per architecture-backlog §1.1. `[PROPOSED — not in source]` for any concrete pick.
- Reconciliation of the resolved set into LiteLLM — owned by B13 (kopf operator).
- Admission validation of the resolved set — owned by A7 (OPA / Gatekeeper).
- The Headlamp rendering of the resolved set — owned by A22 (graphical-editor framework, ADR 0039).

## 3. Context & Dependencies

Upstream consumed: A1 (LiteLLM) is the gateway the resolved set is reconciled into; B13 (kopf
operator) is the reconciler that owns the `CapabilitySet` CRD and computes the resolved state.
Downstream consumers: B13 reconciles the resolved set; A7 validates it at admission; A22 renders
it in the per-Agent unified view; A20 (policy simulator) and A18 (audit) read the resolved set.

ADR decisions honored:
- **ADR 0032** (this): field-level overlay; add-if-not-there / replace-if-there; declared order;
  `overrides` last; deterministic single algorithm.
- **ADR 0013**: the capability CRD model and `CapabilitySet` field set this overlay operates on.
- **ADR 0018**: OPA remains the authoritative restrictor at decision time regardless of what
  overlay authors write (RBAC-as-floor / OPA-as-restrictor).
- **ADR 0030**: any change to the overlay rule is a breaking change requiring CRD/API versioning.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs
`CapabilitySet` (namespaced; owner B13). Fields per Canon: `mcpServers[]`, `a2aPeers[]`,
`ragStores[]`, `egressTargets[]`, `skills[]`, `llmProviders[]`, `opaPolicyRefs[]`. `Agent`
(namespaced; owner A5) supplies `capabilitySetRefs[]` (declared order is significant) and
`overrides` (applied last). This ADR imposes the resolution semantics over those existing fields;
it introduces no new CRD.

### 4.2 APIs / SDK surfaces
The resolution algorithm is a single shared implementation (the kopf operator computes it; other
consumers MUST read the same resolved output, not re-implement a divergent merge). The exact
function signature / library location is **not specified in source** `[PROPOSED — not in source]`.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
Resolution-relevant capability changes surface under `platform.capability.*`
(`platform.capability.changed`, ADR 0013) when a referenced set changes the resolved output. This
ADR emits no new namespace.

### 4.4 Data schemas / connection-secret contracts
N/A — no connection-secret surface; the resolved capability set is an in-memory/CRD-status
artifact, not a backing store.

## 5. OSS-vs-Custom Decision
N/A — ADR (no component build decision). The overlay rule is an invariant; B13 (custom Python
kopf operator) is where the single resolution algorithm lives, per that component's own SPEC.

## 6. Functional Requirements

- REQ-ADR-0032-01: Resolution MUST be field-level overlay: for each top-level field, add it if
  absent in the resolved state, fully replace it if present.
- REQ-ADR-0032-02: List fields MUST be replaced, never concatenated/merged; an overlay that
  intends to extend a parent list MUST restate the full intended list.
- REQ-ADR-0032-03: Referenced `CapabilitySet`s MUST be processed one at a time in
  `capabilitySetRefs[]` declared order.
- REQ-ADR-0032-04: Per-Agent `overrides` MUST be applied last, under the same overlay rule.
- REQ-ADR-0032-05: Given identical Agent spec + identical referenced sets, every consumer (B13,
  A7, A22, A20, A18) MUST compute the same resolved capability set (determinism).
- REQ-ADR-0032-06: The resolution algorithm MUST be a single shared implementation; consumers MUST
  read the resolved output rather than re-derive it with a divergent merge.
- REQ-ADR-0032-07: Any change to overlay semantics (list rule, override ordering, recursion)
  MUST be treated as a breaking change and go through ADR 0030 CRD/API versioning.

## 7. Non-Functional Requirements
- Security/multi-tenancy: determinism is the precondition for sound security review and audit of
  the effective set; OPA remains the runtime restrictor (ADR 0018) independent of overlay output.
- Observability (§6.5): resolved-set changes are auditable via `platform.capability.*`.
- Versioning (ADR 0030): overlay-rule changes are breaking; conversion discipline applies.
- Scale: resolution is O(refs × fields), order-defined, with no merge backtracking.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR. The realizing components (B13 resolver, A7 admission, A22 editor) carry the §14.1
deliverables in their own SPECs; conformance to the overlay rule is verified per §9.

## 9. Acceptance Criteria

- AC-ADR-0032-01: Honored when, for a field present in two stacked sets, the resolved value is the
  later set's value in full (replace), not a merge. (→ REQ-01, REQ-02)
- AC-ADR-0032-02: Honored when a list field in an overlay replaces (not appends to) the parent
  list in the resolved output. (→ REQ-02)
- AC-ADR-0032-03: Honored when reordering `capabilitySetRefs[]` changes the resolved output in the
  order-sensitive case, proving declared order is respected. (→ REQ-03)
- AC-ADR-0032-04: Honored when a per-Agent `override` wins over every referenced set for the same
  field. (→ REQ-04)
- AC-ADR-0032-05: Honored when B13, A7, and A22 each compute byte-identical resolved sets for the
  same fixture inputs. (→ REQ-05, REQ-06)
- AC-ADR-0032-06: Honored when a proposed overlay-rule change is rejected by review unless it
  carries an ADR 0030 version bump. (→ REQ-07)

## 10. Risks & Open Questions
- (med) `[PROPOSED]` — whether `overrides` may grant capabilities absent from referenced sets, and
  nested-`CapabilitySet` recursion, are deferred (architecture-backlog §1.1); resolver behavior
  for those cases is undefined until pinned. Flagged for design-time.
- (low) Replace-not-merge surprises overlay authors who expect additive lists; mitigated by the
  §6.8 worked example and the A22 effective-set preview.
- (med) Divergent re-implementation of the merge in any consumer would silently break determinism;
  REQ-06 mandates a single shared implementation as mitigation.
- Open: validation of missing/deleted references and namespace-boundary rules on cross-references
  — deferred (architecture-backlog §1.1).

## 11. References
- ADR 0032 (this decision). Enforcing/realizing components: B13 (resolution algorithm + reconcile
  to LiteLLM), A7 (admission validation of resolved set), A22 (effective-set rendering), A20
  (simulator reads resolved set), A18 (audit of resolved-set changes).
- architecture-overview.md §6.8 (capability registries; worked overlay example).
  architecture-backlog.md §1.1 (deferred layering sub-decisions), §6 (invariant).
- ADR 0013 (capability CRD model), ADR 0018 (OPA-as-restrictor), ADR 0030 (versioning).
