# SPEC E1 — Operator training track [PROPOSED]

> kind: COMPONENT · workstream: E · tier: T2
> upstream: [] · downstream: [] · adrs: [] · views: []
> canon-glossary: FROZEN · canon-interface: FROZEN

## 1. Purpose & Problem Statement

The Operator training track is the curriculum that takes a platform operator from zero to production-ready operation of the Agentic Execution Platform. Per §12, documentation is necessary but not sufficient: operators need a curriculum that composes the docs into pedagogical order and pairs each topic with hands-on exercises in a sandboxed lab environment. This piece owns the operator-facing modules, their labs, exercises with checks, and knowledge checks.

The problem it solves: an operator who has read the per-product docs (Workstream C) still cannot reliably install, configure, monitor, recover, upgrade, and secure the platform without guided, graded practice. E1 builds that competence by sequencing the operator topics enumerated in §12.1 (platform overview → cluster baseline → install/configure → config patterns → monitoring/alerting → failure modes/runbooks → backup/restore → upgrade → security ops → capacity planning → HolmesGPT incident response) into eleven modules, each with a sandboxed lab namespace that reuses the platform itself (§12, §12.2 closing note: "Labs reuse the platform itself").

E1 is a consumer-tier component (§14, §14.5): it continuously consumes the outputs of Workstreams A and B (the running platform) and Workstream C (docs), and is built after those outputs exist.

## 2. Scope

### 2.1 In scope

- Eleven operator training modules mapped one-to-one to the §12.1 topic list.
- A sandboxed lab namespace per module that reuses the running platform (§12, §12.2 closing note).
- A working starter state, exercises with automated checks, and a knowledge check per module (the §12.2 per-module structure, applied to the operator track).
- Defined learning outcomes per module, expressed as binary competencies an operator demonstrates.
- Module-to-docs linkage: each module references and links into the Workstream C docs rather than restating them (§12).

### 2.2 Out of scope (and where it lives instead — name the owning piece)

- Authoring the underlying product documentation — owned by Workstream C (C1–C9) and the per-product docs of Workstream A components (§14.1, §14.3).
- The Developer training track — owned by E2 (§14.5).
- Operator dashboards themselves — owned by D1 (Operator integrated dashboards) and the per-component Grafana dashboards of Workstream A (§11, §14.4); E1's monitoring module consumes them, it does not build them.
- Building the platform components the labs exercise — owned by Workstreams A and B.
- Cross-cutting and production runbooks — owned by C6 and F6; E1's failure-modes module references them.

## 3. Context & Dependencies

Upstream pieces consumed: none are hard build-time CRD dependencies (the CSV lists no upstream/downstream for E1). As a consumer-tier piece (§14, §14.5), E1 consumes two continuous inputs:
- Workstream C docs (C1–C9) — modules link into tutorials, how-tos, reference, explanation, and runbooks; E1 composes them into pedagogical order (§12).
- The running platform (Workstreams A and B) — labs reuse the platform itself (§12), so each module's lab namespace runs against live platform components (LiteLLM gateway A1, ARK A5, agent-sandbox/Envoy A6, OPA A7, HolmesGPT A14, Headlamp A9, the audit endpoint A18, etc.).

Downstream consumers: none (the CSV lists no downstream; E1 is terminal).

ADR decisions honored: no ADR is directly binding on a training curriculum. Labs that exercise platform behavior inherit the constraints of the components they run against (e.g. multi-tenancy via namespaces per ADR 0016, the audit pipeline per ADR 0034) but E1 imposes no new architectural decisions. `N/A — no ADR governs the training curriculum itself; labs inherit the constraints of the components they exercise.`

## 4. Interfaces & Contracts

E1 is curriculum content plus lab environments; it defines no platform CRDs, APIs, or CloudEvents of its own. Lab namespaces are Approved namespaces (Canon) provisioned the same way any tenant namespace is.

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)

N/A — E1 defines no CRDs or XRDs. Labs apply existing Canon CRDs (`Agent`, `Sandbox`, `CapabilitySet`, `MCPServer`, etc.) as authored by their owning components; E1 authors no new schema.

### 4.2 APIs / SDK surfaces

N/A — E1 exposes no API or SDK surface. Operator labs drive existing surfaces (the `agent-platform` CLI from B9, Headlamp UI from A9, Grafana dashboards) through guided exercises; E1 adds no new surface.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)

N/A — E1 emits and consumes no CloudEvents of its own. Lab exercises may observe existing platform events (e.g. `platform.lifecycle.*`, `platform.audit.*`) as part of monitoring and failure-mode modules, but the events are emitted by the components under exercise, not by E1.

### 4.4 Data schemas / connection-secret contracts

N/A — E1 defines no data schema or connection-secret contract. Lab namespaces consume connection secrets in the uniform substrate shape (Canon: connection secret) produced by the components they exercise.

## 5. OSS-vs-Custom Decision

Custom (build-new content): the curriculum, module structure, exercises, checks, and lab definitions are authored content specific to this platform — no upstream project supplies them. Lab environments are not new software: they reuse the platform itself (§12) and standard Approved namespaces, so there is no fork/wrap of any OSS project. Knowledge-check and exercise-check tooling reuses the existing `agent-platform` CLI (B9) and the three-layer test runners where automated verification is needed, rather than introducing a new framework.

## 6. Functional Requirements

- REQ-E1-01: The track SHALL provide exactly eleven modules, one per topic in §12.1 (platform overview; cluster baseline review; installing and configuring each component group; configuration patterns; monitoring and alerting; common failure modes and runbooks; backup and restore drills; upgrade procedures; security operations; capacity planning; working with HolmesGPT for incident response).
- REQ-E1-02: Each module SHALL declare its module coverage — the named platform components and docs (Workstream A/B/C pieces) it covers — so the union of modules covers the operator-facing surface enumerated in §12.1.
- REQ-E1-03: The "installing and configuring each component" module (§12.1 item 3) SHALL be structured as one sub-module per component group, covering each Workstream A install+configure package.
- REQ-E1-04: Each module SHALL include a sandboxed lab namespace that reuses the running platform (§12, §12.2 closing note), provisioned as an Approved namespace isolated per the platform's namespace multi-tenancy model.
- REQ-E1-05: Each lab SHALL provide a working starter state (a pre-seeded namespace/cluster condition) from which the exercises begin.
- REQ-E1-06: Each module SHALL include exercises with automated checks that verify the operator performed the required action (e.g. component installed, alert acknowledged, restore completed).
- REQ-E1-07: Each module SHALL include a knowledge check that gates module completion.
- REQ-E1-08: Each module SHALL declare explicit learning outcomes as binary, demonstrable operator competencies, and the exercises/checks SHALL map to those outcomes.
- REQ-E1-09: Modules SHALL link into the Workstream C docs (tutorials, how-tos, reference, explanation, runbooks) rather than restating them, composing them into pedagogical order (§12).
- REQ-E1-10: The HolmesGPT incident-response module (§12.1 item 11) SHALL exercise HolmesGPT (the Canon self-management Platform Agent) against a seeded incident in the lab namespace.
- REQ-E1-11: The failure-modes module (§12.1 item 6) SHALL exercise the relevant runbooks (C6 cross-cutting / F6 production runbooks) against induced failure conditions in the lab.

## 7. Non-Functional Requirements

- Security / multi-tenancy (§6.9): lab namespaces SHALL be isolated Approved namespaces enforced by the platform's namespace + RBAC + OPA model (ADR 0016), so a trainee's lab cannot affect another trainee's lab or production tenants.
- Observability (§6.5): monitoring and alerting exercises SHALL use the real operator dashboards (D1) and alert rules without a parallel training-only observability stack; lab activity is auditable like any platform activity (ADR 0034).
- Scale: the track SHALL support concurrent trainees by provisioning one isolated lab namespace per trainee per module; lab teardown SHALL reclaim resources.
- Versioning (ADR 0030): modules SHALL pin the platform/docs version they target and SHALL be revised when the components they cover change CRD/API versions, so a module never teaches a retired surface.

## 8. Cross-Cutting Deliverable Checklist

E1 is a consumer-tier (Workstream E) content component, not a Workstream A install/operate package; the §14.1 standard set is largely N/A. Each item is marked applicable / N/A below.

- Helm/manifests: N/A — E1 ships curriculum content + lab definitions, not a deployed product. Lab namespaces are provisioned via existing tenant-onboarding mechanisms (A21).
- Per-product docs (10.5): N/A — product docs are owned by Workstream A/C; E1 consumes them.
- Runbook (10.7): N/A — E1 references operator runbooks (C6/F6) rather than authoring them.
- Alerts: N/A — alert rules are owned by the Workstream A components; E1's monitoring module consumes them.
- Grafana dashboard (Crossplane XR): N/A — dashboards owned by D1 and Workstream A; E1 consumes them in the monitoring module.
- Headlamp plugin: N/A — E1 ships no plugin; labs use existing Headlamp UI (A9/A22).
- OPA/Rego integration: N/A — E1 contributes no policy; lab namespace isolation reuses existing OPA enforcement.
- Audit emission (ADR 0034): N/A — E1 emits no events of its own; lab actions are audited by the components under exercise.
- Knative trigger flow: N/A — E1 defines no event flows.
- HolmesGPT toolset: N/A — E1 consumes HolmesGPT in module 11; it contributes no toolset.
- 3-layer tests (Chainsaw/Playwright/PyTest): applicable — exercise checks and lab-validity tests reuse the three-layer runners (Chainsaw for CRD/lab-state assertions, Playwright for Headlamp/dashboard UI exercises, PyTest for check logic) via the B9/B14 CLI.
- Tutorials & how-tos: N/A (as authored docs) — E1 links into C2/C3 tutorials and how-tos; it does not author the Diataxis content, it sequences it.

## 9. Acceptance Criteria

- AC-E1-01: The track contains eleven modules whose titles map one-to-one to §12.1's eleven topics. (→ REQ-E1-01)
- AC-E1-02: Each module lists the named components/docs it covers, and the union covers every §12.1 topic with no topic uncovered. (→ REQ-E1-02)
- AC-E1-03: The install/configure module has one sub-module per Workstream A component group. (→ REQ-E1-03)
- AC-E1-04: Each module provisions an isolated lab namespace that runs against live platform components. (→ REQ-E1-04)
- AC-E1-05: Each lab boots into a documented working starter state. (→ REQ-E1-05)
- AC-E1-06: Each module's exercises have automated checks that pass only when the required operator action was performed. (→ REQ-E1-06)
- AC-E1-07: Each module has a knowledge check that must pass to mark the module complete. (→ REQ-E1-07)
- AC-E1-08: Each module declares binary learning outcomes, and every exercise/check traces to a stated outcome. (→ REQ-E1-08)
- AC-E1-09: Every module links to at least one Workstream C doc and restates no doc content verbatim. (→ REQ-E1-09)
- AC-E1-10: The HolmesGPT module's lab runs HolmesGPT against a seeded incident and the check passes when the trainee drives it to the expected diagnosis. (→ REQ-E1-10)
- AC-E1-11: The failure-modes module induces at least one failure and the check passes when the trainee follows the referenced runbook to recovery. (→ REQ-E1-11)

## 10. Risks & Open Questions

- Risk (med): docs drift — if Workstream C docs change, module links and composed order rot. Mitigation: version-pin per REQ-E1-07/§7 and revalidate on doc release. Reconciliation: E1 links rather than restates (REQ-E1-09) to minimize duplicated drift surface.
- Risk (med): lab provisioning load — concurrent trainees each needing an isolated namespace per module could strain the cluster. Mitigation: per-trainee teardown (§7 scale); cap concurrent labs.
- Risk (low): destructive operator exercises (backup/restore, upgrade) could damage shared platform state if isolation leaks. Mitigation: strict namespace isolation per §7; destructive drills confined to dedicated lab substrate.
- Open question (low): does the install/configure module use a dedicated throwaway cluster per trainee, or a shared cluster with namespace-scoped install exercises? §12.1 item 3 implies whole-component installs which are cluster-scoped — `[PROPOSED]` resolution: a per-trainee ephemeral kind substrate (per ADR 0044 substrate label) for install drills, shared platform for the rest. Needs confirmation with the platform-ops owner.
- Open question (low): how are knowledge-check pass thresholds and re-attempts governed? Not specified in §12; `[PROPOSED]` to be set by the training owner.

## 11. References

- architecture-overview.md §12 Training (~line 1577) — curriculum rationale; "Labs reuse the platform itself".
- architecture-overview.md §12.1 Operator track (~line 1581) — the eleven-topic source list.
- architecture-overview.md §12.2 closing note (~line 1611) — per-module structure (lab namespace, starter project, exercises with checks, knowledge check); labs reuse the platform.
- architecture-overview.md §14 / §14.5 Workstream E (~line 1744) — consumer-tier positioning; E1 = "Operator training track — modules with labs".
- architecture-overview.md §14.1 (~line 1647) — Workstream A standard deliverables (referenced for the install/configure module's component-group coverage).
- architecture-overview.md §11 / §11.2 / §14.4 — operator dashboards (D1) consumed by the monitoring module.
- piece-index.csv — E1 row: tier T2, wave consumer, no upstream/downstream.
- _meta/glossary.md — Canon terms (HolmesGPT, Platform Agent, Approved namespace, connection secret, CRDs).
