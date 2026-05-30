# PLAN B2 — LiteLLM custom callbacks

> spec: SPEC-B2 · kind: COMPONENT · tier: T1
> wave: W1 · estimate: L
> upstream-pieces: [A1, A7] · downstream-pieces: []

## 1. Implementation Strategy

Build Python callback classes that register into LiteLLM's `pre-call`/`post-call`/`success`/`failure` hook chain (§6.1), delivering the four gateway gap-fills: PII scrub (Presidio), audit emission via the A18 adapter (ADR 0034), the OPA runtime-decision bridge (§6.6), and content guardrails. Audit and OPA are mandatory and non-bypassable; PII/guardrails are policy-tunable. Implement the structured dry-run path so A20's policy simulator can compose the gateway layer, and the budget-exceeded CloudEvent for the §6.7 flow. Because B2 ships configured into A1's chart but may be authored before A1 integration is ready, build against a documented mock LiteLLM hook host first (placement-fallback) and re-run the same suite against A1 when present. Config rolls via Reloader (ADR 0035) since LiteLLM has no in-process reload.

## 2. Ordered Task List

- **TASK-01:** Define callback host abstraction + mock LiteLLM hook host (placement-fallback) — produces: hook host shim + fixtures — depends-on: [].
- **TASK-02:** Audit-emission callback via A18 adapter (mandatory) — produces: audit callback — depends-on: [TASK-01].
- **TASK-03:** OPA bridge callback (auth/rate/content/budget enforcement) + dry-run mode — produces: OPA callback — depends-on: [TASK-01].
- **TASK-04:** PII-scrub callback (Presidio) — produces: PII callback — depends-on: [TASK-01].
- **TASK-05:** Content-guardrail callback — produces: guardrail callback — depends-on: [TASK-01].
- **TASK-06:** Budget-exceeded CloudEvent emission (`platform.observability.*`) — produces: budget event emitter — depends-on: [TASK-03].
- **TASK-07:** ConfigMap wiring + Reloader rolling-restart against replicas>=2 — produces: chart fragments — depends-on: [TASK-02..05].
- **TASK-08:** `LogLevel` toggle honoring — produces: verbosity wiring — depends-on: [TASK-02..05].
- **TASK-09:** Re-wire against real A1 pipeline; verify suite unchanged — produces: A1 integration — depends-on: [TASK-02..06].
- **TASK-10:** 3-layer tests, dashboard XR, alerts, runbook, how-to — produces: cross-cutting deliverables — depends-on: [TASK-09].

## 3. Dependency Map

### 3.1 Upstream that must ship first (HARD)
- A1 (LiteLLM) — hook pipeline, ConfigMap, replicas>=2 Deployment, spend tracking. **Placement-fallback:** if A1 not ready, develop against mock host (TASK-01); A1 required for final integration (TASK-09).
- A7 (OPA) — runtime decision endpoint + decision shape (incl. dry-run `simulated:true`).
- A18 (audit adapter library) — linked for emission (ADR 0034).

### 3.2 Downstream blocked on this
- None per piece-index.csv. Operationally: §6.7 budget→email flow; A17 MCP services audit path.

### 3.3 Continuous (non-blocking) inputs
- B16 (OPA policy content), B12 (CloudEvent schemas), A20 (policy simulator dry-run shape), B14 test framework, B22 threat model.

## 4. Parallelizable Subtasks
- TASK-02, -03, -04, -05 (the four callbacks) run concurrently after TASK-01.
- TASK-06 depends only on TASK-03; TASK-08 fans across the callbacks.
- Dashboard/alerts/runbook parallel with test authoring (TASK-10).

## 5. Test Strategy
- **PyTest:** AC-B2-01, -02, -04, -05, -06, -08, -11, -12, -13 (callback logic, mock host, dry-run, fail-closed semantics).
- **Playwright:** AC-B2-02, -03, -08 (§13 pattern: "POST to LiteLLM, expect OPA decision, expect audit event").
- **Chainsaw:** AC-B2-07, -09, -10, -11 (CloudEvent on broker, ConfigMap/Reloader rollout, LogLevel toggle).
- **Fakes:** mock LiteLLM hook host (TASK-01), fake OPA returning canned decisions incl. `simulated:true`, fake audit endpoint, in-memory broker assertion. Replace with A1/A7/A18 when landed; re-run unchanged (AC-B2-12).

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/1` (contains A1, A7 specs).
### 6.2 PR — `piece/B2-litellm-callbacks` → base `wave/1`; carries spec-B2 + plan-B2.
### 6.3 Merge order — independent of W1 siblings (B1, B8, B13); wave/1 rolls up to main.

## 7. Effort Estimate
- TASK-01 M, 02 M, 03 L, 04 M, 05 M, 06 S, 07 M, 08 S, 09 M, 10 M. Rollup **L**.
- Critical path: TASK-01 → 03 → 06/09 → 10 (the OPA bridge + budget event + A1 integration).

## 8. Rollback / Reversibility
Remove the callback classes from the LiteLLM ConfigMap and roll the Deployment. Reverting **disables mandatory audit and OPA enforcement on the gateway** — a security/compliance regression, not just a feature loss — so rollback is gated and must be paired with a compensating control or gateway pause. No persistent state owned by B2; downstream budget→email flow and MCP-audit path stop receiving events until restored.
