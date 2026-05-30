# PLAN ADR-0004 — NATS JetStream as the Knative broker backend [PROPOSED]

> spec: SPEC-ADR-0004 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [] · downstream-pieces: [A19;B8;B12]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0004 is enforced by component A4 (single NATS JetStream broker behind Knative Eventing, same install in kind and managed Kubernetes, declarative stream config), with adapters (B8) and the schema registry (B12) building above the Trigger layer. Conformance is proven by: (a) a same-topology diff across substrates; (b) end-to-end dispatch under each `platform.*` namespace; (c) declarative-config-only drift checks; (d) an idempotency/redelivery test. No new build work belongs to this ADR — it is a constraint map over A4.

## 2. Ordered Task List
- TASK-01: Map each REQ to the enforcing component piece — produces: enforcement matrix — depends-on: []
- TASK-02: Specify same-broker-topology check across kind and managed Kubernetes — produces: manifest-diff conformance (A4) — depends-on: [TASK-01]
- TASK-03: Specify per-namespace publish→Trigger→sink dispatch test — produces: e2e suite (A4/B8) — depends-on: [TASK-01]
- TASK-04: Specify declarative-stream-config drift check (GitOps reverts hand edits) — produces: Chainsaw check (A4) — depends-on: [TASK-01]
- TASK-05: Specify at-least-once redelivery / idempotency test — produces: PyTest + e2e (B8) — depends-on: [TASK-01]
- TASK-06: Specify backend-encapsulation assertion (Trigger filters/taxonomy unchanged on swap) — produces: harness test (B12) — depends-on: [TASK-01]

## 3. Dependency Map
### 3.1 Upstream that must ship first (HARD)
- None — A4 is a foundation component.
### 3.2 Downstream blocked on this
- A19, B8, B12.
### 3.3 Continuous (non-blocking) inputs
- ADR 0023 environment-specific sources; B14 test framework; observability stack (§6.5) for broker metrics.

## 4. Parallelizable Subtasks
TASK-02, TASK-03, TASK-04, TASK-05, TASK-06 fan out independently after TASK-01.

## 5. Test Strategy
- AC-01/03 → Chainsaw (topology diff; declarative-config drift revert).
- AC-02/05/06 → PyTest + e2e (per-namespace dispatch; backend-swap encapsulation).
- AC-04 → PyTest (duplicate-redelivery idempotency).
- AC-06 (metrics) → PyTest assertion against the observability stack.
Fixtures: reference idempotent consumer; fake event source until ADR 0023 sources land.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0004-nats-jetstream` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR PRs; rolls up to main.

## 7. Effort Estimate
TASK-01..06 each S. Rollup: S. Critical path: TASK-01 → TASK-03.

## 8. Rollback / Reversibility
Backing out means re-opening the broker-backend choice (Kafka/RabbitMQ/SNS+SQS); because broker selection is encapsulated below the Trigger layer, a swap does not rewrite triggers or adapters, so reversibility is moderate. Downstream breakage is limited to operational (stream sizing/retention/runbooks), not the taxonomy or Trigger contracts.
