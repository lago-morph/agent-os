# SPEC B3 — OPA policy library (framework)

> kind: COMPONENT · workstream: B · tier: T0
> upstream: [A7] · downstream: [B16] · adrs: [0002, 0018, 0030, 0038] · views: [6.6]
> canon-glossary: b0edae10a2e6 · canon-interface: 0ce201d5d5af

## 1. Purpose & Problem Statement

B3 is the **framework** for the platform's OPA policy library: the Rego bundle structure, the
shared library setup (helpers, common decision shapes, test scaffolding), and the
"how-to-add-a-policy" contribution conventions that every other component follows when it
contributes Rego. ADR 0002 commits the platform to a single policy engine (OPA) and a single
policy language (Rego) spanning Kubernetes admission (Gatekeeper), LiteLLM runtime callbacks,
Envoy egress decisions, Headlamp action gating, approval-level elevation, and virtual-key
issuance. Without a shared framework, each of those ~dozen decision points would invent its own
bundle layout, decision-document shape, and test harness — defeating the "one authoring/testing
toolchain" rationale that justified choosing OPA over Kyverno.

B3 ships the *scaffolding only*. The **initial policy content** (the actual restriction rules)
lives in B16. B3 defines where policies go, how they are named, how they are tested, how they are
packaged into SHA-pinned bundles, how they are reconciled by ArgoCD, and the canonical
input/output decision contract — including the structured dry-run mode that ADR 0038's policy
simulator depends on. It is foundation (T0, W1) because every policy-emitting component binds to
its contract.

## 2. Scope

### 2.1 In scope
- Rego **bundle structure**: directory layout, package namespacing, `data` document conventions,
  `.manifest` revision/roots, SHA-pinning, and the per-decision-point bundle partitioning.
- **Shared Rego library**: common helper packages (RBAC-floor evaluation, claim extraction from
  Platform JWT, decision-document constructors, deny/allow aggregation, `simulated` flag handling).
- **Canonical decision contract**: the input document schema each decision point passes to OPA and
  the output/decision document shape OPA returns (allow/deny + reason + `simulated`), uniform across
  admission, gateway, egress, Headlamp, approval, and CLI gates.
- **How-to-add-a-policy** contribution convention: where a component drops its Rego, how it names
  packages/rules, the required unit-test layout, and the CI gates (Rego unit tests, SHA-pin check).
- **Dry-run / simulator hook contract** (ADR 0038): every framework-provided decision entrypoint
  honors a dry-run input and emits the same decision shape with `simulated: true` and no
  enforcement-path side effects.
- Bundle build + SHA-pin tooling and the ArgoCD-reconciled delivery layout for bundles.
- Mapping of bundle packages to the OPA decision points enumerated in §6.6.

### 2.2 Out of scope (and where it lives instead)
- **Actual restriction rules / initial policy content** → **B16** (initial OPA policy library content).
- **OPA / Gatekeeper install + runtime deployment** → **A7**.
- **CapabilitySet overlay resolution semantics** → ADR 0032 / design specs (B13 consumes).
- **The policy simulator service itself** (Headlamp panel + HolmesGPT skill) → **A20** (ADR 0038);
  B3 only defines the dry-run *contract* the simulator composes.
- **LiteLLM custom callbacks** that *invoke* OPA → **B2**.
- **Budget rule content** (BudgetPolicy → OPA data) → reconciled by **B13**; rule content in B16.

## 3. Context & Dependencies

**Upstream consumed:**
- **A7 (OPA / Gatekeeper)** — the OPA engine and Gatekeeper admission integration. B3 consumes
  A7's bundle-loading mechanism and Gatekeeper constraint-template model; B3 defines the bundle
  layout A7 serves and ArgoCD reconciles.

**Downstream consumers:**
- **B16** — fills the framework with the initial restriction rules; must conform to B3's layout,
  decision contract, and test convention.
- Every component with an OPA decision point (B2 LiteLLM callbacks, A6/A3-Envoy egress, B19
  approval elevation, Headlamp action gates via A9/A22, A20 simulator, `agent-platform` CLI B9)
  binds to B3's decision contract.

**ADRs honored:**
- **ADR 0002** — OPA is the single engine, Rego the single language; Rego is treated as code with
  CI unit tests and SHA-pinned bundles; no central UI (Headlamp compensates). B3 realizes the
  "OPA policy library has its own framework component" commitment.
- **ADR 0018** — RBAC-as-floor / OPA-as-restrictor: the framework's decision helpers MUST be
  structurally incapable of *granting* beyond RBAC; OPA may only further restrict.
- **ADR 0030** — bundle/schema versioning; B3 defines how bundle revisions are versioned.
- **ADR 0038** — policy simulator dry-run contract; framework entrypoints honor `simulated`.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
N/A — B3 ships no CRD. It produces Rego bundles consumed by A7/Gatekeeper. Gatekeeper
`ConstraintTemplate` / constraint objects are owned by A7 + B16 content, not defined here.
`[PROPOSED — not in source]` a bundle-revision convention (semver-tagged `.manifest` `revision`)
is proposed below as the versioning mechanism since source does not name a bundle version field.

### 4.2 APIs / SDK surfaces
**Canonical decision contract** (the framework's primary interface). Source (§6.6, ADR 0038)
states only that every decision point emits "the same decision shape" with a `simulated` flag.
The concrete field names below are `[PROPOSED — not in source]`:

- Input document (per decision): `subject` (Platform JWT claims — `platform_tenants`,
  `platform_namespaces`, `platform_roles`, `tenant_roles`, `capability_set_refs`), `action`,
  `target`, `context`, and `dry_run` (bool). `[PROPOSED — not in source]` for all field names.
- Decision document: `allow` (bool), `reasons[]`, `layer` (which enforcement point), and
  `simulated` (bool). `simulated` is the only source-named field (ADR 0038); the rest are
  `[PROPOSED — not in source]`.
- Decision entrypoint convention: one well-known Rego rule per decision point that returns the
  decision document, so the simulator (A20) can compose per-layer dry-runs into one response.

**Shared library packages** (Rego package names are `[PROPOSED — not in source]`): claim
extraction, RBAC-floor predicate, deny-aggregation, decision constructor, dry-run wrapper.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
B3 itself emits none (it is a library/bundle, not a running service). It **defines the contract**
that OPA decision points — including dry-run/simulator runs — are audited under `platform.policy.*`
(§6.6: "Simulator runs ARE audited under `platform.policy.*`"). Concrete per-event-type schemas
live in B12; the emission is performed by the invoking component (B2, A20), not B3.

### 4.4 Data schemas / connection-secret contracts
N/A — no datastore, no connection secret. B3's "data" is OPA `data` documents (e.g.
budget data reconciled by B13, RBAC floor data). The framework defines the `data`-document
namespacing convention only; population is by reconcilers (B13) and content (B16).

## 5. OSS-vs-Custom Decision
**Custom (build-new) framework on top of OSS OPA.** Upstream: Open Policy Agent + Gatekeeper
(via A7; engine OSS, no pinned version stated in source — `[PROPOSED — not in source]` to pin in
the bundle manifest). Approach: **build-new** conventions (bundle layout, shared library, test
scaffold, decision contract); **wrap** OPA's native bundle/`data` model rather than fork it.
Rationale (ADR 0002): one Rego toolchain across all decision points requires a shared framework;
OPA provides the engine but no opinionated library structure.

## 6. Functional Requirements
- **REQ-B3-01:** The framework MUST define a canonical Rego **bundle directory layout** with
  package namespacing per decision point (admission, gateway, egress, headlamp, approval, cli).
- **REQ-B3-02:** The framework MUST ship a **shared Rego library** providing: Platform JWT claim
  extraction, an RBAC-floor predicate, a decision-document constructor, and deny-aggregation.
- **REQ-B3-03:** The framework MUST define a **canonical decision-document shape** returned by
  every decision entrypoint, including the `simulated` flag (ADR 0038).
- **REQ-B3-04:** The framework MUST define a **canonical input-document schema** carrying Platform
  JWT claims (`platform_tenants`, `platform_namespaces`, `platform_roles`, `tenant_roles`,
  `capability_set_refs`), action, target, and a `dry_run` indicator.
- **REQ-B3-05:** Every framework-provided decision entrypoint MUST honor `dry_run`/`simulated`
  with **no enforcement-path side effects** (ADR 0038).
- **REQ-B3-06:** The framework's RBAC-floor predicate MUST make it structurally impossible for a
  policy to **grant** access beyond the RBAC floor — OPA may only further restrict (ADR 0018).
- **REQ-B3-07:** The framework MUST provide a **how-to-add-a-policy** convention: drop location,
  package/rule naming, required unit-test layout, and the CI gate set.
- **REQ-B3-08:** Rego policies MUST be packaged as **SHA-pinned bundles**; the build tooling MUST
  produce the pin and CI MUST verify it (ADR 0002).
- **REQ-B3-09:** Rego **unit tests** MUST run in CI for every contributed policy and block merge
  on failure (ADR 0002).
- **REQ-B3-10:** Bundles MUST be **versioned** via a `.manifest` revision and reconciled by
  **ArgoCD** into the Git-driven delivery path (ADR 0002).
- **REQ-B3-11:** The framework MUST map each bundle package to one of the §6.6 OPA decision
  points so coverage is auditable.

## 7. Non-Functional Requirements
- **Security:** RBAC-floor invariant enforced in the library (REQ-B3-06); bundles SHA-pinned and
  Git-reconciled; no policy may bypass the floor.
- **Multi-tenancy (§6.9):** claim-extraction helpers expose tenant/namespace claims so policies
  decide per-tenant; the framework MUST NOT hardcode tenant identifiers.
- **Observability (§6.5):** decision entrypoints emit a uniform decision document enabling
  `platform.policy.*` audit; dry-run decisions are observable.
- **Scale:** library helpers MUST be side-effect-free and evaluable in the LiteLLM callback hot
  path without per-request bundle reloads.
- **Versioning (ADR 0030):** bundle revisions follow the platform versioning policy; breaking
  decision-contract changes require a coordinated bump because every decision point binds to it.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **applicable** (bundle delivery layout, ArgoCD application).
- Per-product docs (10.5) — **applicable** (how-to-add-a-policy, decision contract reference).
- Runbook (10.7) — **applicable** (bundle rollback, pin-mismatch recovery).
- Alerts — **applicable** (bundle load failure, pin mismatch). `[PROPOSED — not in source]`.
- Grafana dashboard (Crossplane XR) — **N/A** — framework emits no runtime metrics of its own; A7/B2 own OPA decision metrics.
- Headlamp plugin — **N/A** — policy-review/simulator UI is A20/A22; B3 is library-only.
- OPA/Rego integration — **applicable** (this component *is* the Rego framework).
- Audit emission (ADR 0034) — **N/A (defines contract only)** — emission performed by invoking components; B3 defines the `platform.policy.*` decision shape.
- Knative trigger flow — **N/A** — no events emitted by the library itself.
- HolmesGPT toolset — **N/A** — simulator-as-skill is A20.
- 3-layer tests — **applicable** (PyTest for tooling; Rego unit tests; Chainsaw for ArgoCD bundle reconcile).
- Tutorials & how-tos — **applicable** (how-to-add-a-policy is a core deliverable).

## 9. Acceptance Criteria
- **AC-B3-01:** A documented bundle layout exists; a sample policy placed per the layout loads
  into OPA. (→ REQ-B3-01)
- **AC-B3-02:** The shared library exposes claim-extraction, RBAC-floor, decision-constructor,
  and deny-aggregation helpers with passing unit tests. (→ REQ-B3-02)
- **AC-B3-03:** Every decision entrypoint returns a decision document containing the `simulated`
  field. (→ REQ-B3-03)
- **AC-B3-04:** A decision entrypoint correctly reads `capability_set_refs` and the other Platform
  JWT claims from the input document in a test. (→ REQ-B3-04)
- **AC-B3-05:** Invoking an entrypoint with `dry_run=true` returns `simulated: true` and produces
  no enforcement-path side effect (verified by absence of state change). (→ REQ-B3-05)
- **AC-B3-06:** A policy attempting to grant beyond the RBAC floor is rejected by the
  library predicate / a framework test fails. (→ REQ-B3-06)
- **AC-B3-07:** Following the how-to-add-a-policy guide, a new policy + its tests build and pass
  in CI without framework edits. (→ REQ-B3-07)
- **AC-B3-08:** The build tooling emits a SHA pin and CI fails on a mismatched pin. (→ REQ-B3-08)
- **AC-B3-09:** A policy with a failing Rego unit test blocks merge in CI. (→ REQ-B3-09)
- **AC-B3-10:** A bundle revision bump reconciles through ArgoCD to the running OPA. (→ REQ-B3-10)
- **AC-B3-11:** Each bundle package documents the §6.6 decision point it serves; a coverage check
  lists all packages against the decision-point set. (→ REQ-B3-11)

## 10. Risks & Open Questions
- **R1 (high):** The concrete decision-document/input field names are `[PROPOSED — not in source]`.
  If A20/B2/B16 each assume a different shape, the "one decision shape" invariant breaks. Mitigation:
  B3 owns and freezes the contract first; downstreams bind to it. Blast radius: high (cross-component).
- **R2 (med):** RBAC-floor "structural impossibility" (REQ-B3-06) is hard to guarantee in Rego
  purely by convention; it may need a lint/test gate rather than a language guarantee. `[PROPOSED]`
  a CI lint that flags any allow-rule not gated by the floor predicate.
- **R3 (med):** Bundle versioning field is `[PROPOSED — not in source]`; must align with ADR 0030
  per-component ownership and B12's `schemaVersion` style without colliding.
- **R4 (low):** Hot-path performance of shared helpers in LiteLLM callbacks — mitigated by
  side-effect-free helpers and SHA-pinned cached bundles.
- **OQ1:** Does the dry-run input live in the same document as the live input (a `dry_run` flag) or
  a separate entrypoint? Affects A20 composition. `[PROPOSED]` single document + flag.
- **OQ2:** One bundle per decision point or one monorepo bundle with package roots? `[PROPOSED]`
  monorepo bundle, package roots per decision point, single `.manifest`.

## 11. References
- architecture-overview.md §6.6 (line 436; defense-in-depth, audit + OPA hook points, RBAC-floor),
  §6.6 policy simulator (line 544; dry-run `simulated` contract), §6.9 (Platform JWT claims).
- ADR 0002 (OPA + Gatekeeper; Rego-as-code, SHA-pinned bundles, B3/B16 split).
- ADR 0018 (RBAC-floor / OPA-restrictor). ADR 0030 (versioning). ADR 0038 (policy simulators).
- Related pieces: A7 (OPA install), B16 (initial content), B2 (LiteLLM callbacks), A20 (simulator).
