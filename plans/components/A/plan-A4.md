# PLAN A4 — Knative Eventing + NATS JetStream broker

> spec: SPEC-A4 · kind: COMPONENT · tier: T0
> wave: W0 · estimate: L
> upstream-pieces: [] · downstream-pieces: [A19, B8, B12]

## 1. Implementation Strategy

Install Knative Eventing and NATS JetStream via Helm as a single broker that is identical in dev (kind) and prod (managed K8s), then provision the platform `Broker` and the declarative Trigger-filtering contract (including OPA-driven filtering). A4 owns the transport and the routing contract; sources (ADR 0023, owned by consumers) and adapters (B8) are out of scope. Because A4 is W0 foundation with no upstreams, it is built first. Where the audit endpoint (A18) and OPA engine (A7) have not yet landed, A4 uses a stub audit sink and a stub OPA decision endpoint, swapping them when those components land. The two initial trigger flows are validated against B8's adapters as they arrive; A4 ships a reference echo-consumer to prove the path end to end in isolation.

## 2. Ordered Task List

- **TASK-01:** Helm install Knative Eventing (controller + Broker CRDs) — produces: Knative install — depends-on: []
- **TASK-02:** Helm install NATS JetStream — produces: broker backend — depends-on: []
- **TASK-03:** Provision the single platform `Broker` backed by JetStream — produces: Broker instance — depends-on: [TASK-01, TASK-02]
- **TASK-04:** Declarative stream sizing/retention config (GitOps) — produces: stream config — depends-on: [TASK-02]
- **TASK-05:** Trigger-filtering contract by `type` namespace prefix + reference echo-consumer — produces: routing + fixture — depends-on: [TASK-03]
- **TASK-06:** OPA-driven Trigger filtering support (stub OPA until A7) — produces: policy-routing hook — depends-on: [TASK-05]
- **TASK-07:** Idempotency / at-least-once consumer contract + redelivery test harness — produces: delivery contract — depends-on: [TASK-05]
- **TASK-08:** Broker arrival + dispatch-outcome audit emission via adapter (stub until A18) — produces: audit hooks — depends-on: [TASK-03]
- **TASK-09:** Gatekeeper admission wiring for Knative/Trigger CRs — produces: admission — depends-on: [TASK-03]
- **TASK-10:** §14.1 deliverables (docs, runbook, alerts, dashboard XR, Headlamp integration, HolmesGPT toolset, tutorials) — produces: deliverable set — depends-on: [TASK-08]
- **TASK-11:** 3-layer test suite mapping all ACs — produces: tests — depends-on: [TASK-10]

Critical path: TASK-01/02 → 03 → 05 → 07 → 11.

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD)
None (W0 foundation). Upstream Knative + NATS OSS + k8-platform baseline.

### 3.2 Downstream pieces blocked on this
B8 (adapters + initial flows), B12 (schema registry routing target), A19 (Mattermost bridge). Indirectly: every CloudEvent emitter.

### 3.3 Continuous (non-blocking) inputs
- A18 (audit endpoint) — stub until landed.
- A7 (OPA) / B16 (Rego) — stub OPA decision until landed for OPA-driven filtering.
- B12 (schemas) — A4 routes on namespace prefix only, so it does not block on schemas.
- B14 test framework, B22 threat model — continuous inputs.

## 4. Parallelizable Subtasks

- TASK-01 ∥ TASK-02 (independent installs).
- After TASK-03: TASK-04 (stream config) ∥ TASK-05 (routing) ∥ TASK-09 (admission).
- After TASK-05: TASK-06 (OPA filtering) ∥ TASK-07 (idempotency) ∥ TASK-08 (audit).

## 5. Test Strategy

| AC | Layer | Fixtures / fakes |
|---|---|---|
| AC-A4-01,02,03,08,09,10 | Chainsaw | kind + managed-K8s targets; fake source (webhook/PingSource); Gatekeeper |
| AC-A4-04,05,06,07 | PyTest | reference echo-consumer; stub OPA; stub audit endpoint; duplicate-event injector |
| Headlamp Broker/Trigger view | Playwright | seeded Triggers; admin session |

Fakes for not-yet-landed upstreams: stub audit endpoint (A18), stub OPA decision endpoint (A7), reference consumer standing in for B8 adapters.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/0`.
### 6.2 PR — `piece/A4-knative-nats-broker` → base `wave/0`; carries spec-A4.md + plan-A4.md.
### 6.3 Merge order — W0 siblings independent; `wave/0` rolls up to main. B8/A19/B12 build on the merged Broker contract in later waves.

## 7. Effort Estimate

- S: TASK-04, TASK-09. M: TASK-01, TASK-02, TASK-03, TASK-06, TASK-07, TASK-08. L: TASK-05, TASK-10, TASK-11.
- Rollup: **L** (matches piece-index). Critical path ≈ TASK-01/02→03→05→07→11.

## 8. Rollback / Reversibility

Back out by reverting the Knative + NATS Helm releases. Because the broker is **in-flight transport, not a system of record** (ADR 0014), rollback loses only undelivered in-flight events; durable platform state is unaffected. **Reverting A4 breaks the entire eventing fabric** — no Trigger flows, no audit fanout routing, no Mattermost bridge, no AgentRun-from-event or Workflow-from-event adapters (B8). It is a platform-eventing-down event; prefer redeploy-in-place over delete-and-recreate to preserve stream durability config.
