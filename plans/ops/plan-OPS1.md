# PLAN OPS1 — Scale & Cost

> spec: SPEC-OPS1 · kind: COMPONENT (cross-cutting NFR layer) · tier: T0 (NFR)
> wave: post-W4 / consumer (authoring-parallel; real-build after the A/B components it constrains) · estimate: L
> upstream-pieces: [A1, A4, A6, A18, B4, B13, A5] · downstream-pieces: [F5, D1, D2, D3, F1, F4, F6]

## 1. Implementation Strategy

OPS1 is realized as a **cross-cutting NFR contract, not a component**: it adds no deployable. The
strategy is (a) author the scale & cost architecture (capacity model, autoscaling stance,
attribution/showback, quota↔budget envelope, right-sizing) as a spec that *constrains* named owning
pieces; (b) for every REQ, record the **single realization seam** — a CRD field, an OPA/Rego rule,
a `GrafanaDashboard` XR panel set, an F5 measurement, or a CI/review gate — and the owning spec that
must absorb it; (c) surface every genuinely-open decision as a PROPOSED-ADR title (SPEC §10) rather
than deciding it. Because all primitives already exist in Canon (LiteLLM spend, `BudgetPolicy`,
`AgentEnvironment.quotas`, `SandboxTemplate` knobs, `GrafanaDashboard` XR, Mimir/Tempo/Langfuse), the
build work is **wiring + authoring distributed across owning specs + F5 measurement**, and this PLAN
is dominated by a **realization/verification map** (§3, §5) rather than a net-new task ladder. The
layer lands **after W4** so every component it constrains exists and F5 can measure against AWS/EKS.

## 2. Ordered Task List

`TASK-NN: <description> — produces: <artifact> — depends-on: [refs]`

- **TASK-01:** Fix the capacity model — name the four load-bearing paths (A1/A6/A4/A18), state each
  expected v1.0 target (`[PROPOSED]` where unpinned), bind each to its F5 measurement. — produces:
  SPEC §6 REQ-OPS1-01..03 + the F5 cross-ref table — depends-on: [].
- **TASK-02:** Define the autoscaling stance per load-bearing component (scale-eligible vs
  single-instance-by-ADR-0035; trigger-signal class; chokepoint fail-closed constraint; not-autoscaled
  dependencies). — produces: REQ-OPS1-04..07 + PROPOSED-ADR (autoscaler mechanism) — depends-on: [TASK-01].
- **TASK-03:** Specify the cost-attribution chain (`VirtualKey.ownerIdentity`→`capabilitySetRef`→
  `platform_tenants`) and the showback (not chargeback) model. — produces: REQ-OPS1-08..11 + the
  attribution-join contract for B13/A1 + PROPOSED-ADR (attribution join / aggregation site) — depends-on: [].
- **TASK-04:** Define the Cost dashboard panel set (tenant/team/agent/model, budget-vs-actual, burn
  rate, RBAC+OPA visibility) as a `GrafanaDashboard`-XR obligation on the D-series. — produces:
  REQ-OPS1-09 panel contract — depends-on: [TASK-03].
- **TASK-05:** Define the quota↔budget capacity envelope (`AgentEnvironment.quotas` +
  `ResourceQuota`/`LimitRange` + `BudgetPolicy:tenant`), precedence rule, the five quota dimensions
  with `[PROPOSED]` defaults, and the independent-enforcement rule. — produces: REQ-OPS1-12..15 +
  PROPOSED-ADRs (quota realization, default values) — depends-on: [].
- **TASK-06:** Specify right-sizing guidance (`SandboxTemplate` knobs ← F5 numbers) and the Capacity
  dashboard panel set. — produces: REQ-OPS1-16..17 — depends-on: [TASK-01].
- **TASK-07:** Write the realization map — bind every REQ to a (mechanism, owning-spec) pair and record
  which owning specs must absorb an obligation. — produces: PLAN §3 + REQ-OPS1-18..20 — depends-on:
  [TASK-01..06].
- **TASK-08:** Author the test/verification mapping (ACs → Chainsaw/Playwright/PyTest/F5/review). —
  produces: PLAN §5 — depends-on: [TASK-07].
- **TASK-09:** Author the cross-cutting runbooks seam ("cost spike traced through virtual keys";
  capacity planning) feeding F6, and the per-product scale-&-cost doc. — produces: docs/runbook stubs
  routed to F6/C-series — depends-on: [TASK-03, TASK-05].
- **TASK-10:** Finalize PROPOSED-ADR titles + risks; confirm no auto-decided architectural choice. —
  produces: SPEC §10 — depends-on: [TASK-02, TASK-03, TASK-05].

Critical path: TASK-01 → TASK-02/05 (the two highest-blast-radius open decisions) → TASK-07 → TASK-08.

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD — what is consumed)

OPS1 is a **post-build NFR layer**: it constrains pieces that must already exist to be measured/wired.
These are realization dependencies, not wave-build edges into OPS1.

- **A1 (LiteLLM, W0)** — per-key/per-model spend (attribution source of record, REQ-OPS1-08/11), the
  `BudgetPolicy` two-layer enforcement + `platform.observability.*` threshold events (REQ-OPS1-15), the
  ADR-0035 `replicas >= 2` baseline (REQ-OPS1-05).
- **A6 (agent-sandbox, W0)** — `SandboxTemplate.warmPoolSize`/`hibernationEnabled`/`resourceLimits`
  (right-sizing, REQ-OPS1-16); cold-start path (REQ-OPS1-01).
- **A4 (Knative+NATS, W0)** — broker backlog/lag capacity path (REQ-OPS1-01); `platform.observability.*` backlog signal (REQ-OPS1-17).
- **A18 (audit endpoint+adapter, W1)** — audit in-flight + batch keep-up capacity path (REQ-OPS1-01).
- **B13 (kopf operator, W1)** — sole writer of `BudgetPolicy`/`VirtualKey` and the `ownerIdentity`
  resolvability the attribution join depends on (REQ-OPS1-08, R3).
- **B4 (Crossplane, W1)** — owns `AgentEnvironment.quotas` (compute-quota half, REQ-OPS1-12) and the
  `GrafanaDashboard` XR the Cost/Capacity dashboards are realized as (REQ-OPS1-09/17).
- **A5 (ARK, W1)** — `Agent`/`AgentRun` as the counted compute unit for max-agents quota (REQ-OPS1-13).

### 3.2 Downstream pieces blocked on / consuming this

- **F5 (Scale evaluation, T2/consumer)** — measures OPS1's capacity targets (REQ-OPS1-01/02); F5 OQ1/OQ2
  are answered by OPS1's targets + the candidate-SLI PROPOSED-ADR. *Tightest coupling.*
- **D1/D2 (operator dashboards)** — must author the Cost (§11.1 L1556) and Capacity (L1558)
  `GrafanaDashboard` XR panel sets OPS1 mandates (REQ-OPS1-09/17).
- **D3 (test-framework health)** — renders F5's capacity probe metrics.
- **F1 (retention sizing), F4 (DoS baseline), F6 (capacity runbook)** — consume F5's numbers, which trace to OPS1's targets.

### 3.3 Continuous (non-blocking) inputs

- **B14 (test framework)** — the Chainsaw/Playwright concurrency harness F5 + OPS1's quota-admission
  tests run on (ADR 0011); continuously available per waves.md fan-out.
- **B22 (security threat model)** — frames capacity/budget exhaustion as a DoS attack pattern (§6.6 L449);
  OPS1's envelope is the first line, B22/F4 the response. Continuous input.
- **A13 (Mimir/Tempo) + A2 (Langfuse)** — the telemetry surfaces OPS1's attribution/capacity views read; continuously available.

### 3.4 Wave placement rationale

waves.md has no NFR band; the closest fit is **consumer / post-W4** (like F-series): OPS1 constrains
W0–W1 chokepoints and W0 runtime, but can only be *measured and wired* once those plus the D-series
dashboards and F5 exist. It is **authoring-parallel** (the spec can be written now against frozen Canon)
but **real-build after A/B/D and alongside/after F5** — mirroring how F5 itself sits in the consumer band.

## 4. Parallelizable Subtasks

Independent fan-out groups (no inter-dependency):
- **Group A — Capacity & right-sizing:** TASK-01, TASK-06 (share the F5 cross-ref; otherwise independent).
- **Group B — Autoscaling:** TASK-02 (depends only on TASK-01).
- **Group C — Cost/showback:** TASK-03 → TASK-04 (Cost dashboard follows attribution).
- **Group D — Quota envelope:** TASK-05 (fully independent of A/B/C).

Groups A, C, D start concurrently; B follows A; TASK-07/08 (map + tests) join all groups and are the
serialization point; TASK-09/10 (docs + ADR finalization) close out.

## 5. Test Strategy

Map ACs (SPEC §9) → layers. Capacity measurement is **F5's** (Chainsaw/Playwright concurrency, ADR 0011);
OPS1 adds attribution + quota-admission verification. Fixtures noted where an upstream is faked.

| AC | Mechanism / realization | Layer | Fixture / note |
|---|---|---|---|
| AC-OPS1-01 | four-path capacity model bound to F5 | review + F5 cross-ref | F5 measures on AWS/EKS (ADR 0033) |
| AC-OPS1-02 | each target stated; unpinned `[PROPOSED]` + owner | review | confirm-before-gap (mirrors F5 OQ1) |
| AC-OPS1-03 | no SLO asserted | review / lint ("SLO" only as deferred) | — |
| AC-OPS1-04 | per-component scale stance + signal class | review | — |
| AC-OPS1-05 | gateway scale-out keeps ADR-0035 constraints; no chokepoint bypass | **Chainsaw** + cross-ref A6 AC-A6-05 | scale gateway replicas, assert egress still only via LiteLLM/Envoy |
| AC-OPS1-06 | autoscaler choice is PROPOSED-ADR only | review | no HA-beyond-2 committed |
| AC-OPS1-07 | OPA/kopf/NATS/Postgres listed not-autoscaled | review | — |
| AC-OPS1-08 | attribution join `ownerIdentity`→`platform_tenants` | **PyTest** | fake LiteLLM spend rows + JWT claim map; assert every row → one tenant |
| AC-OPS1-09 | Cost `GrafanaDashboard` XR (tenant/team/agent/model; visibility) | **Playwright** + **Chainsaw** | render panels; assert tenant-B can't load tenant-A view (XR `visibility`) |
| AC-OPS1-10 | showback-only / chargeback `[PROPOSED]` out | review | — |
| AC-OPS1-11 | single spend source; enterprise gap via B2 callback | review + **PyTest** | fake A1 spend surface; no second store |
| AC-OPS1-12 | envelope = quota ∘ budget + precedence | review + **Chainsaw** | tenant hits one limit, constrained without the other |
| AC-OPS1-13 | five quota dimensions; defaults `[PROPOSED]` | review | — |
| AC-OPS1-14 | quota/budget admission denies over-limit, never grants beyond RBAC | **Chainsaw** | `ResourceQuota`/Gatekeeper reject over-quota `Agent`/`Sandbox`/`VirtualKey` |
| AC-OPS1-15 | quota=admission-deny; budget=`thresholdActions` next call (independent) | **Chainsaw** (quota) + cross-ref A1 AC-A1-12 (budget) | two independent paths |
| AC-OPS1-16 | right-sizing cites `SandboxTemplate` knobs ← F5 | review + cross-ref F5 AC-F5-02 | — |
| AC-OPS1-17 | Capacity `GrafanaDashboard` XR (warm-pool/cold-start/headroom/backlog/queue) | **Playwright** + **Chainsaw** | — |
| AC-OPS1-18 | no new CRD/XRD/namespace/SDK; non-Canon `[PROPOSED]` | review / lint | — |
| AC-OPS1-19 | every REQ maps to an owning spec | review vs PLAN §3/§6 | no orphan obligation |
| AC-OPS1-20 | open decisions = PROPOSED-ADR titles, undecided | review | — |

Fakes needed where upstream not yet landed: a **LiteLLM spend-row fixture** (A1) for the attribution
PyTest; a **JWT claim-map fixture** (`platform_tenants`) for the join; a **`GrafanaDashboard` XR test
harness** (B4) for the dashboard Chainsaw; a **`ResourceQuota`/Gatekeeper** namespace for quota admission.

### 5.1 Realization map (REQ → enforcement/measurement → owning spec that absorbs it)

| REQ | Enforced/measured by | Owning spec(s) to absorb |
|---|---|---|
| 01–02 (capacity targets) | F5 measurement on AWS/EKS | **F5** (already scopes the four paths) |
| 03 (no SLO) | review/lint | OPS1 self; future §3 |
| 04–07 (autoscaling stance) | deployment shape + ADR; PROPOSED-ADR | **A1/A4/A6/A18 charts** + new ADR |
| 08, 11 (attribution chain) | attribution join on LiteLLM spend | **B13** (`ownerIdentity` resolvability) + **A1** (spend surface) |
| 09 (Cost dashboard) | `GrafanaDashboard` XR | **D1/D2** (panels) + **B4** (XR) |
| 10 (showback-only) | scope statement | OPS1 self |
| 12–15 (quota envelope) | `AgentEnvironment.quotas` + `ResourceQuota` + `BudgetPolicy`; Gatekeeper/OPA | **B4** (`quotas` shape) + **A5/A6** (counted units) + **B3/B16** (Rego) + **A1** (budget action) |
| 16 (right-sizing) | `SandboxTemplate` knobs ← F5 | **A6** + **F5** |
| 17 (Capacity dashboard) | `GrafanaDashboard` XR | **D1/D2** + **B4** |
| 18–20 (non-coining / map / ADRs) | review/lint | OPS1 self |

## 6. PR / Branch Mapping

### 6.1 Stack position

- Base branch = **`wave/ops`** (a new post-W4 NFR band branch off `main` after `wave/4` has rolled up,
  carrying the OPS1..OPS6 NFR layer). Rationale: every piece OPS1 constrains (W0–W4) and the F5/D-series
  realizers exist on `main` by then; the NFR layer stacks cleanly on top with no back-edge into the wave graph.
- `[PROPOSED — not in source]` the `wave/ops` band name — waves.md defines no NFR band; this mirrors the
  consumer-band treatment of F-series. Confirm with the PR-graph owner.

### 6.2 PR

- **PR `piece/OPS1-scale-and-cost` → base `wave/ops`** — carries **only** this piece's two new files
  (`specs/ops/spec-OPS1.md`, `plans/ops/plan-OPS1.md`). It edits no existing spec/plan/source (constraint
  honored). The obligations OPS1 places on owning specs (D-series Cost/Capacity panels, B4 `quotas` sub-shape,
  B13 `ownerIdentity` resolvability) are **recorded as realization seams here** and land as follow-up edits in
  those pieces' own PRs — not in this PR.
- One PR per OPS piece (OPS1..OPS6), each a sibling on `wave/ops`, per `pending-operational-nfr-layer.md`
  ("Produce its own stacked PRs in the graph (one per OPS piece)").

### 6.3 Merge order

- OPS1..OPS6 are **within-band siblings, independent** (each is a self-contained cross-cutting spec+plan);
  they merge into `wave/ops` in any order. `wave/ops` rolls up to `main` once the NFR band is complete.
- The PROPOSED-ADRs OPS1 surfaces are **not** gated by this PR — they are authored only if the user accepts
  the titles (per the layer's "list titles … author only if asked" rule), as their own ADR PRs.

## 7. Effort Estimate

| Task | Size | Note |
|---|---|---|
| TASK-01 capacity model | S | reuses F5's path set |
| TASK-02 autoscaling stance | M | high-blast-radius open decision → PROPOSED-ADR |
| TASK-03 attribution/showback | M | the identity-join is the subtle part (R3) |
| TASK-04 Cost dashboard contract | S | panel set onto D-series XR |
| TASK-05 quota envelope | M | two open PROPOSED-ADRs (realization, defaults) |
| TASK-06 right-sizing + Capacity dash | S | knobs + panel set |
| TASK-07 realization map | M | the cross-cutting binding work |
| TASK-08 test mapping | S | mostly cross-refs to F5 + thin new tests |
| TASK-09 runbook/doc seams | S | routed to F6/C-series |
| TASK-10 ADR/risk finalize | S | titles only, no authoring |

Rollup: **L** (no code, but broad cross-cutting binding + two/three genuinely-open decisions to frame without
deciding). Critical path within the piece: TASK-01 → TASK-02/05 → TASK-07 → TASK-08.

## 8. Rollback / Reversibility

- OPS1 adds **no deployable and edits no existing artifact** — reverting is deleting the two files; nothing
  downstream *breaks at runtime*. What is lost on revert is the **contract**: the capacity targets F5 measures
  against, the attribution/showback obligation on the D-series, and the quota-envelope rule — i.e. those pieces
  would lack a coherent scale/cost spec to absorb, re-opening the dispersed-NFR problem (SPEC §1).
- Because the obligations land as **edits in the owning pieces' own PRs** (not here), reverting OPS1 does not
  un-ship any quota/dashboard that already merged elsewhere; it only removes the governing spec. The PROPOSED-ADRs,
  if accepted later, would need OPS1 (or its successor) as the rationale anchor — so revert-then-reauthor is the
  reversible path, with no migration or data loss (OPS1 owns no state, no CRD, no schema).
