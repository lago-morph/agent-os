# AGENTS.md suggestions — 2026-05-30-8

These are proposed additions to the project's agents file (typically `AGENTS.md` at the
repo root). Each section contains the exact text to paste and the argument for it.
Decide each on its own merits.

---

## Suggestion 1: Freeze a Canon before any parallel authoring fan-out

### Proposed addition

> **Freeze a Canon before fanning out.** Before dispatching three or more agents that
> will independently produce files referencing shared named entities (CRDs, events,
> APIs, modules, glossary terms), first run ONE serial agent to extract a frozen Canon
> into `_meta/` (`glossary.md`, `interface-contract.md`, `piece-index.csv`). Every
> downstream author must read it first and use ONLY its names. Spot-check the Canon
> before freezing — it is the single load-bearing artifact.
>
> *Grounded in: the 21-agent spec/plan fan-out, where the frozen Canon produced zero
> name drift across 238 files.*

### Why this earns its place in your agents file

Independent agents drift deterministically: the same CRD gets three spellings, fields
get invented, ADR numbers get miscited — and these errors look authoritative, so they
propagate. The marginal cost of prevention is one serial agent (~10 minutes) plus a
spot-check. The cost of skipping it is a corpus of mutually-incompatible specs that a
downstream implementer builds conflicting code against. In this session the Canon also
*caught a real bug* (B4 missing from the wave list) before any fan-out wasted budget on
it. One cheap upstream step, large downstream payoff.

---

## Suggestion 2: Tag, don't invent — `[PROPOSED — not in source]`

### Proposed addition

> **Flag gaps, never fabricate.** When authoring a spec/plan/doc from source material,
> use only facts the source states. If you need a name, field, or value the source
> doesn't provide, write it followed by `[PROPOSED — not in source]` rather than
> asserting it as fact. After a run, `grep -rn 'PROPOSED — not in source'` is the
> complete gap list.
>
> *Grounded in: 196 `[PROPOSED]` flags across 95 files became the actionable gap backlog.*

### Why this earns its place in your agents file

A fabricated CRD field is indistinguishable from a real one to a future reader — it's a
landmine. The flag converts silent hallucination into a visible, greppable to-do. The
cost is a few words inline; the payoff is that the entire set of genuine source gaps
becomes a single searchable backlog (in this session, the B20 PV-mapping mechanism, the
missing CloudEvent type names, and the per-primitive secret fields all surfaced this
way) instead of leaking into the corpus as false certainty.

---

## Suggestion 3: Commit-and-push checkpoints during long fan-outs

### Proposed addition

> **Checkpoint progressively during long parallel runs.** When a multi-agent run will
> last more than a few minutes, commit and push completed work in waves rather than only
> at the end. The remote execution sandbox is ephemeral and can restart at any time;
> only committed-and-pushed work survives.
>
> *Grounded in: a mid-run container restart that lost zero committed work because of
> checkpoints 1–14.*

### Why this earns its place in your agents file

This session's container restarted mid-run, killing all in-flight agents and background
pollers. Because work had been committed in 14 progressive checkpoints, nothing was
lost and recovery was a coverage-audit, not a redo. The cost is a `git commit && git
push` every wave (seconds); the avoided cost is regenerating hours of agent output. The
asymmetry is overwhelming for any run long enough to outlast a sandbox's patience.

---

## Suggestion 4: Verify completeness against a manifest, not against agent reports

### Proposed addition

> **Verify fan-out completeness from disk, against a manifest — never from self-reports.**
> After a fan-out expected to produce an enumerable file set, commit the working tree,
> then check each expected path from the manifest for existence AND minimum substance
> (size + required sections). Gap-fill only the genuinely-missing/stub set.
>
> *Grounded in: a manifest check that pinpointed exactly one missing file (ADR-0041 plan)
> among 238 after a restart.*

### Why this earns its place in your agents file

Agents report "done" optimistically and report files "already existing" when they merely
see a peer's work — neither is proof. Only a disk check against an explicit manifest is.
In this session the naive read showed five files "missing"; committing first collapsed
that to one true gap. The cost is a short shell loop; the payoff is not shipping a
silently-incomplete deliverable and not blindly re-running a whole expensive batch.

---

## Suggestion 5: Run a dedicated adversarial planner in parallel for high-stakes plans

### Proposed addition

> **Red-team high-stakes plans in parallel.** Before any execution that dispatches many
> agents or is costly/hard to reverse, run a dedicated adversarial planner alongside the
> cooperative ones. Its findings are gold even when its headline recommendation conflicts
> with explicit instructions — separate its *findings* (adopt the quality controls) from
> its *conclusions* (which user intent may override). Surface the tension at the gate.
>
> *Grounded in: the red-teamer's "freeze a Canon" became the run's most valuable control,
> while its "cut scope" was overridden by user instruction.*

### Why this earns its place in your agents file

A lone planner shares the optimism bias of whoever will execute. A parallel red-teamer
costs one extra agent and routinely finds the failure mode that would have cost the most
— here, the drift/hallucination risk across 119 independent authors. The key discipline
the session demonstrated: a red-teamer's recommendation may conflict with what the user
explicitly asked for, but its *risk controls* are usually separable and worth adopting
regardless. Cheap insurance against expensive rework.

---

## Suggestion 6: Subagents return summaries, not content

### Proposed addition

> **Subagents return compact summaries, not file contents.** When dispatching authoring
> agents that write to disk, instruct each to return only: files written, flags raised,
> and open questions — never the file bodies. This keeps the orchestrator's context
> small enough to coordinate many agents.
>
> *Grounded in: ~21 concurrent authors coordinated without orchestrator context exhaustion.*

### Why this earns its place in your agents file

The orchestrator tracking a wide fan-out is the real bottleneck, not the leaves. If each
agent returns full file bodies, the orchestrator's context saturates and it loses track
of which agents finished. Requiring compact summaries (files + flags + open questions)
let this session run ~21 authors concurrently and survive a restart with a coherent
recovery. The cost is one line in each brief; the payoff is the ability to fan out wide
at all.
