# Spec: `parallel-corpus-sweep`

## Intent

When a large spec/plan corpus (100+ files) needs a uniform terminology change AND a set of decided rulings threaded into the right places, a single sequential agent will run out of context or time. This skill orchestrates a parallel fan-out: one Opus subagent per workstream slice, each given a complete brief covering both the terminology change and the ruling list, writing to non-overlapping file sets. The orchestrator holds the plan, integrates via checkpoint commits, and does a targeted verification pass after all agents complete.

The session that produced this skill swept ~270 Crossplane v1 terminology references out of a 250-file corpus while simultaneously threading 14 separate rulings — in a single PR, merged the same session.

## Trigger

**Direct triggers:**
- "Apply the decisions to the specs"
- "Sweep the corpus for X and update to Y"
- "Thread all these rulings into the spec files"
- "The terminology has changed; update everything"

**Proactive triggers:**
- A HANDOFF or DECISIONS-LOG exists with a list of "apply these to the specs" items covering 50+ files.
- The task description says "re-express" or "update all specs to reflect" a set of decisions.

**Negative triggers:**
- Fewer than ~20 files affected (do it sequentially).
- The changes are architectural decisions (use `architecture-decision-review` instead).

## Inputs

- A HANDOFF or decisions document listing what changes to apply.
- A Canon glossary / interface-contract (if one exists) as the naming authority.
- The corpus directory structure (specs/, plans/, etc.).

## Outputs

- All affected files updated in-place.
- Checkpoint commits after each agent completes (not at the end).
- A verification grep confirming zero residuals.
- Updated meta documents (HANDOFF, DECISIONS-LOG) marking the work done.

## Workflow

### Step 1 — Read before briefing

Read at minimum:
1. The HANDOFF / decisions source document (the full list of what to apply).
2. The heaviest file in each slice (the one with the most occurrences of the old terminology) — to understand vocabulary patterns in context.
3. The Canon glossary and interface-contract, if they exist — these are the authoritative name source.

Do NOT brief subagents from memory. The briefs must embed the full rename map, the full ruling list, and the DO NOT CHANGE exceptions.

### Step 2 — Do structural / mechanical work directly

Before launching subagents, handle anything that is:
- File moves or directory creation (superseded dirs, new ADRs).
- New files all agents will reference (e.g., a new ADR that replaces the old one).
- Short enough to write in < 5 minutes.

Committing these first means subagents can reference real paths.

### Step 3 — Partition the corpus into non-overlapping slices

Typical partition for a 250-file spec/plan corpus:
- **Canon files** (glossary, interface-contract) — one agent, Opus.
- **Workstream A** (A* specs + plans) — one agent, Opus.
- **Workstream B** (B* specs + plans) — one agent, Opus (B is usually heaviest).
- **C/D/E/F + OPS** (smaller workstreams + operational layer) — one agent, Opus.
- **Views** (V6-* specs + plans) — one agent, Opus.
- **ADRs** (ADR-* specs + plans) — one agent, Opus.

**Hard constraint:** no two agents touch the same file. Check the partition before launching.

### Step 4 — Write comprehensive subagent briefs

Each brief must contain:

1. **The full rename map** — every old-name → new-name pair, with "already v2-consistent (no change)" entries explicit.
2. **The full ruling list** — every ruling, stated as what the component must DO (not just the ruling number). Include which components in the slice are owners vs consumers.
3. **DO NOT CHANGE list** — explicit exceptions (e.g., "JWT/OIDC claims — different concept from Crossplane claims; leave all of these").
4. **Judgment instruction** — "This is re-expression of logic, not blind find-and-replace."
5. **Scope boundary** — "Do NOT touch files outside your slice." Named explicitly.
6. **No-commit instruction** — "Do NOT make git commits. The orchestrator commits."

### Step 5 — Launch all agents in parallel

Send all agents in a single message (parallel execution). Use Opus for every slice — re-expression of logic requires judgment.

### Step 6 — Commit on each completion

On each completion notification:
1. Run `git status --short` to see what the agent wrote.
2. Run `git add -A && git commit` immediately — don't wait for other agents.
3. Push to the remote branch.

**Why:** agents write to disk asynchronously. A timed-out agent's completed work is already on disk; a checkpoint commit preserves it.

### Step 7 — Handle timeouts

When an agent hits a stream idle timeout:
1. Check `git status` — the agent likely wrote files before timing out.
2. Commit whatever is there.
3. Run a targeted grep to identify what the timed-out agent missed (do not re-run the full agent).
4. Fix the gaps manually or with a small targeted follow-up agent.

See `subagent-timeout-recovery-spec.md` for the detailed recovery pattern.

### Step 8 — Verification pass

After all agents complete:

```bash
# Check for residual old-terminology names
grep -rn "OldName1\|OldName2\|..." specs/ plans/ _meta/ \
  | grep -v "superseded\|historical\|'no OldName'\|_meta/reviews" \
  | head -30

# Check for residual old-document references
grep -rn "ADR XXXX" specs/ plans/ \
  | grep -v "superseded\|Superseded\|lineage" | wc -l
```

Target: zero residuals (excluding intentional lineage/historical references).

For any residual found:
1. Read the line in context — is it genuinely a v1 term, or is it an exception (JWT claim, plain English, lineage)?
2. If genuinely v1: fix manually.
3. If exception: leave it, but document why.

### Step 9 — Update meta documents

- Mark the task done in DECISIONS-LOG (move from STILL OPEN → COMPLETED).
- Add a SESSION N HANDOFF section to HANDOFF.md noting what's done and what remains.
- Commit and push.

### Step 10 — Open PR

Commit message and PR body should list:
- Zero-count verification result.
- Which agents completed cleanly vs timed out and were manually patched.
- Every ruling threaded (by decision ID).

## Concrete examples

### Example 1 — Crossplane v1→v2 sweep (this session)

**Input:** HANDOFF says ~270 "claim" references need re-expressing as v2 terminology across 250 files; rename map is `XPostgres`→`Postgres`, `XSearchIndex`→`SearchIndex`, etc.; 14 rulings to thread (D-01 through D-08, QN-03, quota schema, etc.). Canon glossary exists.

**Partition:**
- Agent 1: glossary + interface-contract + ADR-0044 spec/plan
- Agent 2: A1–A23 specs + plans
- Agent 3: B1–B22 specs + plans
- Agent 4: C/D/E/F + OPS trim
- Agent 5: V6-01–V6-13 specs + plans
- Agent 6: ADR-0001–0043 specs + plans (except 0041)

**Outcome:** 4 of 6 agents completed cleanly. 2 timed out (A-workstream, Views). Timed-out work was ~80% done (checkpoint commits captured it). Manual follow-up fixed 12 targeted residuals. Final grep: 0 X-prefixed names, 0 Crossplane-sense "claim" language.

### Example 2 — Future terminology sweep (hypothetical)

**Input:** A new API convention replaces `platform.io/environment` with `platform.io/substrate` across 80 files.

**Partition:** Smaller corpus — 3 agents: (A+B workstream), (C/D/E/F+views), (ADRs+Canon).

**Brief excerpt for each agent:**
```
Rename map:
- `platform.io/environment` → `platform.io/substrate`
- `platform.io/environment=kind|aws` → `platform.io/substrate=kind|aws`
Already correct (no change): environment variable names, plain English "environment"
DO NOT CHANGE: `platform.io/environment` in comments that are quoting the old spec
```

**Verification:**
```bash
grep -rn 'platform\.io/environment' specs/ plans/ | grep -v "old spec\|legacy" | wc -l
# Target: 0
```

## Anti-patterns

- **Briefing from memory.** A brief that doesn't embed the full rename map and ruling list produces an agent that misses half the work. The 14 ruling list from this session was ~800 words in each brief; that length was necessary.
- **Waiting for all agents before committing.** Agent 5 (Views) timed out after 45 minutes. If the checkpoint commit strategy had been "commit at the end," all of Agent 4's OPS trim work would have been lost.
- **Overlapping file partitions.** If two agents write to the same file, the second write silently overwrites the first. Check the partition before launching.
- **Not specifying exceptions.** "JWT/OIDC claims" and "plain English claim" were explicit DO NOT CHANGE entries in every brief. Without them, agents would have renamed `platform claim schema` and `clusterOIDCClaimMapping` — incorrect.
- **Re-running a timed-out agent from scratch.** The timed-out Views agent (Agent 5) had already written ~11 of 13 view files. Re-running it would rewrite the completed 11 files (wasting time and potentially introducing drift) and might time out again. Instead: grep for what was missed, fix the 2 gaps manually.

## Acceptance criteria

1. After all agents complete, `grep -rn "OldTerm" specs/ plans/` returns 0 non-exception results.
2. Every ruling from the decision list appears as a requirement or note in at least one spec/plan for its owning component.
3. No two agents wrote to the same file (check via `git log --name-only`).
4. At least one checkpoint commit exists per completed agent (no work lost to timeout).
5. DECISIONS-LOG updated with COMPLETED entry; HANDOFF updated with SESSION N section.

## Files this skill creates / modifies

| Path | Description |
|---|---|
| `specs/**/*.md` | In-place edits: terminology updated, rulings threaded |
| `plans/**/*.md` | In-place edits: same |
| `_meta/glossary.md` | Canon file: XRD renames, new entries |
| `_meta/interface-contract.md` | Canon file: table updates, reference updates |
| `_meta/reviews/DECISIONS-LOG.md` | Mark tasks done |
| `_meta/HANDOFF.md` | Add SESSION N HANDOFF section |
| `adr/NNNN-*.md` | New source ADR if supersession is involved |
| `specs/adr/superseded/` | Superseded spec stubs if ADR is retired |
| `plans/adr/superseded/` | Superseded plan stubs if ADR is retired |
