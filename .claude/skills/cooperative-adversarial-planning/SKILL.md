---
name: cooperative-adversarial-planning
description: Produce a high-stakes plan by running cooperative designer agents AND a dedicated adversarial red-team agent in parallel, then synthesizing — explicitly separating the red-teamer's findings (adopt its quality controls) from its conclusions (which explicit user intent may override). Use before any execution that will dispatch many agents, consume a large budget, or be hard to reverse; or whenever the user signals "scrutinize everything" / "spare no expense" / "red-team this". Trigger phrasings include "plan this with cooperative and adversarial agents", "red-team the plan", "stress-test this approach before we commit", "poke holes in this". Do NOT use for small, cheap, easily-reversible tasks where planning overhead exceeds the work.
---

# Skill: cooperative-adversarial-planning

For a high-stakes plan that will drive a large, expensive, or hard-to-reverse execution, a
single planner — however good — shares the optimism bias of whoever will execute. This
skill runs **cooperative designers and a hard adversarial red-teamer in parallel**, then
synthesizes: the cooperative agents supply the constructive design; the red-teamer attacks
scope, cost, and failure modes; the orchestrator reconciles — explicitly deciding where
user intent overrides the red-teamer and where the red-teamer's quality controls get
folded in regardless.

## When to use

- Before any execution that dispatches many agents / consumes a large budget / is hard to
  reverse.
- Whenever the user signals "scrutinize everything", "spare no expense", "red-team this".
- NOT for small, cheap, easily-reversible tasks.

## Workflow

1. **Decompose** the planning problem into complementary cooperative lenses (e.g.
   orchestration design, mechanics/logistics) plus one dedicated adversarial lens.
2. **Dispatch all planners in parallel**, each with a distinct persona and focused brief.
   The adversarial brief must demand: attack budget realism, scope inflation,
   drift/hallucination risk, orchestration failure modes — and end with a leaner
   counter-proposal.
3. **Collect all outputs.** Do not let any single one dictate.
4. **Synthesize.** Where the red-teamer contradicts an explicit user instruction, keep the
   user's intent but **fold in the red-teamer's quality controls** — they are usually
   separable from the scope recommendation. Where the red-teamer found a genuine design
   flaw, fix it.
5. **Present the synthesized plan at the go/no-go gate**, surfacing the key tension the
   red-teamer raised so the user can overrule with full information.

## The core discipline

Separate the red-teamer's **findings** (gold — adopt the controls) from its
**conclusions** (which explicit user intent may override). A red-teamer may say "cut
scope" when the user said "do all of it"; keep the full scope but adopt the red-teamer's
drift/quality controls. The findings are almost always worth more than the headline.

## Anti-patterns

- Treating the red-teamer's headline recommendation as binding.
- Running adversarial review serially after the cooperative plan is "done" (parallel is
  cheaper and avoids anchoring the red-teamer on the first design).
- Hiding the tension from the user at the gate — the point is *informed* override.
- A single all-purpose planner persona (distinct lenses find distinct problems).

## Acceptance criteria

1. At least one dedicated adversarial planner ran in parallel with the cooperative ones.
2. The synthesized plan states where it follows user intent over the red-teamer and where
   it adopts red-team controls.
3. Genuine design flaws found by the red-teamer are fixed, not deferred.
4. The user gets a go/no-go gate with the key tension surfaced.
