# PLAN OPS6 — Rego (OPA) Human-Factors & Lifecycle

> spec: SPEC-OPS6 · kind: NFR (cross-cutting operational layer) · tier: T0
> wave: post-W4 (operational/NFR layer; authoring-parallel) · estimate: L
> upstream-pieces: [A7, B3, B16, B2, B13, A6, A1, B19, A20, A22, B14, B9, A18, B12, B22]
> downstream-pieces: [—] (cross-cutting; obligations land back into the upstream pieces)

## 1. Implementation Strategy

OPS6 is realized not by building a new component but by **closing the lifecycle around the existing policy plane** and proving the safety invariants the spec asserts. The work is three threads run over the already-landed policy pieces: (1) make the Rego CI gate set **required and complete** (B3 owns the gates; B14 runs them; OPS6 verifies they block) — unit tests, SHA-pin, RBAC-floor conformance, decision-contract conformance, coverage-map; (2) make rollout **staged and reversible** (Gatekeeper audit-mode → enforce promotion over A7, A20 preview, GitOps revert-by-SHA via ArgoCD) and make the **change-authorization** path real (Headlamp/OPA action-gating from B16 + the B19 `Approval` gate for privileged/global changes); (3) build the **anti-lockout** safety net — a pre-promotion self-lockout simulation (over A20) and a documented break-glass recovery contract. Because the change-control model, global-bundle ownership, signing, the promotion fabric, and the break-glass contract are genuinely open, OPS6 ships the **obligations + verification harness** now and gates the mechanism choices behind the five PROPOSED-ADRs; the signing task (TASK-09) is conditional on its ADR. Everything binds to Canon names; nothing edits the upstream specs (their absorbed obligations are recorded as cross-links for the parent to apply).

## 2. Ordered Task List

- **TASK-01:** Inventory the realized policy plane against SPEC §3 — confirm A7 audit-mode/dry-run, B3 gate set + decision contract, B16 coverage map + action-gating + `approval.elevation`, B19 `Approval`, A20 simulator, B14 CI, B9 CLI dry-run all exist and expose what OPS6 binds to — produces: realization-gap matrix — depends-on: [].
- **TASK-02:** Specify the **required CI gate set** as merge-blocking (rego-unit, sha-pin-verify, rbac-floor-conformance, decision-contract-conformance, coverage-map-complete) and the planted-violation fixtures that prove each blocks — produces: CI gate definition + negative fixtures (verifies REQ-02..05) — depends-on: [TASK-01].
- **TASK-03:** Define the **preview-before-commit** loop — wire a policy change through A20 simulator / structured dry-run and assert per-layer delta with `simulated:true`, zero enforcement side effect — produces: preview procedure + test (REQ-06) — depends-on: [TASK-01].
- **TASK-04:** Define the **staged-rollout mechanism** — audit-mode/dry-run stage emitting prospective `platform.policy.*` decisions, then an explicit audited promotion to enforce; document promotion as a distinct action — produces: staged-rollout runbook + Chainsaw flow (REQ-07) — depends-on: [TASK-01]; **mechanism-choice gated on PROPOSED-ADR (promotion fabric)**.
- **TASK-05:** Define **rollback** — `git revert` + ArgoCD re-sync to last-known-good SHA-pinned revision within the rollback-time target; audited — produces: rollback runbook + Chainsaw test (REQ-08) — depends-on: [TASK-04].
- **TASK-06:** Define **change-authorization** — Headlamp/OPA action-gating from Platform JWT roles (B16) for who-may-change-which-surface; route privileged/global changes through the B19 `Approval` (elevate-only) — produces: authorization map + policy-change `actionType` proposal + tests (REQ-09, REQ-10, REQ-14) — depends-on: [TASK-01]; **global-surface ownership gated on PROPOSED-ADR**.
- **TASK-07:** Build the **anti-lockout self-lockout check** — model the policy-plane operability set (author/reviewer/rollback identities, admission webhook liveness, break-glass), and a pre-promotion A20 simulation that blocks a self-lockout — produces: operability-set artifact + pre-promotion gate + test (REQ-11) — depends-on: [TASK-03, TASK-06].
- **TASK-08:** Define the **break-glass recovery contract** — primary path GitOps revert-by-SHA; bounded/namespaced/audited fail-policy or Gatekeeper-exemption fallback; every use emits `platform.security.*` — produces: break-glass runbook + Chainsaw recovery test (REQ-12) — depends-on: [TASK-05, TASK-07]; **fail-policy interaction gated on A7 fail-policy decision + PROPOSED-ADR**.
- **TASK-09 (conditional):** **Bundle signing/verification** — build at load, verify signature at load, refuse on mismatch; key custody → OPS3 — produces: signing step + conditional CI test (REQ-15) — depends-on: [TASK-02]; **conditional on the bundle-signing PROPOSED-ADR being accepted**.
- **TASK-10:** Define the **lifecycle observability** — `platform.policy.*` events for proposed→simulated→audit-mode→enforce→rolled-back with author/approver attribution (via A18/B12); Grafana dashboard obligation for change volume + post-promotion deny-rate delta — produces: event-name proposals + dashboard obligation + test (REQ-13) — depends-on: [TASK-04, TASK-06].
- **TASK-11:** Author the **cross-link obligations** — the exact NFR obligation each of A7/B3/B16/B14/B19/A20 must absorb (no edits here; recorded for the parent) and the operator runbook set — produces: cross-link/obligations note + runbooks — depends-on: [TASK-02..TASK-10].

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD) — id + what is consumed
- **A7** — OPA/Gatekeeper engine, **audit-mode + dry-run admission** (the staging substrate), SHA-pinned GitOps bundle loading, admission fail-policy.
- **B3** — bundle `.manifest` revision + SHA-pin tooling, RBAC-floor predicate, decision contract + dry-run wrapper, how-to-add-a-policy convention, the unit-test + pin CI gates OPS6 elevates to "required + complete".
- **B16** — the v1.0 rule corpus, §6.6 coverage map, Headlamp action-gating policies (who-may-change), `approval.elevation` policy.
- **B19-core** — the `Approval` human gate (Argo suspend/resume, `platform.approval.*`, elevate-only) a privileged policy change routes through.
- **A20** — simulator aggregator/dry-run for preview (TASK-03) and the self-lockout simulation (TASK-07).
- **B14** — `agent-platform` test framework: the CI harness that runs the Rego gates.
- **A18 / B12** — audit adapter + event schemas for `platform.policy.*` lifecycle emission.

### 3.2 Downstream pieces blocked on this — ids
None as build pieces (cross-cutting). The obligations flow **back** into A7/B3/B16/B14/B19/A20 as NFR absorptions (recorded by TASK-11 for the parent to apply); B22's threat-model output (ADR 0027) lands as new policy targets the lifecycle must accommodate without lockout.

### 3.3 Continuous (non-blocking) inputs
- **B22 (security threat model)** — feeds policy targets continuously (ADR 0027); OPS6's change-control must absorb additions without rewrite.
- **B14 (test framework)** — continuous CI substrate.
- **B9 (`agent-platform` CLI)** — the promotion-gate/dry-run surface used in staged rollout and preview.
- **A22 / ADR 0039** — Headlamp editors authors use; non-blocking for the verification harness.
- **Sibling OPS pieces:** **OPS3** (signing-key custody for TASK-09), **OPS5** (day-2 upgrade/promotion ops) — coordinate, non-blocking.

## 4. Parallelizable Subtasks

- After TASK-01, three independent fan-out groups:
  - **Gating group:** TASK-02 (→ TASK-09 conditional).
  - **Rollout group:** TASK-03 → TASK-04 → TASK-05.
  - **Authz group:** TASK-06.
- **Convergence:** TASK-07 (anti-lockout) needs TASK-03 + TASK-06; TASK-08 (break-glass) needs TASK-05 + TASK-07; TASK-10 needs TASK-04 + TASK-06. TASK-11 closes after all. TASK-09 runs in parallel with the rollout group **iff** its ADR is accepted.

## 5. Test Strategy (ACs → layers)

- **PyTest (logic / Rego unit):** AC-OPS6-02, -03, -04, -05 (gate set + planted-violation fixtures); -06 (dry-run no-side-effect); -09 (authz role checks); -10 (elevate-only); -11 (self-lockout logic); -13 (lifecycle event emission); -14 (CapabilitySet-snippet parity). Fakes: where a real upstream layer's dry-run is not exercised in CI, use a fake dry-run honoring `simulated:true` (per A20 R1 pattern).
- **Chainsaw (operator/CRD):** AC-OPS6-01 (only-GitOps-mutates-bundle), -07 (audit-mode→enforce promotion), -08 (revert-by-SHA rollback within target), -11 (self-lockout caught at admission promotion), -12 (break-glass recovery from a broken enforce state).
- **Playwright (UI/e2e):** AC-OPS6-06 (Headlamp simulator panel preview), -09 (Headlamp action-gating denies an out-of-scope author).
- **CI (conditional):** AC-OPS6-15 — bundle signature-verification refuse-on-mismatch; **marked deferred until the signing ADR is accepted**.
- **Fixtures/fakes for not-yet-landed paths:** reuse A20's fake-layer harness and B16's planted-granting-rule fixture; a deliberately-broken enforce bundle for the break-glass test; an out-of-scope tenant JWT for the authz tests.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch
Base = **`wave/4`** (the highest real-build wave; OPS6 is a post-W4 cross-cutting layer and consumes pieces landing through W4 — A23/B5/B19-ui plus the W0–W3 policy plane). Per `_meta/pending-operational-nfr-layer.md`, the operational/NFR layer runs after the 119-piece spec/plan work, verification, and PR graph are complete.

### 6.2 PR
`piece/OPS6-rego-lifecycle` → base `wave/4`; carries **only** `specs/ops/spec-OPS6.md` + `plans/ops/plan-OPS6.md` (this piece's two new files). No edits to existing specs/plans/source — the cross-link obligations (TASK-11) are recorded in this plan for the parent to apply as separate follow-ups, not committed as edits here.

### 6.3 Merge order
OPS6 is one of six sibling OPS PRs (OPS1–OPS6), each its own stacked PR off the post-W4 base, **mutually independent** (no OPS piece consumes another; OPS3/OPS5 are coordinate, not blocking). The OPS layer rolls up after the component waves merge to main. Within OPS6, all changes are in the two authored files, so the PR is atomic.

## 7. Effort Estimate

- TASK-01 S · TASK-02 M · TASK-03 S · TASK-04 M · TASK-05 S · TASK-06 M · TASK-07 L · TASK-08 M · TASK-09 M (conditional) · TASK-10 S · TASK-11 S.
- **Rollup: L.** Critical path: TASK-01 → TASK-03 → TASK-07 → TASK-08 → TASK-11 (the anti-lockout/break-glass safety chain is the long pole; it depends on both the preview loop and the authorization map and is the highest-blast-radius work).

## 8. Rollback / Reversibility

Backing OPS6 out removes the two authored files only; no running system changes (it adds no deployable). If the **obligations** have already been absorbed by upstream pieces (via the TASK-11 follow-ups), reverting OPS6 does not by itself loosen any gate — but it removes the *requirement* that those gates stay required/complete, so the platform could regress to ungated/un-staged policy changes over time. The high-value, hard-to-revert artifacts are the **anti-lockout self-lockout check (TASK-07)** and the **break-glass contract (TASK-08)**: if those are removed after the policy plane has come to rely on them, a subsequent bad enforce-promotion regains platform-wide lockout potential (SPEC R1). Therefore TASK-07/-08 should be reverted only together with a documented alternative recovery path. The signing step (TASK-09) is independently reversible (drop the verification step; bundles fall back to SHA-pin-only, the ADR 0002 baseline).
