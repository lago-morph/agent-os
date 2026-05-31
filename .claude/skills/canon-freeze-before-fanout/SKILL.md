---
name: canon-freeze-before-fanout
description: Extract a frozen "Canon" (glossary + interface contract + piece index) from source material before dispatching parallel authoring agents, so independent agents cannot drift on shared names or invent fields. Use whenever three or more agents will independently produce files that reference a shared set of named entities — CRDs, APIs, event types, modules, glossary terms — and naming consistency across their outputs matters. Trigger phrasings include "freeze the canon", "build a shared vocabulary before fanning out", "make a glossary/interface contract for the agents", or proactively right before any multi-agent authoring fan-out over a common domain. Do NOT use for single-agent tasks or fan-outs whose outputs share no vocabulary.
---

# Skill: canon-freeze-before-fanout

When many independent authoring agents write parts of one coherent artifact set in
parallel, they drift: the same entity gets spelled three ways, fields get invented,
section or ADR numbers get miscited — and the errors look authoritative, so they
propagate. This skill eliminates that failure class by extracting an authoritative,
frozen **Canon** from the source *once, serially, before any fan-out*, then binding every
downstream agent to it.

## When to use

- Before dispatching ≥3 agents that will independently produce files referencing shared
  named entities (CRDs, APIs, events, modules, terms).
- Proactively at the start of any parallel authoring run over a common domain.
- NOT for single-agent tasks, or fan-outs whose outputs share no vocabulary.

## Workflow

1. **Identify the source's authoritative naming surfaces** — inventory tables, glossary
   sections, schema definitions, decision records.
2. **Dispatch one serial agent** (foreground) to extract the Canon into `_meta/`:
   - `glossary.md` — every proper noun an author could misspell, with canonical casing +
     a one-line meaning. Marked FROZEN.
   - `interface-contract.md` — the cross-piece registry: every shared entity → owning
     unit + the fields the source *actually states*. Gaps marked "not specified in
     source", **never invented**.
   - `piece-index.csv` — one row per unit (id, kind, tier, dependencies, …).
   - Optionally `waves.md` — a dependency-derived build/authoring layering.
   Instruct the agent explicitly: extract what the source *says*; do not invent; mark
   gaps; capture exact casing.
3. **Require a validation summary** from the agent: entity count, row count, and any
   contradictions/gaps it found in the source (these are high-value findings).
4. **Spot-check the Canon yourself** — counts, a few spellings, the gaps section. Fix
   extraction errors before freezing. The Canon is the single load-bearing artifact; an
   error in it propagates to every downstream file.
5. **Commit the Canon.** Treat it as immutable for the run. A mid-run change invalidates
   work built on it; if it must change, that is a deliberate Canon revision, not a silent
   edit.
6. **Bind every downstream author** in its prompt: "MANDATORY FIRST STEP — read the Canon
   files. Use ONLY names from the Canon. Anything not in the Canon must be tagged
   `[PROPOSED — not in source]`, never asserted."
7. **After the run, `grep -rn 'PROPOSED — not in source'`** — that is your complete,
   deduplicated list of real source gaps.

## Anti-patterns

- Letting each author re-derive vocabulary from the raw source (wastes budget, produces
  divergent readings).
- Allowing the Canon to be edited mid-run.
- Inventing fields to fill gaps instead of flagging them.
- Skipping the human spot-check of the Canon before freezing.

## Acceptance criteria

1. A frozen Canon exists and is committed before any parallel authoring begins.
2. The Canon's entity list matches the source's authoritative inventory (verifiable count).
3. Every author prompt references the Canon and the `[PROPOSED]` tagging rule.
4. Post-run, a single grep produces the complete gap list.
5. No two output files spell the same shared entity differently.
