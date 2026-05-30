# SPEC F4 — Security review

> kind: COMPONENT · workstream: F · tier: T2
> upstream: [B22, B16, A6, A7, B13, A18, A17] · downstream: [] · adrs: [0027, 0002, 0003, 0018, 0013, 0034, 0029, 0033] · views: [6.6]
> canon-glossary: b0edae10a2e6 · canon-interface: 0ce201d5d5af

## 1. Purpose & Problem Statement

F4 is the production-readiness **security review**: the end-of-v1.0 pass that exercises the defense-in-depth controls the architecture committed to (§6.6) against the adversarial threat model B22 produced (ADR 0027), and that performs the operational security drills v1.0 names — credential rotation, secret-handling audit, sandbox-escape testing, OPA policy-bundle review, and scoping a penetration test. ADR 0027 fixes the v1.0 *stance* (primary modeled threat = unintentional excess access; B22 is the design-spec prerequisite that ships before implementation); F4 *verifies* that the shipped platform meets B22's mandatory test cases, security standards, and dashboard signals.

F4 does not author the threat model (that is B22, already landed) and does not write new policy bundles (B16). It validates: it runs the drills, audits secret handling, reviews the bundle for gaps against B22 targets, attempts sandbox escape, and produces a pen-test scope document plus a findings/remediation list routed to the owning components and to F6.

## 2. Scope

### 2.1 In scope
- **Credential rotation drills**: rotate platform credentials/secrets (LiteLLM virtual keys via `VirtualKey` `ttl`, MCP-service creds via ESO, provider keys) and verify rotation propagates without outage and old material is invalidated.
- **Secret-handling audit**: trace every secret from cloud provider (AWS Secrets Manager) → ESO → workload (gateway, agents, MCP services), confirming no secret is committed to Git, logged, or exposed in audit/trace payloads.
- **Sandbox escape testing**: adversarial attempts against the agent-sandbox kernel isolation (gVisor / Kata, ADR 0003-adjacent A6) and Envoy egress allowlist, mapping to B22's "sandbox escape" and "capability escape" attack patterns.
- **Policy-bundle review**: review the B16 OPA Rego bundle against B22's OPA policy targets — admission for all v1.0 CRDs, runtime gateway authorization, egress policy, RBAC-floor/OPA-restrictor enforcement, Headlamp action gating — for gaps/unreachable rules.
- **Pen-test scope**: a scoping document defining what an external/internal penetration test would cover for v1.0, including trust boundaries (§6.6) and out-of-scope items, plus rules of engagement.
- **Findings register**: prioritized findings + residual-risk notes routed to owning components and F6.

### 2.2 Out of scope (and where it lives instead)
- **The adversarial threat-model design** → **B22** (ships before implementation; F4 verifies against it, ADR 0027).
- **Writing/owning OPA policy content** → **B16** (initial policy library) / **B3** (framework); F4 reviews, files gaps.
- **The sandbox runtime / Envoy proxy themselves** → **A6**; F4 attacks them, does not build them.
- **The audit pipeline / tamper-resistance mechanism** → **A18 / ADR 0034**; F4 verifies audit integrity, does not build it.
- **Executing the penetration test** → out of v1.0 build scope; F4 produces the *scope*, the test is an engagement.
- **Continuous container security / image signing / re-scan** → future-enhancements.md §2 (deferred); build-time Trivy/Grype scan is B15/§8.
- **Identity federation chain config** → ADR 0028/0029, B1/Keycloak; F4 verifies claim enforcement, does not configure IdP.
- **Cross-layer policy-conflict / drift analysis tooling** → future-enhancements.md §13 (deferred); F4 does a manual bundle review, not automated conflict analysis.

## 3. Context & Dependencies

**Upstream consumed:**
- **B22 (threat model design spec)** — the security standards, mandatory test cases, mandatory dashboard signals, and OPA policy targets F4 verifies the platform against (ADR 0027). This is F4's acceptance contract.
- **B16 (initial OPA policy library content)** — the Rego bundle F4 reviews for coverage against B22 targets.
- **A6 (agent-sandbox + Envoy egress proxy)** — the kernel isolation + egress allowlist F4 attempts to escape.
- **A7 (OPA / Gatekeeper)** — the policy engine whose admission/runtime decisions F4 probes.
- **B13 (kopf operator)** — reconciles `VirtualKey` (rotation via `ttl`) and capability CRDs F4 exercises during rotation/capability-escape testing.
- **A18 (audit endpoint + adapter)** — the audit integrity F4 verifies (tamper-resistance, completeness of emission points §6.6).
- **A17 (initial MCP services)** — credential-bearing integrations F4 audits for secret handling.

**Downstream consumers:** none directly; F6 consumes the findings + drill runbooks.

**ADRs honored:**
- **ADR 0027** — v1.0 threat-model stance: primary threat is unintentional excess access; B22 is the prerequisite design spec; the dedicated adversarial model refines §6.6's defense-in-depth. F4 verifies against B22's outputs.
- **ADR 0002 / 0018** — OPA + Gatekeeper; RBAC-as-floor / OPA-as-restrictor. F4 reviews the bundle and verifies the enforcement model holds.
- **ADR 0003** — Envoy egress proxy as CNI-agnostic egress control; F4 tests the FQDN allowlist.
- **ADR 0013** — capability CRD model; F4 tests for capability escape (agent acquiring a capability not in its set).
- **ADR 0034** — audit emission durability/integrity; F4 verifies audit-tampering resistance and emission completeness.
- **ADR 0029** — required Keycloak JWT claim schema; F4 verifies claim enforcement at LiteLLM/OPA/Headlamp.
- **ADR 0033** — AWS (EKS) + GitHub initial targets; pen-test scope is framed against this deployment.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
F4 introduces **no new CRD/XRD**. It exercises existing ones during drills: `VirtualKey` (`ttl`, rotation), `CapabilitySet` / `MCPServer` / `A2APeer` / `EgressTarget` (capability-escape testing — Canon §1.4), `Sandbox` / `SandboxTemplate` (`runtime` gVisor/Kata — Canon §1.3). It asserts behavior, sets no schema.

### 4.2 APIs / SDK surfaces
N/A — F4 introduces no API/SDK surface. Drills are run through the `agent-platform` CLI / test framework (B9/B14) and adversarial probes. The Platform JWT claim schema (Canon: `platform_tenants`, `platform_namespaces`, `platform_roles`, `tenant_roles`, `capability_set_refs`) is the identity surface F4 verifies enforcement of.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
F4 consumes `platform.security.*` (sandbox-escape signal, repeated authn failures, policy-bypass attempts — Canon §2) and `platform.policy.*` (OPA decisions/violations) as the detection signals it verifies fire during attack simulation. It also reads `platform.audit.*` to confirm attempted actions are audited. F4 mints no new event types; whether a probe *should* raise a `platform.security.*` event is a B12/B22 schema question — `[PROPOSED — not in source]` where a specific event name is needed.

### 4.4 Data schemas / connection-secret contracts
F4 audits the connection-secret contract (Canon §4) for leakage: secrets MUST remain in the ESO→workload path and never appear in Git, logs, audit, or traces. It verifies, adds no fields.

## 5. OSS-vs-Custom Decision
**Review + drills using existing tooling and the test harness — no new build.** Rotation uses native `VirtualKey` `ttl` + ESO resync; secret-handling audit is inspection + B22 checklist; sandbox-escape testing uses adversarial probes against gVisor/Kata + Envoy; bundle review is manual Rego inspection against B22 targets (automated conflict analysis is deferred, future §13). Orchestration is **config/wrap** of B9/B14. No fork, no new service. Rationale: ADR 0027 places the *design* in B22 and the *controls* in A/B components; F4 is the verification gate, not new machinery.

## 6. Functional Requirements
- **REQ-F4-01:** F4 MUST perform a credential rotation drill (`VirtualKey` `ttl`/reissue, ESO-managed MCP/provider secrets) and verify the new credential works, the old is invalidated, and no request outage occurs during propagation.
- **REQ-F4-02:** F4 MUST audit the full secret path (AWS Secrets Manager → ESO → workload) and confirm no secret material is present in Git, container logs, audit records, or traces.
- **REQ-F4-03:** F4 MUST attempt sandbox escape against gVisor/Kata isolation and the Envoy egress allowlist, and confirm escape attempts fail and are detected/audited (`platform.security.*`).
- **REQ-F4-04:** F4 MUST attempt capability escape (agent invoking a capability not in its CapabilitySet) and confirm OPA/gateway denies and the attempt is audited (ADR 0013/0018).
- **REQ-F4-05:** F4 MUST review the B16 OPA bundle against B22's OPA policy targets and produce a coverage matrix (admission per CRD, gateway runtime, egress, RBAC-floor/OPA-restrictor, Headlamp gating) flagging gaps/unreachable rules.
- **REQ-F4-06:** F4 MUST verify audit integrity: emission points (§6.6 — LiteLLM, Gatekeeper, Envoy, broker, ARK, sandbox controller, ESO, ArgoCD, LibreChat) emit, and audit records resist tampering (ADR 0034 system-of-record).
- **REQ-F4-07:** F4 MUST produce a **pen-test scope** document covering trust boundaries (§6.6), in/out-of-scope assets, adversary classes (B22), and rules of engagement for the ADR 0033 deployment.
- **REQ-F4-08:** F4 MUST produce a prioritized **findings register** with residual-risk notes, routed to owning components (A6/A7/B16/A18/...) and to F6.
- **REQ-F4-09:** F4 MUST confirm Keycloak JWT claim enforcement (ADR 0029 schema) at the consuming surfaces (LiteLLM, OPA, Headlamp, LibreChat) — a forged/over-broad claim is rejected.

## 7. Non-Functional Requirements
- **Security:** F4 is itself a privileged activity; drills MUST run in a controlled environment, MUST NOT exfiltrate real secrets, and MUST record only pass/fail + redacted evidence. Sandbox-escape probes MUST be contained to a throwaway cluster.
- **Multi-tenancy (§6.9):** F4 MUST verify tenant↔tenant and internal-A2A↔external-A2A boundaries (B22 trust boundaries) — a probe from tenant A MUST NOT reach tenant B's data/capabilities.
- **Observability (§6.5):** F4 MUST verify B22's *mandatory dashboard signals* exist and light up under attack simulation (e.g., repeated authn failures, policy-bypass attempts surface on dashboards).
- **Scale:** F4 MUST include a DoS-class probe (budget/capability exhaustion — B22 attack pattern) confirming `BudgetPolicy`/quota controls degrade gracefully, coordinated with F5 load numbers.
- **Versioning (ADR 0030):** the bundle-review coverage matrix MUST note the policy-bundle version reviewed; a later bundle change re-opens the relevant review rows.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **N/A — F4 ships no deployable; it drills/reviews existing-deployed controls.**
- Per-product docs (10.5) — **applicable** (security-review report, coverage matrix, pen-test scope, secret-handling audit).
- Runbook (10.7) — **applicable** (credential rotation runbook; secret-leak response; sandbox-escape response). Feeds **F6**.
- Alerts — **applicable (verify, not author)** — F4 confirms B22 mandatory security alerts fire; gaps are findings. `[PROPOSED]` new alerts route to owning components.
- Grafana dashboard (Crossplane XR) — **applicable (verify)** — F4 confirms B22 mandatory dashboard signals exist; missing signals are findings (owned by the relevant A/D component).
- Headlamp plugin — **N/A — review activity, not an interactive CRD surface.**
- OPA/Rego integration — **applicable (review)** — F4 reviews B16's bundle coverage; it files gaps, B16 owns the Rego.
- Audit emission (ADR 0034) — **applicable** — F4 verifies emission completeness + tamper-resistance; a security-review-action audit record is `[PROPOSED]`.
- Knative trigger flow — **applicable (verify)** — F4 confirms `platform.security.*` detection signals route (e.g., toward HolmesGPT per §6.7); introduces no new flow.
- HolmesGPT toolset — **N/A — F4 verifies detection signals reach HolmesGPT; it adds no toolset.** `[PROPOSED]` security-posture read tool.
- 3-layer tests — **applicable** (Chainsaw for admission-deny / capability-escape-deny; PyTest for rotation propagation + secret-leak scans; Playwright for Headlamp action-gating / claim enforcement).
- Tutorials & how-tos — **applicable** (how-to: rotate a credential; how-to: read the security-review coverage matrix).

## 9. Acceptance Criteria
- **AC-F4-01:** A rotated `VirtualKey`/secret takes effect, the old value is rejected, and in-flight traffic sees no outage. (→ REQ-F4-01)
- **AC-F4-02:** A scan of Git, logs, audit records, and traces finds zero secret material along the ESO→workload path. (→ REQ-F4-02)
- **AC-F4-03:** A sandbox-escape attempt fails and produces a `platform.security.*` signal + audit record. (→ REQ-F4-03)
- **AC-F4-04:** An agent call to a capability outside its CapabilitySet is denied and audited. (→ REQ-F4-04)
- **AC-F4-05:** A coverage matrix maps every B22 OPA target to a bundle rule (or a flagged gap), with the reviewed bundle version. (→ REQ-F4-05)
- **AC-F4-06:** Each §6.6 emission point is shown emitting; a tamper attempt on an audit record is detectable. (→ REQ-F4-06)
- **AC-F4-07:** A pen-test scope document exists covering §6.6 trust boundaries, B22 adversary classes, and rules of engagement. (→ REQ-F4-07)
- **AC-F4-08:** A prioritized findings register exists, each finding assigned to an owning component and referenced from F6. (→ REQ-F4-08)
- **AC-F4-09:** A forged/over-broad Keycloak claim is rejected at LiteLLM/OPA/Headlamp/LibreChat. (→ REQ-F4-09)

## 10. Risks & Open Questions
- **R1 (high):** A real exploitable finding (e.g., a working sandbox escape or capability escape) is a GA blocker. That is F4's purpose, but late discovery is high blast radius. Mitigation: run F4 early enough to remediate; B22 fed forward into components already. Blast radius: high.
- **R2 (high):** F4 depends on B22 being complete and authoritative; if B22's targets are thin, F4's review under-covers. Mitigation: F4 treats B22 as the contract and flags any B22 target it cannot test as a B22 gap. Blast radius: high.
- **R3 (med):** Adversarial probes could damage shared infra or leak data if run against a live cluster. Mitigation: throwaway cluster + redacted evidence (NFR). Blast radius: med.
- **R4 (med):** Manual bundle review may miss cross-layer conflicts (automated analysis is deferred, future §13). Mitigation: explicitly note the limitation; recommend future-§13 tooling. Blast radius: med.
- **OQ1:** Does v1.0 require an external pen test executed, or only scoped? `[PROPOSED]` scope only in v1.0 (execution is an engagement decision); flag for security owner.
- **OQ2:** Per-chokepoint fail-open vs fail-closed (future §1) is unresolved — should F4 assert OPA-down behavior? `[PROPOSED]` record observed OPA-down behavior as a finding; the policy decision is deferred.

## 11. References
- architecture-overview.md §6.6 (threat model v1.0 stance, defense-in-depth, audit + OPA hook points, emission points lines 515–525), §14.2 B22 line 1716 (threat model design ships before first wave), §14.6 line 1760 (F4 scope), §8 (security-first CI), §6.7 (AlertManager → HolmesGPT detection flow).
- future-enhancements.md §2 (continuous container security deferred), §13 (cross-layer policy-conflict / drift deferred), §1 (per-chokepoint fail-open/closed deferred).
- Canon interface-contract §1.3 (`Sandbox`/`SandboxTemplate` runtime), §1.4 (`VirtualKey`/capability CRDs/`BudgetPolicy`), §2 (`platform.security.*`, `platform.policy.*`), §4 (connection secret), §5 (audit integrity). Canon glossary (Platform JWT claim schema).
- ADR 0027 (threat-model scope + B22 prerequisite), ADR 0002/0018 (OPA + RBAC-floor/OPA-restrictor), ADR 0003 (Envoy egress), ADR 0013 (capability model), ADR 0034 (audit durability/integrity), ADR 0029 (JWT claim schema), ADR 0033 (AWS/GitHub targets).
- Related pieces: B22, B16, B3, A6, A7, B13, A18, A17, B1, F5, F6.
