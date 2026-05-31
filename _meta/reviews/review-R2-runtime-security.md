# Review R2 — Runtime / Identity / Security / MCP / Tenant-Onboarding (second pass)

Reviewer: independent red-team (R2). Scope: T0 contract pieces **A5, A11, A17, A18, A22**.
Canon bound: `_meta/glossary.md`, `_meta/interface-contract.md`, `_meta/waves.md`, `_meta/piece-index.csv`,
plus source ADR-0020 (`adr/0020-initial-mcp-services.md`, `specs/adr/spec-ADR-0020.md`) and
`architecture-overview.md` §14.1. Cross-pieces consulted: A21, A20, A9, A16, A14.

Severity legend: **HIGH** = contract/interface break or untestable load-bearing claim; **MED** =
ambiguity/blast-radius risk needing owner sign-off; **LOW** = polish / traceability.

---

## A5 — ARK agent operator — Verdict: **SOUND**

Clean. Every REQ has a covering AC (REQ-01..15 ↔ AC-01..15, 1:1), CRD field table matches
interface-contract §1.2 verbatim, CloudEvents correctly confined to `platform.lifecycle.*` /
`platform.audit.*`, and the bootstrap-degradation posture (audit endpoint + OPA land after A5 in
W1) is explicit and tested. `[PROPOSED]` flags are all low-blast-radius (ARK chart version, status
sub-resource shapes, optional admin HTTP surface) — correctly deferred to ARK upstream, not coined.

- **LOW** (spec-A5.md §4.3 / REQ-A5-08, AC-A5-08): §4.3 "Emitted" bullet lists
  "`Sandbox-handoff`/`MemoryStore`-related lifecycle events" among ARK emissions. Sandbox lifecycle
  is A6-owned (glossary; interface-contract §1.3) and `MemoryStore` is a B4 XR; ARK emitting under
  those resource lifecycles risks double-emission / ownership blur. Fix: scope ARK's
  `platform.lifecycle.*` emissions to `Agent`/`AgentRun`/`Team` and the *handoff request* event only;
  state explicitly that Sandbox/MemoryStore lifecycle events are emitted by their owners (A6 / B4).
- **LOW** (spec-A5.md §6 REQ-A5-11 / plan TASK-10): REQ-A5-11 binds the ARK Headlamp plugin to "the
  A9/A22 framework," but A5 is **W1** and A22 is **W2** (waves.md). The plugin cannot build on A22 at
  A5's build time. Plan TASK-10 lists depends-on `[TASK-02]` only and omits the A9/A22 cross-wave
  ordering. Fix: note A22-framework consumption as a later-wave rewire (mirror the audit/OPA
  bootstrap-tolerance already used), or build the plugin on A9 alone for v1.0.

---

## A11 — OpenSearch — Verdict: **SOUND**

Advisory-only / reproducibility invariant is stated crisply and is the spec's backbone; the
`XSearchIndex` field set and substrate-agnostic status (`ready`/`endpoint`/`version`) match
interface-contract §1.6 / §4. REQ-01..12 ↔ AC-01..12 are 1:1 and testable. B4 ownership boundary
(A11 operates the product, B4 owns the Composition) is correctly drawn and flagged as a joint-contract
risk (R1).

- **MED** (spec-A11.md §4.4 / REQ-A11-08 / AC-A11-08): The connection-secret claim is internally
  inconsistent on `port`. §4.4 maps OpenSearch onto "endpoint, user, password, optional TLS material"
  and says the canonical `dbname` is N/A — but REQ-A11-08 and AC-A11-08 assert the written secret has
  keys "endpoint/host, **port**, user, password, optional TLS." interface-contract §4 fixes the
  canonical five as `host, port, user, password, dbname`. The spec never resolves whether OpenSearch's
  secret carries discrete `host`+`port` or a single `endpoint`, and AC-A11-08 ("identical keys … a
  consumer binds to the same keys on both substrates") is the load-bearing cross-substrate test — A18
  and A10 bind to this. Fix: pin the exact key set (recommend canonical `host`/`port` + drop `dbname`,
  or document `endpoint`-only) and make AC-A11-08 assert that exact set, since downstreams branch on it.
- **LOW** (spec-A11.md header / §2.2): downstream is `[A10, A18]` (matches CSV), but A17's
  system-mediated OpenSearch MCP also consumes A11's connection secret/cluster (A17 §3, REQ-A17-05).
  Not a contract break — A17 is correctly listed as a de-facto consumer in §2.2 — but the secret-key
  resolution above (MED) also gates A17. Worth surfacing in the A11↔A17 contract note.

---

## A17 — Initial MCP services integration — Verdict: **NEEDS-WORK**

Thorough and correctly bound to ADR 0020; the no-bypass / ESO-only / CapabilitySet+OPA posture is
sound and REQ-01..16 ↔ AC-01..16 are 1:1. The blocking issue is a genuine **Canon-vs-Canon
contradiction** the spec correctly flags but cannot self-resolve.

- **HIGH** (spec-A17.md §4.1 + R1; REQ-A17-05 / AC-A17-05): The frozen `interface-contract.md` §1.4
  defines `MCPServer.authMode` as **`(system/user-cred)`** — two values. But ADR 0020 is a settled T0
  decision mandating **three** credential modes; both `specs/adr/spec-ADR-0020.md` §4.1 and the raw
  `adr/0020-initial-mcp-services.md` ("a third distinct credential shape alongside system- and
  user-credential") enumerate **system-mediated**, and REQ-A17-05 / REQ-ADR-0020-03 require it for
  OpenSearch. So the FROZEN interface contract and a FROZEN T0 ADR disagree on the `authMode` enum.
  A17 tags this `[PROPOSED]` and defers the enum to B13 — but B13 cannot widen a *frozen* contract by
  fiat, and A17's OpenSearch system-mediated mode (REQ/AC-05) is **untestable** until the enum is
  authoritative. Blast radius: the `MCPServer` CRD schema (B13-reconciled) + every consumer of §1.4.
  Fix (Canon-level, not A17's to make): issue a Canon revision adding `system-mediated` to
  interface-contract §1.4 `authMode` to match ADR 0020, then drop the `[PROPOSED]`. **Open question
  for the contract owner — do not auto-resolve.**
- **MED** (spec-A17.md §2.1, REQ-A17-01, AC-A17-01): The "seven services" framing is inconsistent with
  the actual count. §1, REQ-01, and ADR-0020 REQ-01 enumerate **eight** registrable servers (GitHub,
  Google Drive, Context7, OpenSearch, Postgres, MongoDB, web-search, web-scrape); the prose repeatedly
  says "seven services" (treating web-search+web-scrape as one). AC-A17-01 says "All seven services
  exist as … `MCPServer` CRDs" — a literal conformance test will mis-count. Fix: standardize on "eight
  `MCPServer` services" (or "seven service families / eight servers") and make AC-01 assert the exact
  set, so the schema-conformance test is unambiguous. (ADR-0020 AC-01 itself says "all eight services.")
- **MED** (spec-A17.md §4.1, REQ-A17-06/07/15): `XAgentDatabase` per-principal provisioning has **no
  quota/budget gate** (acknowledged in R4 as `[PROPOSED]`). This is the *first runtime-state XR*;
  unbounded per-agent/tenant/user DB minting can exhaust the substrate. No REQ mandates a cap and no AC
  exercises exhaustion. Fix: add at minimum a REQ that DB provisioning is gated by an OPA/budget
  decision (even if the threshold value is deferred to F1), plus a fail-safe AC — otherwise the
  capability ships with an unbounded-resource hole.
- **LOW** (spec-A17.md §3 / OQ3, plan §3.1): B4 is a hard build-order upstream (XRDs for
  Postgres/Mongo/OpenSearch-system-mediated) but is absent from the CSV `upstream` cell (`A1;A7;B13`).
  Spec + plan both flag this `[PROPOSED]`. Consistent with A18's identical treatment; recommend the CSV
  be corrected (B4 added to A17 upstream) rather than carried as a perpetual `[PROPOSED]`. Surfaced,
  not resolved.

---

## A18 — Audit endpoint + audit adapter library — Verdict: **NEEDS-WORK**

The most-depended-on T0 surface, and the spec rightly specifies the *behavioral* contract
exhaustively and separates source-fixed invariants (numbered 1–6, §4.2) from `[PROPOSED]` concrete
shapes. Failure modes (Postgres-down fail-closed, OpenSearch-down survives, S3-verify-then-delete)
are precise and testable. REQ-01..15 ↔ AC-01..13 — but coverage is not fully 1:1 (see below). The
structural risk is that the *entire concrete adapter surface every other component binds to* is
`[PROPOSED]`, by the spec's own admission (R1/R2 both HIGH).

- **HIGH** (spec-A18.md §4.2, §4.4; R1, R2): The adapter method/envelope field names **and** the
  `audit_events` column schema are `[PROPOSED — not in source]` — only the `emit`-style behavior, the
  `platform.audit.*` typing, and the table *name* are source-fixed. Every emitting component (§6.6
  list: LiteLLM, Gatekeeper, Envoy, Knative, ARK, agent-sandbox, ESO, ArgoCD, LibreChat, Headlamp,
  approval, Coach, HolmesGPT, B13) and every reader (Headlamp audit view, HolmesGPT tool, F1 retention)
  binds to surfaces that do not yet exist. This is correctly self-identified as platform-wide blast
  radius. It is not a *defect* in A18 (source genuinely defers this) but it **is** a release blocker for
  the corpus: A18 must freeze and publish the adapter contract + `audit_events` schema **before** any
  downstream wave author emits. Plan TASK-01 commits to "freeze + publish first," which is the right
  mitigation — but nothing in the spec makes the freeze a *gated AC*. Fix: add an AC that the adapter
  contract + migration schema are version-tagged and published as the binding surface, and make
  downstream emission depend on that artifact, not on A18 code.
- **MED** (spec-A18.md §6 REQ-A18-13 vs §9 AC coverage): **No AC maps to REQ-A18-15's pair** cleanly
  and **REQ-A18-13 (Envoy egress) is covered by AC-A18-11**, REQ-A18-14 by AC-A18-12, REQ-A18-15 by
  AC-A18-13 — those are fine. However REQ-A18-12 (versioning, both library *and* endpoint) maps only to
  AC-A18-10, which tests the **endpoint `/v1` route deprecation** but **not the library semantic-version
  deprecation/compat-matrix**. The library half of REQ-12 (the higher-blast-radius half) is untested.
  Fix: extend AC-A18-10 (or add an AC) to assert the adapter library advertises a semantic version and
  a compatibility matrix against the endpoint, with a deprecation window.
- **MED** (spec-A18.md §4.2 delivery semantics / OQ1 / R3): At-least-once vs at-most-once and
  buffering-on-transient-endpoint-failure are `[PROPOSED]`. The threat model (§7 "audit integrity is a
  named asset") *relies* on completeness guarantees, yet the spec leaves whether events can be dropped
  on transient endpoint (not Postgres) failure undefined. Source fixes only "fail-closed on
  Postgres-down" and "survive OpenSearch-down." Fix: A18 must pick a delivery guarantee and state it as
  a REQ+AC (recommend at-least-once with caller-visible failure on endpoint-unreachable), because the
  security posture cannot be validated against an undefined semantic. Surfaced as an open decision.
- **LOW** (spec-A18.md §4.1): `AuditLog` is described as "owner B4-composed, defined/consumed here,"
  and §4.1 says "ownership of the `AuditLog` versioning lifecycle sits with A18." interface-contract
  §1.6 lists the XRD owner as Crossplane/B4 and §1.1 ties versioning-lifecycle ownership to "the
  component that owns its reconciler." A18 claiming the versioning lifecycle while B4 owns the
  Composition reconciler is a latent ownership ambiguity (parallels A21/B4 OQ-A21-1). Fix: reconcile
  with B4 — either A18 owns the `AuditLog` *claim shape* lifecycle with B4 owning Composition
  internals, stated explicitly, or defer the lifecycle to B4. Confirm; do not assume.

---

## A22 — Headlamp graphical-editor framework — Verdict: **NEEDS-WORK**

Strong spec: CRD-target table matches interface-contract §1 verbatim, PR-only/no-cluster-write is a
hard invariant with a real AC (AC-08, API-server-audit assertion), the 11-editor initial set matches
ADR 0039 / §14.1 exactly, and REQ-01..16 ↔ AC-01..16 are 1:1. The A22↔A21↔B4 ownership overlap is the
live risk and is handled honestly but not closed.

- **HIGH** (spec-A22.md §2.2, §4.1, REQ-A22-09, AC-A22-13; vs A21 §2.2/OQ-A21-2; §14.1): The
  **A22↔A21 `TenantOnboarding` editor ownership is doubly-declared and unreconciled across both specs.**
  §14.1 (authoritative source, line 1687/1688) states two things that are in tension: A22 "ships the
  framework plus the initial editor set" *including* `TenantOnboarding`, **and** "A21 owns its own
  CRD's editor; A22 owns only the cross-cutting framework / shared widgets." A22 REQ-A22-09 hard-mandates
  shipping a `TenantOnboarding` editor; A21 REQ-A21-07 hard-mandates A21 building *the* `TenantOnboarding`
  editor on A22. Both pieces tag this `[PROPOSED]` (A22 OQ-A22-1, A21 OQ-A21-2) and propose the same
  "A22 baseline, A21 extends/supersedes" split — but neither owns the decision, so today **two pieces
  are each on the hook to ship a `TenantOnboarding` editor**, with AC-A22-13 (A21 consumes A22's lib)
  and AC-A21-07 (A21 builds the editor) both binding. Wave order (A22 W2 → A21 W3) makes "A22 baseline
  first, A21 supersedes" *feasible*, but feasibility ≠ a settled contract. Fix (cross-piece, owner
  decision): pick one of (a) A22 ships **framework only**, no `TenantOnboarding` editor, A21 ships the
  sole editor — which contradicts ADR 0039's initial set; or (b) A22 ships a *non-extensible reference*
  `TenantOnboarding` editor explicitly marked "superseded by A21," and A21's editor is the production
  one — and amend ADR 0039's initial-set wording. **Open question — do not auto-resolve.**
- **MED** (spec-A22.md §1 vs §2.2): §1 names `BudgetPolicy` among the "schema-heavy constrained fields"
  whose YAML hand-editing the framework exists to fix, then §2.2 **excludes `BudgetPolicy`** from the
  v1.0 editor set (deferred per ADR 0039). Defensible (problem named, scope deferred) but reads as a
  contradiction and could mislead an implementer into building the editor. Fix: in §1 add a parenthetical
  "(BudgetPolicy editor deferred — ADR 0039)" so the motivation list and the shipped set don't appear to
  conflict.
- **MED** (spec-A22.md §4.2, REQ-A22-13, OQ-A22-2; vs §14.1 / interface-contract §3): The **framework
  widget/form-generation library is the load-bearing downstream contract** (A21 and every per-component
  editor team bind to it), yet its API surface is entirely `[PROPOSED — not in source]` and §14.1 ascribes
  "shared widgets" to **A9**, while A22 owns "the editor layer." So the boundary of *which* widgets are
  A9's vs A22's is undefined precisely where A21/B-track consume it. interface-contract §3 names no such
  SDK surface. This mirrors A18's adapter-surface risk at smaller scale: a downstream-binding surface
  that does not yet exist. Fix: A22 + A9 must jointly fix and publish the widget-library API boundary
  (which widgets ship from A9 vs A22) before A21 (W3) builds on it; add an AC that the library API is
  versioned/published, since AC-A22-13 already presumes A21 consumes it.
- **LOW** (spec-A22.md §4.1 `TenantOnboarding`/`AuditLog`/`GrafanaDashboard` owner column): the table
  lists these XRD targets' reconciler as "Crossplane (B4)," consistent with interface-contract §1.6.
  But the `TenantOnboarding` row also parenthetically says "A21 owns CRD-specific behaviour" — fine —
  while `AuditLog` is shown as plain B4 with no note that A18 claims its versioning lifecycle (see A18
  LOW above). Editors render against the served OpenAPI schema regardless of who owns versioning, so no
  functional break — but for consistency, footnote that `AuditLog` claim-shape evolution is A18-driven.

---

## Cross-cutting notes

- **A16↔A14 edge-direction (flagged in the brief):** does **not** surface in my pieces. Verified anyway
  — CSV (`A14 → A16`), spec-A16 (upstream `[A1, A14]`) and spec-A14 (downstream `[A16, B10]`) all agree.
  No issue.
- **A22↔A21↔B4 composition/editor overlap (flagged):** real and unresolved — see A22 HIGH + the parallel
  A21 OQ-A21-1 (B4 owns `TenantOnboarding` Composition runtime while A21 owns the component/composition
  behaviour) and OQ-A21-2 (the editor double-build). Both A21 OQs are `[PROPOSED]` boundaries needing the
  same owner decision as the A22 HIGH.
- **A17 / ADR-0020 MCP secret shapes (flagged):** the `authMode` enum mismatch (A17 HIGH) is the concrete
  manifestation. Per-service ESO secret *key shapes* (GitHub/Drive/Context7/external-OpenSearch) are
  genuinely deferred (backlog §1.11) and correctly `[PROPOSED]` — low blast radius, not a contract break.
- **B4 absent from CSV upstream** for both A17 and A18 (XRD provider) is a consistent CSV gap, not a spec
  defect; recommend a CSV correction rather than perpetual `[PROPOSED]` carry. Surfaced for the index owner.

---

## Summary

| Piece | Verdict | HIGH | MED | LOW |
|---|---|---|---|---|
| A5  | SOUND      | 0 | 0 | 2 |
| A11 | SOUND      | 0 | 1 | 1 |
| A17 | NEEDS-WORK | 1 | 2 | 1 |
| A18 | NEEDS-WORK | 1 | 2 | 1 |
| A22 | NEEDS-WORK | 1 | 2 | 1 |
| **Total** | | **3** | **7** | **6** |

No BLOCKER-verdict pieces. The three HIGH findings are each cross-piece / Canon-level and require an
owner decision (not unilateral spec edits): (1) A17 `authMode` enum — frozen interface-contract §1.4
contradicts frozen ADR 0020 (system-mediated missing); (2) A18 adapter contract + `audit_events` schema
are entirely `[PROPOSED]` yet platform-wide-binding — must be frozen/published before downstream emission;
(3) A22↔A21 `TenantOnboarding` editor doubly-owned across both specs with no settled split.
