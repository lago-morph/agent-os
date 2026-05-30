# SPEC ADR-0035 — Dynamic log-level and trace-granularity toggle with staged restart for non-reloadable services `[PROPOSED]`

> kind: ADR · workstream: — · tier: T1
> upstream: [A1; A6; A13] · downstream: [A18; A14; B6; B9; B13; A22] · adrs: [0035] · views: [6.5]
> canon-glossary: FROZEN · canon-interface: FROZEN

## 1. Purpose & Problem Statement

ADR 0035 is a settled decision: every Platform Component declares a **toggle pattern** for raising
log verbosity and trace granularity on demand and lowering it again, preserving the "no performance
impact when observability is off" invariant. Two patterns are supported — an **in-process toggle**
(preferred) and a **rolling-restart toggle** with staged-restart guardrails for runtimes that cannot
reload in place (LiteLLM). This SPEC states what honoring the decision requires of each component,
HolmesGPT, the Test CLI, and Headlamp — it does not re-argue the toggle model.

The problem the decision solves: loggers and tracers are conditionally instantiated, so a runtime
level change only takes effect after a component re-reads config; components differ in reload
capability. One decision covers both cases so admins/HolmesGPT can raise verbosity on a specific
component during an incident and drop it afterward, without always-on tracing.

## 2. Scope

### 2.1 In scope
- The per-component "toggle pattern" declaration as a required component-template deliverable.
- The in-process toggle path via the `LogLevel` CRD (or per-component admin endpoint), SIGHUP-style re-read.
- The rolling-restart toggle path with the staged-restart guardrails for non-reloadable runtimes.
- Trace granularity following the same conditional-instantiation model as log level.
- One toggle mechanism, two callers: HolmesGPT diagnostics and the Test CLI debug/tracing flag.
- Headlamp surfacing current effective level + the same write path; audit of every level change.

### 2.2 Out of scope (and where it lives instead)
- The shared in-process reload implementation — provided by the Platform SDK (component B6); each component links it.
- The audit pipeline that records level changes — owned by ADR 0034 / component A18.
- Trace/metric backends (Tempo/Mimir/Langfuse) that consume the resulting telemetry — owned by A13 / ADR 0015.
- HolmesGPT runbook content and the Test CLI build — owned by A14 and B9 respectively.

## 3. Context & Dependencies

Upstream consumed: A1 (LiteLLM — the canonical rolling-restart-toggle component); A6 (agent-sandbox
+ Envoy — components carrying the toggle); A13 (Tempo + Mimir — trace/metric sinks granularity feeds).
Downstream consumers: A18 (audit endpoint — in-process toggle target + records level changes), A14
(HolmesGPT — primary toggle caller), B6 (SDK — shared reload impl), B9 (Test CLI — debug flag), B13
(kopf operator — in-process toggle target), A22 (Headlamp editor surface for `LogLevel`).

ADR decisions honored:
- **ADR 0035** (this): per-component toggle-pattern declaration; in-process preferred; rolling-restart with guardrails for non-reloadable runtimes.
- **ADR 0030**: `LogLevel` CRD is versioned per-component-owner like every CRD.
- **ADR 0034**: every level change is an auditable event; the audit endpoint is itself an in-process toggle target.
- **ADR 0011**: the Test CLI gains a debug/tracing flag using the same `LogLevel` surface; re-asserts default on teardown.
- **ADR 0012**: HolmesGPT is the primary automated caller; raising a level is a policy-controlled write (§6.10).
- **ADR 0015**: raising granularity activates spans that were not being created (not sampling always-on spans), so Tempo/Langfuse cost scales with verbosity, not traffic.
- **ADR 0006**: the kopf operator is an in-process toggle target.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
- **`LogLevel` (CRD)**, namespaced, reconciled per-component (in-process or rolling-restart). Source-stated
  fields: `componentSelector`, `level`, `traceGranularity`, `scope` (component / tenant / eventClass),
  `expiresAt`. Versioned per the owning component (ADR 0030). A per-component admin endpoint is an
  accepted alternative surface to the CRD for in-process components.

### 4.2 APIs / SDK surfaces
- The Platform SDK (B6) provides the shared in-process reload implementation so components do not
  re-invent it; it re-reads logging + tracing config in place (SIGHUP-style) on a `LogLevel` change.
- One mechanism, two callers: HolmesGPT (A14) and the Test CLI (B9) both write the same `LogLevel`
  CRD / admin-endpoint surface. The Test CLI debug/tracing flag activates the toggle for one test
  run and re-asserts the default on teardown.
- Concrete reload-method signatures are **not specified in source** `[PROPOSED — not in source]`.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- Every level change is an auditable event recorded via the audit adapter under **`platform.audit.*`**
  (ADR 0034): who raised verbosity, when, on what, for how long. `[PROPOSED — not in source]` whether
  a level change also emits under `platform.observability.*`; the audit obligation is the stated one.

### 4.4 Data schemas / connection-secret contracts
N/A — no connection-secret surface. The toggle carries level/granularity/scope/expiry via the
`LogLevel` CRD (§4.1), not a data store.

## 5. OSS-vs-Custom Decision
N/A — ADR (cross-component toggle pattern). The shared reload path is custom Python in the SDK (B6);
LiteLLM (A1) is wrapped with the rolling-restart guardrails rather than modified. Those calls live
in the realizing components' SPECs.

## 6. Functional Requirements

- REQ-ADR-0035-01: Every Platform Component MUST declare a toggle pattern (in-process or rolling-restart) as part of its component definition; a component PR that does not declare its pattern is incomplete.
- REQ-ADR-0035-02: Components whose runtime supports it MUST use the in-process toggle: a `LogLevel` CRD or admin endpoint instructs the component to re-read logging+tracing config in place, with no pod restart and no in-flight request impact.
- REQ-ADR-0035-03: Returning the level to default MUST drop the re-instantiated loggers/tracers via conditional instantiation, preserving the "no impact when off" invariant.
- REQ-ADR-0035-04: Components that cannot reload in process (LiteLLM) MUST use the rolling-restart toggle: update a ConfigMap/env var and trigger a rolling Deployment restart, with guardrails `replicas >= 2`, readiness probes that drop the pod from Service endpoints before SIGTERM, `preStop: sleep 10–15s`, and `terminationGracePeriodSeconds` sized to the worst-case streaming response (60–300s).
- REQ-ADR-0035-05: A single-replica LiteLLM MUST NOT be a supported configuration once this ADR lands.
- REQ-ADR-0035-06: Trace granularity MUST follow the same model: raising granularity activates spans that were not being created, NOT sampling on always-on spans (ADR 0015), so Tempo/Langfuse cost scales with verbosity not traffic.
- REQ-ADR-0035-07: There MUST be one toggle mechanism with two callers — HolmesGPT (A14) and the Test CLI (B9) MUST use the same `LogLevel` CRD / admin-endpoint surface.
- REQ-ADR-0035-08: The Test CLI debug/tracing flag MUST activate the toggle for one test run and re-assert the default on teardown (forgetting to disable is not possible).
- REQ-ADR-0035-09: Raising a level via HolmesGPT MUST be a policy-controlled write like any other (§6.10).
- REQ-ADR-0035-10: Every level change MUST be emitted as an auditable event under `platform.audit.*` regardless of pattern (ADR 0034).
- REQ-ADR-0035-11: Headlamp MUST surface the current effective level per component and provide the same write path admins would otherwise use via `kubectl`.

## 7. Non-Functional Requirements
- Performance: "no impact when off" is invariant — conditional instantiation, no always-on sampled tracing.
- Availability: rolling-restart guardrails ensure a verbosity change drains in-flight streaming responses (up to `terminationGracePeriodSeconds`) rather than dropping them; readiness dropped before SIGTERM.
- Security/multi-tenancy (§6.9): `LogLevel.scope` supports component / tenant / eventClass granularity; HolmesGPT writes are policy-gated.
- Observability (§6.5, §6.10): Headlamp shows effective level; HolmesGPT runbooks treat the declared pattern as authoritative.
- Versioning (ADR 0030): `LogLevel` is versioned per owning component.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR. The "toggle pattern" declaration becomes a review checkpoint embedded in every
COMPONENT's own §14.1 deliverable set; conformance to this ADR is verified per §9. The shared reload
impl (B6), the LiteLLM guardrails (A1), the Test CLI flag (B9), and the Headlamp surface (A22) carry
their deliverables in their own SPECs.

## 9. Acceptance Criteria

- AC-ADR-0035-01: Honored when a component-review gate rejects a component that ships without a declared toggle pattern. (→ REQ-01)
- AC-ADR-0035-02: Honored when an in-process component re-reads config on a `LogLevel` change with no pod restart and no in-flight request impact, and returns to "no impact when off" when the level is lowered. (→ REQ-02, REQ-03)
- AC-ADR-0035-03: Honored when LiteLLM's Deployment carries `replicas >= 2`, readiness-before-SIGTERM, `preStop: sleep 10–15s`, and the raised `terminationGracePeriodSeconds`, and a verbosity change drains an in-flight streaming response without dropping it. (→ REQ-04)
- AC-ADR-0035-04: Honored when a single-replica LiteLLM configuration is rejected as unsupported. (→ REQ-05)
- AC-ADR-0035-05: Honored when raising trace granularity is shown to create spans that were previously not created at all, not to enable sampling on existing spans. (→ REQ-06)
- AC-ADR-0035-06: Honored when HolmesGPT and the Test CLI are shown to drive the same `LogLevel` surface, and the Test CLI re-asserts the default on teardown after a single run. (→ REQ-07, REQ-08)
- AC-ADR-0035-07: Honored when a HolmesGPT-initiated level change is shown to pass through the §6.10 policy gate. (→ REQ-09)
- AC-ADR-0035-08: Honored when every level change appears as an audit event under `platform.audit.*` recording who/when/what/duration. (→ REQ-10)
- AC-ADR-0035-09: Honored when Headlamp displays the current effective level per component and an admin can change it through Headlamp's write path. (→ REQ-11)

## 10. Risks & Open Questions
- (med) Components whose runtime can neither reload in process nor tolerate a rolling restart reopen this decision (ADR 0035 revisit trigger); none known in v1.0.
- (low) Streaming responses in flight at a verbosity change take up to `terminationGracePeriodSeconds` to drain — visible as a slightly delayed old-pod shutdown, not a dropped response.
- (low) `[PROPOSED]` — whether a level change also emits under `platform.observability.*`, and the exact reload-method signatures, are not specified in source; flagged for B6 / B12 design.

## 11. References
- ADR 0035 (this decision). Enforcing/realizing components: A1 (LiteLLM rolling-restart), B6 (shared reload impl), A18 (audit endpoint toggle + level-change audit), A14 (HolmesGPT caller), B9 (Test CLI flag), B13 (kopf operator), A22 (Headlamp `LogLevel` surface).
- architecture-overview.md §6.5 (observability), §6.10 (HolmesGPT). ADR 0006, 0011, 0012, 0015, 0030, 0031, 0034.
