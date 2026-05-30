# PLAN V6-07 — Eventing architecture (Knative + NATS JetStream) `[PROPOSED]`

> spec: SPEC-V6-07 · kind: VIEW · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [] · downstream-pieces: []

## 1. Implementation Strategy
This view is realized, not built. The realization map follows the wave layering: **A4 (Knative Eventing + NATS JetStream broker)** lands in W0 as the event mesh + broker backend; **B8 (Knative event adapter services)** lands in W1 (depends on A4) as the thin field-mapping adapters; **B12 (CloudEvent schema registry)** lands in W2 (depends on A4/B8) as the Git JSON-schema registry under the ten namespaces. The view holds when the same-broker-everywhere, environment-specific-source, Trigger-does-filtering, adapters-have-no-logic, closed-ten-namespace, event-versioning, initial-trigger-flows, and bidirectional-Mattermost invariants all hold end-to-end. Verification confirms broker parity across kind/AWS, the two initial flows fire, and a conformance check rejects any event outside the ten namespaces.

## 2. Ordered Task List
- **TASK-01:** Confirm A4 establishes Knative Eventing on NATS JetStream as the same broker in dev and prod, with Triggers + filters as the only filtering surface (ADR 0004) — produces: broker+trigger checklist — depends-on: []
- **TASK-02:** Confirm environment-specific sources (PingSource/ApiServerSource/AwsSqsSource/webhook) are NOT Crossplane-wrapped (ADR 0023) — produces: source-exception checklist — depends-on: [TASK-01]
- **TASK-03:** Confirm B8 adapters are pure field-mapping (CloudEvent→`AgentRun` CR; CloudEvent→Argo Workflow) with no decision logic — produces: adapter-purity checklist — depends-on: [TASK-01]
- **TASK-04:** Confirm B12 registry holds schemas under the closed ten namespaces, every event carries `specversion`+`schemaVersion`, breaking changes mint new event types (ADR 0030/0031) — produces: taxonomy+versioning checklist — depends-on: [TASK-03]
- **TASK-05:** Confirm the two initial trigger flows (AlertManager→HolmesGPT; budget-exceeded→email) and bidirectional Mattermost with OPA Trigger filtering (ADR 0036) — produces: initial-flows checklist — depends-on: [TASK-02, TASK-03]
- **TASK-06:** Define end-to-end view verification mapping AC-01..08 — produces: view acceptance suite mapping — depends-on: [TASK-04, TASK-05]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- A4 — Knative + NATS JetStream broker (consumed: mesh, broker, Triggers).
- B8 — adapter services (consumed: CloudEvent→`AgentRun`/Argo field-mapping).
- B12 — CloudEvent schema registry (consumed: namespace schemas + versioning).
### 3.2 Downstream pieces blocked on this
- None directly (view imposes constraints; each component designs its own trigger flows per its component spec). A19 (Mattermost adapter), A14 (HolmesGPT), A3 (Argo Workflows) consume the fabric.
### 3.3 Continuous (non-blocking) inputs
- B14 test framework; B22 threat model (A2A-lateral-movement, event-spoofing patterns shape REQ-03/05); the per-component "design your own trigger flows" expectation is a continuous downstream obligation, not a wave dependency.

## 4. Parallelizable Subtasks
- TASK-02 (source exception) and TASK-03 (adapter purity) run concurrently once TASK-01 is green. Fan-out group: {TASK-02, TASK-03}. TASK-04 follows TASK-03; TASK-05 joins TASK-02+TASK-03.

## 5. Test Strategy
- **Chainsaw (operator/CRD):** Trigger filter change reroutes events via GitOps without adapter changes (AC-03); CloudEvent→`AgentRun` adapter creates the CR (AC-04).
- **Playwright (UI/e2e):** Mattermost message in / platform event out through the same broker with OPA channel routing (AC-08).
- **PyTest (logic):** broker parity kind/AWS (AC-01); source not Crossplane-wrapped (AC-02); adapter no-logic assertion (AC-04); namespace-membership conformance + reject-outside-set (AC-05); `specversion`+`schemaVersion` + new-event-type-on-break (AC-06); AlertManager→HolmesGPT + budget→email flows (AC-07).
- Fixtures/fakes: in-memory broker or kind NATS JetStream; fake AlertManager alert; fake LiteLLM budget callback CloudEvent; stub OPA for Trigger filtering; B12 schema fixtures.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/auth` (views authoring band)
### 6.2 PR — `piece/V6-07-eventing-architecture` → base `wave/auth`; carries spec-V6-07.md + plan-V6-07.md
### 6.3 Merge order — independent of sibling view PRs; rolls up to main with the other V6-0x views.

## 7. Effort Estimate
- TASK-01 S · TASK-02 S · TASK-03 S · TASK-04 S · TASK-05 S · TASK-06 M. Rollup: S (authoring). Critical path: TASK-01 → TASK-03 → TASK-04 → TASK-06.

## 8. Rollback / Reversibility
Reverting the view doc has no runtime effect (authoring artifact). If an invariant is found wrong, amend the SPEC and re-flag `[PROPOSED]`; the realizing component SPECs (A4/B8/B12) carry the enforceable change. No downstream code breaks from reverting the view itself.
