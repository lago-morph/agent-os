# SPEC B16 — Initial OPA policy library content

> kind: COMPONENT · workstream: B · tier: T0
> upstream: [B3, A1, A6, A5, B13] · downstream: [] · adrs: [0002, 0018, 0013, 0032, 0017, 0044, 0038, 0030, 0027] · views: [6.6, 6.8, 6.9]
> canon-glossary: b0edae10a2e649ba06e2b184dc938235aab758e3 · canon-interface: 0ce201d5d5af5cffcf09b647ea4a902a47596d36

## 1. Purpose & Problem Statement

B16 ships the **initial restriction-rule content** for the platform's OPA policy library: the
actual Rego that every decision point enforces at v1.0. B3 defines the *framework* (bundle layout,
shared library, decision-document contract, dry-run hook, test convention); B16 fills that
framework with the concrete policies. ADR 0002 commits to one engine (OPA) and one language (Rego)
spanning Kubernetes admission (Gatekeeper), LiteLLM runtime callbacks, Envoy egress, Headlamp
action gating, approval-level elevation, and virtual-key/capability decisions. B16 is the v1.0
content for those surfaces.

Every policy B16 writes obeys the platform-wide **RBAC-as-floor / OPA-as-restrictor** invariant
(ADR 0018): a B16 policy may only deny or raise the bar, never grant. B16 is T0/W2 — it depends on
the framework (B3) plus the runtime decision-point owners (A1 LiteLLM, A6 Envoy/sandbox, A5 ARK,
B13 kopf) being present to bind against. B16's targets are also the **canonical landing zone for
B22's threat-model output** (ADR 0027): B22 updates "the OPA policy library targets in B16."

## 2. Scope

### 2.1 In scope
- **Admission policies (Gatekeeper)** for **all v1.0 CRDs/XRDs** enumerated as OPA decision points
  in §6.6: `Agent`, `AgentRun`, `Sandbox`, `SandboxTemplate`, `Memory`, `MemoryStore`, `MCPServer`,
  `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`, `CapabilitySet`, `VirtualKey`, `BudgetPolicy`,
  `Approval`, `LogLevel`, and the Crossplane XRs (`AgentEnvironment`, `SyntheticMCPServer`,
  `GrafanaDashboard`, `AuditLog`, `TenantOnboarding`, `AgentDatabase`, and substrate XRDs
  `Postgres`/`SearchIndex`/`ObjectStore`/`MongoDocStore`).
- The **substrate-mismatch admission rule** (ADR 0044): reject any XR whose target substrate
  has no matching Composition for the cluster's `platform.io/environment` label (B4 supplies the
  requirement + input shape; B16 authors the Rego).
- **Runtime LiteLLM-callback policies**: tool/model authorization, rate limiting, content checks,
  **budget enforcement** (reading `BudgetPolicy`→OPA data reconciled by B13), and the
  dynamic-registration decision (may a Platform Agent register an A2A/MCP interface in its namespace).
- **Egress policies** (Envoy egress OPA): which destinations are allowed for which agent class,
  layered over the FQDN allowlist derived from `EgressTarget` CRDs.
- **RBAC-floor / OPA-restrictor enforcement content** (ADR 0018): the per-decision narrowing rules
  that compose with the B3 floor predicate; structurally incapable of granting.
- **Headlamp action-gating policies**: which admin may approve suggestion cards, issue virtual keys,
  edit budgets, perform privileged operations (§6.6 OPA decision points).
- The **`approval.elevation` policy surface** (ADR 0017): may only **elevate** the required approval
  level, never lower it (B19-core consumes this).
- Multi-tenancy narrowing (§6.9): cross-tenant restriction rules; capability cross-namespace
  publication checks (ADR 0013); per-tenant scoping reading Platform JWT claims.
- Rego **unit tests** for every policy, plus the **dry-run/`simulated`** behaviour (ADR 0038) on
  every B16 entrypoint, conforming to B3's contract.

### 2.2 Out of scope (and where it lives instead)
- **Framework / bundle layout / shared library / decision contract / dry-run hook** → B3.
- **OPA + Gatekeeper install / engine / ConstraintTemplate runtime** → A7.
- **The LiteLLM callback code that *invokes* OPA** → B2 (B16 supplies the Rego it calls).
- **The policy simulator service / Headlamp panel / HolmesGPT skill** → A20 (ADR 0038); B16's
  entrypoints honor the dry-run flag the simulator composes.
- **`BudgetPolicy`/`VirtualKey`/capability CRD reconciliation into OPA data + LiteLLM** → B13.
- **CapabilitySet overlay resolution algorithm** → ADR 0032 / B13; B16 reads the *resolved* set.
- **The adversarial threat model itself** → B22 (B16 is a *consumer* of B22's policy targets).
- **The `Approval` CRD + Argo + CloudEvent mechanics** → B19-core; B16 owns only `approval.elevation`
  Rego.
- **Per-event-type CloudEvent schemas** for `platform.policy.*` decisions → B12.

## 3. Context & Dependencies

**Upstream consumed (HARD):**
- **B3** — the bundle layout, shared Rego library (claim extraction, RBAC-floor predicate,
  decision constructor, deny-aggregation, dry-run wrapper), decision-document/input-document
  contract, and test scaffold. B16 content MUST conform to all of it.
- **A1 (LiteLLM)** — the runtime callback decision point B16's gateway policies target.
- **A6 (agent-sandbox + Envoy egress)** — the egress decision point B16's egress policies target.
- **A5 (ARK)** — the CRDs (`Agent`, `AgentRun`, `Team`, `Tool`, `Memory`, `Evaluation`, `Query`)
  whose admission B16 gates.
- **B13 (kopf operator)** — reconciles capability/`VirtualKey`/`BudgetPolicy` CRDs into OPA `data`;
  B16's budget/capability/virtual-key policies read that data.

**Downstream consumers:** None as a build piece (B16 has no `downstream` in the CSV). Its policies
are *enforced* by A7/Gatekeeper, B2 (LiteLLM callbacks), A6 (Envoy), A9/A22 (Headlamp gates),
B19-core (approval elevation), A20 (simulator dry-runs B16 entrypoints).

**ADRs honored:**
- **ADR 0002** — Rego-as-code, SHA-pinned bundles, B3/B16 split; B16 is the content half.
- **ADR 0018** — every B16 policy is a restrictor, never a grantor.
- **ADR 0013** — capability/CapabilitySet model; cross-tenant publication is OPA-checked.
- **ADR 0032** — B16 reads the *resolved* CapabilitySet, not the layered inputs.
- **ADR 0017** — `approval.elevation` may only elevate.
- **ADR 0044** — substrate-mismatch admission rejection.
- **ADR 0038** — every entrypoint honors dry-run with `simulated: true`, audited under
  `platform.policy.*`, no enforcement-path side effects.
- **ADR 0030** — bundle/policy versioning per the framework's convention.
- **ADR 0027** — B22's threat-model output updates B16's policy targets; B16 carries the v1.0
  baseline and absorbs B22 additions as they publish.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
N/A — B16 defines no CRD. It writes **Rego that decides over** the CRDs/XRDs in interface-contract
§1.2–§1.6. It reads CRD fields by their Canon names (e.g. `BudgetPolicy.limits`,
`CapabilitySet.opaPolicyRefs[]`, `Approval.defaultLevel`, `EgressTarget.fqdn`,
`MemoryStore.accessMode`, `platform.io/environment`). Gatekeeper `ConstraintTemplate`/constraint
objects are owned by A7 + B16 content together (per B3 §4.1); B16 supplies the Rego bodies.

### 4.2 APIs / SDK surfaces

The **OPA decision-document format is owned and defined by B3 (D-03)**; B16 does not own or redefine it — B16 authors policy content that evaluates to documents in the B3-owned format.

**Policy-bundle ownership / signing / staged rollout:**
- **#24 (ownership):** the **platform security team** owns the global/cross-tenant OPA bundle. Tenants propose changes by pull request; a security reviewer MUST approve before merge. B16 does not unilaterally own global bundle content.
- **#25 (signing):** policy bundles are cryptographically signed and verified at load time; the engine refuses to load unsigned or tampered bundles.
- **#26 (staged rollout):** a new or changed bundle first runs in **audit mode** (logs what it would block) and flips to enforce only after review; the Kargo promotion pipeline gates that flip.
- Any security-relevant event B16 policies detect (e.g. signature-verification failure, tampered-bundle rejection) MUST also be emitted under `platform.security` (schema owned by A7), in addition to local handling.
B16 emits **decision documents** in B3's canonical shape (input: Platform JWT claims
`platform_tenants`/`platform_namespaces`/`platform_roles`/`tenant_roles`/`capability_set_refs`,
action, target, context, dry-run; output: `allow`, `reasons[]`, `layer`, `simulated`). The field
names other than `simulated` are `[PROPOSED — not in source]` and **owned by B3** — B16 binds to
whatever B3 freezes; it does not define new ones. Each decision point has one well-known entrypoint
rule per B3 convention so A20 can compose per-layer dry-runs.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
OPA decisions and violations flow under **`platform.policy.*`** (OPA decisions, policy violations,
dynamic-registration accept/deny — interface-contract §2). **Simulator dry-run decisions ARE audited
under `platform.policy.*`** (§6.6). B16 itself is Rego (not a running service): emission is performed
by the invoking component (B2, A6, Gatekeeper, B19, A20); B16 defines the decision content that gets
emitted. Concrete per-event-type schemas live in **B12**. `approval.elevation` outcomes ultimately
surface on `platform.approval.*` via B19, not emitted by B16.

### 4.4 Data schemas / connection-secret contracts
N/A — no datastore, no connection secret. B16's "data" is OPA `data` documents (budget data,
capability/resolved-set data, RBAC-floor data) **populated by B13** reconciliation; B16 reads them
under the namespacing convention B3 defines.

## 5. OSS-vs-Custom Decision
**Custom (build-new) Rego content on OSS OPA/Gatekeeper (A7), authored against the B3 framework.**
No upstream policy bundle is forked; B16 writes platform-specific rules. Rationale (ADR 0002):
one Rego toolchain across all decision points; the content is platform policy, inherently custom.

## 6. Functional Requirements
- **REQ-B16-01:** B16 MUST provide a Gatekeeper **admission policy for every v1.0 CRD/XRD** listed
  as an OPA decision point in §6.6 (the full kind list above), each conforming to B3's layout and
  decision contract.
- **REQ-B16-02:** B16 MUST implement the **substrate-mismatch admission rule**: reject any XRD
  XR whose target substrate has no matching Composition for the cluster's
  `platform.io/environment` label (ADR 0044), using B4's supplied input shape.
- **REQ-B16-03:** B16 MUST provide **LiteLLM runtime-callback policies** for tool/model
  authorization, rate limiting, content checks, and **budget enforcement** that reads
  `BudgetPolicy` data reconciled by B13 (§6.6).
- **REQ-B16-04:** B16 MUST provide the **dynamic-registration policy**: whether a Platform Agent may
  register an A2A or MCP interface in its namespace (§6.6).
- **REQ-B16-05:** B16 MUST provide **egress policies** deciding allowed destinations per agent
  class, composing with the `EgressTarget`-derived FQDN allowlist at the Envoy egress proxy (§6.6).
- **REQ-B16-06:** B16 MUST provide **Headlamp action-gating policies** for suggestion-card approval,
  **virtual-key issuance**, budget edits, and other privileged admin operations (§6.6).
- **REQ-B16-07:** B16 MUST provide the **`approval.elevation`** policy that may only **elevate** an
  `Approval`'s required level and never lower it (ADR 0017 / 0018), consumed by B19-core.
- **REQ-B16-08:** B16 MUST provide **virtual-key-issuance and capability-access** policies that
  narrow (never widen) what an RBAC-permitted requestor may issue/bind (§6.6, ADR 0013/0018).
- **REQ-B16-09:** B16 MUST provide **multi-tenancy narrowing** rules reading Platform JWT claims,
  including the **cross-namespace capability-publication** check (default namespace-private,
  cross-tenant opt-in OPA-checked — ADR 0013, §6.9).
- **REQ-B16-10:** **Every** B16 policy MUST be a **restrictor** — structurally incapable of granting
  beyond the RBAC floor (uses B3's floor predicate; ADR 0018).
- **REQ-B16-11:** **Every** B16 decision entrypoint MUST honor the **dry-run** flag, returning
  `simulated: true` with no enforcement-path side effect, audited under `platform.policy.*`
  (ADR 0038).
- **REQ-B16-12:** Every B16 policy MUST ship **Rego unit tests** following B3's convention; tests
  block merge on failure (ADR 0002).
- **REQ-B16-13:** B16 MUST maintain a **coverage map** of every §6.6 OPA decision point to the
  policy package(s) that serve it, so coverage is auditable; and MUST provide an **extension point**
  where B22's threat-model output adds policy targets (ADR 0027).

## 7. Non-Functional Requirements
- **Security:** B16 *is* the platform's runtime/admission security content; the RBAC-floor invariant
  (REQ-B16-10) is the central guarantee. Bundles are SHA-pinned and Git-reconciled via B3's pipeline.
  B16 carries the v1.0 baseline against "unintentional access excess" (§6.6) and absorbs B22's
  adversarial additions (ADR 0027).
- **Multi-tenancy (§6.9):** policies decide per-tenant from Platform JWT claims; no tenant identifier
  is hardcoded; cross-tenant access is deny-by-default and OPA-checked on publication.
- **Observability (§6.5):** every decision (including dry-run) emits the uniform decision document
  enabling `platform.policy.*` audit; coverage map makes enforced surfaces auditable.
- **Scale:** policies MUST be side-effect-free and evaluable in the LiteLLM callback hot path
  without per-request bundle reloads (inherits B3's library discipline).
- **Versioning (ADR 0030):** B16 policies version with the bundle revision; a decision-contract
  change is owned by B3 — B16 follows.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **applicable** (Rego packages + Gatekeeper constraints delivered via B3's
  ArgoCD-reconciled bundle layout).
- Per-product docs (10.5) — **applicable** (policy-coverage reference, per-decision-point rule docs).
- Runbook (10.7) — **applicable** (false-deny triage, policy rollback, budget-policy mismatch).
- Alerts — **applicable** (policy-violation spikes, budget-exceeded, dynamic-registration denies).
  `[PROPOSED — not in source]` for concrete thresholds.
- Grafana dashboard (Crossplane XR) — **applicable** — policy-decision/violation dashboard authored
  as a `GrafanaDashboard` XR (panels are also a B22 mandatory-signal landing zone).
- Headlamp plugin — **N/A** — the policy-review/simulator UI is A20/A22; B16 is content-only.
- OPA/Rego integration — **applicable** — this component *is* the initial Rego content.
- Audit emission (ADR 0034) — **applicable (defines content)** — emission performed by invoking
  components; B16 defines the `platform.policy.*` decision content audited (incl. dry-run).
- Knative trigger flow — **N/A** — B16 mints no event types; `platform.policy.*` schemas are B12.
- HolmesGPT toolset — **N/A** — simulator-as-skill is A20.
- 3-layer tests — **applicable** (Rego unit tests + PyTest for data-fixture logic; Chainsaw for
  Gatekeeper admit/deny against real CRDs; Playwright **N/A** — no UI).
- Tutorials & how-tos — **applicable** ("read a CRD field in a policy", "add a tenant-scoped rule").

## 9. Acceptance Criteria
- **AC-B16-01:** Every v1.0 CRD/XRD in the §6.6 list has an admission policy; applying a violating
  CR is denied and a conforming CR is admitted, for each. (→ REQ-B16-01)
- **AC-B16-02:** An XR on a `kind` cluster with no matching Composition is **denied at
  admission** with a substrate-mismatch reason. (→ REQ-B16-02)
- **AC-B16-03:** A LiteLLM call exceeding its `BudgetPolicy` limit is denied by the budget policy;
  an in-budget call is allowed; tool/model-authorization and rate-limit rules deny/allow per fixture.
  (→ REQ-B16-03)
- **AC-B16-04:** An agent attempting to register an MCP/A2A interface outside policy is denied; a
  permitted registration is allowed. (→ REQ-B16-04)
- **AC-B16-05:** An egress request to a destination not allowed for the agent class is denied; an
  allowed `EgressTarget` destination passes. (→ REQ-B16-05)
- **AC-B16-06:** A non-authorized admin is denied virtual-key issuance / budget edit / card approval;
  an authorized one is allowed, per Keycloak claims. (→ REQ-B16-06)
- **AC-B16-07:** `approval.elevation` raises an `Approval`'s required level for a sensitive-data
  action and **never** lowers a level below its default in any test case. (→ REQ-B16-07)
- **AC-B16-08:** A virtual-key/capability-binding request beyond the requestor's RBAC is denied; a
  within-RBAC request is narrowed-or-allowed but never widened. (→ REQ-B16-08)
- **AC-B16-09:** A cross-namespace capability reference without OPA-checked publication is denied;
  a published one is allowed; tenant-scoping reads JWT claims in a test. (→ REQ-B16-09)
- **AC-B16-10:** A static/lint+test gate proves no B16 allow-rule grants beyond the B3 floor
  predicate (a planted granting rule fails the gate). (→ REQ-B16-10)
- **AC-B16-11:** Every B16 entrypoint invoked with dry-run returns `simulated: true`, produces no
  state change, and is recorded under `platform.policy.*`. (→ REQ-B16-11)
- **AC-B16-12:** A policy with a failing Rego unit test blocks merge in CI. (→ REQ-B16-12)
- **AC-B16-13:** The coverage map lists every §6.6 OPA decision point against its serving package
  with no gaps; the B22 extension point is documented and exercised by a placeholder target.
  (→ REQ-B16-13)

## 10. Risks & Open Questions
- **R1 (high):** B16 binds to B3's `[PROPOSED]` decision-document/input field names. If B3's contract
  shifts after B16 lands, every policy breaks. Mitigation: B3 freezes first; B16 binds; coordinated
  bump if it changes (ADR 0030).
- **R2 (high):** "Structurally incapable of granting" (REQ-B16-10) is hard to guarantee in Rego by
  language alone; relies on a CI lint/test gate (inherited from B3 R2). Residual risk if the gate
  has gaps.
- **R3 (med):** B22 (ADR 0027) will add policy targets after B16's baseline; B16 must absorb them
  without a structural rewrite. Mitigation: explicit extension point (REQ-B16-13). Blast radius
  scales with how much B22 adds.
- **R4 (med):** Budget-enforcement rules depend on B13 reconciling `BudgetPolicy`→OPA data in a
  shape B16 expects; the data-document shape is a B3/B13 contract (`[PROPOSED]` field names).
- **R5 (low):** Some LiteLLM budget features may be enterprise-only (§6.6); if so, enforcement still
  routes through OPA — B16 unaffected, but the spend-tracking side may degrade.
- **OQ1:** Are content-check policies (§6.6 "content checks") in v1.0 scope as real rules or
  placeholders? `[PROPOSED]` ship a minimal content-check entrypoint + hook, real rules deferred to
  B22 output.
- **OQ2:** Is the substrate-mismatch deny emitted as `platform.policy.*` or `platform.security.*`?
  Mirror B4 OQ2 — `[PROPOSED]` `platform.policy.*`; B12 owns schema.

## 11. References
- architecture-overview.md §6.6 (line 436: threat-model stance; lines 532–542: OPA decision points —
  authoritative target list; line 544: policy simulator dry-run / `simulated` contract), §6.8
  (capability registries, resolved-set reads), §6.9 (multi-tenancy, cross-tenant publication).
- ADR 0002 (OPA + Gatekeeper; B3/B16 split), 0018 (RBAC-floor / OPA-restrictor), 0013 (capability
  model), 0032 (overlay — resolved set), 0017 (approval elevation), 0044 (substrate-mismatch
  admission), 0038 (policy simulator dry-run), 0030 (versioning), 0027 (B22 feeds B16 targets).
- interface-contract.md §1.2–§1.6 (CRD/XRD field names B16 decides over), §2 (`platform.policy.*`).
- Related pieces: B3 (framework), A7 (OPA install), B2 (LiteLLM callbacks), A6 (Envoy), A5 (ARK),
  B13 (kopf data), A20 (simulator), B19-core (approval elevation), B22 (threat-model targets),
  B12 (event schemas), B4 (substrate-mismatch input shape).
