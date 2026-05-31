# PLAN C7 — Maintainer / extender documentation for all custom code  [PROPOSED]

> spec: SPEC-C7 · kind: COMPONENT · tier: T1
> wave: authoring-parallel (consumer) · estimate: M
> upstream-pieces: [C1] · downstream-pieces: [C8]

## 1. Implementation Strategy
Stand up the §10.6 maintainer-documentation section and a per-component template enforcing the
four artifacts (usage docs; maintainer/extender docs; API reference; test + contribution guide),
then drive coverage across every custom component named in §10.6. Because docs are constructed
before implementation as the source of test cases (§10), C7 ships the template/section early so
each custom component (B-series + A22) authors its maintainer content against its own spec, with
partial early authoring encouraged. Co-develop the page template / front-matter with C8 for
preprocessing-free re-indexing, enforce Canon-name usage via a lint, and validate builds through
C1's GitHub Actions docs check. Component-specific maintainer content lands with each component;
C7 owns the standard, the coverage checklist, and curation/consistency.

## 2. Ordered Task List
- **TASK-01:** Define the per-component maintainer-doc template (four §10.6 artifacts) + front-matter, agreed with C8 — produces: template — depends-on: [].
- **TASK-02:** Build the custom-component coverage checklist from §10.6 / §14.2 (kopf operator, Headlamp plugins, platform SDK, agent base images, callbacks, Coach, HolmesGPT toolsets, glue services, Crossplane Compositions) — produces: coverage matrix — depends-on: [TASK-01].
- **TASK-03:** Author/curate API-reference conventions per ADR 0030 (CRD/XRD versions, SDK semver + compat matrix, URL-path HTTP versioning) — produces: API-ref convention — depends-on: [TASK-01].
- **TASK-04:** Author anchor maintainer docs for the highest-leverage custom components — kopf operator (ADR 0006 subchart), platform SDK (`memory.*`/`rag.*`), agent base images (`langgraph`/`deep-agents`, ADR 0019), Crossplane Compositions (ADR 0044 connection-secret) — produces: maintainer doc pages — depends-on: [TASK-02, TASK-03].
- **TASK-05:** Author/curate remaining maintainer docs (Headlamp plugins, callbacks, Coach, HolmesGPT toolsets, glue services) — produces: maintainer doc pages — depends-on: [TASK-02, TASK-03].
- **TASK-06:** Add Canon-name lint + boundary cross-links to C4 (reference) — produces: lint config + cross-links — depends-on: [TASK-04, TASK-05].
- **TASK-07:** Validate builds/links via C1 docs check + joint C8 re-index + KB-retrieval smoke — produces: passing checks + retrieval verification — depends-on: [TASK-06].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- **C1** — portal + reserved §10.6 section + Markdown/MkDocs conventions and docs CI check.
### 3.2 Downstream pieces blocked on this
- **C8** — re-indexes maintainer docs and validates KB retrieval jointly.
### 3.3 Continuous (non-blocking) inputs
- Every custom component (B2, B4, B5, B6, B7, B10, B13, A22, glue services, custom HolmesGPT toolsets) feeds its own maintainer content per docs-before-implementation (§10); C4 for the reference-boundary cross-link; B14 test framework for link/build checks.

## 4. Parallelizable Subtasks
- After TASK-01: TASK-02 and TASK-03 run concurrently.
- After TASK-02+TASK-03: TASK-04 and TASK-05 fan out per component (independent per page).
- TASK-06 joins; TASK-07 is the final serial validation gate.

## 5. Test Strategy
- **Chainsaw:** N/A — C7 owns no CRD.
- **Playwright:** AC-C7-05 (all maintainer docs render in the §10.6 section), portal nav/search.
- **PyTest / link-lint (via C1 check):** AC-C7-01 (template), AC-C7-02 (coverage matrix complete), AC-C7-03 (required sections present, sampled), AC-C7-04 (versioning stated), AC-C7-06 (Canon-name lint passes), AC-C7-08/-09/-10 (kopf subchart, SDK values, connection-secret documented).
- **Joint with C8:** AC-C7-07 (re-index without preprocessing) and the KB-retrieval smoke.
- **Fixtures/fakes:** stub `platform-knowledge-base` (C8 fake) for retrieval smoke before C8 lands; component-spec stubs as sources for anchor maintainer docs authored pre-implementation.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/<C1-wave>` (contains spec-C1, the portal skeleton).
### 6.2 PR — `piece/C7-maintainer-docs` → base of the C-content wave; carries spec-C7.md + plan-C7.md.
### 6.3 Merge order — independent of C-content siblings (C2–C6, C9); C8 merges after to exercise re-index; wave rolls up to main.

## 7. Effort Estimate
- TASK-01 S · TASK-02 S · TASK-03 M · TASK-04 M · TASK-05 M · TASK-06 S · TASK-07 S.
- Rollup: **M**. Critical path: TASK-01 → TASK-03 → TASK-04 → TASK-06 → TASK-07.

## 8. Rollback / Reversibility
Back out by reverting the C7 PR; the §10.6 maintainer-docs section returns to the empty C1 slot
and the template/coverage matrix are removed. Downstream impact: C8 indexes no maintainer content
(KB cannot answer "how does our custom code work / how do I extend it"); external extenders lose
the standard. No runtime state; reversible at any time. Component-authored maintainer pages, if
already merged with their components, remain but lose the shared template/section structure.
