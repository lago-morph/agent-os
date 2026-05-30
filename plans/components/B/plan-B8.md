# PLAN B8 — Knative event adapter services

> spec: SPEC-B8 · kind: COMPONENT · tier: T1
> wave: W1 · estimate: M
> upstream-pieces: [A4] · downstream-pieces: [B12]

## 1. Implementation Strategy

Build two thin, stateless Python services that sit downstream of Knative Triggers: CloudEvent → AgentRun (creates an `AgentRun` CR for ARK) and CloudEvent → Workflow (submits an Argo Workflow). They are pure field-mappers with no decision logic — all filtering stays at the Trigger so it remains audited and policy-checked (§6.7). Make creation idempotent against at-least-once redelivery, source-agnostic across AWS/kind (ADR 0023), version the HTTP receivers `/v1/...` (ADR 0030), and emit audit via the A18 adapter (ADR 0034). Ship reference Trigger manifests as examples. Since B8 is upstream of B12, B8's mapping work informs the `platform.lifecycle.*` schemas B12 will register.

## 2. Ordered Task List

- **TASK-01:** CloudEvents HTTP receiver scaffold (`/v1/...`, structured+binary) — produces: receiver lib — depends-on: [].
- **TASK-02:** CloudEvent → AgentRun field-mapper + K8s create — produces: AgentRun adapter — depends-on: [TASK-01].
- **TASK-03:** CloudEvent → Workflow field-mapper + Argo submit — produces: Workflow adapter — depends-on: [TASK-01].
- **TASK-04:** Idempotency keying on CloudEvent identity — produces: dedupe layer — depends-on: [TASK-02, TASK-03].
- **TASK-05:** traceId propagation into AgentRun — produces: trace wiring — depends-on: [TASK-02].
- **TASK-06:** Audit emission via A18 adapter — produces: emission wiring — depends-on: [TASK-02, TASK-03].
- **TASK-07:** Failure-surfacing (non-2xx / dead-letter signaling) — produces: error contract — depends-on: [TASK-02, TASK-03].
- **TASK-08:** Reference Trigger manifests for both flows — produces: example Triggers — depends-on: [TASK-02, TASK-03].
- **TASK-09:** Helm/manifests (two Deployments, RBAC, ServiceAccounts) — produces: chart — depends-on: [TASK-02..07].
- **TASK-10:** 3-layer tests, dashboard XR, alerts, runbook, tutorial — produces: cross-cutting deliverables — depends-on: [TASK-09].

## 3. Dependency Map

### 3.1 Upstream that must ship first (HARD)
- A4 (Knative Eventing + NATS JetStream broker) — broker, Trigger machinery, delivery contract.
- A18 (audit adapter library) — linked for emission. A5 (ARK) / A3 (Argo) — the reconcilers of created resources (needed for end-to-end Chainsaw, mockable earlier).

### 3.2 Downstream blocked on this
- B12 (CloudEvent schema registry) — consumes B8 as the first concrete adapter exercising lifecycle schemas.

### 3.3 Continuous (non-blocking) inputs
- B14 test framework, B22 threat model.

## 4. Parallelizable Subtasks
- TASK-02 (AgentRun) and TASK-03 (Workflow) run concurrently after TASK-01.
- TASK-05/06/07 fan across both adapters once they exist.
- Dashboard/alerts/runbook/tutorial parallel with test authoring (TASK-10).

## 5. Test Strategy
- **Chainsaw:** AC-B8-01, -02, -04, -06, -07, -09, -10, -11 ("emit CloudEvent on broker → expect AgentRun CR / Workflow / audit event / lifecycle event").
- **PyTest:** AC-B8-03, -04, -05, -10 (field-mapping, idempotency, source-agnosticism, failure-surfacing logic).
- **Playwright:** AC-B8-08 (HTTP receiver `/v1/...` flow + version rejection).
- **Fakes:** fake broker/Trigger delivery harness; mock ARK (accept AgentRun create) and mock Argo (accept submit) until A5/A3 land; fake audit endpoint. Re-run against real A5/A3/A18 when present.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/1` (contains A4 spec).
### 6.2 PR — `piece/B8-knative-event-adapters` → base `wave/1`; carries spec-B8 + plan-B8.
### 6.3 Merge order — independent of W1 siblings (B1, B2, B13); wave/1 rolls up to main. B12 (W2) bases on wave/1.

## 7. Effort Estimate
- TASK-01 S, 02 M, 03 M, 04 M, 05 S, 06 S, 07 S, 08 S, 09 S, 10 M. Rollup **M**.
- Critical path: TASK-01 → 02 → 04 → 09 → 10.

## 8. Rollback / Reversibility
Scale the two adapter Deployments to zero / remove the reference Triggers; inbound CloudEvents then have no AgentRun/Workflow sink and are dead-lettered or dropped by the broker. No persistent state owned by B8. Downstream: B12 loses its first concrete adapter exemplar (spec/test reference only, not a runtime break); the §6.7 trigger flows that target these sinks stop firing until restored.
