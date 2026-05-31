---
name: coverage-manifest-gapfill
description: Verify that a parallel fan-out actually produced its full expected file set by checking on-disk reality against a manifest (existence AND minimum substance), then gap-fill only the genuinely missing or stub outputs — instead of trusting agent self-reports or blindly re-running the whole batch. Use after any fan-out expected to produce an enumerable set of files, and ALWAYS after a container restart or any agent reporting a stall, rate-limit, or partial failure mid-batch. Trigger phrasings include "make sure everything's covered", "what's missing?", "gap-fill the batch", "verify completeness", "did all the agents finish?". Do NOT use for tasks with no enumerable expected output set.
---

# Skill: coverage-manifest-gapfill

When a large fan-out is supposed to produce a known set of output files, "the agents said
they're done" is not proof. Agents stall, hit rate limits, the container restarts, and
files silently never get written. This skill drives a fan-out to *verified* completeness:
check actual on-disk files against a manifest, compute the exact missing set, and dispatch
narrowly-scoped gap-fill agents.

## When to use

- After any fan-out expected to produce an enumerable file set.
- ALWAYS after a container restart, or after any agent reports a rate-limit / stall /
  partial failure mid-batch.
- NOT for tasks with no enumerable expected output set.

## Workflow

1. **Read the manifest** (e.g. `_meta/piece-index.csv`); derive every expected file path
   from each unit ID + the naming convention. Handle prefix variants (e.g. plain
   component IDs vs `V6-`/`ADR-` prefixed paths).
2. **Commit the working tree first.** Otherwise newly-written-but-uncommitted files look
   "missing" or get conflated, producing false positives and negatives both.
3. **Check each expected path**: exists? If yes, is it above a stub threshold (byte size)
   and does it contain the required template sections? Build the precise missing/stub set.
4. **If the set is empty**, print "N/N verified", commit, done.
5. **Otherwise dispatch gap-fill agents** scoped to *only* the missing/stub IDs, each
   bound to the same Canon/templates as the original run. Prefer several small agents over
   one large one (rate-limit resilience — if one fails you don't lose the whole retry).
6. **Re-check until coverage is verified complete.** Then commit + push.

## Useful commands

```bash
# stub audit — smallest files first
find specs -name 'spec-*.md' -printf '%s %p\n' | sort -n | head
# required-section check
for f in $(find specs -name 'spec-*.md'); do grep -q '## 9. Acceptance' "$f" || echo "no AC: $f"; done
```

## Anti-patterns

- Trusting agent self-reports of "done", or "already exists" (an agent often merely sees a
  peer's file). Only a disk check against an explicit manifest is proof.
- Re-running the entire fan-out to fill a few gaps (wasteful; risks overwriting good work).
- Checking coverage before committing the working tree.
- Counting files without a substance threshold (a half-written file counts as "present").
- One giant gap-fill agent (a rate limit loses the whole retry).

## Acceptance criteria

1. Coverage is computed from on-disk reality against an explicit manifest, not reports.
2. The working tree is committed before the check runs.
3. A stub threshold distinguishes real files from placeholders.
4. Gap-fill is scoped to exactly the missing/stub set.
5. The final state is a clean tree at verified full coverage, pushed.
