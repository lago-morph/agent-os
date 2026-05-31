# DECISIONS NEEDED — consolidated register

> Synthesizes the five T0 red-team reviews (`review-R1..R5*.md`), the consistency
> sweep (`consistency-existence.md`, `csv-reciprocity.md`), and the §10 open-questions
> of the six Operational/NFR specs (`specs/ops/spec-OPS1..6.md`).
> Each item has a stable ID and a decision-owner hint. Nothing here is auto-decided —
> these are surfaced for a human ruling. See the cited source files for full detail
> (severity breakdowns, AC ids, line refs).

---

## 1. Canon-level contradictions & blockers (resolve before build)

| ID | Title | Source | Sev | The conflict | Recommended direction |
|----|-------|--------|-----|--------------|-----------------------|
| **D-01** | `MCPServer.authMode: system-mediated` contradiction | R2, R5 (A17 / ADR-0020 / interface-contract §1.4) | HIGH | `interface-contract.md §1.4` (FROZEN) lists only `system`/`user-cred`; FROZEN `ADR-0020` mandates a third `system-mediated` mode and silently coins it (untagged). Two frozen Canon docs conflict; A17's system-mediated AC is untestable until the enum is authoritative. | Ratify the third enum value in interface-contract §1.4 (Canon revision) **or** drop it from ADR-0020. Pick one source of truth. |
| **D-02** | B19 `platform.approval.*` events are ownerless | R4 (**BLOCKER**) | BLOCKER | B19 attributes the approval event types to B12, but B12 `REQ-B12-09` disclaims inventing names, and Canon says "B12 owns the registry; each component owns its event types." The 4 approval events that A23/B5/B10/A14 bind to have no owner and no authoring task. | Assign authorship to **B19** (contributed into B12's registry); add the authoring task to B19's plan. |
| **D-03** | OPA decision-document ownership collision | R1 | HIGH | A7 *claims* to own the OPA dry-run/decision-document shape, but A20 and B3 both place ownership in **B3**; A6 and A1 also bind. One contract, four conflicting owners. | Designate **B3** as owner (per A20/B3); make A7/A6/A1 consumers. |
| **D-04** | B4 `MemoryStore` connection-secret unspecified | R3 | HIGH | B11 binds (`REQ-B11-07`/`AC-B11-06`) to a connection-secret B4 never promises; the `MemoryStore` XR lists only `accessMode`/`backendType`, no `connectionSecretRef`. B4's per-primitive secret fields beyond the 5 canonical keys are all `[PROPOSED]` and load-bearing for A18+B11. | B4 spec must concretely define the `MemoryStore` connection-secret keys. Couples with **D-09**. |
| **D-05** | A18 audit-adapter contract + `audit_events` schema entirely `[PROPOSED]` | R2 | HIGH | The adapter interface and `audit_events` schema bind every emitting/reading component platform-wide, yet are unspecified and no gated AC enforces a freeze. Corpus release blocker (not an A18 defect — source defers it). | Freeze & publish the adapter interface + `audit_events` schema before any downstream emission; add a freeze-gate AC. |
| **D-06** | A22↔A21 `TenantOnboarding` editor double-ownership | R2 (+ A21 OQ-1/-2) | HIGH | §14.1 says both "A22 ships the initial editor set incl. TenantOnboarding" **and** "A21 owns its CRD's editor." Both specs hard-mandate building it; both tag it `[PROPOSED]`. | Settle the split — recommend A21 owns its own editor; A22 ships the framework + non-A21 editors. Also resolves the A22↔A21↔B4 overlap. |
| **D-07** | 3 CRDs referenced but absent from Canon glossary | C1 (consistency-existence) | HIGH | `SearchIndex`, `MongoDocStore`, `ObjectStore` are used throughout specs but are not in `glossary.md` (specs reference them via Crossplane **v1** "claim-form" framing — see D-11). | Add them to the glossary (as Crossplane v2 composite types) **or** rename references to existing kinds; fold into D-11. *(You deferred this Canon edit to your ruling.)* |
| **D-08** | Co-owned-but-undefined egress/budget contracts | R1, R5 (A6↔B13, A7↔B13; B13 rates both high blast-radius) | HIGH | The `EgressTarget`→Envoy wiring mechanism (A6↔B13) and the budget OPA-data document shape (A7↔B13) are `[PROPOSED]` on **both** producer and consumer. R5 adds: the audit-endpoint egress conflicts with the `EgressTarget` allowlist with no declared targets. | Joint freeze of both contracts + declare the audit endpoint's egress target. |
| **D-11** | Specs written against Crossplane **v1**; platform is **v2** | terminology sweep (this session) | HIGH | ~270 Crossplane-context refs use the v1 `claim` / `X`-prefixed-composite model; ADR-0041 codifies an "XR↔claim naming convention." Crossplane v2 removed claims (XRs are namespaced directly), so the specs describe a resource model the platform won't use. | Coordinated v2-conformance pass: revise/supersede ADR-0041 to the v2 model, then sweep the corpus to re-express terminology **and** logic in v2 terms (not a blind `s/claim/XR/`). |

---

## 2. DAG / index questions

| ID | Item | Source | Status / action |
|----|------|--------|-----------------|
| **D-09** | `B4↔B11` edge direction | C3 (AMBIGUOUS) | csv has B4 downstream=B11 but B11's upstream omits B4; unclear whether the real edge is `B4→B11` or `B11→B4`. Needs a ruling; couples with **D-04** (MemoryStore secret). |
| **D-10** | 4 safe csv reciprocity fixes | C3 | **APPLIED this session** — `B4` added to the `upstream` of A18/A21/A23/B19. Listed for traceability; canonical `piece-index.csv` updated, ambiguous B4↔B11 left untouched. |

---

## 3. Proposed ADRs (genuinely open decisions — 28, grouped)

Collected from OPS1–OPS6 §10. Titles are proposals; each gates the listed pieces.

**Secrets / keys**
- KMS / envelope-encryption provider for secrets at rest — *OPS3; gates all connection-secrets.*
- Secret/key rotation cadence policy (per-class intervals + overlap windows) — *OPS3; signing-key/OIDC-root cadence is the top open Q.*
- Rotation-metadata location (ESO config vs CRD field vs external policy) — *OPS3.*
- User-credential OAuth token lifecycle (persist/refresh/revoke in LiteLLM) — *OPS3; A1/A17.*

**Tenancy / isolation**
- Tenant compute-quota realization: `AgentEnvironment.quotas` vs native `ResourceQuota`/`LimitRange` vs both, and where admission denial fires — *OPS1 #2 + OPS4 #A (same decision); gates B4/A5/A6 + every onboarding.*
- Default/minimum sandbox `RuntimeClass` policy per tenant tier (gVisor/Kata/runc) + substrate fallback — *OPS4.*
- Soft vs hard multi-tenancy: node-sharing + node-pool entitlement scheme — *OPS4; sets escape blast radius (biggest OPS4 Q).*
- Pod-layer `NetworkPolicy` enforcement contract across substrates (CNI capability + deny-by-default baseline) — *OPS4.*
- Default tenant-scoped quota values (max agents/sandboxes/virtual keys/cost budget/rate limit) + override path — *OPS1.*

**Scale / cost**
- Platform autoscaling mechanism & per-component scaling stance (HPA vs KEDA vs single-instance per ADR-0035) — *OPS1.*
- Tenant cost-attribution join + showback aggregation site (dashboard-time query vs stored rollup) — *OPS1.*
- Promote v1.0 capacity baselines to candidate SLIs feeding future SLO mgmt — *OPS1 (ties OPS5 SLO ADR).*

**Disaster recovery**
- Backup/DR policy attachment: out-of-band operator config vs first-class XRD field — *OPS2.*
- Authoritative per-data-class RPO/RTO targets for v1.0 — *OPS2; source fixes no numbers, yet they gate every DR drill (biggest OPS2 Q).*
- etcd/control-plane backup posture: GitOps-reconcile-only vs etcd snapshots — *OPS2.*
- v1.0 cross-region/AZ failover model: backup-restore-only vs active-passive vs active-active — *OPS2.*
- Per-tenant restore granularity: whole-cluster vs per-tenant PITR — *OPS2.*

**Day-2 / migration**
- CRD/XRD production migration strategy (stored-version migration cadence, webhook HA/cert rotation, deprecation-calendar unit) — *OPS5; highest-blast-radius Day-2 Q.*
- Platform SLO/SLI targets & ownership — *OPS5.*
- Adopt the Crossplane `Operation` type for declarative Day-2 imperative actions — *OPS5.*
- Declarative recurring health/synthetic-probe + capacity-baseline mechanism (`ScheduledTest`/`HealthCheck` XRD) — *OPS5.*
- ArgoCD drift-remediation stance: auto-heal+prune vs detect-and-alert per resource class — *OPS5.*

**Policy / Rego**
- Policy change-control model (who can change policy, review/approval flow) — *OPS6.*
- Ownership of the global/cross-tenant OPA bundle — *OPS6; combined security + availability SPOF (biggest OPS6 Q).*
- Policy-bundle signing + load-time verification — *OPS6.*
- Staged-rollout/promotion mechanism for bundles (audit-mode→enforce; whether Kargo Stages gate policy) — *OPS6.*
- Anti-lockout / break-glass recovery contract — *OPS6.*

---

## 4. Quality nits (non-blocking)

| ID | Item | Source | Note |
|----|------|--------|------|
| **QN-01** | OPS5 has 16 ACs for 22 REQs | OPS5 | ~6 REQs may lack a dedicated covering AC; add ACs or document shared coverage. |
| **QN-02** | Inconsistent `canon-*` front-matter hash formatting | C1 | Lengths 8/12/16/40 + mixed "FROZEN"/file-ref styles. Cosmetic; normalize in a sweep (all bind the same frozen Canon). |
| **QN-03** | B12 registry models 10 event namespaces but only ~1 has a landing task | R4 | The other 8 (incl. approval — see D-02) need authoring tasks before emitters land. |
| **QN-04** | Untestable / weak ACs | R4, R5 | B16 "structurally-incapable-of-granting" invariant tests only a planted canary; B16 self-referential admission-coverage map; V6-06 conformance ACs depend on non-frozen source enumerations. |
| **QN-05** | "platform release" deprecation-calendar unit undefined | R5, OPS5 | Makes ~5 versioning ACs untestable and gates OPS5 migration ACs; define the unit (ties to the CRD/XRD migration ADR). |

---

### Cross-corroboration note
Two items were independently flagged by two different reviewers, raising confidence they are real: **D-01** (authMode) by R2 *and* R5; **D-04 / D-09** (B4↔B11 / MemoryStore secret) by R3 *and* the C3 csv asymmetry.
