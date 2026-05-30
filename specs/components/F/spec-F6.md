# SPEC F6 — Production runbook compilation

> kind: COMPONENT · workstream: F · tier: T2
> upstream: [C6, F1, F2, F4, F5, C1] · downstream: [] · adrs: [0008, 0012, 0040, 0026, 0035] · views: [6.5, 6.10]
> canon-glossary: b0edae10a2e6 · canon-interface: 0ce201d5d5af

## 1. Purpose & Problem Statement

F6 is the final production-readiness pass on runbooks (§14.6): a compilation step that takes the cross-cutting/integrated runbooks (C6) and the per-component runbooks (each Workstream A component's 10.7 deliverable), folds in the new runbooks the other F-pieces produce (F1 retention-change, F2 DR/restore, F4 security drills, F5 capacity), and **verifies that every per-component runbook has been exercised at least once**. It is the gate that turns a pile of authored runbooks into an operationally-trusted, cross-referenced runbook set.

F6 authors no new infrastructure and owns no new product. It consumes C6's runbooks and the F1/F2/F4/F5 runbook outputs, reconciles them for consistency, fills cross-cutting gaps (incident flows that span components), and produces a verification record showing each runbook was walked through (in a drill, a test, or a real exercise). Its primary downstream consumer is the operator (and, via the Knowledge Base, HolmesGPT).

## 2. Scope

### 2.1 In scope
- **Final compilation pass** on the cross-cutting/integrated runbooks (C6) — consistency, cross-references, single index, gap-fill for incidents that span components.
- **Per-component runbook exercise verification** — confirm each Workstream A component's 10.7 runbook has been exercised at least once (via a drill, a 3-layer test, or an F-piece activity) and record the evidence.
- **Fold-in of F-piece runbooks** — F1 (retention change / batch-stall), F2 (Postgres restore / OpenSearch reindex / secret recovery / full restore), F4 (credential rotation / secret-leak / sandbox-escape response), F5 (capacity / backlog-recovery / cold-start tuning).
- **Cross-cutting incident flows** — runbooks for incidents that cross components (e.g., gateway-down spanning A1/OPA/audit; broker-backlog spanning A4/B8/B12; chokepoint failure per future §1 hard-error reality).
- **Knowledge Base alignment** — ensure compiled runbooks are indexed into the `platform-knowledge-base` RAGStore (via C8) so HolmesGPT and operators can retrieve them.
- **Verification record** — a matrix mapping each runbook → exercise evidence → owning component.

### 2.2 Out of scope (and where it lives instead)
- **Per-component runbooks (authoring)** → each **Workstream A** component (10.7 deliverable); F6 verifies exercise, does not author them.
- **The cross-cutting runbooks (authoring)** → **C6**; F6 does the final pass/compilation on them.
- **The documentation portal infrastructure** → **C1** (Material for MkDocs, ADR 0008); F6 publishes into it, does not build it.
- **The Knowledge Base indexing pipeline** → **C8**; F6 ensures runbooks are indexed, does not build the pipeline.
- **The individual F-drills** → **F1/F2/F4/F5**; F6 folds in their runbook outputs and references their drill records.
- **HolmesGPT itself / its toolsets** → **A14 / B10**-adjacent; F6 ensures runbook *content* is retrievable, does not build toolsets.
- **Ongoing DR cadence / SLO runbooks** → future-enhancements.md §1, §3 (deferred).

## 3. Context & Dependencies

**Upstream consumed:**
- **C6 (cross-cutting / integrated runbooks)** — the primary input; F6 does the final compilation/consistency/cross-reference pass on them.
- **F1 / F2 / F4 / F5** — each produces new runbooks (retention, DR/restore, security drills, capacity) that F6 folds into the compiled set and references the drill/finding records of.
- **C1 (documentation portal infrastructure)** — Material for MkDocs portal (ADR 0008) F6 publishes the compiled runbook set into.
- **(per-component A runbooks)** — each Workstream A component's 10.7 runbook; F6 verifies each was exercised (not a CSV-listed upstream edge, but the verification target).
- **C8 (Knowledge Base indexing)** — indexes the compiled runbooks into `platform-knowledge-base`.

**Downstream consumers:** none (terminal piece); operators and HolmesGPT consume the output at runtime.

**ADRs honored:**
- **ADR 0008** — Material for MkDocs is the documentation portal; the compiled runbooks live there with search/versioning.
- **ADR 0012** — HolmesGPT is a first-class Platform Agent; runbooks must be retrievable to it (via the Knowledge Base) for triage.
- **ADR 0040** — Kargo promotion; runbooks for promotion/rollback flows are part of the cross-cutting set.
- **ADR 0026** — independent-cluster topology; runbooks are per-cluster, no federation flows.
- **ADR 0035** — rolling-restart-as-reload; operational runbooks (e.g., log-level toggle, config change) reflect the staged-restart reality.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
F6 introduces **no new CRD/XRD**. Runbooks reference existing CRDs/XRDs operationally (e.g., `LogLevel` for verbosity changes per ADR 0035, `AuditLog` for retention per F1, `Approval` for gated remediation). F6 sets no schema.

### 4.2 APIs / SDK surfaces
N/A — F6 introduces no API/SDK surface. It is documentation compilation + verification. Runbooks may reference the `agent-platform` CLI (B9) operator commands; F6 cites them, defines none.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
N/A — F6 emits/consumes no CloudEvents. Runbooks *document* responses to events under `platform.observability.*` / `platform.security.*` / `platform.lifecycle.*`, but F6 itself is not on an event path. (A "runbook updated" notification, if ever wanted, would be `platform.capability.*` via the Knowledge Base content change — `[PROPOSED — not in source]`, owned by C8.)

### 4.4 Data schemas / connection-secret contracts
N/A — F6 introduces no data schema or connection-secret. Compiled runbooks are documents indexed into the Knowledge Base RAGStore via C8's existing pipeline.

## 5. OSS-vs-Custom Decision
**Documentation compilation + verification — no build.** F6 uses the existing Material for MkDocs portal (C1, ADR 0008) and the C8 indexing pipeline; it is **config/content** work (compile, cross-reference, verify, publish, index). No fork, no new service. Rationale: §14.6 scopes F6 as a "final pass" and "verify exercised" activity; building tooling would exceed its remit. The verification matrix is authored content, not a new system.

## 6. Functional Requirements
- **REQ-F6-01:** F6 MUST produce a single compiled, cross-referenced runbook set from the C6 cross-cutting runbooks plus the F1/F2/F4/F5 runbook outputs, with a unified index.
- **REQ-F6-02:** F6 MUST verify that **every** Workstream A per-component runbook (10.7) has been exercised at least once, recording the exercise evidence (drill / 3-layer test / real exercise) and owning component.
- **REQ-F6-03:** F6 MUST produce a verification matrix mapping each runbook → exercise evidence → owner, and flag any unexercised runbook as a gap to its owning component.
- **REQ-F6-04:** F6 MUST author cross-cutting incident runbooks for incidents that span components (at minimum: chokepoint failure per future §1, broker backlog, gateway/audit/OPA-correlated failure, DR invocation).
- **REQ-F6-05:** Compiled runbooks MUST be published into the Material for MkDocs portal (C1, ADR 0008) with search/versioning.
- **REQ-F6-06:** Compiled runbooks MUST be indexed into the `platform-knowledge-base` RAGStore via C8 so HolmesGPT (ADR 0012) and operators can retrieve them.
- **REQ-F6-07:** F6 MUST reconcile terminology/cross-references to Canon names (glossary) and link each runbook to the relevant ADR(s) and component spec(s).
- **REQ-F6-08:** F6 MUST record runbook content versions and ensure the published set reflects the as-shipped v1.0 component behavior (ADR 0035 staged-restart reality, single-instance defaults per future §1).

## 7. Non-Functional Requirements
- **Security:** runbooks MUST NOT embed secret material or bypass-the-control instructions; security-response runbooks (from F4) MUST follow least-privilege guidance and reference approval-gated (`Approval`) remediation where applicable.
- **Multi-tenancy (§6.9):** runbooks that touch tenant resources MUST state the namespace/tenant-scoping and RBAC required; cross-tenant operations are called out explicitly.
- **Observability (§6.5):** each operational runbook MUST name the signal(s) that trigger it (alert/dashboard/event) so it is actionable; F6 verifies the signal exists (linking F4/F5 findings).
- **Scale:** capacity/backlog runbooks (from F5) MUST cite the measured baselines so operators have concrete numbers, not just procedures.
- **Versioning (ADR 0030):** the compiled set carries a version aligned to the v1.0 release; runbook revisions follow the portal's (C1) versioning.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **N/A — F6 ships no deployable; it compiles/publishes runbooks.**
- Per-product docs (10.5) — **applicable** — F6 *is* documentation: the compiled runbook set + verification matrix.
- Runbook (10.7) — **applicable** — F6 is the runbook-compilation piece itself; output is the unified runbook set. (Meta-runbook: "how to add/exercise a runbook.")
- Alerts — **N/A — F6 references the alerts that trigger runbooks; owning components author them.**
- Grafana dashboard (Crossplane XR) — **N/A — F6 references dashboards in runbooks; D-stream/owning components own them.**
- Headlamp plugin — **N/A — documentation compilation, no interactive CRD surface.**
- OPA/Rego integration — **N/A — F6 documents policy-response procedures; B16 owns the Rego.**
- Audit emission (ADR 0034) — **N/A — F6 is documentation; it documents audit-response runbooks, emits nothing.**
- Knative trigger flow — **N/A — F6 documents event-response runbooks; introduces no flow.**
- HolmesGPT toolset — **applicable (content)** — F6 ensures runbooks are retrievable by HolmesGPT via the Knowledge Base (ADR 0012); it authors no toolset.
- 3-layer tests — **applicable (verification harness)** — F6 uses the exercise evidence from 3-layer tests as runbook-exercised proof (REQ-F6-02); a doc-lint/link-check test for the compiled set is `[PROPOSED]`.
- Tutorials & how-tos — **applicable** (how-to: navigate/use the runbook set; how-to: exercise and sign off a runbook).

## 9. Acceptance Criteria
- **AC-F6-01:** A single cross-referenced runbook set with a unified index exists, incorporating C6 + F1/F2/F4/F5 runbooks. (→ REQ-F6-01)
- **AC-F6-02:** A verification matrix shows every Workstream A per-component runbook with exercise evidence and owner; no runbook is unaccounted-for. (→ REQ-F6-02, REQ-F6-03)
- **AC-F6-03:** Any unexercised runbook is flagged as a gap with an assigned owning component. (→ REQ-F6-03)
- **AC-F6-04:** Cross-cutting incident runbooks exist for at least chokepoint failure, broker backlog, correlated gateway/audit/OPA failure, and DR invocation. (→ REQ-F6-04)
- **AC-F6-05:** The compiled set is published and searchable in the Material for MkDocs portal. (→ REQ-F6-05, REQ-F6-08)
- **AC-F6-06:** A known runbook fact is retrievable from the `platform-knowledge-base` RAGStore (and thus by HolmesGPT). (→ REQ-F6-06)
- **AC-F6-07:** Each runbook links to its Canon terms, relevant ADR(s), and component spec(s); a link-check passes. (→ REQ-F6-07)

## 10. Risks & Open Questions
- **R1 (med):** "Exercised at least once" may be only weakly evidenced (a test that touches the path ≠ a real operator walkthrough). Mitigation: F6 grades evidence (drill > automated-test-touch > review) and flags weak evidence as a residual gap. Blast radius: med (false confidence).
- **R2 (med):** F6 depends on C6 and F1/F2/F4/F5 runbooks all existing; if any upstream runbook is missing/late, the compiled set has holes. Mitigation: the verification matrix makes holes explicit; F6 ships with gaps flagged rather than hidden. Blast radius: med.
- **R3 (low):** Runbooks drift from as-shipped behavior (esp. ADR-0035 staged-restart vs assumed hot-reload). Mitigation: REQ-F6-08 reconciles to as-shipped; link-check + review. Blast radius: low.
- **R4 (low):** Indexing into the Knowledge Base could lag, so HolmesGPT retrieves stale runbooks. Mitigation: verify C8 indexing on publish (AC-F6-06). Blast radius: low.
- **OQ1:** What evidence bar counts as "exercised" — a passing 3-layer test that traverses the runbook path, or a human walkthrough? `[PROPOSED]` accept either, but grade and flag test-only evidence; confirm with operators.
- **OQ2:** Are chokepoint-failure runbooks meaningful given fail-open/closed is undecided (future §1)? `[PROPOSED]` document the v1.0 hard-error reality and the manual recovery, noting the deferred policy decision.

## 11. References
- architecture-overview.md §14.3 C6 line 1729 (cross-cutting/integrated runbooks), §14.6 line 1762 (F6 scope: final pass + verify per-component runbooks exercised), §10.7 (per-component runbook deliverable), §6.5 (observability signals), §6.7 (AlertManager → HolmesGPT), §6.10/V6-10 (HolmesGPT self-management).
- future-enhancements.md §1 (single-instance defaults, chokepoint hard-error, DR drill = F2; fail-open/closed deferred), §3 (SLO runbooks deferred).
- Canon glossary (Knowledge Base = platform docs, runbooks, pinned vendor docs; HolmesGPT). Canon interface-contract §1.5 (`LogLevel`, `Approval`), §2 (`platform.observability.*`/`platform.security.*`).
- ADR 0008 (Material for MkDocs portal), ADR 0012 (HolmesGPT first-class agent), ADR 0040 (Kargo promotion/rollback), ADR 0026 (independent cluster), ADR 0035 (staged-restart reality).
- Related pieces: C6, C1, C8, F1, F2, F4, F5, A14 (HolmesGPT), and every Workstream A component (per-component runbook owners).
