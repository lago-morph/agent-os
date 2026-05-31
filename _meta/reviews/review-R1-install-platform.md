# Review R1 — Install / Platform Core (A1, A4, A6, A7)

> Reviewer: independent red-team, second pass · Date: 2026-05-31
> Scope: T0 contract pieces A1 (LiteLLM gateway), A4 (Knative + NATS JetStream),
> A6 (agent-sandbox + Envoy egress), A7 (OPA / Gatekeeper).
> Canon basis: `_meta/glossary.md`, `_meta/interface-contract.md`, `_meta/waves.md`,
> `_meta/piece-index.csv` rows A1/A4/A6/A7.
> Method: interface soundness vs interface-contract.md + named up/downstream pieces;
> AC verifiability + REQ→AC coverage; load-bearing `[PROPOSED]` blast radius;
> internal / dependency-direction contradictions vs waves.md + csv.

Severity legend: HIGH (must fix before freeze / breaks a cross-piece contract or leaves
a REQ unverifiable), MED (real risk, fix soon), LOW (polish / tracking).

---

## A1 — LiteLLM (gateway) — verdict: NEEDS-WORK

A1 is well-aligned with Canon: CRD field tables in §4.1 reproduce interface-contract §1.4
verbatim, the closed CloudEvent namespaces are respected, `XPostgres` consumption matches
§1.6/§4, and the "B13 sole writer / kopf as subchart" model matches ADR 0006 and the B13 spec.
The problems are coverage gaps and one cross-piece contract that A1 leaves implicit.

- **HIGH — REQ-A1-15 observability is partly unverifiable as written / metrics not in any AC.**
  REQ-A1-15 asserts OTel→Tempo, LLM-traces→Langfuse, and "Prometheus metrics scraped into Mimir."
  AC-A1-15 checks correlated Tempo+Langfuse spans and "gateway metrics appear in Mimir," but
  "metrics appear in Mimir" depends on A13 (Mimir/Tempo) and A2 (Langfuse), which A1 lists only as
  non-blocking continuous inputs (§3). In an isolated W0 build (per plan-A1 fakes) there is no
  Mimir/Tempo/Langfuse to assert against, so the AC is not objectively verifiable at A1's own gate.
  Fix: split AC-A1-15 into (a) A1-side assertion — gateway *exposes* a Prometheus `/metrics`
  endpoint and emits OTel spans with a stable `trace_id` (verifiable in isolation), and (b) an
  integration assertion gated on A2/A13 presence. State the dependency explicitly.

- **HIGH — A1↔A4 broker dependency is contradictory across specs and unmodeled in the csv.**
  A1 §4.3 and REQ-A1-12/13/14 require A1 to *emit* CloudEvents (`platform.gateway.*`,
  `platform.observability.*`, `platform.policy.*`) and the budget-exceeded→email flow is routed "by
  Triggers (A4)." Emitting onto the broker is a hard runtime dependency on A4's Broker contract, yet
  **A4 does not appear in A1's upstream and A1 does not appear in A4's downstream** (csv: A1 upstream
  empty; A4 downstream = A19;B8;B12). Both are W0 so build order is fine, but the *interface*
  dependency (A1 produces events onto A4's broker) is undocumented on both sides. A1 §4.3 hand-waves
  this ("if the broker is down, A1 buffers/drops non-critical events") without naming A4 as the
  transport owner. Fix: A1 §3/§4.3 should name A4 as the (non-blocking, W0-sibling) broker the
  gateway publishes to, and the budget→email flow should cite A4+B8 as the routing owners. Surface
  the csv edge gap rather than silently relying on it. (See A4 HIGH below — same edge, other side.)

- **MED — REQ-A1-04 "only the B13 identity may write" has no defined identity mechanism.**
  REQ-A1-04 / AC-A1-04 require the virtual-key admin API to accept writes *only* from "the B13 kopf
  operator identity" and reject "a cluster-admin by hand." Nothing in A1 (or the interface-contract)
  defines how that identity is established and verified at the admin surface (mTLS SA? Platform JWT
  role? network policy?). As written the AC is testable in spirit but the mechanism is invented at
  test time, so two implementers could satisfy it incompatibly. Fix: tag the auth mechanism
  `[PROPOSED — not in source]` and pin one approach, or defer explicitly to B13's `/v1` admin
  contract and reference it. Currently neither A1 nor B13 names the mechanism.

- **MED — Load-bearing `[PROPOSED]`: callback function signatures (§4.2, R-A1-3).**
  A1's entire B2 integration surface (REQ-A1-09, AC-A1-09 "hooks each fire exactly once") rests on
  the pre/post/on-failure callback contract whose exact signatures are `[PROPOSED — not in source]`
  ("follow upstream LiteLLM's callback interface"). Blast radius: MED-HIGH — B2 is the sole consumer
  and A1 correctly defers signatures to upstream, but AC-A1-09's "exactly once" semantics presume a
  specific invocation contract that is not pinned. The design *depends* on this being stable. Fix:
  keep the flag, but make AC-A1-09 assert against a named upstream LiteLLM callback version pin
  (recorded in the runbook per §5), so "exactly once" is checkable against a fixed contract.

- **MED — EgressTarget enforcement split (A1 §4.1 row / R-A1-6) is an open governance gap.**
  A1 §4.1 notes "egress-target enforcement is shared with A6 Envoy," and R-A1-6 raises the open
  question "is any agent egress reachable that bypasses both A1 and A6?" This is a genuine,
  unresolved security boundary question spanning A1+A6+B22 — correctly surfaced, not auto-resolvable
  here. Flagging so the reviewer above tracks it as a cross-piece open item, not a per-spec defect.

- **LOW — `platform.security.*` gateway events (§4.3, R-A1-8) `[PROPOSED]`.** Namespace-mapped only;
  concrete types deferred to B12. Low blast radius (additive). No action beyond keeping the flag.

- **LOW — REQ-A1-20 bundles many deliverables into one REQ/AC pair.** AC-A1-20 is a checklist
  ("Docs, runbook, alert rules, ... all exist"). Acceptable for a cross-cutting deliverable, but
  "exist" is weakly verifiable (e.g. a rendering `GrafanaDashboard` XR vs a present-but-broken one).
  Fix: make each sub-item independently assertable (XR renders; Headlamp plugin loads; alerts fire on
  a synthetic condition) — partly already implied; tighten the AC wording.

---

## A4 — Knative Eventing + NATS JetStream broker — verdict: SOUND (minor)

A4 is the cleanest of the four. It correctly disclaims owning event sources (ADR 0023), schemas
(B12), and the audit pipeline (A18); it routes the full closed namespace set; it does not pose as a
system of record (ADR 0014). CRD handling is correct — Knative `Broker`/`Trigger` are upstream
kinds, not platform CRDs, and A4 introduces none.

- **HIGH — A4's downstream set omits its largest class of real consumers (csv vs reality).**
  csv A4 downstream = `A19;B8;B12`. But A4 §4.3 states A4 transports **all ten namespaces** and §3
  acknowledges "every component that emits/consumes CloudEvents (cross-cutting, non-blocking): A1,
  A7, A18, ARK (A5), etc." A1, A6, and A7 (this review's pieces) all emit onto A4's broker. The csv
  models these as non-edges, consistent with waves.md "continuously available inputs, not
  wave-depth-setting" — so this is *defensible* as a wave-layering choice, **but** it means the A4↔
  {A1,A6,A7,A18,...} producer interface is invisible in the dependency graph. Recommend A4 §3/§4.2
  explicitly state "the Broker is a non-blocking publish target for every CloudEvent-emitting
  component; these are not wave edges but ARE interface consumers" so the contract surface is not
  lost. This is the mirror of the A1 HIGH above; resolving one resolves both. (Not a BLOCKER because
  build order is unaffected — all parties are W0/early — but it is a documented-interface gap.)

- **MED — AC-A4-06 is weakly verifiable ("detectable as non-conformant").**
  REQ-A4-06 forbids introducing a new top-level namespace. AC-A4-06: "An event with a `type` outside
  the ten namespaces is detectable as non-conformant (no Trigger routes it)." "No Trigger routes it"
  is a *negative* and is satisfied trivially by any event for which no Trigger happens to match —
  it does not prove A4 *detects/rejects* a non-conformant namespace. As written, an off-taxonomy
  event is silently un-routed, indistinguishable from a valid event with no subscriber. Fix: make the
  AC assert a positive signal — e.g. a Gatekeeper/Trigger-admission or a broker-side validation that
  *rejects or flags* a `type` whose prefix is not one of the ten (and emits an observability/security
  signal). Otherwise REQ-A4-06's "SHALL NOT introduce a new namespace" is unenforced at runtime.

- **MED — Load-bearing `[PROPOSED]`: stream-config mechanism CRD-vs-Helm (§4.1, R1).**
  REQ-A4-03 (declarative stream config) and AC-A4-03 (out-of-band change reconciled back) depend on
  the GitOps mechanism, which is `[PROPOSED — not in source]` (Helm-values assumed). Blast radius:
  MED — the AC's "reconciled back" behavior differs materially between a CRD reconciler and Helm/
  ArgoCD drift correction. The design depends on *a* declarative path existing; either is workable,
  but AC-A4-03's reconcile semantics aren't pinnable until one is chosen. Keep flag; pick one in plan.

- **LOW — OQ1 dead-letter handling `[PROPOSED]`.** Genuinely open (broker DLQ policy not in source).
  Interacts with REQ-A4-05 at-least-once + AC-A4-09 "loses only in-flight-undelivered events." Surface
  to the platform owner; do not auto-resolve. Low blast radius for v1.0.

- **LOW — NATS client-credential secret shape `[PROPOSED]` (§4.4, R3).** Correctly flagged; event
  sources are documented ADR-0041 exceptions so no connection-secret XRD is expected. No action.

---

## A6 — agent-sandbox + Envoy egress proxy — verdict: NEEDS-WORK

A6 correctly *owns* the `Sandbox`/`SandboxTemplate` reconciler and versioning (matches
interface-contract §1.3 owner=A6 and glossary line 120) and correctly *consumes* `EgressTarget`/
`CapabilitySet` (owned by B13). The defense-in-depth framing and ADR mapping are accurate. Two
cross-piece contract gaps and one unverifiable-in-isolation AC pull it to NEEDS-WORK.

- **HIGH — The `EgressTarget`→Envoy-config contract is `[PROPOSED]` on BOTH sides and owned by
  neither — load-bearing for A6's core function.** A6 §4.4 treats Envoy per-agent-class config as
  "B13-produced config" and tags the exact shape `[PROPOSED — not in source]`; REQ-A6-10 / AC-A6-10
  require an `EgressTarget` to become reachable "with no hand-edit of Envoy config." On the other
  side, B13 (`specs/components/B/spec-B13.md` §interfaces line ~123–124 and R2) says "The wiring
  mechanism is `[PROPOSED — not in source]`" and rates its blast radius **high**. So A6's single most
  important enforcement input (REQ-A6-06 FQDN allowlist) rides on a contract that *both* owners
  defer and *neither* defines. Blast radius: HIGH — without an agreed wiring mechanism, A6's plan
  TASK-06 "seed fixture until B13" can diverge from what B13 actually emits, and AC-A6-10 cannot be
  validated end-to-end. Fix: A6 and B13 must co-author and freeze the `EgressTarget`→Envoy config
  contract (config map? xDS? file mount?) before either freezes; until then mark it a shared open
  question, not a per-side `[PROPOSED]` each assumes the other will resolve.

- **HIGH — Dry-run decision-shape ownership contradicts A20 and B3 (cross-piece).**
  A6 §4.2 + R3 say Envoy dry-run field names are `[PROPOSED]` to "reconcile with A20/ADR 0038."
  But A20 (`specs/components/A/spec-A20.md` §4.2 line ~61 and scope line ~25) states the decision
  contract / `simulated` shape is owned by **B3**, and B3 (`specs/components/B/spec-B3.md` §"Canonical
  decision contract") confirms it owns "the output/decision document shape ... uniform across" all
  decision points. So A6 names A20 as the reconcile partner, A20 points to B3, and A7 (below) claims
  to *own* a parallel contract. Three pieces, three different owners. Fix: A6 (and A7) should bind to
  **B3's** canonical decision document; A20 only *composes* it. Re-point A6 R3 / §4.2 at B3.

- **MED — AC-A6-05 / AC-A6-13 "no bypass" is the load-bearing security AC but is underspecified.**
  R1 (high) correctly identifies that the "no other egress path" invariant (ADR 0003) is only as
  strong as the pod network policy. REQ-A6-05 says "no other egress path ... SHALL exist," but A6's
  scope and tasks (plan TASK-09) put the *network-policy* that actually prevents direct egress inside
  A6 without stating which layer enforces it (CNI NetworkPolicy? sandbox netns? Envoy as the only
  route?). AC-A6-05 "a direct outbound attempt is blocked" is verifiable only once that mechanism is
  named. Fix: name the no-bypass enforcement mechanism explicitly and make AC-A6-05/13 assert it per
  substrate (the plan already requires the two-substrate bypass-probe — good; the spec REQ should
  name the mechanism so the probe has a defined target).

- **MED — Load-bearing `[PROPOSED]`: sandbox-lifecycle event namespace `lifecycle.*` vs `audit.*`
  (§4.3, R5).** REQ-A6-09 + AC-A6-09 require sandbox create/destroy/hibernate audit; §4.3 also emits
  them under `platform.lifecycle.*`, flagging the namespace choice `[PROPOSED]` ("both apply"). This
  is genuinely ambiguous from the glossary (lifecycle namespace explicitly covers "Sandbox ...
  lifecycle") and ADR 0034 (audit). Low-to-MED blast radius: a downstream Trigger filtering on the
  wrong prefix would miss the events. Surface to B12 (per-type naming) — do not auto-resolve; but A6
  should at minimum commit that the *audit* copy always goes under `platform.audit.*` (REQ-A6-09 is
  an audit requirement) and the lifecycle copy is additive.

- **LOW — AC-A6-11 asserts "no enforcement-path audit (only the simulator-run `platform.policy.*`
  record)."** This couples A6's dry-run to A7/A20's audit semantics; verifiable but only with the
  A20 contract present. Consistent with A7 AC-A7-06 — keep, but note it cannot pass in isolation
  without the stub A20/A7 path the plan provides.

- **LOW — Envoy mTLS material shape `[PROPOSED]` (§4.4).** Correctly deferred to B13-produced config;
  same root as the HIGH wiring-contract item. Resolving that resolves this.

---

## A7 — OPA / Gatekeeper — verdict: NEEDS-WORK

A7 is architecturally central and mostly faithful: it correctly installs upstream OPA/Gatekeeper
(introduces no platform CRD), enumerates the full admission set matching interface-contract §1.x
owners, and honors ADR 0018 (restrictor-only), 0041 (substrate-claim rejection), 0002 (no image
signing). Two cross-piece ownership conflicts and one unresolved fail-policy default are the issues.

- **HIGH — A7 claims to OWN the dry-run / decision-shape contract; B3 owns it (direct
  contradiction).** A7 §2.1 ("ships ... the dry-run/decision-shape contract"), §4.2, §5 ("Custom
  code: the dry-run decision-shape contract"), and R3 all assert A7 owns or authors the decision
  shape and that A20/A1/A6 "depend on a single agreed shape." But B3 (`spec-B3.md` §"Canonical
  decision contract": "the input document schema each decision point passes to OPA and the
  output/decision document shape OPA returns ... uniform across" points) and A20 (§4.2/§scope) both
  place ownership in **B3**. This is a true ownership collision on the platform's single most
  reused contract. Blast radius: HIGH — A1, A6, A20, B19, B20 all bind to "the decision shape"; if
  A7 and B3 both think they own it, two divergent shapes can ship. Fix: A7 should *consume/serve*
  B3's canonical decision document and the substrate-claim constraint should reference B3's shape;
  remove A7's ownership claim. (Same root as the A6/A20 HIGHs above — one resolution fixes all.)

- **HIGH — Fail-open vs fail-closed admission default is `[PROPOSED]` and is load-bearing for every
  CR write.** §7 (Availability) and R1 (high) flag the Gatekeeper admission failure policy as
  `[PROPOSED — not in source]`, recommending fail-closed. This is not a cosmetic gap: the entire
  platform's CR-write safety depends on it, and **downstream pieces assume a value** — e.g.
  plan-A7 §8 and the A1/A6 fail-closed runtime posture (REQ-A1-10, REQ-A6-07) presume admission also
  fails closed. No AC covers it. Blast radius: HIGH (security correctness of admission). Fix: this is
  a genuine open decision — surface it to the platform owner for an explicit ruling, then add a REQ +
  AC ("a constructed webhook outage denies a policy-relevant CR create"). Do not let it stay an
  un-ACed `[PROPOSED]` given how many pieces lean on it.

- **MED — REQ-A7-12 (OPA-bridge callback ownership) oscillates with landing order; no clean handoff.**
  REQ-A7-12 / AC-A7-12 say A7 carries the LiteLLM OPA-bridge callback "if A7 lands before A1." A1 §2
  and plan-A1 also describe the OPA-bridge/audit callbacks "riding A1 if it precedes A7." R4 (med)
  admits ownership "oscillates between A1/A7/B2." Both A1 and A7 are W0 (waves.md), so neither is
  guaranteed first — the conditional has no deterministic resolver, and B2 (the real owner of
  callback *handlers*) is W1. Risk: a gap where neither ships the bridge, or both do. Fix: make the
  handoff explicit and unconditional — e.g. "B2 owns the bridge handler; whichever of A1/A7 lands
  first ships a temporary no-op/allow shim that B2 replaces," with an AC on the shim. Currently the
  contract is "whoever is first," which is not objectively schedulable.

- **MED — REQ-A7-05 budget-through-OPA depends on `[PROPOSED]` OPA-data shape co-owned with B13.**
  REQ-A7-05 / AC-A7-05 require OPA to evaluate "current spend + `BudgetPolicy`," and §4.4 + R6 flag
  the OPA-data document shape for budgets/capabilities as `[PROPOSED — not in source]`, to reconcile
  with B13/B16. B13 (spec-B13 R2) independently rates "OPA-data document paths ... `[PROPOSED]`" as
  **high** blast radius. So the budget-decision contract (consumed by A1 REQ-A1-12) is undefined and
  owned jointly by A7+B13+B16. Blast radius: MED-HIGH. Fix: A7+B13 must freeze the OPA-data document
  paths together; AC-A7-05 should assert against that agreed shape, not an ad-hoc one.

- **MED — AC-A7-02 "Creating each v1.0 CRD/XR triggers a ... decision; a policy-violating instance of
  each is denied" is broad and only partly verifiable in isolation.** It requires a policy-violating
  instance *of each* of ~30 kinds, but the Rego content that defines "violating" is B3/B16 (W1+),
  not A7. At A7's own gate there is no real policy to violate. Fix: scope AC-A7-02 to "admission
  *fires* for each kind (constraint bound)" at A7's gate, and defer the "denied on violation"
  assertion to a reference constraint A7 ships (it already ships the substrate-claim constraint —
  use that as the concrete admit/deny proof, AC-A7-07, and reference it).

- **LOW — R5 approval-elevation event emitter (A7 vs B19) `[PROPOSED]`.** §4.3 flags whether A7 or
  B19 emits the `platform.approval.*` elevation event. Genuinely open; reconcile with B19/ADR 0017.
  Low blast radius (one event's emitter). Surface; do not resolve.

- **LOW — REQ-A7-13 / AC-A7-13 (no image-signature verification) is a doc-check AC.** Verifiable as
  "no such constraint ships," which is fine, but it encodes an accepted *security gap* (R2 high). The
  AC is sound; the residual risk is correctly pushed to B22. No change.

---

## Cross-piece summary (for the reviewer above)

- **The single biggest issue is one contract, surfaced four ways:** the OPA **decision /
  dry-run document shape**. A7 claims ownership; A6 and A1 say "reconcile with A20"; A20 says
  "bind to B3"; B3 actually owns it. Recommend a Canon clarification: **B3 owns the canonical
  decision document; A7 serves it, A20 composes it, A1/A6 bind to it.** Resolving this clears one
  HIGH in A7, one in A6, and de-risks A1's callback ACs.
- **Second cross-cutting gap:** the `EgressTarget`→Envoy-config wiring mechanism and the budget
  OPA-data document shape are each `[PROPOSED]` on both producer (B13) and consumer (A6 / A7) sides,
  with B13 itself rating both **high** blast radius. These need joint freezes, not parallel deferrals.
- **Wave/csv note (not a blocker):** A1/A6/A7 all publish onto A4's broker, but the csv models no
  A4 edges to them (consistent with waves.md "non-wave-depth" fan-out). Defensible for build order,
  but the *interface* dependency is undocumented on both sides — recommend a one-line note in A4 and
  in each emitter so the contract surface isn't invisible.

---

## Verdict & counts

| Piece | Verdict | HIGH | MED | LOW |
|---|---|---|---|---|
| A1 | NEEDS-WORK | 2 | 3 | 2 |
| A4 | SOUND (minor) | 1 | 2 | 2 |
| A6 | NEEDS-WORK | 2 | 2 | 2 |
| A7 | NEEDS-WORK | 2 | 3 | 2 |
| **Total** | — | **7** | **10** | **8** |

No piece is a hard BLOCKER (none has an internally fatal, unfixable contradiction); all four are
fixable in-place. The two HIGHs that gate downstream freezes are the **decision-shape ownership
collision (A7 vs B3/A20)** and the **EgressTarget→Envoy wiring + budget OPA-data shapes co-owned but
undefined (A6/A7 ↔ B13)** — these should be resolved before any of A1/A6/A7/A20/B3/B13 freeze.
