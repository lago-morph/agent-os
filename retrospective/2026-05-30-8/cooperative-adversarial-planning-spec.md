# Spec: `cooperative-adversarial-planning`

## Intent

For a high-stakes plan that will drive a large, expensive execution, a single planner —
however good — has blind spots and an optimism bias. This skill produces a plan by
running **cooperative designers and a hard adversarial red-teamer in parallel**, then
synthesizing: the cooperative agents supply the constructive design, the red-teamer
attacks scope, cost, and failure modes, and the orchestrator reconciles — explicitly
deciding where user intent overrides the red-teamer and where the red-teamer's quality
controls get folded in regardless. In the session that produced this skill, the
red-teamer's attack ("119 independent agents will drift and hallucinate; freeze a
Canon") became the single most valuable design input, even though its headline
recommendation (cut scope to ~40) was overridden by explicit user instruction.

## Trigger

- Direct: "plan this with cooperative and adversarial agents", "red-team the plan",
  "stress-test this approach before we commit".
- Proactive: before any execution that will dispatch many agents / consume a large
  budget / be hard to reverse; whenever the user signals "spare no expense" or
  "scrutinize everything."
- Negative: skip for small, cheap, easily-reversible tasks where planning overhead
  exceeds the work.

## Inputs

- The goal and constraints (including explicit user decisions/instructions).
- The relevant source material the planners must ground in.
- The budget/time envelope.

## Outputs

- 2–3 parallel planning documents (cooperative designs + one adversarial critique).
- A synthesized meta-plan presented to the user that explicitly states where it follows
  user intent over the red-teamer and where it adopts the red-teamer's controls.

## Workflow

1. Decompose the planning problem into complementary cooperative lenses (e.g.
   orchestration design, mechanics/logistics) plus one dedicated adversarial lens.
2. Dispatch all planners **in parallel**, each with a distinct persona and a focused
   brief. The adversarial brief must demand: attack budget realism, scope inflation,
   drift/hallucination risk, failure modes, and end with a leaner counter-proposal.
3. Collect all outputs. Do not let any single one dictate.
4. **Synthesize.** Where the red-teamer contradicts an explicit user instruction, keep
   the user's intent but **fold in the red-teamer's quality controls** (the controls are
   usually separable from the scope recommendation). Where the red-teamer found a genuine
   design flaw, fix it.
5. Present the synthesized plan to the user for the go/no-go gate, surfacing the key
   tension the red-teamer raised so the user can overrule with full information.

## Concrete examples

**Example 1 (this session).** Three meta-planners ran concurrently: a delivery-architect
(orchestration), a GitOps engineer (stacked-PR mechanics), and a red-teamer. The
red-teamer argued the 119-piece count was inflated and that parallel agents would
hallucinate CRD fields. Synthesis: kept all 119 pieces (explicit user "yes on #1") but
adopted the red-teamer's **frozen-Canon** and **`[PROPOSED]`-tagging** controls and its
**tiered effort** model (hard adversarial review only on high-blast-radius pieces). The
result honored user intent while neutralizing the red-teamer's correctly-identified risk.

**Example 2 (generalized).** Planning a data migration: one cooperative agent designs
the cutover, the red-teamer attacks rollback gaps and dual-write races and proposes a
shadow-read validation phase. Synthesis keeps the cutover but inserts the shadow-read
gate the red-teamer demanded.

## Anti-patterns

- **Treating the red-teamer's headline recommendation as binding.** Its *findings* are
  gold; its *conclusions* may conflict with user intent. Separate the two.
- **Running adversarial review serially after the cooperative plan is "done."** Parallel
  dispatch is cheaper and avoids anchoring the red-teamer on the first design.
- **Hiding the tension from the user.** The whole point of the gate is informed
  override; surface the disagreement, don't paper over it.
- **A single all-purpose planner persona.** Distinct lenses find distinct problems.

## Acceptance criteria

1. At least one dedicated adversarial planner ran in parallel with the cooperative ones.
2. The synthesized plan explicitly states where it follows user intent over the
   red-teamer and where it adopts red-team controls.
3. Genuine design flaws found by the red-teamer are fixed, not deferred.
4. The user gets a go/no-go gate with the key tension surfaced.

## Files this skill creates / modifies

- Typically none persisted beyond the chat-level meta-plan, unless the user asks for the
  plan to be written to disk. The artifacts are the parallel planning outputs and the
  synthesized plan presented at the gate.
