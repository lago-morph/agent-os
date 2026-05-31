# SPEC B22 — Security threat model design specification

> kind: COMPONENT · workstream: B · tier: T0
> upstream: [] · downstream: [A, B] (non-blocking fan-out) · adrs: [0027, 0002, 0003, 0018, 0016, 0034, 0013, 0030] · views: [6.6]
> canon-glossary: b0edae10a2e649ba06e2b184dc938235aab758e3 · canon-interface: 0ce201d5d5af5cffcf09b647ea4a902a47596d36

## 1. Purpose & Problem Statement

B22 consumes `platform.evaluation` as a read-only consumer: it takes an explicit dependency on B14 (owner of the `platform.evaluation` namespace per QN-03) for the schema and does not co-own the namespace.

B22 is a **design specification, not code**: the platform's dedicated **adversarial threat model**.
The architecture's built-in defense-in-depth is optimized primarily against *unintentional access
excess* (§6.6, ADR 0027). B22 is the different artifact — a structured analysis treating **malicious
actors** as the primary modeling subject: adversary classes, asset inventory, trust boundaries, and
specific attack patterns, whose **output is a set of security standards every component must meet**
(acceptance criteria, mandatory test cases, mandatory dashboard signals, OPA policy targets).

B22 is **foundation (T0, W0)** and a **required pre-implementation deliverable** (ADR 0027): it ships
before the first wave of A/B implementation lands so its requirements feed forward into every
component's deliverable list, OPA targets (B16), and observability signals. It is a **non-blocking
fan-out input** (`B22 → A`, `B22 → B`, `_meta/waves.md`): continuously available, not a
wave-depth-setting predecessor. Each component completes a **security review against B22's
standards** before its implementation is considered complete.

## 2. Scope

### 2.1 In scope (this is an authoring/design deliverable)
- **Adversary classes** (§6.6 minimum set): malicious agent author, compromised agent, compromised
  tenant, compromised admin, compromised LLM or MCP provider, supply-chain compromise (skill
  artifacts, container images, CRDs), prompt injection from incoming text.
- **Asset inventory**: tenant data, secrets, audit integrity, compute, capability-registry integrity,
  identity tokens, the platform's own self-management agents (HolmesGPT, Coach).
- **Trust boundaries**: tenant ↔ tenant, agent ↔ gateway, gateway ↔ provider, sandbox ↔ host kernel,
  control plane ↔ data plane, internal A2A ↔ external A2A.
- **Attack catalog** (§6.6 minimum patterns): capability escape, audit tampering, OPA bypass,
  sandbox escape, A2A-based lateral movement, exfiltration via approved tools, secret extraction via
  prompt injection, denial-of-service via budget or capability exhaustion.
- **Per-attack mapping**: which controls apply (mapped to the ADR 0002/0003/0018 defense-in-depth
  layers), what residual risk remains, what observability detects exploitation.
- **The output standards** (the load-bearing deliverable): mandatory **acceptance criteria**,
  mandatory **test cases**, mandatory **dashboard signals**, and **OPA policy targets** — plus the
  **enforcement map** stating which component must satisfy each standard.
- The **forward-feed updates**: the edits B22 makes to §6.6, to the per-component deliverable lists in
  Workstream A, and to the OPA policy library targets in **B16** (ADR 0027).

### 2.2 Out of scope (and where it lives instead)
- **Any implementation / code** — B22 is design only (ADR 0027). Controls are implemented by their
  owning components (A7 OPA, A6 Envoy/sandbox, A18 audit, B13 capability registry, etc.).
- **The initial OPA policy *content*** → B16 (B22 sets the *targets*; B16 writes the Rego).
- **The framework the policies sit in** → B3.
- **Audit pipeline implementation** → A18 / ADR 0034 (B22 sets audit-integrity standards).
- **Continuous runtime posture** — re-scanning running images, image signing + admission
  verification, runtime drift detection — explicitly **out of v1.0** (ADR 0027,
  future-enhancements §2). B22 may identify additional gaps that get added to that backlog.
- **The security review *execution* per component** → each component's own checklist (F4 security
  review compiles the program); B22 provides the standards they review against.
- **Generalized everyday-mistake controls** — those remain the architecture's primary v1.0 modeled
  threat (§6.6); B22 refines/extends, does not replace them.

## 3. Context & Dependencies

**Upstream consumed:** None — B22 is W0 foundation with no internal dependencies (CSV: empty
upstream). It binds to the frozen architecture context (§6.6, the ADRs) rather than consuming build
outputs.

**Downstream consumers (non-blocking fan-out — `_meta/waves.md`):**
- **Every Workstream A and B component** — B22's standards update each component's deliverable list,
  acceptance criteria, mandatory tests, and dashboard signals. The edges `B22 → A`, `B22 → B` are
  *continuously available inputs*, not "must precede"; delays in B22 directly delay each dependent
  component's **security-complete** state.
- **B16** specifically — B22 updates the **OPA policy library targets** B16 must implement.
- **Workstream A observability (A13/D1/D2)** — B22's per-attack "what observability detects
  exploitation" becomes the canonical source for alert rules and dashboard panels.
- **F4 (security review)** — compiles the per-component reviews against B22's standards.

**ADRs honored:**
- **ADR 0027** — primary: B22 is design not code, a required pre-implementation deliverable, sets
  standards for every component, updates §6.6 + Workstream A deliverables + B16 targets; the v1.0
  stance (everyday-mistake primary) is a scope decision not a permanent posture; continuous runtime
  posture is out of v1.0.
- **ADR 0002 / 0003 / 0018** — the defense-in-depth chain (OPA+Gatekeeper / Envoy egress /
  RBAC-floor+OPA-restrictor) that B22 refines and extends rather than replaces.
- **ADR 0016** — multi-tenancy via namespaces + RBAC + OPA + network enforcement (tenant↔tenant
  boundary analysis).
- **ADR 0034** — audit pipeline (audit-integrity / audit-tampering analysis).
- **ADR 0013** — capability-registry integrity (capability-escape analysis).
- **ADR 0030** — B22's standards documents are versioned design artifacts.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
N/A — B22 defines no CRD/XRD. It analyses the existing CRD/XRD set (interface-contract §1) as assets
and trust-boundary surfaces and sets standards over them; it invents no fields.

### 4.2 APIs / SDK surfaces
N/A — B22 ships no runnable surface. Its "interface" is the **standards artifact** consumed by other
specs: the enforcement map (attack → control → owning component → mandatory AC/test/dashboard/OPA
target). This is a document contract, not an API.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
B22 emits no events. It **specifies which `platform.security.*` and `platform.policy.*` signals must
exist** as detection for each attack (e.g. sandbox-escape signal, repeated authn failures,
policy-bypass attempts are the source-stated `platform.security.*` contents — interface-contract §2).
The concrete per-event-type schemas remain B12's; B22 sets the requirement that they exist and are
surfaced as dashboard signals. `[PROPOSED — not in source]`: the specific new event-type names B22
may require are deferred to B12, not minted here.

### 4.4 Data schemas / connection-secret contracts
N/A — no datastore, no connection secret. B22 sets the **audit-integrity standard** (audit tampering
is in the attack catalog) over the A18/ADR 0034 pipeline but defines no schema itself.

## 5. OSS-vs-Custom Decision
N/A — B22 is a design specification, not a component with an OSS-vs-custom build choice. (Per the
template's COMPONENT note, this is the explicit N/A.) Threat-modeling *method* may draw on standard
frameworks (e.g. STRIDE/attack-tree style analysis) but B22 commits to no tool or upstream project;
the deliverable is the platform-specific analysis and standards.

## 6. Functional Requirements
> B22 "functional requirements" are authoring requirements — what the design specification must
> contain and produce.
- **REQ-B22-01:** B22 MUST enumerate the **adversary classes** at least covering the §6.6 set
  (malicious agent author, compromised agent/tenant/admin, compromised LLM/MCP provider,
  supply-chain compromise, prompt injection).
- **REQ-B22-02:** B22 MUST produce an **asset inventory** at least covering tenant data, secrets,
  audit integrity, compute, capability-registry integrity, identity tokens, and the self-management
  agents (HolmesGPT, Coach).
- **REQ-B22-03:** B22 MUST document the **trust boundaries** at least covering the six §6.6 pairs
  (tenant↔tenant, agent↔gateway, gateway↔provider, sandbox↔host-kernel, control↔data plane,
  internal↔external A2A).
- **REQ-B22-04:** B22 MUST produce an **attack catalog** covering at least the eight §6.6 patterns
  (capability escape, audit tampering, OPA bypass, sandbox escape, A2A lateral movement, exfiltration
  via approved tools, prompt-injection secret extraction, budget/capability-exhaustion DoS).
- **REQ-B22-05:** For **each** attack, B22 MUST produce a **per-attack mapping**: applicable controls
  (referenced to the ADR 0002/0003/0018 layers), **residual risk**, and **detecting observability**.
- **REQ-B22-06:** B22 MUST output, for each standard, a set of **mandatory acceptance criteria,
  mandatory test cases, mandatory dashboard signals, and OPA policy targets** that components must
  meet (ADR 0027).
- **REQ-B22-07:** B22 MUST produce an **enforcement map** binding each standard/attack to the
  **owning component(s)** responsible for satisfying it, so every component knows its security
  obligations.
- **REQ-B22-08:** B22 MUST specify the **forward-feed updates** it makes: to §6.6, to the
  per-component deliverable lists in Workstream A, and to the **OPA policy library targets in B16**
  (ADR 0027).
- **REQ-B22-09:** B22 MUST scope **continuous runtime posture** (image signing + admission
  verification, continuous re-scanning, runtime drift detection) as **out of v1.0**, recording any
  identified gaps into future-enhancements §2 (ADR 0027).
- **REQ-B22-10:** B22 MUST state that its v1.0 stance (everyday-mistake primary) is a **scope
  decision, not a permanent posture**, and define the revisit trigger (as adversarial controls
  mature, the primary modeled threat shifts and §6.6 is updated).
- **REQ-B22-11:** B22 MUST define the **per-component security-review gate**: a component's
  implementation is not "complete" until reviewed against B22's standards (ADR 0027), and components
  that begin before B22's first publication wait on it to close their security checklist.

## 7. Non-Functional Requirements
- **Security:** B22 *is* the security NFR source for the platform; it sets the standards the rest of
  the platform's security NFRs derive from. It refines but does not weaken the ADR 0002/0003/0018
  defense-in-depth commitment.
- **Multi-tenancy (§6.9 / ADR 0016):** the tenant↔tenant trust boundary and cross-tenant attack
  paths (capability publication abuse, A2A lateral movement, shared-memory leakage) are first-class
  in the analysis.
- **Observability (§6.5):** B22's per-attack detection signals become the canonical source for alert
  rules and dashboard panels the observability stack must surface (ADR 0027) — a design output, not a
  runtime metric B22 itself emits.
- **Scale:** N/A as a runtime property — B22 is design. The DoS-via-exhaustion attack pattern does
  set scale-related detection/limit standards other components implement.
- **Versioning (ADR 0030):** the standards documents are versioned design artifacts; as the threat
  posture shifts (REQ-B22-10) new revisions update §6.6 and downstream targets.

## 8. Cross-Cutting Deliverable Checklist
> B22 is a design specification (not code); the §14.1 standard set is **N/A as build artifacts** —
> B22 instead *defines the security portion of* these checklists for other components.
- Helm/manifests — **N/A — design spec, no deployable.**
- Per-product docs (10.5) — **applicable** — B22 *is* a document; the threat model + standards + 
  enforcement map are its deliverable, surfaced in the docs portal (and Knowledge Base).
- Runbook (10.7) — **N/A — no running service** (incident runbooks for the modeled attacks are
  authored by the owning components / Workstream C/F, informed by B22).
- Alerts — **applicable (specifies, does not ship)** — B22 specifies the mandatory alert/dashboard
  signals; A13/D1/D2 implement them.
- Grafana dashboard (Crossplane XR) — **applicable (specifies, does not ship)** — B22 specifies the
  mandatory dashboard signals; dashboards are authored as `GrafanaDashboard` claims by the
  observability components.
- Headlamp plugin — **N/A — design spec.**
- OPA/Rego integration — **applicable (specifies targets)** — B22 sets the OPA policy targets B16
  implements.
- Audit emission (ADR 0034) — **applicable (sets standard)** — B22 sets the audit-integrity /
  audit-tampering standard; emission is A18 + every component's adapter.
- Knative trigger flow — **N/A — design spec.**
- HolmesGPT toolset — **N/A** — though HolmesGPT/Coach are *assets* B22 analyses.
- 3-layer tests — **applicable (specifies mandatory cases)** — B22 defines mandatory test cases each
  component must implement; B22 ships no test code itself.
- Tutorials & how-tos — **applicable** — a "how to run a component security review against B22"
  how-to is part of feeding the standards forward.

## 9. Acceptance Criteria
> Binary pass/fail on the design deliverable's completeness and forward-feed.
- **AC-B22-01:** The threat model enumerates all §6.6 adversary classes. (→ REQ-B22-01)
- **AC-B22-02:** The asset inventory covers all §6.6 assets including HolmesGPT and Coach.
  (→ REQ-B22-02)
- **AC-B22-03:** All six §6.6 trust boundaries are documented. (→ REQ-B22-03)
- **AC-B22-04:** The attack catalog covers all eight §6.6 patterns. (→ REQ-B22-04)
- **AC-B22-05:** Every catalogued attack has a per-attack mapping with controls, residual risk, and
  detecting observability — no attack is unmapped. (→ REQ-B22-05)
- **AC-B22-06:** Every standard yields a concrete mandatory AC, mandatory test case, mandatory
  dashboard signal, and (where applicable) an OPA policy target. (→ REQ-B22-06)
- **AC-B22-07:** The enforcement map binds every standard/attack to at least one owning component;
  no standard is ownerless. (→ REQ-B22-07)
- **AC-B22-08:** B22 specifies concrete edits to §6.6, the Workstream A deliverable lists, and the
  B16 OPA policy targets. (→ REQ-B22-08)
- **AC-B22-09:** Continuous runtime posture is explicitly listed as out-of-v1.0 and any newly
  identified gaps are recorded into future-enhancements §2. (→ REQ-B22-09)
- **AC-B22-10:** The document states the v1.0 stance is a scope decision and defines the revisit
  trigger. (→ REQ-B22-10)
- **AC-B22-11:** The per-component security-review gate is defined and referenceable by every A/B
  component's security checklist. (→ REQ-B22-11)

## 10. Risks & Open Questions
- **R1 (high):** B22 is a fan-out input whose **delay delays the security-complete state of every
  dependent component** (ADR 0027). Mitigation: B22 is W0 foundation, authored first; partial
  publication (per-attack) can unblock incrementally rather than big-bang.
- **R2 (med):** B22 sets B16's OPA targets *after* B16 may have started its baseline; B16 must absorb
  B22 additions via an extension point (cross-ref B16 REQ-B16-13). Coordination risk if the two drift.
- **R3 (med):** The set of new `platform.security.*` event types B22 may require is `[PROPOSED]` and
  owned by B12; if B22 over-specifies event names it collides with B12's authority. Mitigation: B22
  specifies the *signal requirement*, B12 mints the type.
- **R4 (med):** Scope creep into continuous runtime posture (out of v1.0). Mitigation: REQ-B22-09
  hard-fences it to future-enhancements §2.
- **R5 (low):** B22's standards could conflict with the "lean v1.0" stance of other ADRs (e.g.
  approval system no-escalation). Mitigation: B22 refines/extends the existing ADR commitments, does
  not override them (ADR 0027).
- **OQ1:** Does B22 use a named methodology (STRIDE / attack trees) as the canonical structure, or a
  bespoke one? `[PROPOSED]` STRIDE-per-trust-boundary plus an attack-tree per catalogued attack; the
  source mandates the *contents*, not the method.
- **OQ2:** Is the enforcement map a standalone registry or embedded per-component? `[PROPOSED]`
  standalone canonical map B22 owns, referenced (not copied) by each component's security checklist,
  to avoid drift.

## 11. References
- architecture-overview.md §6.6 (line 436: threat-model v1.0 stance; lines 442–453: the design-spec
  mandate, adversary classes, asset inventory, trust boundaries, attack patterns, per-attack mapping,
  output standards), §14.2 (Workstream B / component B22).
- ADR 0027 (threat-model scope + B22 prerequisite — primary), 0002 (OPA+Gatekeeper), 0003 (Envoy
  egress), 0018 (RBAC-floor / OPA-restrictor), 0016 (multi-tenancy), 0034 (audit pipeline), 0013
  (capability-registry integrity), 0030 (versioning of the standards documents).
- interface-contract.md §1 (CRD/XRD assets analysed), §2 (`platform.security.*` / `platform.policy.*`
  detection namespaces).
- future-enhancements.md §2 (continuous container security — out-of-v1.0 fence).
- Related pieces: B16 (OPA targets), B3 (policy framework), A18 (audit), A13/D1/D2 (observability
  signals), F4 (security review program), and every A/B component (security-review gate).
