# Spec: `spec-corpus-conformance-sweep`

## Intent

Sometimes a large authored corpus is discovered to be built on the **wrong version or
convention** of a foundational technology — and the error is pervasive and load-bearing,
not a stray word. In the session that produced this spec, ~270 references across the
corpus used the **Crossplane v1** resource model ("claims", the `X`-prefixed
composite/claim split, "claim admission"), codified in a foundational ADR, while the
target platform is **Crossplane v2** (claims removed, composite resources namespaced
directly). A blind find-and-replace would corrupt the meaning, because some passages
describe genuine v1 *behavior* (the two-object admission flow, conversion webhooks) that
must be **re-expressed**, not renamed. This skill runs that correction as a coordinated,
verifiable pass: handle the governing ADR the right way, fan out the re-expression across
the corpus, and prove the result.

## Trigger

- Direct: "the specs are written against the wrong version of X", "sweep the corpus to
  v2", "we're on the wrong convention everywhere — fix it", "conformance pass".
- Proactive: when a decision-review or grep reveals a foundational technology/convention
  is used pervasively in a form the platform won't use, and an ADR codifies the wrong one.
- Negative: a handful of references (just edit them); a pure rename with no logic change
  (a scripted replace + review suffices, no skill needed).

## Inputs

- The corpus to sweep (specs/plans/ADRs) and the count + locations of the offending
  references (from a grep).
- The authoritative new model (e.g. "Crossplane v2: namespaced XRs, no claims").
- The governing ADR(s) that codify the old convention.
- Any frozen Canon (glossary, interface-contract) that also encodes the old terms.

## Outputs

- A new ADR adopting the correct model; the old ADR marked `Superseded` and **moved to a
  `superseded/` subdirectory** so future agents don't read the stale one.
- The corpus re-expressed in the correct model (terminology **and** logic), on a branch.
- A verification report: residual-term grep (expect zero), a logic re-read pass, a
  Canon-reconciliation check.
- One PR for review.

## Workflow

1. **Quantify first.** Grep the whole corpus for the offending terms; record the count
   and the files. Separate genuine offenders from false positives (e.g. JWT "claims" is
   unrelated to Crossplane "claims" — context-filter before counting or editing).
2. **Handle the ADR via the standard lifecycle.** Create a new ADR for the correct model;
   set the old ADR's status to `Superseded by <new>`; **move the superseded ADR (and its
   spec/plan, if the project pairs them) into an `adr/superseded/` subdirectory**; add a
   one-line guard banner to the new ADR. (Moving beats guard-text-only: it keeps curious
   agents from reading the stale file at all.)
3. **Fan out the sweep by slice.** Dispatch parallel subagents, one per corpus slice
   (e.g. per workstream / per directory). Use a strong model — this is judgment work, not
   substitution. Each subagent: re-express terminology AND the affected *logic*
   (admission, naming, versioning) in the new model; where a passage describes real
   old-model behavior, rewrite the behavior, not just the noun. Subagents write to disk;
   the orchestrator commits centrally.
4. **Fold in adjacent decided cleanups** that touch the same files (cheaper than a second
   pass), e.g. adding missing types to the glossary.
5. **Verify.** Grep for residual offending terms (target zero, minus the known
   false-positive class); do at least one full re-read pass focused on the re-expressed
   logic; reconcile every cited type/event/ADR against Canon.
6. **Deliver one PR**; do not merge unilaterally.

## Concrete examples

**Example A — the ADR move.** `ADR-0041` codified the v1 "XR ↔ claim naming convention."
Handling: create `adr/00NN-crossplane-v2-resource-model.md`; set ADR-0041 status to
`Superseded by 00NN`; `git mv adr/0041-*.md adr/superseded/` (and its spec/plan); the new
ADR opens with a guard note. Result: an agent browsing `adr/` no longer trips over the v1
convention.

**Example B — re-expression, not replace.** A v1 line read: *"All XRDs are namespaced
claims (the `X`-prefixed name is the cluster-scoped composite)."* A blind replace of
"claim"→"XR" yields nonsense. The correct re-expression: *"All composite resources (XRs)
are namespaced directly (Crossplane v2); there is no separate claim object and no
`X`-prefix duality."* The *logic* (two objects → one) changed, not just the word.

## Anti-patterns

- **Blind find-and-replace** of the offending term. The logic, not just the vocabulary,
  is wrong in places.
- **Context-blind grepping.** "claim" also means JWT/OIDC claims — don't sweep those.
  Filter by surrounding context before counting or editing.
- **Editing the old ADR in place** instead of superseding + moving it — loses the
  decision history and leaves a stale file agents will read.
- **One mega-agent for the whole corpus.** It will run out of context or time; slice it.
  (A sibling session's single consolidator agent timed out reading ~100 KB.)
- **Skipping the logic re-read.** The residual-term grep proves the words are gone; only a
  read proves the behavior is now correct.

## Acceptance criteria

1. The governing ADR is superseded (status set) and physically moved to `superseded/`; a
   new ADR states the correct model.
2. A residual-term grep returns zero genuine offenders.
3. A full re-read confirms re-expressed *logic* is correct, not just renamed.
4. All cited types/events/ADRs reconcile against Canon.
5. The change ships as one reviewable PR; nothing merged without approval.

## Files this skill creates / modifies

- `adr/00NN-<new-model>.md` — the new ADR (create).
- `adr/superseded/<old-adr>.md` (+ paired spec/plan) — moved (git mv).
- The corpus files in scope — re-expressed (edit, via subagents).
- A verification note (in the PR body or a `…/conformance-verify.md`).
