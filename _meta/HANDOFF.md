# HANDOFF — Spec + Plan Generation Run (Agentic Execution Platform)

> Original run branch: `claude/spec-plan-review-parallel-zR9Qr` · Date: 2026-05-30
> **Status (updated 2026-05-31 Session 3): ALL decisions applied. Corpus is Crossplane v2.
> All rulings from DECISIONS-LOG.md are threaded into spec/plan files. OPS layer trimmed.
> Awaiting PR review + merge. No open decisions remain.**
>
> Session 3 branch: `claude/gracious-cray-YbkhX` · PR: open for review.
> §§1–7 are background on the original authoring run. SESSION 2 HANDOFF is retained for
> history. SESSION 3 HANDOFF below is the current state.

---

## SESSION 3 HANDOFF — All Decisions Applied (2026-05-31): current state

**Everything in the SESSION 2 list below is DONE.** All four task groups completed:

1. **Crossplane v1→v2 pass** ✅ — ADR-0044 created; ADR-0041 superseded + moved; full corpus swept; zero residual v1 XRD names or claim language.
2. **OPS-layer trim** ✅ — OPS2/OPS5 retired; OPS1/OPS3/OPS4/OPS6 trimmed to in-scope slivers.
3. **Rulings threaded into specs** ✅ — All D-01..D-08, QN-03, #5/#9/#23/#24/#25/#26, observability correction, secrets, D-04, D-07 applied to every affected spec/plan.
4. **Cleanup** ✅ — QN-02 (hash normalization), QN-04 (untestable ACs), QN-05 (platform release unit), A21 quota consistency.

**PR:** `claude/gracious-cray-YbkhX` — open for review. No merge conflicts expected (additive changes to spec/plan files, OPS files trimmed, new ADR-0044 files).

**What remains for future sessions:**
- Merge the PR (human review).
- Build the per-piece PR graph (119 PRs, one per piece, stacked on wave/N branches — see §6.4 of original plan).
- Targeted adversarial review on T0 contract pieces (see §6.3).
- The Future-version backlog items (#11, #18, #20, #27 — see DECISIONS-LOG).

---

## SESSION 2 HANDOFF — Review & Decisions (2026-05-31): start here

A full review + decision pass ran over the 119-piece corpus. Read these, in order:

1. **`_meta/reviews/DECISIONS-LOG.md`** — the authoritative record: every ruling (Decided),
   the **Future-version** backlog, the **Out-of-scope** list, the resolved **event-namespace
   ownership** table, and the **OPS-layer consequence**. *This is the single source of truth
   for what was decided.*
2. **`_meta/reviews/OPEN-DECISIONS.md`** — plain-language rationale; 5 of 6 now marked decided.
3. T0 reviews: `review-R1..R5*.md` + `DECISIONS-NEEDED.md` / `-PLAIN.md`; consistency:
   `consistency-existence.md`, `csv-reciprocity.md`.

**Repo / PR state.** Merged to `main`: PR #12 (OPS layer), #13 (reviews + registers),
#14 (consistency + 4 csv fixes). **Open, NOT merged: PR #15** on branch
`claude/scrub-crossplane-v1-terms` — carries the decisions log, the v1→v2 finding, the
v1-term scrub of the registers, the D-09 csv fix, and the resolved-QN-03 record. Merge or
rebase PR #15 first in the next session.

### The ONE remaining open decision → the next session's main job: Crossplane v1 → v2

The specs were authored on the **Crossplane v1** resource model (~270 references to "claim",
the `X`-prefixed-composite/claim split, "claim admission"), codified in **ADR-0041**. The
platform is **Crossplane v2** (claims removed; composite resources are namespaced directly).
Agreed plan (one PR):
- **ADR mechanics:** create a new ADR (next free number, ~0044) — *"Crossplane v2 resource
  model"*; set **ADR-0041 → `Superseded`**; **move ADR-0041 (and its spec/plan) into an
  `adr/superseded/` subdirectory** so agents don't read the stale one; new ADR carries a guard note.
- **Sweep:** parallel **opus** subagents, one per slice (Workstream A, B, C–F, views, ADRs),
  **re-expressing** v1 terminology *and logic* (admission, naming, versioning) in v2 terms —
  re-expression, not blind find-and-replace. Fold in D-07 (add `SearchIndex`/`MongoDocStore`/
  `ObjectStore` to the glossary as v2 composite types).
- **Verify:** grep for residual Crossplane-context "claim"/`X`-prefix (target zero) + a re-read pass.

### Honest answer to "is Crossplane the only TODO before implementing?"

**It's the last open *question*, but not the last *task*.** The decisions live in the log,
not yet in the spec files. Before implementing the corpus, a **"apply the decisions to the
specs"** phase is needed (naturally bundled with the Crossplane pass, since it touches most
of the same files):

1. **Crossplane v1→v2 pass** (above) — largest.
2. **OPS-layer trim** — retire/trim OPS1–OPS6 to the surviving in-scope slivers (see the log's
   "Consequence for the OPS layer"): OPS2/OPS5 essentially gone; OPS1/OPS4 → quota+sandbox
   sliver; OPS3 → "consume ESO + reloader + LiteLLM entry"; OPS6 → RBAC granularity.
3. **Thread the remaining rulings into the actual specs** (currently only in the log): D-02
   (B19 authors `platform.approval.*` + add the task), D-03 (B3 owns the OPA decision-doc;
   A1/A6/A7 are consumers), D-05 (audit-adapter + `audit_events` freeze-gate), QN-03 (the ten
   event-namespace owners **+ the cross-cutting "on a security event, also emit to the bus"
   requirement in every component**), the **observability correction** (remove the direct
   `AlertManager→HolmesGPT` trigger from glossary/§6.7 → bus-mediated), **#5/#9 + quota schema**
   (extensible typed `quotas` list on `TenantOnboarding`; admission requires `cpu`+`memory`
   entries *present*, value may be explicit `unlimited:true`; composition emits
   `ResourceQuota`/`LimitRange`), #23 (distinct RBAC perms), D-08 (Envoy stays; sandbox
   `NetworkPolicy` L3/L4 baseline; egress **floor**, not a ceiling; OAuth = a LiteLLM-GUI
   secret), D-01 (`system`/`system-mediated` meanings; retire `user-cred`; system agents use a
   service-account identity), secrets (ESO + reloader startup-gate + LiteLLM PushSecret flow),
   D-04 (memory = design-time abstraction owned by the memory adapter B11).
4. **My-scope cleanup:** QN-02 (normalize `canon-*` front-matter hashes), QN-04 (rewrite the
   untestable ACs), QN-05 (reword/drop the "platform release"-unit ACs).

### Guardrails the user set (honor these in every edit)
- **Plain language** in any human-facing doc; never lead with bare piece codes.
- **Crossplane v2 only** — the word "claim" and the v1 model must not appear.
- **Egress baseline is a floor (extensible), never a closed/mandatory ceiling** — watch
  mandatory-vs-optional wording; one word flips intent.
- **One owner per event namespace**; others take an explicit dependency (no co-ownership).
- **Quotas: mandatory *presence* of cpu+memory** (value may be explicit `unlimited`), not
  mandatory limits.
- **Do not scope-creep** into baseline/platform/cluster concerns (see the Out-of-scope list).

---

## 1. What this run produced

For **every one of the 119 "pieces"** of the platform architecture, a detailed
implementation **SPEC** and **PLAN** were authored to disk and committed:

| Area | Pieces | Spec+Plan | Treatment |
|---|---|---|---|
| Workstream A (install/operate) | A1–A23 (23) | ✅ | Full build spec+plan; T0 pieces exhaustive |
| Workstream B (custom dev) | B1–B22 (22) | ✅ | Full build spec+plan; T0 pieces exhaustive |
| Workstream C (docs) | C1–C9 (9) | ✅ | Content/process spec+plan |
| Workstream D (dashboards) | D1–D3 (3) | ✅ | `GrafanaDashboard` XR delivery |
| Workstream E (training) | E1–E2 (2) | ✅ | Module/lab coverage |
| Workstream F (prod readiness) | F1–F6 (6) | ✅ | Hardening/verification |
| Architectural views | V6-01–V6-13 (13) | ✅ | Shaped: integration-contract + realization-map |
| ADRs | ADR-0001–0041 (41) | ✅ | Shaped: honor-requirements + enforcement-map |
| **Total** | **119** | **238 files** | ~2.8 MB content |

No stubs: smallest spec ≈ 6.5 KB, smallest plan ≈ 2.8 KB. Every spec has the full
11-section template incl. numbered `REQ-*` / `AC-*` (each AC maps to a REQ); every
plan has the full 8-section template incl. dependency map + PR/branch mapping.

## 2. How it was built (method)

1. **Frozen Canon first** (`_meta/`): `glossary.md` (exact CRD/product casings),
   `interface-contract.md` (31 CRDs/XRDs → owning component, CloudEvent taxonomy,
   SDK surfaces, connection-secret contract, audit-adapter interface),
   `piece-index.csv` (119 rows: id/kind/tier/wave/upstream/downstream/estimate),
   `waves.md` (W0–W4 build layering + cycle resolution). Every author bound to it.
2. **~24 parallel authoring subagents**, each a persona matched to its pieces
   (platform/security/runtime/observability/SDK/docs/SRE/architect…). Each did a
   cooperative draft → **self-adversarial red-team pass** → fix, before finalizing.
3. **Anti-drift discipline**: authors used ONLY Canon names; anything not in the
   source was tagged `[PROPOSED — not in source]` rather than silently invented.
   Result: **196 `[PROPOSED]` flags across 95 files** — a deliberate, greppable
   to-do list of genuine source gaps (see §5).
4. **Progressive commits** (checkpoints 1–14) so the ephemeral sandbox never lost
   work; survived a mid-run container restart with zero data loss.

## 3. Directory map

```
_meta/                       frozen Canon + templates + this handoff
  glossary.md  interface-contract.md  piece-index.csv  waves.md
  templates/{spec,plan}.md
  pending-operational-nfr-layer.md   ← queued OPS1–OPS6 phase (see §6)
specs/  components/{A..F}/spec-<ID>.md   views/spec-V6-NN.md   adr/spec-ADR-NNNN.md
plans/  components/{A..F}/plan-<ID>.md   views/plan-V6-NN.md   adr/plan-ADR-NNNN.md
```
Source (unchanged): `architecture-overview.md`, `architecture-backlog.md`,
`future-enhancements.md`, `adr/`, `todo.md`.

## 4. Build-wave layering (from the §14.7 DAG — authoritative in `_meta/waves.md`)

- **W0 foundation** (max day-one parallelism): A1,A2,A3,A4,A6,A7,A8,A9,A11,A13,A14,A15,B22
- **W1**: A5,A10,A18,A19,A20,B1,B2,B3,B4,B8,B11,B13
- **W2**: A16,A17,A22,B6,B9,B12,B16,B17,B20
- **W3**: A21,B7,B10,B14,B19(core)
- **W4**: A23,B5,B15,B18,B21
- **consumer band** (author-parallel, real-build after A/B): C*, D*, E*, F*
- Views + ADRs: authoring-parallel context (no build edges).

## 5. Key open questions & contradictions surfaced (act on these)

These were found by the authoring agents and recorded in the relevant specs' §10.
The high-blast-radius ones:

- **DAG edge gaps (CSV vs prose):** several real dependencies are missing from the
  §14.7 hard edges — **B11→B4** (memory adapter needs the `MemoryStore` composition),
  **A10→Postgres/B4**, **A16↔A14 edge direction** (call-dep vs shared-KB peer). The
  `piece-index.csv` was patched for B4 (T0, W1, downstream A18;A21;A23;B19;B11) but a
  full DAG reconciliation against the prose is still owed.
- **B19 → A23 → B5 apparent cycle:** resolved by splitting **B19-core** (W3: CRD +
  OPA + Argo + CloudEvent) from **B19-ui** (W4: Headlamp approval queue, ships with
  B5). Confirm this split is acceptable.
- **B20 persistent-volume mapping mechanism** is entirely absent from the source —
  the spec proposes an `AgentVolumeMapping` CRD (all fields `[PROPOSED]`). Needs an
  A6 design agreement + likely a new ADR before build.
- **A22 ↔ A21 ↔ B4 editor/composition ownership** overlap (who owns the
  `TenantOnboarding` editor and composition). Reconciled in-spec; needs sign-off.
- **CloudEvent per-event-type names** are deferred to B12 throughout — many specs
  use closest-fit namespaces from the closed taxonomy and flag the exact type names.
- **CRD field gaps**: per-primitive connection-secret fields (B4), `Approval` schema
  details (B19/ADR-0017), MCP per-service secret/auth shapes (A17/ADR-0020),
  tenant-quota fields (ADR-0016) — all `[PROPOSED]`, all traceable via grep.

Find them all: `grep -rn 'PROPOSED — not in source' specs plans`
and `grep -rln '## 10' ... ` then read each §10 "Risks & Open Questions".

## 6. What is still TO DO (in order)

1. **Finish the in-flight gap-fill commit.** Three gap-fill agents were regenerating
   A1/A4/A6/A7/A11 (fresh T0 quality) and finalizing the C1/C5/ADR-0040/ADR-0041
   plans when the container restarted; the shell entered a degraded state. **Verify
   `plans/adr/plan-ADR-0041.md` is a full plan (not the ~119-byte stub seen mid-write)**,
   then commit + push. (Re-run a gap-fill agent for any file still thin.)
2. **Consistency normalization sweep** (deterministic + 1 agent): reconcile
   `piece-index.csv` upstream/downstream reciprocity; verify every cited CRD/event/
   ADR exists in Canon; normalize the truncated `canon-*` hash lengths in
   front-matter (cosmetic; all bind to the same frozen Canon). No real drift found
   in spot-checks, but a full pass is owed.
3. **Targeted adversarial review on the ~32 T0 contract pieces** (A1,A4,A5,A6,A7,
   A11,A17,A18,A22,B3,B4,B6,B7,B12,B13,B16,B19,B22 + T0 views/ADRs): dedicated
   red-team agents pushing hard on interface soundness, untestable ACs, and the
   `[PROPOSED]` decisions that are load-bearing. (Self-red-team was done per piece;
   this is the independent second pass.)
4. **Build the per-piece PR graph** (user chose ~119 PRs, one per piece, stacked on
   `wave/N` integration branches per `_meta/waves.md`; spec+plan for a piece travel
   together; merges are conflict-free since each piece is its own files). Recipe and
   branch/PR naming are in the meta-plan; not yet executed — everything is currently
   on the single working branch.
5. **Operational / NFR architecture layer** (queued in
   `_meta/pending-operational-nfr-layer.md`): **OPS1** scale & cost, **OPS2** DR,
   **OPS3** key/secret management, **OPS4** multi-tenant compute isolation, **OPS5**
   day-2 ops, **OPS6** Rego human-factors & lifecycle. Each: cross-cutting NFR spec +
   realization/verification plan, **hard adversarial review**, and **proposed-ADR
   titles surfaced** where decisions are genuinely open (don't auto-decide).
6. **Open the umbrella PR** to `main` once the graph is built (currently no PR exists
   for this branch).

## 7. Provenance / integrity notes

- Source architecture docs were **not modified** — all output is additive under
  `specs/`, `plans/`, `_meta/`.
- The run survived a container restart; all committed work intact. The only at-risk
  artifact at handoff time is the ADR-0041 plan (see §6.1) — verify before relying.
- Effort estimates in plans are **real-build days** (S≤0.5d, M≤2d, L≤5d, XL>5d),
  not authoring time.
