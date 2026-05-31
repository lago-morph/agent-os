# AGENTS.md suggestions — 2026-05-31-19

These are proposed additions to the project's agents file (`AGENTS.md` at the repo root). Each section contains:

1. **Proposed addition** — the exact text to paste.
2. **Why this earns its place in your agents file** — the argument for doing it, grounded in something that happened (or nearly happened).

Decide each on its own merits. Skip ones that don't apply to your operating posture; copy-paste the ones that do.

---

## Suggestion 1: Commit before investigating a timeout

### Proposed addition

> **Commit before investigating.** When a subagent hits a stream idle timeout, run `git add -A && git commit` *before* running any greps, reading any files, or assessing the damage. The sandbox is ephemeral. The completed work is on disk. Commit first, investigate second — always.
>
> *Grounded in: two agent timeouts during the Crossplane v2 corpus sweep (Agents 2 and 5).*

### Why this earns its place in your agents file

During the Crossplane v2 sweep, Agents 2 and 5 both hit stream idle timeouts after 45+ minutes. Both had written ~80% of their work to disk. The checkpoint commit strategy — committing immediately on each completion notification — preserved both agents' partial work. If even one more tool call (a grep, a read) had triggered sandbox recycling before that commit, an hour of Opus-agent work would have vanished silently.

The cost of this rule: one `git add -A && git commit` before you do anything else. The cost of not having it: losing everything the timed-out agent wrote. The asymmetry is extreme. This rule should be a reflex, not a judgment call.

---

## Suggestion 2: Never re-run a timed-out agent from scratch

### Proposed addition

> **Do not re-run timed-out agents from scratch.** If a subagent timed out after writing partial work, scope recovery to the gap only. Use `grep` to identify what was missed, then fix manually (< 15 lines) or with a mini-agent scoped to gap files only. Re-running the full agent wastes time re-doing completed files and may timeout again at the same choke point.
>
> *Grounded in: Agent 5 (Views) timed out with 11 of 13 view files complete; recovery took 15 minutes vs re-running would have taken 45+ and might have timed out again.*

### Why this earns its place in your agents file

Agent 5 in the corpus sweep timed out after writing 22 of 26 files. The remaining gap was 6 lines across 3 files. A targeted grep identified them in under 2 minutes; manual edits fixed them in 5. The full recovery was under 15 minutes. Re-running the full agent would have spent 45+ minutes re-expressing the 22 already-correct files, potentially introducing drift on completed work, and would likely have hit the same timeout at the same resource-intensive point. The gap-first pattern was strictly better on every dimension: faster, safer, and more predictable.

---

## Suggestion 3: Classify "claim" hits before editing (JWT vs Crossplane)

### Proposed addition

> **Classify before editing.** When grepping for terms that exist in multiple semantic contexts (e.g., "claim" for Crossplane vs JWT/OIDC), read 3–5 lines of context for each hit before editing. Always maintain an explicit DO NOT CHANGE list covering false positives. `clusterOIDCClaimMapping`, `platform_tenants` claim schema, and similar Keycloak/OIDC constructs are never renamed — they are JWT claims, a completely different concept from Crossplane claims.
>
> *Grounded in: ~700 "claim" occurrences in the corpus, ~680 of which were correct JWT/OIDC references that must not change.*

### Why this earns its place in your agents file

The Crossplane v2 sweep targeted "claim" as Crossplane-context usage (XRC, claim admission, claim shape). But the same word appears in `clusterOIDCClaimMapping`, `platform_tenants` claim schema, JWT claim validation, and plain English phrases like "we claim responsibility." All of those are correct and must not change. Without explicit DO NOT CHANGE entries in every agent brief, agents would have renamed `clusterOIDCClaimMapping` — a Keycloak field — to something that doesn't exist and would break authentication. Explicit classification at the start of each edit pass costs 2 minutes and prevents an entire class of silent correctness errors.

---

## Suggestion 4: Embed the full rename map and ruling list in every subagent brief

### Proposed addition

> **Full brief, every agent.** Do not brief subagents from memory or with abbreviated lists. Each brief must embed: (1) the complete rename map with "already correct (no change)" entries explicit, (2) the complete ruling list stated as what the component must DO, (3) the explicit DO NOT CHANGE exceptions. The ruling list from the Crossplane sweep was ~800 words per brief; that length was necessary — agents working from abbreviated briefs missed half the rulings.
>
> *Grounded in: 14 rulings across 6 parallel agents; abbreviated briefs in early drafts produced agents that applied only 4–6 of 14 rulings.*

### Why this earns its place in your agents file

The temptation when briefing 6 parallel agents is to keep briefs short — summarize the rename map, reference the decisions log by number. In early draft briefs for this session, that approach was considered. It was rejected in favor of embedding the full list in each brief, adding ~800 words of ruling content per brief. The payoff: agents that completed cleanly (1, 3, 4, 6) threaded all 14 rulings correctly on the first pass. A brief that says "apply D-01 through D-08" produces an agent that guesses at what those rulings mean. A brief that says "D-01: modes are `system` and `system-mediated`; `user-cred` is retired; thread this into §2 auth-mode table and §3 REQ-*-NN" produces correct output.

---

## Suggestion 5: Partition before launching — check for overlap

### Proposed addition

> **Verify partition before launching parallel agents.** Before launching N parallel agents over a file corpus, explicitly list each agent's file set and confirm there is no overlap. Two agents writing to the same file will produce silent overwrites — the second agent's output replaces the first's with no warning, no merge conflict, no error.
>
> *Grounded in: 6-agent corpus sweep over 250 files; any overlap would have silently destroyed work from one agent.*

### Why this earns its place in your agents file

Parallel agents write to disk independently. There is no locking mechanism, no merge process, no "conflict detected" warning when two agents touch the same file. The second one to finish wins, silently. In the corpus sweep, the partition was carefully defined (A-workstream / B-workstream / C–F+OPS / Views / ADRs / Canon), and the partition was checked before launch. Had any ADR file been assigned to both Agent 4 and Agent 6, one agent's threading work would have been overwritten. The check takes 2 minutes. Rediscovering the overwrite after the fact (if you even notice) takes far longer.

---

## Suggestion 6: Do structural work synchronously before launching subagents

### Proposed addition

> **Structural work before subagents.** File moves, directory creation, new files that agents will reference — do these synchronously before launching any subagents. Subagents reading real paths is correct; subagents referencing paths that don't exist yet produces brittle briefs and subtle errors. Rule: if the work involves moving a file, creating a directory, or creating a new reference document, do it yourself first and commit before briefing agents.
>
> *Grounded in: ADR-0041 supersession and ADR-0044 creation done synchronously before the 6-agent fan-out; agents referenced `specs/adr/spec-ADR-0044.md` because it existed.*

### Why this earns its place in your agents file

In the Crossplane sweep, the ADR mechanics (creating `specs/adr/superseded/`, moving ADR-0041 spec/plan there, creating the new ADR-0044 source file, authoring `specs/adr/spec-ADR-0044.md`) were done manually and committed before any subagent was launched. This meant every subagent brief could say "reference `specs/adr/spec-ADR-0044.md`" and agents could read it. If this work had been delegated to Agent 1 (Canon + ADR-0044) while the other 5 were simultaneously briefed, Agents 2–6 would have referenced a file that didn't exist when they started. The fix cost is not "one small edit later" — it's re-running each agent that made decisions based on a non-existent file. Structural work is a dependency; treat it as one.

---

## Suggestion 7: Push after every checkpoint commit, not just at the end

### Proposed addition

> **Push on every checkpoint commit.** After each `git commit` during a long parallel operation, immediately run `git push`. The sandbox is ephemeral — an idle-timeout or container recycle between your last commit and a push loses all commits since the last push. Commit + push is the atomic unit of "this work is safe."
>
> *Grounded in: 7 checkpoint commits across a 3-hour session; each was immediately pushed; no work was lost despite two agent timeouts and one near-recycle event.*

### Why this earns its place in your agents file

During the corpus sweep, 7 checkpoint commits were made. Each was pushed immediately after the commit. This meant that even when Agent 5 timed out 45 minutes in, the preceding checkpoint commit (capturing Agent 4's OPS-trim work) was already on the remote. A "commit locally and push at the end" strategy would have put 45+ minutes of committed work at risk of sandbox recycling. The cost of pushing after each commit: ~5 seconds per push. The cost of not doing it: losing everything since the last push if the sandbox recycles. On an ephemeral sandbox, commit-and-push is the unit of durable work.

---

## Suggestion 8: Read the heaviest file in each slice before briefing

### Proposed addition

> **Read before briefing.** Before launching subagents over a corpus slice, read at minimum: (1) the heaviest file in the slice (the one with the most occurrences of what you're changing), and (2) the Canon / authoritative source if one exists. This grounds each brief in actual vocabulary patterns and prevents briefs that describe changes the files don't actually need.
>
> *Grounded in: reading `spec-B4.md` (39 "claim" occurrences) before briefing revealed the concrete vocabulary patterns that informed all 6 agent briefs.*

### Why this earns its place in your agents file

Before the 6 agents were launched in the corpus sweep, three files were read: the ADR-0041 spec (to understand the concrete v1 claim vocabulary), the B4 spec (the heaviest file with 39 "claim" occurrences, to understand real usage patterns in context), and the Canon glossary (the authoritative naming source). This pre-reading took 10 minutes. It surfaced vocabulary patterns that weren't obvious from the decisions log alone — for example, that "claim shape" and "claim admission" were the specific phrases needing re-expression, while "claim" in "we claim this requirement" was plain English and correct. Without those reads, the rename map in each brief would have been abstract; agents would have applied it inconsistently. With them, the map was concrete and four of six agents completed cleanly.
