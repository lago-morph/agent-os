# Review R5 — Red-team (2nd pass): T0 contract VIEWS and ADRs

**Date:** 2026-05-31
**Reviewer role:** Independent red-team (second pass) on contract soundness, AC testability, load-bearing `[PROPOSED]` decisions, and ADR↔spec contradictions.
**Canon read:** `_meta/interface-contract.md`, `_meta/glossary.md`, `_meta/waves.md`, the templates, plus canonical ADR text under `adr/` for cross-checks (0017, 0034).
**Constraint honored:** no spec/plan/source file edited; this review file is the only write.

---

## Scope selection

Filtering `_meta/piece-index.csv` for `kind ∈ {VIEW, ADR}` AND `tier = T0`:

- **T0 views (6):** V6-01, V6-02, V6-06, V6-11, V6-12, V6-13.
- **T0 ADRs (10):** ADR-0006, ADR-0013, ADR-0017, ADR-0019, ADR-0020, ADR-0030, ADR-0031, ADR-0034, ADR-0040, ADR-0041.

That is 16, over the ~12 budget. Applying the task's prioritization rule (hot spots + realization-map views), the **14 reviewed pieces** are:

- **All 6 T0 views** (the realization-map/integration-contract surface): V6-01, V6-02, V6-06, V6-11, V6-12, V6-13.
- **Named hot-spot ADRs:** ADR-0017 (Approval), ADR-0020 (MCP secrets), and **ADR-0016** (tenant-quota multi-tenancy) — included despite ADR-0016 being tier **T1** in the CSV/front-matter because the task explicitly names it a hot spot. (Tier mismatch noted under ADR-0016.)
- **Structural T0 "contract" ADRs the views bind to:** ADR-0013 (capability CRDs), ADR-0030 (versioning), ADR-0031 (CloudEvent taxonomy), ADR-0034 (audit pipeline), ADR-0041 (substrate abstraction), ADR-0040 (Kargo — read to adjudicate the ADR-0017 contradiction; verdict included).

**Deprioritized (read-only context, not separately verdicted):** ADR-0006 and ADR-0019 — small, settled, narrow-blast packaging/SDK-naming decisions already cross-referenced and reconciled by V6-01/V6-02; no hot-spot flag. Listed here for completeness; no findings surfaced against them during cross-checks.

Cross-cutting note on Canon hashes: the front-matter `canon-glossary`/`canon-interface` fields are inconsistent across these 14 pieces (`FROZEN`, `see _meta/...`, and 8/12-hex truncations all appear). Already logged by the prior consistency review (`consistency-existence.md` Task 4); not re-litigated here beyond noting it weakens any "spec pinned to a known Canon revision" guarantee — see X-HIGH-1.

---

## Cross-cutting findings (affect multiple pieces)

### X-HIGH-1 (HIGH) — Audit-endpoint egress vs the `EgressTarget` allowlist is an unreconciled contract gap
**Where:** ADR-0034 §4.x / REQ-ADR-0034-11; V6-01 REQ-V6-01-01; V6-02 REQ-V6-02-03; V6-06 REQ-V6-06-03; ADR-0013 REQ-ADR-0013-04 (Envoy allowlist sourced from `EgressTarget`).
**Problem:** ADR-0034 REQ-11 (and the canonical ADR `adr/0034-…` Consequences) require the audit endpoint's egress to S3 and AWS-managed OpenSearch to traverse the Envoy egress proxy "the endpoint is not a special case." But every egress contract in the views states the Envoy path is reachable **only for `EgressTarget`-declared FQDNs** (V6-02 REQ-03, V6-06 REQ-03), and ADR-0013 REQ-04 makes `EgressTarget` CRDs the single source of truth for Envoy allowlists. No reviewed piece declares an `EgressTarget` for S3 / managed-OpenSearch, and it is unstated whether the FQDN-allowlist model applies to **platform infrastructure components** (audit endpoint, Crossplane-provisioned backends) or **only to Platform Agents**. If the allowlist applies to infra, audit egress is silently blocked until someone authors those `EgressTarget`s; if it does not, the "everything via Envoy is allowlisted" invariant (a stated security property) has an unenumerated carve-out. Either way the contract is currently underspecified at exactly a security boundary.
**Fix:** Add one sentence to V6-06 (and/or ADR-0034 §4.4) stating whether platform-infra egress is governed by `EgressTarget` allowlisting or by a distinct infra-egress policy, and if the former, require the `AuditLog` XRD / A18 to emit the S3 + managed-OpenSearch `EgressTarget`s. Surface as an open question for the V6-06 / ADR-0034 owners; do not auto-resolve.

### X-HIGH-2 (HIGH) — "platform release" deprecation unit is undefined yet load-bearing for every versioning AC
**Where:** ADR-0030 REQ-03 / OQ-1 / AC-ADR-0030-03; V6-13 REQ-V6-13-03 / OQ (med); V6-12 (inherits); ADR-0041 REQ-07; ADR-0013 REQ-11; ADR-0017/0020/0034 versioning clauses.
**Problem:** The deprecation window for every CRD/XRD/HTTP surface is "≥1 minor **platform release**," but ADR-0030 OQ-1 concedes the calendar definition of "platform release" is deferred (backlog §1.18) **while there is explicitly no synchronized platform version** (REQ-02). V6-13's own self-red-team (§10 med) flags this as a tension and adopts a "cadence-marker" reading, but that reading is not ratified in ADR-0030 or Canon. Consequence: AC-ADR-0030-03, AC-V6-13-03, AC-ADR-0041-07 all assert "deprecated for ≥1 minor platform release" — i.e. they are **measured in an undefined unit**, so they are not currently binary-testable. This is a single root cause propagating into ~5 ACs across the contract layer.
**Fix:** ADR-0030 owner ratifies the cadence-marker definition (a coordinated cut of the component set, not an API version) in Canon, or defines the unit. Until then, mark the affected ACs as blocked-on-OQ-1 rather than passable. Open question — route to ADR-0030 owners.

### X-MED-3 (MED) — Platform SDK compatibility matrix pins to "Letta," coupling a T0 contract to a T1 backend choice
**Where:** ADR-0030 REQ-07 / AC-06; V6-13 REQ-V6-13-05 / AC-05; interface-contract §3.1.
**Problem:** Both the T0 versioning ADR and the T0 versioning view require the Platform SDK to "ship a version-pinned compatibility matrix against **gateway / ARK / Letta** versions." Letta (A10) is tier **T1** and the memory backend is itself abstracted behind `MemoryStore` (ADR 0025/0005). Naming Letta by product in a frozen T0 contract bakes a specific T1 backend into the SDK's compatibility surface; if the memory backend is ever swapped (which the `MemoryStore` abstraction exists to allow), the T0 contract text is stale. The interface-contract §3.1 reproduces the same wording, so this is Canon-sourced, not a spec drift — but it is a latent coupling worth flagging.
**Fix:** Consider rewording to "…against gateway / ARK / the memory backend (Letta in v1.0)" in ADR-0030 / V6-13, or accept the coupling explicitly. Low blast radius today (Letta is the only v1.0 backend); flagged so it is a conscious choice. Open question for ADR-0030 owner.

---

## Per-piece verdicts and findings

### V6-01 — Gateway architecture — **SOUND**
Integration-contract reconciles cleanly with interface-contract §1.4 (B13 CRDs reproduced verbatim), §2 (5 emitted CloudEvent namespaces all in the closed set), §3.3 (URL-path admin API), §5 (audit via adapter). REQ↔AC coverage is 1:1 and each AC is binary. Realization map (A1/B2/B13/B1) matches the piece-index downstream edges.
- **V6-01-LOW-1 (LOW):** REQ-V6-01-07 enumerates failover semantics with an architecture-line cite but no Canon anchor (failover detail is not in interface-contract). It is self-consistent and testable (AC-07 reproduces each row), so this is a provenance note only — confirm the §6.1 line range is authoritative for the zero-provider-rejected-at-admission claim.
- **V6-01-LOW-2 (LOW):** REQ-V6-01-11 asserts "eight initial MCP service classes" while ADR-0020's set, counted as listed, is {GitHub, Google Drive, Context7, OpenSearch, Postgres, MongoDB, web-search, web-scrape} = 8 if web-search and web-scrape are counted separately (ADR-0020 AC-01 also says "all eight services"). Consistent — noted only because the prose elsewhere says "web search + web scrape" as one clause; keep the count explicit so the conformance test asserts 8 distinct `MCPServer` CRDs.

### V6-02 — Agent runtime architecture — **SOUND**
Reconciles with interface-contract §1.2 (`Agent`/`AgentRun`/… fields verbatim), §1.3 (`Sandbox`/`SandboxTemplate`), §1.4 (`EgressTarget`/`Skill` consumed), §3.2 (agent SDK), and the closed event taxonomy. REQ↔AC is 1:1 and binary. The two-path egress invariant is stated crisply.
- **V6-02-MED-1 (MED):** AC-V6-02-07 ("a non-supported harness image complying with the interface is still fully governed") is **weakly testable as written** — "fully governed" is asserted but the test would need an actual adversarial harness that *attempts* bypass, not merely a compliant one. REQ-07's strength is the bypass-resistance claim, but the AC only checks a cooperative image. Strengthen AC-07 to require a harness that actively tries an out-of-band LLM/egress call and is still contained, otherwise the AC under-verifies its own REQ.
- Cross-ref: V6-02 participates in **X-HIGH-1** (egress allowlist scope).

### V6-06 — Security and policy architecture — **NEEDS-WORK**
The defense-in-depth and RBAC-floor/OPA-restrictor invariants are well-formed and the enumerated-point conformance (REQ-05/06) is the right shape. But two enforcement points are named only by reference to architecture-line ranges, not to an enumerable Canon list, which undercuts the central "no enumerated point may be silently omitted" guarantee.
- **V6-06-HIGH-1 (HIGH):** REQ-V6-06-05 and REQ-V6-06-06 require that "every enumerated audit-emission point" and "every enumerated OPA decision point" be covered, and AC-05/AC-06 say "a scan finds no enumerated point missing." But **the enumeration itself lives only in `architecture-overview.md` §6.6 (lines ~515–542), not in Canon or in this spec**. There is no machine-checkable list in the reviewed corpus, so the conformance AC cannot actually be run — it depends on a source enumeration the spec does not reproduce and the interface-contract does not hold. This is the load-bearing AC of the whole security view and it is currently untestable without dereferencing a non-frozen source. **Fix:** reproduce the audit-point and OPA-decision-point enumerations inline in V6-06 §4 (or have B22/B16 own a frozen list the test asserts against), so AC-05/06 have a concrete target. The view itself flags R-1 (high) on exactly this risk but does not close it.
- **V6-06-MED-2 (MED):** AC-V6-06-03 says a capability outside the set "would be contained at the kernel — each layer independently denies." "Would be contained" is hypothetical, not a binary observation; the kernel layer (gVisor/Kata) does not "deny a capability" the way admission/gateway/egress do — it isolates. The AC conflates four heterogeneous layers under one "each layer independently denies" assertion that is not literally true for the kernel layer. Reword AC-03 to test the three policy layers as denials and the kernel layer as containment/isolation separately.
- **V6-06-MED-3 (MED):** REQ-V6-06-09 lists the simulator's per-layer dry-run set but ADR-0038 (the simulator's owning ADR) was not in T0 scope; the view consumes it as settled. No contradiction found, but the "RBAC, Gatekeeper audit-mode, LiteLLM OPA callback, Envoy egress OPA, approval elevation, CLI promotion gates" list is a six-item contract the simulator must implement — confirm A20/ADR-0038 enumerate the same six, else REQ-09 over-specifies relative to the component.
- Cross-ref: **X-HIGH-1** (egress allowlist scope) lands most directly here.

### V6-11 — Identity federation — **SOUND**
This is the strongest contract piece. The §4.4 claim table reproduces the Platform JWT schema (glossary + ADR-0029) verbatim and exhaustively; the three-flow topology, mapper boundary, and per-environment OIDC availability are crisp. REQ↔AC mostly 1:1.
- **V6-11-MED-1 (MED):** **AC coverage gap.** REQ-V6-11-08 (the three federation roles — cluster OIDC root / IRSA cloud adapter / Keycloak platform adapter — MUST remain distinct) and REQ-V6-11-10 (trust bootstrap via ESO over IRSA-bound SA) have **no dedicated AC**. AC-V6-11-01..07 cover REQ-01..07 and REQ-09, but REQ-08 and REQ-10 are unverified. REQ-08 is an architectural-separation invariant (hard to test but at least assert "no component collapses two roles into one trust token") and REQ-10 is concretely testable (bootstrap secret fetch path). Add AC-V6-11-08 (→REQ-10, bootstrap fetch) and an AC for REQ-08.
- **V6-11-LOW-2 (LOW):** REQ-V6-11-09 commits the platform to enabling AKS OIDC flags and shipping a kind bootstrap utility, but ADR-0033 scopes v1.0 to AWS+GitHub+kind (Azure not exercised). The view correctly flags Azure as "architecture-supported, not exercised" (R, med) — but REQ-09 phrases the AKS commitment as a MUST. Soften REQ-09's AKS clause to match ADR-0033's "pattern fixed, wiring deferred," or the REQ asserts a v1.0 obligation the program-scope ADR excludes.

### V6-12 — CRD inventory — **SOUND** (with one open Canon question carried forward)
The inventory matches interface-contract §1.2–1.6 one-to-one; the §10 divergence log is exactly the right artifact and is honest about the naming nuances. The cross-check methodology is sound.
- **V6-12-MED-1 (MED, open question — confirms prior review):** The three derived claim spellings `SearchIndex` / `ObjectStore` / `MongoDocStore` are correctly tagged `[PROPOSED]` here (§4.1 + §10), but they are **referenced as if canonical elsewhere in the corpus** — the prior `consistency-existence.md` review found `MongoDocStore` used in V6-12 itself and `SearchIndex`/`ObjectStore` in B4. Blast radius: MED — these are deterministic prefix-drops, not invented names, so the risk is a Canon-completeness gap, not a contradiction. **Fix (Canon, not spec):** add the three claim forms to `glossary.md` alongside `AuditLog`/`GrafanaDashboard`, then drop the `[PROPOSED]` tags. Open — route to ADR-0041 / glossary owners.
- **V6-12-LOW-2 (LOW):** `LogLevel`'s "no single owner" for the ADR-0030 versioning column is correctly flagged (§10 med, open) and forwarded to V6-13. Consistent handling; no action beyond the routed question.

### V6-13 — Versioning policy — **SOUND** (gated on X-HIGH-2)
Six surface classes each map to interface-contract §1.1/§2/§3, REQ↔AC is 1:1 and binary, and the self-red-team on the "platform release" tension is candid.
- Carries **X-HIGH-2** (platform-release unit undefined → AC-V6-13-03 not testable) and **X-MED-3** (Letta coupling). No piece-local additional findings; the view's own §10 already surfaces both tensions and routes them — good practice, but they remain open and gate the affected ACs.

### ADR-0013 — Capability CRD model — **SOUND**
Enforcement points are concrete: Gatekeeper admission (REQ-05), OPA gating against `capability_set_refs` (REQ-05), RBAC bind-scoping, LiteLLM/Envoy/OPA as the single source of truth (REQ-04). Not hand-wavy. The notification-is-not-enforcement separation (REQ-08/09) is exactly right and AC-09 tests it well (suppress notification, confirm enforcement still denies).
- **ADR-0013-LOW-1 (LOW):** REQ-ADR-0013-08 mandates a `refresh_capabilities()` SDK poll fallback but §4.2 correctly tags the signature `[PROPOSED — not in source]`. AC-08 tests "within one turn via `refresh_capabilities()`" — testable in spirit, but the one-turn staleness bound is an assertion not traceable to Canon. Low blast (enforcement is independent per REQ-09). Confirm the one-turn bound with B6.
- **ADR-0013-LOW-2 (LOW):** OQ-1 (high, overlay semantics deferred to ADR-0032) correctly marks AC-02 (layering) as partial. This is honestly disclosed; the high rating is the ADR's own, and it is the right call — layering ACs cannot fully pass until ADR-0032 lands. Carry as a known partial, not a defect.

### ADR-0016 — Multi-tenancy via namespaces — **SOUND** (tier-label caveat)
Four-layer enforcement, three visibility modes, and claim-driven scoping all map to concrete points (Gatekeeper, OPA-at-LiteLLM, Envoy network policy, RBAC). REQ↔AC 1:1 and binary. Honors ADR-0018/0013/0028/0029/0037/0026 without contradiction.
- **ADR-0016-MED-1 (MED):** REQ-ADR-0016-03 / AC-03 assert the four layers are **independent** ("a misconfiguration in one MUST still be bounded by the others"), and R-1 concedes "a shared misconfiguration source (e.g. a bad OPA bundle deployed everywhere) reduces independence." The REQ states independence as a MUST while the risk note admits two of the four layers (Gatekeeper admission and OPA-at-LiteLLM) **share the OPA engine and bundle** — so they are not failure-independent. AC-03 only tests the RBAC-vs-rest case (over-broad RBAC still bounded), not the shared-OPA-bundle case. The independence claim is partially false as written. **Fix:** weaken REQ-03 to "at least two structurally-independent layers (RBAC floor + Envoy network policy) bound any single-layer OPA/Gatekeeper misconfiguration," and add an AC exercising a bad-bundle scenario to confirm network policy still holds.
- **ADR-0016-LOW-2 (LOW):** Tier mismatch — CSV and front-matter both label ADR-0016 **T1**, but it is one of the three named hot spots and is consumed by T0 views (V6-06 out-of-scopes tenancy to "V6-09 / ADR-0016"). Not a defect in the ADR, but the T1 label undersells its contract weight; flag for tier review.
- **ADR-0016-LOW-3 (LOW):** Tenant-scoped quotas are `[PROPOSED]`/deferred (OQ-1, backlog §1.2). Given the task framing of ADR-0016 as the "tenant-quota" hot spot, note that **quotas are explicitly NOT decided here** — anyone treating ADR-0016 as the quota contract will find only a deferral. Blast radius MED for downstream pieces expecting a quota field set (none exists; `[PROPOSED]` is correctly applied).

### ADR-0017 — Generalized approval system — **NEEDS-WORK**
Four-part contract (Approval CRD + Argo + OPA-elevate-only + Headlamp) maps to concrete enforcement (REQ-04 OPA elevate-only is the security spine; AC-04 tests that a lower-it/grant-beyond-RBAC policy is rejected). Mostly sound, but one REQ contradicts the canonical ADR's own consumer enumeration.
- **ADR-0017-MED-1 (MED) — contradiction with canonical ADR + ADR-0040:** REQ-ADR-0017-07 states "Coach skill changes and HolmesGPT remediation **MUST be the two initial use cases**." But the canonical `adr/0017-…` (Decision + Consequences) names **three** internal consumers (Coach, HolmesGPT remediation, **HolmesGPT upon-approval state** — "a third concrete consumer") **plus Kargo as "the first external-system consumer"** (also v1.0 per ADR-0040 REQ-05/08, which MUST use the `Approval` CRD as its human gate). So "the two initial use cases" under-counts the v1.0 consumer set the same ADR mandates two REQs later (REQ-07 itself covers upon-approval; REQ-08 mandates Kargo). The spec is internally inconsistent (REQ-07 says "two" while REQ-07/08 enumerate four). **Fix:** reword REQ-07 to "Coach skill changes, HolmesGPT remediation, and HolmesGPT upon-approval are the initial platform-internal consumers; Kargo (REQ-08) is the first external consumer," matching canon. Low real risk but a true decision-vs-spec contradiction in a T0 ADR.
- **ADR-0017-MED-2 (MED) — testability degraded by deferred schema:** OQ-1 (med) admits the exact `Approval` schema (first-class fields vs metadata blob), timeout/escalation behavior, and shared-vs-per-type Argo template are all design-time, and concedes "several ACs are partial until B19 design lands." AC-02 ("action attributes … sufficient for an approver to decide independently") is **inherently non-binary** — "sufficient to decide independently" is a judgment, not a pass/fail. Tighten AC-02 to a checkable proxy (e.g. `actionType` + `actionAttributes` + `evidenceRefs[]` all populated and non-empty) and acknowledge the "sufficiency" property is reviewed, not tested.
- **ADR-0017-LOW-3 (LOW):** REQ-09 (no M-of-N/delegation/escalation/routing in v1.0) is a negative requirement; AC-09 ("no such code ships") is testable by absence but only via review/grep. Acceptable for a scope-fence REQ; noted.

### ADR-0020 — Initial MCP services set — **NEEDS-WORK**
Strong enforcement mapping: each service is an `MCPServer` CRD (REQ-01), three credential modes are first-class (REQ-02/03), `XAgentDatabase` fronts DB provisioning (REQ-04), no-bypass invariant (REQ-09). Reconciles with interface-contract §1.4 and §4 (connection secret). But the load-bearing decisions are heavily deferred.
- **ADR-0020-HIGH-1 (HIGH) — load-bearing `[PROPOSED]`, wide blast radius:** OQ-1 (high) defers **per-service auth flows, scopes, secret shapes, revocation, budget enforcement, per-XRD field schemas, AND the search/scrape OPA bundle contents** all to design-time (backlog §1.11), and states "many ACs are partial until A17/B4 design lands." This is the single most load-bearing deferral in the reviewed set: the credential-mode contract (REQ-02/03) and the OPA-policed web egress (REQ-06) are *named as first-class and load-bearing* in §7, yet their actual content is unspecified. AC-02/03/06 therefore test mode *existence* but not mode *correctness* (e.g. AC-06 tests "a disallowed scrape target is blocked" but the disallow-list / OPA bundle that defines "disallowed" is `[PROPOSED]`). Blast radius HIGH: this ADR is the day-one exercised integration surface (§1 rationale) and three early agents (A16/A14/B10) compose against it. **Fix:** acceptable to defer the *contents*, but ADR-0020 should explicitly mark which ACs are existence-only vs correctness, and require A17/B4 design to land the search/scrape OPA bundle before the web-scrape service is enabled (it is the simulator's "primary target" per §7 precisely because false-negatives leak egress). Surface as the top open question for A17.
- **ADR-0020-MED-2 (MED):** REQ-ADR-0020-03 introduces credential mode **`system-mediated`** (and §4.1 lists `authMode` as system/user-cred/**system-mediated**), but interface-contract §1.4 states `MCPServer.authMode` accepts only **`(system/user-cred)`**. This is a **third enum value not in Canon** and it is **not tagged `[PROPOSED — not in source]`** in §4.1 (it is presented as canonical). Either Canon's `authMode` enumeration is incomplete or the ADR is coining a value silently — exactly the failure mode the `[PROPOSED]` discipline exists to catch. **Fix:** tag `system-mediated` as `[PROPOSED — not in source]` in ADR-0020 §4.1 and route an interface-contract §1.4 update to add the third `authMode` value, or reframe system-mediated as a sub-case of `system`. (Note: V6-01 §4.1 and ADR-0013 §4.1 both reproduce `authMode (system/user-cred)` only — so ADR-0020 diverges from two other T0 pieces on the same field.)
- **ADR-0020-LOW-3 (LOW):** REQ-01's "exactly eight" set is binary and good (AC-01 tests presence + Firecrawl absence). Consistent with V6-01 REQ-11. No issue; noted as the positive counterpart to V6-01-LOW-2.

### ADR-0030 — CRD/API versioning policy — **SOUND** (gated on X-HIGH-2)
Enforcement points are concrete and CI-shaped: new `vN` group + conversion webhook (REQ-02, AC-02), both substrate Compositions on XRD change (REQ-05, AC-04), new-event-type-on-breaking (REQ-06, AC-05), JWT-schema/mapper lockstep (REQ-11, AC-10). REQ↔AC 1:1. The mapper-lockstep blast-radius reasoning (NFR §7, R-2) is a genuinely good catch.
- Carries **X-HIGH-2** (the "platform release" unit, which the ADR itself owns and defers in OQ-1) and **X-MED-3** (Letta). The ADR is otherwise the cleanest of the structural contracts.
- **ADR-0030-LOW-1 (LOW):** REQ-11 / AC-10 require the cluster-OIDC mapper bundles to version "in lockstep, pinned to the schema version," depending on "ADR 0029's platform-CI mapper tests." Those tests are asserted to exist but ADR-0029 was not in T0 scope; confirm the mapper-test gate is real, else AC-10's second half (bundle bump verification) has no harness.

### ADR-0031 — CloudEvent taxonomy — **SOUND**
The closed ten-namespace set is reproduced verbatim from interface-contract §2; conformance is CI-shaped (AC-01 rejects out-of-set types; AC-05 blocks a new namespace without an ADR). REQ-07 (audit vs security namespace distinctness) is concrete and matches ADR-0034. Clean.
- **ADR-0031-LOW-1 (LOW):** OQ med — "the exact CI/registry mechanism that enforces the closed set (conftest vs registry admission) is not specified in source." Correctly `[PROPOSED]`. AC-01/05 assume a "registry/CI conformance check" and an "architecture-conformance gate" exist; these are the enforcement points and they are deferred to B12 design. Blast LOW (the mechanism is an implementation detail of a well-defined invariant), but note both load-bearing ACs depend on an unbuilt gate.
- **ADR-0031-LOW-2 (LOW):** The open item "which namespace a chatops invocation lands under (ADR-0036 says 'likely `platform.lifecycle.*`')" is left genuinely open. Fine for a taxonomy ADR; flagged so the chatops binding is not assumed settled.

### ADR-0034 — Audit pipeline — **SOUND**
Excellent contract: the SoR topology (Postgres+S3 / OpenSearch advisory), the verify-then-delete batch cycle (REQ-04/05), graceful kind degradation (REQ-06), and reproducibility (REQ-07) are all concrete and testable; AC-03/04 (S3 verify + failure-retain) are strong binary tests. Reconciles with interface-contract §5 verbatim and the canonical ADR text.
- **ADR-0034-HIGH-1 (HIGH) — escalates X-HIGH-1 + a flagged operational gap:** REQ-11 (audit egress via Envoy) collides with the `EgressTarget` allowlist model — see **X-HIGH-1**. Additionally, the ADR's own §10 (high) flags that the in-flight Postgres `audit_events` table "grows unbounded while S3 is unrecoverable" and "the alerting threshold is an A18 design detail, not specified here." So the failure-bounding NFR ("no data loss") is honored, but the **operational backstop (alert before disk-full) is unspecified at the contract level** where it arguably belongs as an obligation on A18. Fix: add a REQ obliging A18 to ship an `audit_events` growth alert (threshold deferrable), so "no data loss" does not silently become "no data loss until the disk fills."
- **ADR-0034-LOW-2 (LOW):** `audit_events` column set and adapter/endpoint method signatures are `[PROPOSED]` (OQ med) — correctly tagged, blast LOW (schemas owned by A18/B12), AC-02 tests row-appears not column-shape.

### ADR-0040 — Kargo promotion fabric — **SOUND** (adjudicated for the ADR-0017 contradiction)
Read primarily to resolve the ADR-0017 consumer-count question. Enforcement points are concrete: `Approval` CRD as the human gate (REQ-05), OPA gates promotion actions (REQ-06), audit via adapter (REQ-10), uniform claim promotion via ADR-0041 (REQ-08). The two-kind CI test (REQ-11) is a strong, testable conformance. No contradiction with ADR-0017 from Kargo's side — Kargo correctly *consumes* the `Approval` CRD and ADR-0040 REQ-05 matches canonical ADR-0017's "first external consumer" framing. The inconsistency is entirely on the ADR-0017 spec side (ADR-0017-MED-1).
- **ADR-0040-LOW-1 (LOW):** Kargo-native Warehouse/Stage/Promotion objects are correctly noted as **not** §6.12 platform CRDs and tagged `[PROPOSED]` for any platform-CRD treatment. Good boundary discipline; noted only to confirm V6-12's inventory should NOT be expected to list them (it does not — consistent).

---

## Summary

| Piece | Verdict | HIGH | MED | LOW |
|---|---|---|---|---|
| V6-01 | SOUND | 0 | 0 | 2 |
| V6-02 | SOUND | 0 | 1 | 0 |
| V6-06 | NEEDS-WORK | 1 | 2 | 0 |
| V6-11 | SOUND | 0 | 1 | 1 |
| V6-12 | SOUND | 0 | 1 | 1 |
| V6-13 | SOUND | 0 | 0 | 0 |
| ADR-0013 | SOUND | 0 | 0 | 2 |
| ADR-0016 | SOUND | 0 | 1 | 2 |
| ADR-0017 | NEEDS-WORK | 0 | 2 | 1 |
| ADR-0020 | NEEDS-WORK | 1 | 1 | 1 |
| ADR-0030 | SOUND | 0 | 0 | 1 |
| ADR-0031 | SOUND | 0 | 0 | 2 |
| ADR-0034 | SOUND | 1 | 0 | 1 |
| ADR-0040 | SOUND | 0 | 0 | 1 |
| **Cross-cutting** | — | 2 | 1 | 0 |
| **Total** | — | **6** | **10** | **18** |

**Verdict tally:** 11 SOUND, 3 NEEDS-WORK, 0 BLOCKER.

**No BLOCKERs.** The contract layer is unusually disciplined (verbatim Canon reuse, honest `[PROPOSED]` tagging, self-red-team sections). The HIGH findings are gaps/under-specifications at security and versioning boundaries, not structural breaks.

**HIGH findings (6):**
- X-HIGH-1 — audit-endpoint egress vs `EgressTarget` allowlist unreconciled (V6-02/V6-06/ADR-0013/ADR-0034).
- X-HIGH-2 — "platform release" deprecation unit undefined, making ~5 versioning ACs untestable (ADR-0030/V6-13/V6-12/ADR-0041/ADR-0013).
- V6-06-HIGH-1 — the audit-point/OPA-decision-point enumerations the conformance ACs depend on exist only in non-frozen source, not in Canon or the spec.
- ADR-0020-HIGH-1 — credential modes + search/scrape OPA bundle (load-bearing, day-one) are `[PROPOSED]`; ACs verify existence not correctness.
- ADR-0034-HIGH-1 — egress-via-Envoy (X-HIGH-1) + unspecified `audit_events` unbounded-growth alert backstop.

**Top open questions to route (do not auto-resolve):** Does `EgressTarget` allowlisting apply to platform-infra egress (X-HIGH-1)? Ratify the "platform release" cadence-marker definition (X-HIGH-2). Add `system-mediated` to Canon `MCPServer.authMode` or tag it `[PROPOSED]` (ADR-0020-MED-2). Add the three derived claim forms to the glossary (V6-12-MED-1). Reconcile ADR-0017 REQ-07's "two initial use cases" with the canonical four-consumer set (ADR-0017-MED-1).
