# Spec: `coverage-manifest-gapfill`

## Intent

When a large fan-out is supposed to produce N×M deterministic output files, "the agents
said they're done" is not proof. Agents stall, hit rate limits, the container restarts,
and some files silently never get written. This skill drives a fan-out to *verified*
completeness by checking actual on-disk files against a manifest, computing the exact
missing set, and dispatching narrowly-scoped gap-fill agents — instead of trusting
self-reports or blindly re-running the whole batch. In the session that produced this
skill, a mid-run container restart killed several agents; a manifest check revealed the
true state was 119/119 specs but 118/119 plans, pinpointing a single missing file
(ADR-0041's plan) that a focused agent then filled.

## Trigger

- Direct: "make sure everything's covered", "what's missing?", "gap-fill the batch",
  "verify completeness".
- Proactive: after any fan-out expected to produce a known, enumerable set of files;
  always after a container restart or any agent reporting a rate-limit/stall mid-batch.
- Negative: skip for tasks with no enumerable expected output set.

## Inputs

- A manifest of expected units (e.g. `_meta/piece-index.csv`) with stable IDs.
- The naming convention mapping each ID → expected file path(s).
- A minimum-substance threshold (byte size or required-section check) to distinguish a
  real file from a stub.

## Outputs

- A printed coverage report (have / expected, the exact missing list).
- Gap-fill agents dispatched only for genuinely-missing or stub files.
- A clean commit once coverage is verified complete.

## Workflow

1. Read the manifest; derive every expected file path from each unit ID + the naming
   convention. Handle prefix variants (e.g. components vs `V6-`/`ADR-` prefixed paths).
2. **Commit any uncommitted working-tree files first** — otherwise newly-written files
   look "missing" or get conflated. (This session's pre-commit read produced false
   "missing" entries that vanished after committing.)
3. For each expected path: exists? If yes, is it above the stub threshold and does it
   contain the required template sections? Build the precise missing/stub set.
4. If the set is empty, print "N/N verified", commit, done.
5. Otherwise dispatch gap-fill agents scoped to *only* the missing/stub IDs, each bound
   to the same Canon/templates as the original run. Prefer several small agents over one
   large one (rate-limit resilience).
6. Re-run the check until coverage is verified. Then commit + push.

## Concrete examples

**Example 1 (this session).** After a restart, the naive check reported A1/A4/A6/A7/A11
"missing." Committing stragglers first collapsed that to a single true gap:
`plans/adr/plan-ADR-0041.md`. A focused agent produced a full 9.9 KB enforcement-map
plan; coverage reached 119/119 + 119/119; final commit `8e30fc2`. The size audit
(`find … -printf '%s %p' | sort -n | head`) confirmed no stubs — smallest spec 6.5 KB.

**Example 2 (generalized).** A batch generates `report-{01..50}.md`. The check finds 47
present, 2 missing (12, 39), 1 stub (`report-22.md`, 180 bytes). Dispatch one agent for
{12, 22, 39}; re-check; commit at 50/50.

## Anti-patterns

- **Trusting agent self-reports of "done."** Several agents in this session reported
  files "already existed" — true, but only verifiable by checking disk, not by belief.
- **Re-running the entire fan-out to fill a few gaps.** Wasteful and risks overwriting
  good work. Scope to the missing set.
- **Checking coverage before committing the working tree.** Produces false positives and
  false negatives both.
- **Counting files without a substance threshold.** A 119-byte half-written file counts
  as "present" but isn't. Check size + required sections.
- **One giant gap-fill agent.** If it hits a rate limit you lose the whole retry. Many
  small agents degrade gracefully.

## Acceptance criteria

1. Coverage is computed from on-disk reality against an explicit manifest, not reports.
2. The working tree is committed before the check runs.
3. A stub threshold distinguishes real files from placeholders.
4. Gap-fill is scoped to exactly the missing/stub set.
5. The final state is a clean tree at verified full coverage, pushed.

## Files this skill creates / modifies

- (reads) `_meta/piece-index.csv` or equivalent manifest.
- (writes) only the previously-missing/stub output files, via gap-fill agents.
- A coverage-verified commit on the working branch.
