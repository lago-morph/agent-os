# PLAN A2 — Langfuse

> spec: SPEC-A2 · kind: COMPONENT · tier: T1
> wave: W0 · estimate: M
> upstream-pieces: [] · downstream-pieces: [B1, B6, B9, B10]

## 1. Implementation Strategy
Install Langfuse unmodified via Helm as a single ArgoCD release, wire its state to the shared
Postgres through B4's `Postgres` connection secret, expose its ingestion endpoint as the
LLM-grade trace sink, and enforce the ADR 0015 `trace_id` correlation invariant on the receiving
side. Because A2 is a W0 foundation piece with no hard upstreams, build proceeds against fakes for
not-yet-landed consumers (B6/B10) and a local Postgres fixture standing in for `Postgres`. Deliver
the standard §14.1 set: dashboard XR, alerts, OPA admission, audit emission of admin actions,
observability self-instrumentation, a HolmesGPT lookup tool, three-layer tests, and docs.

## 2. Ordered Task List
- **TASK-01:** Author Helm values + ArgoCD application for Langfuse — produces: install manifests — depends-on: [].
- **TASK-02:** Wire Langfuse to `Postgres` connection secret (host/port/user/password/dbname) — produces: store wiring — depends-on: [TASK-01].
- **TASK-03:** Configure ingestion endpoint + key issuance for SDK/gateway emitters — produces: ingest config — depends-on: [TASK-01].
- **TASK-04:** Implement/verify `trace_id`-preserving ingest (no rewrite) + Tempo join contract — produces: correlation contract test — depends-on: [TASK-03].
- **TASK-05:** Configure Grafana/Tempo→Langfuse deep-link targets — produces: deep-link config — depends-on: [TASK-04].
- **TASK-06:** Implement `LogLevel` honoring (rolling-restart pattern) on ingest path — produces: ADR 0035 wiring — depends-on: [TASK-03].
- **TASK-07:** SSO handoff surface for oauth2-proxy (B1) + JWT-claim UI scoping — produces: SSO surface — depends-on: [TASK-01].
- **TASK-08:** `GrafanaDashboard` XR + alert rules (ingest/store/backlog) — produces: dashboard XR + alerts — depends-on: [TASK-01].
- **TASK-09:** OPA admission Rego for A2 workloads + audit emission of admin actions — produces: Rego + audit wiring — depends-on: [TASK-01].
- **TASK-10:** HolmesGPT Langfuse-lookup toolset entry — produces: toolset — depends-on: [TASK-04].
- **TASK-11:** Three-layer tests + per-product docs/runbook/tutorial — produces: tests + docs — depends-on: [TASK-04, TASK-06, TASK-07, TASK-08].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- None (W0). Soft/foundation: B4 `Postgres` connection secret (fake with a local Postgres until B4 lands).
### 3.2 Downstream pieces blocked on this
- B1 (SSO in front of Langfuse), B6 (SDK OTel emission target), B9 (CLI trace links), B10 (Coach trace observation).
### 3.3 Continuous (non-blocking) inputs
- B14 test framework; B22 security threat-model standards; dotted toolset edge into A14 (HolmesGPT).

## 4. Parallelizable Subtasks
- After TASK-01: TASK-02, TASK-03, TASK-07, TASK-08, TASK-09 run concurrently (independent fan-out).
- TASK-04→TASK-05 and TASK-04→TASK-10 are serial on the correlation contract.
- TASK-06 parallel with TASK-04/05.

## 5. Test Strategy
- **Chainsaw:** AC-A2-01 (install/Ready), AC-A2-07 (LogLevel rolling restart), AC-A2-09 (XR + alert).
- **Playwright:** AC-A2-06 (deep link), AC-A2-08 (SSO + claim scoping).
- **PyTest:** AC-A2-02, -03, -04, -05, -10 (store, ingest, trace_id correlation, payload filtering, HolmesGPT tool).
- **Fixtures/fakes:** local Postgres standing in for `Postgres`; fake emitter posting spans with a fixed `trace_id`; stub Tempo span for the join assertion; mock oauth2-proxy/Keycloak for SSO.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/0`.
### 6.2 PR — `piece/A2-langfuse` → base `wave/0`; carries spec-A2.md + plan-A2.md.
### 6.3 Merge order — independent of W0 siblings (A1/A3/A13 etc.); wave/0 rolls up to main.

## 7. Effort Estimate
- TASK-01 S · TASK-02 S · TASK-03 M · TASK-04 M · TASK-05 S · TASK-06 M · TASK-07 S · TASK-08 M · TASK-09 M · TASK-10 S · TASK-11 M.
- Rollup: **M**. Critical path: TASK-01 → TASK-03 → TASK-04 → TASK-11.

## 8. Rollback / Reversibility
Back out by removing the ArgoCD application; Langfuse state in Postgres is retained or dropped by
choice. Downstream impact: LLM-grade tracing and Langfuse deep-links go dark; B10 (Coach) loses its
observation source and B6 OTel emission to Langfuse no-ops. Tempo/Mimir (A13) and audit (A18) are
unaffected — correlation is one-directional convenience, not a hard dependency for those paths.
