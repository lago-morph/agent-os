# PLAN B12 — CloudEvent schema registry

> spec: SPEC-B12 · kind: COMPONENT · tier: T0
> wave: W2 · estimate: M
> upstream-pieces: [A4, B8] · downstream-pieces: [B19]

## 1. Implementation Strategy
B12 is a Git-versioned JSON-Schema registry plus validation/publish tooling that enforces ADR 0031's
closed ten-namespace taxonomy and ADR 0030's dual versioning (`specversion` + `schemaVersion`,
mint-on-break). The work is contract-first: freeze the registry layout, schema-id convention, and
conformance/versioning rules before seeding content, because B19 and every emitter bind to them. B12
does not author other components' event names — it hosts and enforces. Seed only the Canon/exemplar
schemas: `platform.capability.changed` (B6/B7 contract) and the two v1.0 trigger-flow exemplars. The
namespace-conformance validator (exactly-one Canon namespace, reject ad-hoc top-level), the versioning
validator (additive-minor / new-type-on-break), and a contribution CI gate are the load-bearing
deliverables. Build-new tooling wrapping JSON-Schema + CloudEvents; never coin a top-level namespace.

## 2. Ordered Task List
- **TASK-01:** Freeze the **registry layout**: directory/namespace structure for the ten Canon
  namespaces, schema-id convention, registry index, envelope rules (`specversion` + `schemaVersion`)
  — produces: registry layout spec + skeleton — depends-on: [].
- **TASK-02:** Implement the **namespace-conformance validator** (every type under exactly one Canon
  namespace; reject non-Canon top-level) — produces: conformance validator — depends-on: [TASK-01].
- **TASK-03:** Implement the **versioning validator** (additive→minor `schemaVersion`; breaking→new
  event type; backward-compat detection) — produces: versioning validator — depends-on: [TASK-01].
- **TASK-04:** Build the **contribution CI gate + publishing surface** (drop-a-schema → validate →
  publish for emitter/external-subscriber consumption) — produces: CI gate + publish tooling —
  depends-on: [TASK-02, TASK-03].
- **TASK-05:** Author the **`platform.capability.changed`** schema (affected Agent/CapabilitySet
  identifiers) as the B6/B7 contract — produces: capability-changed schema — depends-on: [TASK-01].
- **TASK-06:** Author the **two v1.0 exemplar** flow schemas (budget-exceeded; AlertManager→HolmesGPT)
  under Canon namespaces — produces: exemplar schemas — depends-on: [TASK-01].
- **TASK-07:** **Docs**: registry layout, namespace catalog, how-to-add-an-event-schema, versioning
  rules, schema-validation-failure runbook — produces: §10.5 docs + runbook — depends-on: [TASK-04].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- **A4 (Knative Eventing + NATS JetStream broker)** — transport for the schematized CloudEvents.
- **B8 (Knative event adapter services)** — consume CloudEvents and field-map; validate against B12.

### 3.2 Downstream pieces blocked on this
- **B19** (`platform.approval.*` schemas). Indirectly every emitter (B2, B13, A18, A20, ARK) and the
  capability-change consumers B6/B7.

### 3.3 Continuous (non-blocking) inputs
- **B14** test framework; **B22** threat model (`platform.security.*` event requirements).

## 4. Parallelizable Subtasks
After TASK-01: TASK-02 and TASK-03 run concurrently; TASK-05 and TASK-06 (content seeding) run
concurrently with the validators. TASK-04 joins TASK-02+TASK-03. TASK-07 is the serial tail.

## 5. Test Strategy
- **PyTest:** conformance validator rejects non-Canon namespace and missing-envelope schemas
  (AC-B12-02, AC-B12-03); versioning validator passes additive / fails breaking-without-new-type
  (AC-B12-04); contribution CI gate end-to-end (AC-B12-05); publish/retrieve round-trip (AC-B12-06);
  `platform.capability.changed` validates a sample event (AC-B12-07); exemplar schemas validate
  (AC-B12-08); no-invented-names review check (AC-B12-09); registry-loads check (AC-B12-01).
- **Chainsaw:** N/A — no CRD/operator.
- **Playwright:** N/A — no UI.
- **Fixtures/fakes:** sample valid + invalid event schemas, a sample `platform.capability.changed`
  event payload (the B6/B7 contract fixture), stub published-schema store for the subscriber test.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/2` (contains A4/B8 specs)
### 6.2 PR — `piece/B12-cloudevent-schema-registry` → base `wave/2`; carries spec-B12 + plan-B12
### 6.3 Merge order — independent of W2 siblings (B6/B9/B16/B17); wave/2 rolls up to main; B19 (W3)
branches after B12 merges.

## 7. Effort Estimate
TASK-01 M · TASK-02 M · TASK-03 M · TASK-04 M · TASK-05 S · TASK-06 S · TASK-07 S. Rollup: **M**.
Critical path: TASK-01 → TASK-03 → TASK-04 → TASK-07.

## 8. Rollback / Reversibility
Registry is Git-versioned; back out by reverting the schema/tooling commits. Reverting a published
schema is constrained by the mint-on-break rule (never mutate a published type) — roll forward with a
new event type instead. Downstream emitters/subscribers pin a `schemaVersion`, so a tooling revert
does not invalidate already-published schemas. B19 blocks only on the approval-namespace schemas it
authors here.
