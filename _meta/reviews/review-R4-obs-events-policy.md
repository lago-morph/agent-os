# Review R4 — Observability / Events / Policy contract pieces (second pass red-team)

**Reviewer role:** independent red-team (second pass), T0 contract pieces
**Date:** 2026-05-31
**Pieces:** B12, B13, B16, B19 (+B19-core/B19-ui split), B22
**Canon consulted:** glossary.md, interface-contract.md (§1.4, §1.5, §2, §5, §6), piece-index.csv rows
(B12/B13/B16/B19/B22 + A7/A18/A23/B4/B5/F1), waves.md, architecture-overview.md §6.7/§6.13 lines
604/621/988 + DAG lines 1885–1899.

**Method:** read each spec (11-section) + plan (8-section); traced every "schema/name lives in B12"
claim back to an owning task; checked REQ→AC coverage; rated each `[PROPOSED — not in source]`;
cross-checked wave/CSV/DAG direction. Sibling reviews `csv-reciprocity.md` and
`consistency-existence.md` referenced where they overlap (B4→B19 edge).

---

## Cross-cutting finding (drives several per-piece HIGHs)

**The `platform.*` per-event-type *name + schema* ownership is split incoherently between B12 and its
emitters, and for `platform.approval.*` it is ORPHANED.**

Canon is explicit and consistent on the division of labour (architecture-overview.md line 988;
interface-contract.md §2 + §6): **"B12 owns the registry; each component owns its event types."**
B12 hosts + enforces conformance/versioning; the *emitting component authors the concrete event-type
names and schemas* and lands them in B12. interface-contract §2 names only event *concepts* per
namespace, never the exact type strings (e.g. it says "Approval requested, OPA-elevated, decided,
timed out" — not `platform.approval.requested`).

B12's spec models this correctly (REQ-B12-09: "MUST NOT invent per-event-type names"; only seeds
`platform.capability.changed` + 2 exemplars). **But the consuming specs invert it:** B19 §2.1/§3/§4.3/
§7/§11 repeatedly states the `platform.approval.*` schemas are **"owned by B12"** / **"live in B12"** and
treats B12 as the party that must *freeze* them (R5). That contradicts Canon (the *names* are B19's to
author) and — critically — **no plan task anywhere authors them**: B12 plan TASK-05/06 author only
capability.changed + the two exemplars; B19 plan TASK-05/06 *emit against* `platform.approval.*` but list
"B12 `platform.approval.*` schemas faked" as a fixture. So four event types every approval consumer (B5,
B10, A14, A23, the requesting agent) binds to have **no owner and no authoring task**. This is the single
biggest contract hole in the set and is logged as a BLOCKER under B19 and a HIGH under B12.

The same latent ambiguity exists for B13 (`platform.capability.changed` *is* Canon-named, so B13 is
fine), B16 (`platform.policy.*` — B16 §4.3 correctly says "defines the decision content… emission by the
invoking component"; names still unauthored but B16 doesn't claim B12 authors them), and B22
(`platform.security.*` — B22 §4.3 correctly defers *name minting* but the names still have no owner).
B12 is named the registry for all ten namespaces yet has authoring tasks for exactly one namespace +
two exemplars; the other eight namespaces' first concrete schemas have no landing task in any plan in
this set. That is a corpus-level gap, surfaced here because B12 is the convergence point.

---

## B12 — CloudEvent schema registry — verdict: **NEEDS-WORK**

Strong, disciplined spec: the contract-first stance (freeze layout before content), the "host don't
invent" boundary (REQ-B12-09), and the dual-versioning enforcement are exactly right. REQ→AC coverage
is complete (AC-B12-01..09 each map a REQ 1:1). Problems are about *what B12 promises others* vs what it
plans to deliver.

- **[HIGH] B12 is Canon-designated registry for ten namespaces but plans content for one + two
  exemplars; downstreams read "lives in B12" as "B12 will deliver it."** (spec §1/§4.2/§4.3; plan
  TASK-05/06.) B12 correctly disclaims authoring names, but every emitter spec (esp. B19) says its
  schemas "live in B12" without B12 or the emitter owning a task to *land the first concrete schema*.
  Fix: B12 spec/plan should state explicitly that an emitter's piece-branch contributes its schema files
  *into* the registry via the TASK-04 contribution gate, and that B12's own deliverable is the
  layout+gate+capability.changed+exemplars only. Add a one-line "contribution responsibility" note so no
  reader infers B12 authors approval/policy/security/lifecycle/gateway/tenant/evaluation schemas.
- **[HIGH] Load-bearing `[PROPOSED]`: registry layout / schema-id convention / index format (R1, blast
  radius high — self-rated).** B19 and *every* emitter bind to these. Concur with the high rating; the
  mitigation (freeze first) is sound but the plan must gate all downstream emitter branches on TASK-01
  merging. Currently only B19's plan notes the dependency; B13/B16/B22 emit too and don't. Fix: make
  TASK-01's frozen layout a published artifact other plans cite.
- **[MED] `platform.capability.changed` payload field set `[PROPOSED]` beyond "affected Agent/
  CapabilitySet identifiers" (R3); B6/B7 deserialize it.** This is the one schema B12 *does* own as a
  hard cross-piece contract (B6/B7 in W2/W3 bind to it). Fix: freeze the field list in TASK-05 and treat
  it as a frozen interface, not a `[PROPOSED]`; coordinate with B6/B7 specs (out of this review's set).
- **[MED] Exemplar trigger-flow event-type *names* `[PROPOSED]` (R4) risk preempting B2/A14.** B12
  flags this correctly; ensure TASK-06 lands them as clearly-labelled exemplars that B2/A14 can rename
  on adoption without a mint-on-break violation (i.e. don't publish them as load-bearing types).
- **[LOW] CSV `B12.downstream = [B19]` understates reach.** Per spec §3 and DAG line 1899, B13/A18/B2/
  A20/ARK/B6/B7 all bind to B12's registry contract; CSV lists only B19. Defensible (only B19 is a
  *build-order* blocker; the rest bind to the contract, not the artifact), but the asymmetry should be
  noted so reviewers don't read B12 as having a single consumer. Not a fix-now.
- **[LOW] OQ1/OQ2 (runtime serving? self-emission on registry change?) genuinely open** — correctly left
  open; no action, just confirming they should NOT be auto-resolved.

---

## B13 — Custom Python kopf operator for LiteLLM — verdict: **SOUND** (minor)

Best-anchored spec in the set: all eight CRD field sets are reproduced verbatim from interface-contract
§1.4, ownership boundaries (kopf vs Crossplane/B4, content vs B3/B16, gateway events vs A1) are clean,
and REQ→AC coverage is complete and faithful (15 REQs → AC-B13-01..14, each AC cites its REQ; REQ-B13-01
and -02 share AC-B13-01 which is acceptable as the apply-and-reconcile criterion).

- **[MED] OPA-data document paths + Envoy-allowlist wiring mechanism `[PROPOSED]` (R2, self-rated high
  blast radius).** B16 budget/capability policies (REQ-B16-03/08) read exactly these data paths, and A6
  consumes the allowlist wiring. This is a real two-way contract and B16's own R4 flags the same seam
  from the other side. Concur with HIGH blast radius on the decision; downgraded to MED here only
  because both specs flag it and the fix (align to B3's `data`-namespacing) is identified. Fix: B13 and
  B16 must co-freeze the OPA `data` path convention before either's content lands; name B3 as the
  arbiter explicitly in both plans (B13 plan §3.3 lists B3 only as non-blocking — that's too weak for a
  contract this load-bearing).
- **[MED] CRD `status`/condition field names `[PROPOSED]` (R1); Headlamp (B5/A22) binds to them.** Plan
  TASK-01 commits to "freeze the status subresource," which is the right move. Fix: ensure the frozen
  status shape is published as the B5/A22 contract, not left internal.
- **[LOW] `platform.gateway.*` (MCP health) emission ownership genuinely ambiguous (R4)** — B13 defaults
  to "A1 owns it." Correct call; leave open, do not auto-resolve.
- **[LOW] CSV reciprocity note:** sibling `csv-reciprocity.md` shows `B4→B19` is a missing-reciprocal
  TYPO; B13 is unaffected (its upstream A1 is reciprocal). No B13 action; noted for completeness.

---

## B16 — Initial OPA policy library content — verdict: **NEEDS-WORK**

Comprehensive and correctly framed (content half of B3/B16; restrictor-only invariant; reads *resolved*
CapabilitySet; B22 extension point). REQ→AC coverage is complete (REQ-B16-01..13 → AC-B16-01..13, 1:1).
The issues are testability of the central security invariant and one untestable AC.

- **[HIGH] REQ-B16-10 "structurally incapable of granting" is the platform's central security guarantee
  but is only *behaviourally* asserted, and its AC is weaker than the REQ.** Spec R2 admits Rego can't
  guarantee this by language alone; AC-B16-10 reduces it to "a planted granting rule fails the gate."
  That tests the *gate's sensitivity to one planted rule*, not the *absence of granting paths* across the
  real policy set — a classic "test proves the canary, not the mine." This leaves the load-bearing
  invariant effectively untestable as written. Fix: strengthen AC-B16-10 to require (a) the floor
  predicate be the *sole* allow-constructor (enforced by lint forbidding bare `allow` rules), and (b) a
  property/fuzz check over representative inputs that no entrypoint returns allow where the B3 floor
  denies. Acknowledge residual risk explicitly rather than implying the gate closes it.
- **[HIGH] REQ-B16-01 asserts admission policies for "all v1.0 CRDs/XRDs … in §6.6" but the kind list is
  transcribed in the spec, not pinned to a single source of truth — and it silently omits the `Approval`
  reconciler-owner nuance and any A2A/MCP runtime kinds.** If §6.6's list and interface-contract §1
  drift, B16's coverage map (REQ-B16-13) can read "complete" while missing a kind. The AC (AC-B16-01,
  -13) checks the map against *B16's own list*, not against the Canon inventory — a self-referential
  coverage proof. Fix: bind the coverage map's kind set to interface-contract §1.2–§1.6 as the
  authority and add an AC that fails if any §1 kind lacks a policy. (Severity HIGH because B16 is the
  sole admission-policy author; a silent gap = an unguarded admission surface.)
- **[MED] `approval.elevation` is a two-way contract with B19-core but the *input/output shape* B19
  passes is unspecified on both sides.** B16 §4.2 binds to B3's decision-document shape; B19 §4.2 calls
  it a "dry-run surface" with `[PROPOSED]` fields. Neither pins what an `Approval`'s elevation query
  *input* looks like (which `Approval` fields are passed) or the *output* level vocabulary. B16 R-? and
  B19 R3 both flag the level vocabulary but not the query shape. Fix: B16 + B19 co-specify the
  `approval.elevation` input (the `Approval` fields + JWT claims) and output (resolved level) as a frozen
  mini-contract before either lands.
- **[MED] Decision-document field names `[PROPOSED]`, owned by B3 (R1, self-rated high).** Correct that
  B3 owns it; flagged appropriately. No B16-side fix beyond "bind after B3 freezes," but the plan should
  gate TASK-01 on B3's frozen contract (it lists B3 as HARD upstream — good).
- **[LOW] OQ1 (content-check policies real vs placeholder) and OQ2 (substrate-mismatch event namespace)
  genuinely open** — correctly deferred to B22/B12 respectively; no auto-resolve.

---

## B19 — Generalized approval system (B19-core) — verdict: **BLOCKER**

The spec is otherwise excellent — the W3/W4 split is modelled carefully, REQ-B19-11 + AC-B19-11 are a
well-designed guard that the split holds, and the Approval CRD fields match interface-contract §1.5
verbatim. REQ→AC coverage is complete (REQ-B19-01..11 → AC-B19-01..11, 1:1). It is a BLOCKER solely
because of the orphaned `platform.approval.*` schema ownership, which is a real, currently-unfillable
contract hole, plus an under-concrete Approval schema. The split itself is sound.

- **[HIGH / BLOCKER] `platform.approval.*` per-event-type names + schemas have no owner and no authoring
  task — and B19 mis-attributes them to B12, contradicting Canon.** (spec §2.1, §3, §4.3, §7, R5, §11;
  plan TASK-05/06 + §5 fixtures.) Per architecture-overview line 988, *B19 owns these event types*; B12
  only hosts. B19 instead says they are "owned by B12 / live in B12" and waits on B12 to "freeze" them
  (R5) — but B12's plan never authors them and B12's spec disclaims inventing names (REQ-B12-09). Result:
  the four approval events (requested / OPA-elevated / decided / timed-out) that A23, B5, B10, A14, and
  the requesting agent all bind to are **ownerless**. AC-B19-05/06 assert emission of a
  `platform.approval.*` CloudEvent but the *type string and payload schema being emitted is undefined and
  unassigned.* Fix (BLOCKER — resolve before either B12 or B19 lands): (1) correct B19 spec to state B19
  *authors* the `platform.approval.*` event-type names + schemas and *contributes them into* B12's
  registry via the B12 contribution gate; (2) add a B19 plan task "author + contribute
  `platform.approval.{requested,elevated,decided,timed-out}` schemas to B12"; (3) align B12 spec wording
  so "lives in B12" means "hosted in, authored by emitter." This is the cross-cutting finding above,
  instantiated.
- **[HIGH] The Approval CRD schema (ADR 0017) is concrete enough on *fields* but the *load-bearing
  semantics* are all `[PROPOSED]` and several are interdependent.** §4.1 defers (per ADR 0017):
  first-class-fields-vs-metadata-blob, timeout value/behaviour, shared-vs-per-`actionType` Argo template,
  approve/reject audit detail, and the level vocabulary beyond `operator`/`org-admin`. B19 proposes
  reasonable defaults, but **the `status` shape (resolved level + workflow ref + timeout deadline) is the
  contract A23 and B5/B19-ui read**, and it is `[PROPOSED]`. A23 (W4) and B5/B19-ui (W4) both bind to it;
  if it flexes after they start, the split's "core lands first" benefit is lost. Fix: promote the
  `Approval.status` shape and the Kargo-handshake fields (R4) from `[PROPOSED]` to a frozen B19-core
  deliverable in TASK-01, published as the A23/B5 contract. ADR-0017 concreteness verdict: **adequate on
  spec fields, under-specified on the status/handshake contract that downstreams actually bind to.**
- **[MED] B19-core vs B19-ui split is coherent — but the CSV/edge encoding has one real defect.** The
  split logic in waves.md and B19 spec §1/§3 is sound and acyclic (B19-core W3 → A23 W4; A23 W4 →
  B5/B19-ui W4 co-land). REQ-B19-11/AC-B19-11 correctly guard it. However: (a) per sibling
  `csv-reciprocity.md`, `B4→B19` is a missing-reciprocal TYPO — B19.upstream omits B4 though B19 consumes
  B4 XRDs (B19 §4.4 "any backing store… via B4 XRDs"); add B4 to B19.upstream. (b) The CSV carries the
  *full* B19 edge set on one W3 row including `B5` upstream, while the *actual* B5→B19 edge is W4-only
  (UI). The spec narrates this correctly (§3 note), but a reader of the CSV alone sees a W4 piece (B5) as
  upstream of a W3 piece (B19) — an apparent backward edge. Fix: not a spec change, but the CSV comment/
  waves.md should flag that the `B5` entry in B19.upstream is the deferred-UI edge, else it reads as a
  W4→W3 violation. (Severity MED: the resolution exists and is documented; the encoding is just
  ambiguous.)
- **[MED] `Approval.defaultLevel` / resolved-level vocabulary must align with Keycloak claims to be
  enforceable (R3).** The resolved-level filter (REQ-B19-04) and OPA elevation (REQ-B19-02) are
  unenforceable unless the level names map to `platform_roles`/`tenant_roles`. B19 flags this; the level
  vocabulary is shared with B16's `approval.elevation` output (see B16 MED above) — co-freeze.
- **[LOW] OQ1 (queue derived from CRD list vs separate index) and OQ2 (Kargo creates Approval directly
  vs via adapter) genuinely open** — both have sensible `[PROPOSED]` defaults; leave open.

---

## B22 — Security threat model design specification — verdict: **SOUND**

Correctly typed as a design/authoring deliverable (not code); the N/A justifications across §4–§8 are
principled, the W0-foundation + non-blocking-fan-out framing matches waves.md exactly, and the
authoring-style REQ/AC framing is appropriate. REQ→AC coverage is complete (REQ-B22-01..11 →
AC-B22-01..11, 1:1) and the ACs are genuinely binary (coverage of §6.6 sets, no-ownerless-rows, edits
exist). No HIGH findings.

- **[MED] The B22→B16 timing coupling is a real ordering hazard, not just a risk note.** B22 (W0) sets
  B16's (W2) OPA targets, but B22's plan (TASK-09) applies the B16 forward-feed edits late and B22 may
  publish incrementally. B16 R3 + REQ-B16-13 build an extension point to absorb late additions — good —
  but **neither spec states what happens if B22's security-complete gate for a B16 policy arrives after
  B16 has shipped its baseline.** It's bounded (additive via extension point) but the "absorb without
  structural rewrite" guarantee is asserted, not tested. Fix: add a concrete contract — B22's
  enforcement-map rows targeting B16 reference B16 policy-package slots, and B16 has an AC that a
  placeholder B22 target lands additively (AC-B16-13 partially does this; make the B22→B16 handshake
  explicit on both sides).
- **[MED] B22 "MUST update §6.6 / Workstream A deliverable lists / B16 targets" (REQ-B22-08, AC-B22-08)
  mandates edits to *source/Canon-adjacent* documents — a process hazard.** §6.6 is frozen architecture
  context; B22 editing it forward is sanctioned by ADR 0027, but the spec doesn't say how that
  reconciles with the "do not edit source" discipline the rest of the corpus follows. Fix: clarify
  whether B22's §6.6 edits are Canon revisions (gated) or downstream-doc annotations; otherwise this REQ
  invites uncontrolled source edits. (Open question — flagging, not resolving.)
- **[LOW] `platform.security.*` event-type names B22 requires are `[PROPOSED]`, deferred to B12 (R3).**
  B22 handles this correctly (specifies the *signal requirement*, lets B12/the emitter mint the type) —
  this is the *right* model and is exactly what B19 got wrong. No fix; cited as the positive contrast.
- **[LOW] Downstream `[A, B]` non-blocking fan-out is correctly modelled** per waves.md; no
  wave-depth-setting claim. Confirmed sound.

---

## Severity roll-up

| Piece | Verdict | HIGH | MED | LOW |
|---|---|---|---|---|
| B12 | NEEDS-WORK | 2 | 2 | 2 |
| B13 | SOUND | 0 | 2 | 2 |
| B16 | NEEDS-WORK | 2 | 2 | 1 |
| B19 | **BLOCKER** | 2 (1 is the BLOCKER) | 3 | 1 |
| B22 | SOUND | 0 | 2 | 2 |
| **Total** | | **6 HIGH** | **11 MED** | **8 LOW** |

**BLOCKER (1):** B19 — `platform.approval.*` event-type names + schemas are ownerless (B19 mis-attributes
them to B12, contradicting Canon line 988; no plan task authors them; all approval consumers bind to
undefined type strings). Must be assigned (to B19, contributed into B12) before B12 or B19 lands.

**Open questions surfaced (not resolved):** B12 runtime-serving + self-emission (OQ1/OQ2); B13
gateway-event ownership; B16 content-check scope + substrate-mismatch event namespace; B19 queue-store +
Kargo-create mechanics; B22 §6.6-edit-vs-frozen-source process question; the corpus-wide question of
which plan lands the *first* concrete schema for each of the eight non-capability `platform.*` namespaces.
