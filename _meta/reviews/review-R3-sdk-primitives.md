# Review R3 — SDK / Dev Primitives (B3, B4, B6, B7)

> Reviewer: independent red-team, second pass · Date: 2026-05-31
> Scope: T0 "contract" pieces B3 (OPA policy framework), B4 (Crossplane v2 Compositions),
> B6 (Platform SDK), B7 (Agent SDK). Canon read: glossary.md, interface-contract.md,
> piece-index.csv rows (B3/B4/B6/B7/B11/A5/A6/A7/A1/A2), waves.md.
> Method: interface soundness vs interface-contract.md + upstream/downstream pieces;
> AC/REQ coverage; load-bearing `[PROPOSED]` blast-radius; contradictions vs waves.md + CSV.
> Probe focus (per brief): the B11→B4 dependency and whether B4's connection-secret fields
> are concretely specified or `[PROPOSED]`.

---

## Severity legend
HIGH = blocks a consumer or breaks a tested architectural invariant if unaddressed.
MED = real gap/contradiction, locally containable. LOW = polish / latent risk.

---

## B4 — Crossplane v2 Compositions  ·  VERDICT: NEEDS-WORK

B4 is the most carefully written of the four and correctly disciplines source-stated vs
`[PROPOSED]` fields. Two HIGH findings concern the **`MemoryStore` ↔ B11 seam** the brief asked
me to probe; the connection-secret contract is concretely specified **only for the four substrate
XRDs**, not for the composed XRs that B11 actually consumes.

- **[HIGH] B4 never specifies the connection-secret `MemoryStore` exposes.**
  File: `specs/components/B/spec-B4.md` §4.1 (composed-XR table, lines 127–133), §4.4 (lines 153–166),
  REQ-B4-12 / AC-B4-12. The tested connection-secret invariant (§4.4, REQ-B4-12) is scoped to the
  **substrate** XRDs (`XPostgres`/`XSearchIndex`/`XObjectStore`/`XMongoDocStore`). `MemoryStore` is
  a *composed* XR; §4.1 lists its fields as only `accessMode`, `backendType` — **no
  `connectionSecretRef`, no statement of what secret it writes/surfaces**. B11 (`spec-B11.md` §4.4,
  REQ-B11-07, AC-B11-06) binds hard to "the uniform connection secret … written by the
  `XPostgres`/`XSearchIndex`/`XObjectStore` Compositions that back the `MemoryStore`." B4 never
  states that `MemoryStore` composes those primitives, nor whether it re-surfaces their secret or
  writes its own. This is an unclosed interface seam between the two pieces.
  Fix: B4 must add, for `MemoryStore`, either (a) an explicit `connectionSecretRef` + the substrate
  primitives it composes, or (b) a statement that B11 reads the composed `XPostgres`/`XSearchIndex`
  secret directly and `MemoryStore` surfaces it. Add an AC asserting B11 can resolve the secret
  from a `MemoryStore` claim. (Open question — do not auto-resolve which of a/b.)

- **[HIGH] Per-primitive connection-secret field lists are load-bearing `[PROPOSED]`.**
  File: `spec-B4.md` §4.4 (lines 156–160), R1 (line 284). B4 itself rates this HIGH: only
  `host/port/user/password/dbname` are Canon; the `XObjectStore` (and `MemoryStore`-backing search)
  equivalents (`bucket`/`accessKeyId`/`secretAccessKey`, etc.) are `[PROPOSED]`. Consumers A18 and
  **B11** bind to these keys. interface-contract §4 + §6 explicitly leave per-primitive field lists
  to component design, so this is a legitimately-open Canon gap, not a spec defect — but it is the
  single highest-blast-radius decision in B4's surface and is currently unfrozen.
  Blast radius: HIGH (A18 audit pipeline + B11 memory both read these secrets).
  Fix: land TASK-02 (freeze the per-primitive mapping table) as a hard predecessor to A18/B11; have
  both consumers cite the frozen table by reference.

- **[MED] CSV dependency-edge asymmetry: B4→B11 vs B11.upstream.**
  `piece-index.csv`: B4 row lists `B11` in downstream; **B11 row lists upstream = `A10` only**, B4
  absent. B4 §3 and plan §3.2/§6.3 ("must merge before … B11 land") treat B11 as a HARD consumer;
  B11 §3 + its own R4 flag the same gap. The two specs agree on intent but the CSV edge is
  one-directional, so any tooling that reads `upstream` (not `downstream`) will miss the B4→B11
  ordering constraint. Fix: add `B4` to B11's CSV `upstream` column (Canon CSV edit — flag to Canon
  owner; I did not edit it).

- **[MED] B4 and B11 are both placed in W1 despite a hard B4→B11 build edge.**
  `waves.md` W1 band + `piece-index.csv` (both rows `wave=W1`). A same-wave hard dependency is
  permissible only if intra-wave ordering is honored; B4 plan §6.3 asserts it ("must merge before …
  B11"), but waves.md gives no intra-W1 ordering and itself flags B4 as the "single wave-assignment
  gap in §14." Consumers A18/A21 are also W1. This is a latent scheduling hazard, not a
  contradiction. Fix: note the intra-W1 ordering (B4 before A18/A21/B11) explicitly in waves.md or
  accept B4 as a sub-foundation; surface to wave owner.

- **[MED] `MemoryStore.backendType` value grammar undefined but B11 switches on it.**
  `spec-B4.md` §4.1 (line 131) carries `backendType` verbatim from interface-contract §1.6 with no
  enumeration; B11 §4.1 ("reads `backendType` to select the Letta path") branches on it. Neither
  piece enumerates the legal values. Fix: B4 should `[PROPOSED]` an enumerated `backendType` set (or
  state it's Letta-only in v1.0) so B11's selection logic has a contract to test against.

- **[LOW] Claim-form kinds `SearchIndex`/`ObjectStore`/`MongoDocStore` not in Canon.**
  `spec-B4.md` §4.1 lines 110–112 use these claim forms; they are absent from glossary.md /
  interface-contract.md (only `AuditLog`, `GrafanaDashboard`, `AgentDatabase` claim forms are
  Canonized). Already logged in `_meta/reviews/consistency-existence.md` Task 1 as a Canon
  incompleteness; restating for cross-reference. Fix: Canon owner adds the three claim forms.

- **[LOW] `status.substrateDetail` and `substrateClass` value grammar are `[PROPOSED]`** (§4.1
  lines 114–116, 135–139; R2/R3). Contained: lint/admission-validate as the spec proposes.

AC/REQ coverage for B4: every REQ-B4-01..14 has a covering AC-B4-01..14; mappings are sound and
mostly testable. The two HIGH items are *missing coverage* for the `MemoryStore` secret seam, not
weak existing ACs.

---

## B6 — Platform SDK  ·  VERDICT: SOUND (with one HIGH watch-item it self-declares)

B6 is internally consistent, correctly marks every method signature `[PROPOSED — not in source]`
(interface-contract §3.1 authorizes exactly this), and correctly treats `platform.capability.changed`
+ `refresh_capabilities()` as the only Canon-named symbols. §4.4 correctly says N/A for
connection-secret (SDK holds no datastore). No dependency-direction errors.

- **[HIGH] All four surface signatures `[PROPOSED]`; B7/B10/B21 bind to them.**
  `spec-B6.md` §4.2 + R1 (line 234). B6 self-rates this HIGH and proposes the right mitigation
  (freeze-first, treat as versioned contract, REQ-B6-08). This is the correct posture — flagging
  only because it is the single load-bearing decision and its freeze must precede B7 (W3). No fix
  beyond executing TASK-01 before any consumer binds. Blast radius: HIGH but mitigated by semver.

- **[MED] AC-B6-06 conflates two behaviors and uses a soft bound ("within one turn").**
  `spec-B6.md` AC-B6-06 (lines 216–218) covers REQ-B6-06's two distinct guarantees (event-callback
  fires; poll fallback returns updates) in one criterion, and "within one turn" is undefined for a
  library with no turn concept. Fix: split into two ACs and replace "within one turn" with a
  concrete, testable bound (e.g. "returns the updated set on next call").

- **[MED] REQ-B6-12 ("both LangGraph primitives and Deep Agents idioms") duplicates B7's REQ-B7-03
  — ownership ambiguity.** `spec-B6.md` REQ-B6-12 / AC-B6-12 vs `spec-B7.md` REQ-B7-03 / AC-B7-03
  state nearly the same obligation. B6 §2.1 also says bindings live in B7 ("Agent-authoring harness
  … → B7"). Having B6 *require* the dual-idiom binding while B7 *also* requires it risks double
  ownership / double implementation. Fix: B6 should own only the single language-neutral API
  contract; the idiom binding belongs to B7. Reword REQ-B6-12 to "MUST expose one API contract that
  B7 can bind into both idioms" and drop the binding obligation from B6.

- **[LOW] OQ1 (memory→Letta direct vs via LiteLLM) is unresolved and affects AC-B6-01's meaning.**
  §10 OQ1 + §4.2 `memory.*`. The "round-trips through the platform memory path" AC is testable
  either way, so non-blocking, but the path choice should be settled with A10/B11 before TASK-04.

AC/REQ coverage for B6: REQ-B6-01..13 each have a covering AC-B6-01..13. Sound except the AC-B6-06
split above.

---

## B7 — Agent SDK (LangGraph + Deep Agents)  ·  VERDICT: SOUND

B7 correctly defines no CRD, treats `Agent.sdk` values `langgraph`/`deep-agents` as Canon, defers
signatures to B6, and routes all enforcement externally. Dependency *direction* is correct (B6/A5/A6
upstream → B7). One real cross-piece risk and a branch-mapping inaccuracy.

- **[MED] `Agent.sdk` admission-validation is `[PROPOSED]` and lands in another owner (A5).**
  `spec-B7.md` §4.1 (lines 90–92), R3 (line 213). Source says the enum grows "without a schema
  break" but names no validation mechanism. B7 cannot enforce this (A5 owns `Agent`), and the spec
  rightly makes the harness reject unknown `sdk` as a backstop. Real risk: an unenrolled `sdk` value
  admitted by A5 reaches the harness. Fix: raise an explicit coordination item with A5 to
  admission-validate `Agent.sdk` against the allowed set; keep harness-side rejection as defense.
  (Open — do not auto-resolve where validation lives.)

- **[MED] Plan branch-base note misstates wave membership.**
  `plans/components/B/plan-B7.md` §6.1: base branch `wave/3` "contains B6/A5/A6 specs." Per CSV,
  **B6 is W2, A5 is W1, A6 is W0** — none are in W3. The dependency order is fine (earlier waves →
  W3), but the parenthetical is factually wrong and would mislead a brancher. Fix: reword to "base
  `wave/3`; upstreams B6 (W2)/A5 (W1)/A6 (W0) merged in earlier waves." (Same loose phrasing appears
  in `plan-B6.md` §6.1 — "wave/2 … contains A1/A2 specs"; A1/A2 are W0 — and `plan-B3.md` §6.1 —
  "wave/1 … contains A7 spec"; A7 is W0. Logged once here; all three are LOW individually, MED as a
  pattern.)

- **[LOW] REQ-B7-07/AC-B7-07 (additive enrollment, no schema break) tests a *mock* sdk value only.**
  Acceptable for v1.0 but worth noting the guarantee is only exercised against a stub, not a real
  second SDK. Non-blocking.

AC/REQ coverage for B7: REQ-B7-01..10 each have a covering AC-B7-01..10; all testable. Sound.

---

## B3 — OPA policy library (framework)  ·  VERDICT: SOUND

(Not strictly an "SDK primitive" — it's the Rego framework — but in the assigned set.) Internally
consistent; correctly splits framework (B3) vs content (B16); marks the whole decision-document
field set `[PROPOSED]` save the one source-named field `simulated`. Dependency direction correct
(A7 → B3 → B16).

- **[MED] The decision-document/input contract is fully `[PROPOSED]` yet ~6 consumers bind to it.**
  `spec-B3.md` §4.2 (lines 84–99), R1 (line 193). Only `simulated` (ADR 0038) is source-named;
  `subject/action/target/context/dry_run/allow/reasons[]/layer` are all `[PROPOSED]`. B2, A20, B16,
  B19, Envoy gates, and B9 all bind to it (§3, §6.6). B3 self-rates HIGH and proposes freeze-first;
  the mitigation is sound, so I rate the residual MED. Fix: execute TASK-01 as a hard predecessor to
  any consumer; publish the frozen contract as the versioned artifact (REQ-B3-03/04).

- **[MED] REQ-B3-06 ("structurally impossible to grant beyond RBAC floor") is not achievable by
  Rego convention alone.** `spec-B3.md` REQ-B3-06 + R2 (line 196). The spec's own R2 concedes this
  needs a CI lint rather than a language guarantee, but REQ-B3-06 and AC-B3-06 are still worded as a
  structural impossibility; AC-B3-06 then tests it as "a policy attempting to grant … is rejected by
  the library predicate / a framework test fails" — the "/ a framework test fails" escape hatch makes
  the AC pass via the lint, contradicting the REQ's "structurally incapable" wording. Fix: reword
  REQ-B3-06 to "the framework MUST provide a lint/test gate that rejects any allow-rule not gated by
  the RBAC-floor predicate" so REQ and AC describe the same (achievable) mechanism.

- **[LOW] B3 is the only one of the four with a hardcoded canon hash in front-matter**
  (`canon-glossary: b0edae10a2e6 · canon-interface: 0ce201d5d5af`) vs the "see _meta/…" form the
  other three use. Cosmetic; matches the corpus-wide hash-format inconsistency already logged in
  `consistency-existence.md` Task 4.

AC/REQ coverage for B3: REQ-B3-01..11 each have a covering AC-B3-01..11; all testable except the
REQ-B3-06 wording mismatch above.

---

## Cross-piece summary

| Piece | Verdict | HIGH | MED | LOW |
|---|---|---|---|---|
| B4 | NEEDS-WORK | 2 | 3 | 2 |
| B6 | SOUND | 1 | 2 | 1 |
| B7 | SOUND | 0 | 2 | 1 |
| B3 | SOUND | 0 | 2 | 1 |
| **Total** | — | **3** | **9** | **5** |

**The single dominant theme:** every one of these four T0 pieces is a *contract* whose load-bearing
fields are legitimately `[PROPOSED — not in source]` (SDK signatures, Rego decision document,
per-primitive secret keys). Each spec correctly flags freeze-first as the mitigation. The platform
risk is **scheduling**: these contracts MUST be frozen before their W2/W3 consumers branch, and the
specs assert this but the wave/CSV machinery does not encode intra-wave ordering.

**Probe answer (B11→B4 / B4 connection-secret):** B4's connection-secret fields are concretely
specified **only for the four substrate XRDs** and only as the five canonical keys; the
per-primitive equivalents and — critically — **the secret that the composed `MemoryStore` XR
actually exposes to B11 are NOT specified** (B4 §4.1/§4.4). B11 binds to a secret B4 never promises.
This is the most actionable interface gap in the set (B4 HIGH #1).

## Open questions surfaced (not resolved here)
1. Does `MemoryStore` write its own connection secret, or surface the backing `XPostgres`/
   `XSearchIndex` secret? (B4 §4.1) — must be decided with B11.
2. Where does `Agent.sdk` admission validation live — A5 admission, harness, or both? (B7 R3)
3. Does B6 `memory.*` reach Letta directly or via LiteLLM? (B6 OQ1) — affects B11 seam.
4. Should the dual-idiom binding obligation sit in B6 or B7? (B6 REQ-B6-12 vs B7 REQ-B7-03)
5. Canon CSV: add `B4` to B11's `upstream`; add the three claim-form kinds to glossary.

No spec/plan/source files were edited; only this review file was written.
