# SPEC A3 — Argo Workflows

> kind: COMPONENT · workstream: A · tier: T1
> upstream: [] · downstream: [B19] · adrs: [0017, 0031, 0035, 0034, 0030] · views: [6.7]
> canon-glossary: b0edae10a2e6 · canon-interface: 0ce201d5d5af

## 1. Purpose & Problem Statement

A3 installs and operates **Argo Workflows**, the platform's workflow engine for triggered and
long-running orchestration (architecture-overview.md §6.7, §7.2). When an incoming CloudEvent
"needs orchestration" rather than a single agent run, a thin Knative adapter submits an **Argo
Workflow** (§6.7, §7.2 sequence). Argo Workflows handles fan-out, multi-step plans, and durable
long-running execution; agent-sandbox provides pause/resume and checkpoints land in Postgres.

Argo Workflows is also the **mechanics layer of the generalized approval system** (ADR 0017): the
`Approval` CRD is reconciled by an **Argo Workflow + component B19**, where Argo handles the
workflow mechanics (wait-for-decision, timeout), a Headlamp plugin provides the approval UI, and
OPA decides who may approve. A3 is a W0 foundation component so B19 (W3) can build the approval
workflow on a working engine.

## 2. Scope

### 2.1 In scope
- Argo Workflows Helm install + values, branding, operation as a platform service.
- Keycloak OIDC config for the Argo Workflows console (per §9 OSS-gap table — basic multi-tenant
  console, high-level cross-environment views deferred to Headlamp plugins).
- Workflow-controller wiring so the Knative workflow adapter (B8) can submit Argo Workflows
  (the adapter itself is field-mapping-only and owned by B8; A3 provides the submit target).
- Support for **fan-out** (parallel workflow steps) and **long-running/durable** execution that
  cooperates with agent-sandbox pause/resume and Postgres checkpoints (§7.2).
- The Argo-side workflow mechanics that B19's approval system composes onto (wait-for-`Approval`,
  timeout firing) — A3 ships the engine + the integration surface; B19 owns approval logic.
- `LogLevel`-driven log/trace granularity on the workflow controller (ADR 0035).
- Standard §14.1 Workstream-A deliverable set (see §8).

### 2.2 Out of scope (and where it lives instead)
- The CloudEvent→Argo Workflow adapter — **Knative event adapter services (B8)**.
- The `Approval` CRD, OPA elevation, decision CloudEvent emission, approval-queue UI —
  **B19 (B19-core W3 / B19-ui W4)**; A3 only provides the workflow engine it runs on.
- Agent execution itself — **ARK (A5)**; Argo orchestrates, it does not run agents directly.
- Sandbox pause/resume + checkpointing — **agent-sandbox (A6)** + Postgres (B4 `XPostgres`).
- GitOps sync — **ArgoCD** (baseline / referenced); A3 is *Argo Workflows*, a distinct product.
- Promotion across environments — **Kargo (A23)**.

## 3. Context & Dependencies

**Upstream consumed:** none (W0 foundation).

**Downstream consumers:**
- **B19** — Generalized approval system composes with Argo Workflows for human-gate mechanics
  (ADR 0017); B19's upstream set includes A3.

**ADR decisions honored:**
- **ADR 0017** — generalized approval system via `Approval` CRD + Argo Workflows + OPA elevation.
  A3 supplies the Argo Workflows mechanics; the model is kept intentionally thin (no M-of-N, no
  escalation timers in v1.0).
- **ADR 0031** — workflow lifecycle CloudEvents fall under `platform.lifecycle.*` (Workflow
  created/started/completed/failed) and approval events under `platform.approval.*` (owned by
  B19). Per-event names deferred to B12.
- **ADR 0035** — workflow-controller verbosity is gated by `LogLevel`; in-process vs
  rolling-restart pattern is determined by Argo's reload behavior `[PROPOSED — not in source:
  Argo's hot-reload capability not stated]`.
- **ADR 0034** — Argo audit-relevant actions (workflow submit, step outcomes) emit via the audit
  adapter; no direct writes to audit stores.
- **ADR 0030** — A3 ships no platform CRD (Argo's own `Workflow` CRD is upstream-owned, not a
  platform CRD); versioning applies to any glue HTTP surface.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
- A3 defines **no platform CRD**. Argo Workflows ships its own upstream `Workflow` CRD — this is
  vendor-owned, not in the platform Canon CRD inventory (§6.12); reproduce no invented fields.
- A3 **consumes** `Approval` (owner: Argo Workflow + B19) — `requestingAgent`, `actionType`,
  `actionAttributes`, `defaultLevel`, `evidenceRefs[]`, `decision`, `decidedBy`, `decidedAt` —
  the Argo Workflow waits on the `decision`/timeout. A3 provides the engine; B19 owns the CRD.
- A3 consumes `LogLevel` (ADR 0035) and delivers a `GrafanaDashboard` XR (B4-owned XRD).

### 4.2 APIs / SDK surfaces
- **Argo Workflows submit/query API** (upstream product API) — the target B8's workflow adapter
  submits to; surfaced to the console via OIDC. No new platform HTTP API beyond Argo's own.
- Argo Workflows console — OIDC against Keycloak; cross-environment views via Headlamp (B5).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- **Emitted:** `platform.lifecycle.*` for Workflow created/started/completed/failed
  `[PROPOSED — not in source: specific event names deferred to B12]`.
- **Consumed:** workflow-submission is driven by the Knative Trigger→adapter (B8) path; the
  adapter submits an Argo Workflow (§7.2). Approval-decision consumption is B19's domain.
- Per-event-type names/schemas owned by **B12**.

### 4.4 Data schemas / connection-secret contracts
- Long-running workflow checkpoints land in **Postgres** (§7.2) via `XPostgres` connection secret
  (host/port/user/password/dbname) where A3 needs persistence `[PROPOSED — not in source: Argo's
  own persistence/archive backing is design-time]`.
- No platform-owned data schema introduced.

## 5. OSS-vs-Custom Decision
- **Named project:** Argo Workflows (glossary: "Workflow engine (A3)").
- **Mode:** install + configure + operate (Helm); unmodified upstream, no fork.
- **Version:** pin a tested version `[PROPOSED — not in source: exact version design-time]`.
- **ADR linkage:** ADR 0017 (approval mechanics); §6.7/§7.2 (orchestration role).
- **Rationale:** chosen workflow engine for fan-out and durable long-running orchestration; the
  approval system reuses its mechanics rather than building a bespoke state machine.

## 6. Functional Requirements
- **REQ-A3-01:** A3 SHALL install Argo Workflows via Helm with platform values as a single
  ArgoCD-synced release.
- **REQ-A3-02:** A3 SHALL configure the Argo Workflows console with Keycloak OIDC SSO.
- **REQ-A3-03:** A3 SHALL provide a submit target the Knative workflow adapter (B8) can call to
  launch an Argo Workflow from a CloudEvent (§6.7, §7.2).
- **REQ-A3-04:** A3 SHALL support fan-out workflows (parallel steps) and report per-step outcomes.
- **REQ-A3-05:** A3 SHALL support long-running/durable workflows that cooperate with agent-sandbox
  pause/resume and Postgres checkpointing (§7.2) — a paused workflow resumes without re-running
  completed steps.
- **REQ-A3-06:** A3 SHALL provide the workflow mechanics B19 composes onto for human gates: a
  workflow can wait on an `Approval` `decision` and fire on timeout (ADR 0017) — without
  implementing approval policy itself.
- **REQ-A3-07:** A3 SHALL honor a `LogLevel` CR adjusting workflow-controller log/trace
  granularity (ADR 0035).
- **REQ-A3-08:** A3 SHALL emit audit events for workflow submit and step outcomes via the audit
  adapter (ADR 0034) and `platform.lifecycle.*` CloudEvents for workflow lifecycle.
- **REQ-A3-09:** A3 SHALL deliver a per-component Grafana dashboard (`GrafanaDashboard` XR) and
  alert rules for controller failure conditions (controller down, workflow stuck/failed surge).
- **REQ-A3-10:** A3 SHALL contribute a HolmesGPT toolset entry for querying workflow status.

## 7. Non-Functional Requirements
- **Security / multi-tenancy (§6.9):** console OIDC-gated; workflows run namespace-scoped; RBAC
  floor + OPA restriction (ADR 0018). Per-tenant workflow isolation via namespace.
- **Observability (§6.5):** controller emits OTel spans/metrics through the standard collector;
  workflow spans carry `trace_id` for Tempo correlation where applicable.
- **Scale:** support concurrent fan-out and long-running workflows; concrete concurrency target
  deferred to F5. Long-running termination must not drop in-flight steps.
- **Versioning (ADR 0030):** pin tested upstream version; URL-path versioning on any glue surface.

## 8. Cross-Cutting Deliverable Checklist (§14.1)
- Helm/manifests — **applicable**.
- Per-product docs (10.5) — **applicable**.
- Runbook (10.7) + backup/restore — **applicable** (workflow archive/checkpoints in Postgres
  inherit its backup/PITR posture).
- Alerts — **applicable** (controller down, stuck/failed workflows).
- Grafana dashboard (Crossplane XR) — **applicable**.
- Headlamp plugin — **N/A as A3-owned editor** — cross-environment workflow views are B5's
  cross-cutting plugin scope; no A3 row in ADR 0039's editor set. Native Argo console covers
  operational views.
- OPA/Rego integration — **applicable** (admission on workflow resources; who-may-approve gating
  is B19/OPA, not A3).
- Audit emission (ADR 0034) — **applicable** (workflow submit/step outcomes).
- Knative trigger flow — **applicable** (the "needs orchestration → submit Argo Workflow" flow,
  §7.2; emits `platform.lifecycle.*`).
- HolmesGPT toolset — **applicable** (workflow-status query tool).
- 3-layer tests (Chainsaw/Playwright/PyTest) — **applicable**.
- Tutorials & how-tos — **applicable** ("trigger an agent on a schedule / from an event",
  "set up a long-running agent with checkpointing" — §10.1).

## 9. Acceptance Criteria
- **AC-A3-01 (REQ-A3-01):** ArgoCD sync brings Argo Workflows controller to Ready as one release.
  *(Chainsaw)*
- **AC-A3-02 (REQ-A3-02):** Unauthenticated console access redirects to Keycloak; an OIDC login
  succeeds. *(Playwright)*
- **AC-A3-03 (REQ-A3-03):** A simulated B8 adapter call submits a Workflow that runs to completion.
  *(PyTest + Chainsaw)*
- **AC-A3-04 (REQ-A3-04):** A fan-out workflow with N parallel steps completes and reports N step
  outcomes. *(Chainsaw)*
- **AC-A3-05 (REQ-A3-05):** A long-running workflow paused mid-run resumes from checkpoint without
  re-executing completed steps. *(Chainsaw + PyTest)*
- **AC-A3-06 (REQ-A3-06):** A workflow waiting on an `Approval` proceeds when `decision=approved`
  is written and fires the timeout branch when no decision arrives in the configured window.
  *(Chainsaw, using a stub `Approval` CRD until B19 lands)*
- **AC-A3-07 (REQ-A3-07):** Applying a `LogLevel` CR raises controller verbosity and reverting
  restores baseline. *(Chainsaw + PyTest)*
- **AC-A3-08 (REQ-A3-08):** A workflow submit produces an audit event via the adapter and a
  `platform.lifecycle.*` CloudEvent. *(PyTest)*
- **AC-A3-09 (REQ-A3-09):** The `GrafanaDashboard` XR renders; a simulated controller-down fires
  the alert. *(Chainsaw + PyTest)*
- **AC-A3-10 (REQ-A3-10):** HolmesGPT can query a running workflow's status via the toolset.
  *(PyTest)*

## 10. Risks & Open Questions
- **R1 (med):** ADR 0035 reload pattern for the Argo controller is unstated → `[PROPOSED]`;
  in-process vs rolling-restart confirmed during implementation.
- **R2 (med):** Long-running pause/resume correctness depends on agent-sandbox (A6) checkpoint
  semantics; A3↔A6 cooperation contract is `[PROPOSED — not in source]` at field level. Blast
  radius: durable workflows; mitigated by stubbing A6 in tests.
- **R3 (low):** Argo's own persistence/archive backing (Postgres) shape is design-time
  `[PROPOSED]`.
- **R4 (low):** B19 builds the approval workflow onto A3; A3 must keep the wait/timeout surface
  stable. Coordinated at the `Approval` CRD contract level (interface-contract §1.5).

## 11. References
- architecture-overview.md §6.7 (eventing, lines ~553–630); §7.2 (triggered and long-running,
  lines ~1078–1155); §9 OSS-gap table (Argo Workflows row, line ~1411); §14.1 (A3 row, line ~1669).
- ADR 0017 (generalized approval system — Argo mechanics); ADR 0031 (CloudEvent taxonomy);
  ADR 0035 (log/trace toggle); ADR 0034 (audit); ADR 0030 (versioning).
- View V6-07 (Eventing architecture).
- Related pieces: B19 (approval system), B8 (workflow adapter), A6 (sandbox pause/resume),
  A5 (ARK), B4 (XPostgres/GrafanaDashboard), A4 (Knative broker).
