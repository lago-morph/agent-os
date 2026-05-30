# PLAN A3 — Argo Workflows

> spec: SPEC-A3 · kind: COMPONENT · tier: T1
> wave: W0 · estimate: M
> upstream-pieces: [] · downstream-pieces: [B19]

## 1. Implementation Strategy
Install Argo Workflows unmodified via Helm as a single ArgoCD release, configure Keycloak OIDC for
its console, and expose a clean submit target for B8's CloudEvent→Workflow adapter. Prove the three
load-bearing capabilities the architecture relies on: fan-out (parallel steps), durable
long-running execution cooperating with agent-sandbox pause/resume + Postgres checkpoints, and the
wait-on-`Approval`/timeout mechanics B19 composes onto. As a W0 foundation piece with no hard
upstreams, build against stubs for `Approval` (B19), the workflow adapter (B8), and sandbox
checkpoint behavior (A6). Deliver the §14.1 set: dashboard XR, alerts, OPA admission, audit
emission, lifecycle CloudEvents, HolmesGPT toolset, three-layer tests, docs.

## 2. Ordered Task List
- **TASK-01:** Helm values + ArgoCD application for Argo Workflows — produces: install manifests — depends-on: [].
- **TASK-02:** Keycloak OIDC config for the workflow console — produces: SSO config — depends-on: [TASK-01].
- **TASK-03:** Submit-target wiring for the B8 workflow adapter — produces: submit surface — depends-on: [TASK-01].
- **TASK-04:** Fan-out workflow template + step-outcome reporting — produces: fan-out template + test — depends-on: [TASK-03].
- **TASK-05:** Long-running/durable workflow + checkpoint cooperation with agent-sandbox — produces: durable template + resume test — depends-on: [TASK-03].
- **TASK-06:** Wait-on-`Approval` + timeout mechanics surface for B19 — produces: approval-mechanics surface — depends-on: [TASK-03].
- **TASK-07:** `LogLevel` honoring on the workflow controller (ADR 0035) — produces: toggle wiring — depends-on: [TASK-01].
- **TASK-08:** Audit emission (submit/step outcomes) + `platform.lifecycle.*` CloudEvents — produces: audit + event wiring — depends-on: [TASK-03].
- **TASK-09:** `GrafanaDashboard` XR + alert rules + OPA admission Rego — produces: dashboard/alerts/Rego — depends-on: [TASK-01].
- **TASK-10:** HolmesGPT workflow-status toolset entry — produces: toolset — depends-on: [TASK-03].
- **TASK-11:** Three-layer tests + docs/runbook/tutorials — produces: tests + docs — depends-on: [TASK-04, TASK-05, TASK-06, TASK-08, TASK-09].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- None (W0).
### 3.2 Downstream pieces blocked on this
- B19 (approval system composes with Argo Workflows mechanics — ADR 0017).
### 3.3 Continuous (non-blocking) inputs
- B8 (workflow adapter — submit caller; stub until landed), A6 (sandbox checkpoint behavior; stub),
  B14 test framework, B22 threat-model standards, dotted toolset edge into A14.

## 4. Parallelizable Subtasks
- After TASK-01: TASK-02, TASK-03, TASK-07, TASK-09 run concurrently.
- After TASK-03: TASK-04, TASK-05, TASK-06, TASK-08, TASK-10 fan out independently.

## 5. Test Strategy
- **Chainsaw:** AC-A3-01 (install), AC-A3-04 (fan-out), AC-A3-05 (pause/resume), AC-A3-06 (approval wait/timeout via stub `Approval`), AC-A3-07 (LogLevel), AC-A3-09 (XR/alert).
- **Playwright:** AC-A3-02 (console OIDC).
- **PyTest:** AC-A3-03 (adapter submit), AC-A3-08 (audit + CloudEvent), AC-A3-10 (HolmesGPT tool).
- **Fixtures/fakes:** stub `Approval` CRD (B19 not landed); fake B8 adapter submitting a Workflow;
  stubbed agent-sandbox checkpoint hook; mock Keycloak.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/0`.
### 6.2 PR — `piece/A3-argo-workflows` → base `wave/0`; carries spec-A3.md + plan-A3.md.
### 6.3 Merge order — independent of W0 siblings; wave/0 rolls up to main.

## 7. Effort Estimate
- TASK-01 S · TASK-02 S · TASK-03 M · TASK-04 M · TASK-05 L · TASK-06 M · TASK-07 S · TASK-08 M · TASK-09 M · TASK-10 S · TASK-11 M.
- Rollup: **M**. Critical path: TASK-01 → TASK-03 → TASK-05 → TASK-11 (durable/resume is the longest pole).

## 8. Rollback / Reversibility
Remove the ArgoCD application to back out. Downstream impact: B19's approval workflow has no engine
to run on (B19-core blocked); triggered/long-running orchestration falls back to single AgentRun
path only (no fan-out, no durable multi-step). Knative broker (A4) and ARK (A5) single-run path are
unaffected.
