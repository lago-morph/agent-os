# SPEC A6 — agent-sandbox + Envoy egress proxy

> kind: COMPONENT · workstream: A · tier: T0
> upstream: [] · downstream: [A5, A20, B16, B7, B20] · adrs: [0003, 0002, 0018, 0034, 0030, 0032, 0035, 0013] · views: [6.2, 6.6]
> canon-glossary: 6aadcc2a4f383a26 · canon-interface: 54f5ede58e5f8c05

## 1. Purpose & Problem Statement

A6 delivers two coupled perimeter controls for Platform Agents: the **agent-sandbox** runtime operator (reconciling `Sandbox` and `SandboxTemplate`, running agent pods under gVisor/Kata kernel isolation, with a warm-pool model) and the **Envoy egress proxy** — the single chokepoint for **all non-LiteLLM agent outbound HTTP** (ADR 0003). LiteLLM already governs LLM/MCP/A2A traffic; Envoy governs everything else, enforcing the FQDN allowlist derived from each agent's `EgressTarget` capabilities, calling OPA for runtime decisions, and emitting audit on every connection.

The problem A6 solves is **kernel-level isolation + the network perimeter** in the defense-in-depth model (§6.6): per-agent capabilities are enforced declaratively (Agent CRD + CapabilitySet), at the gateway (LiteLLM), at the egress proxy (Envoy FQDN allowlist + OPA), and at the kernel (gVisor/Kata). A6 owns the last two of those four layers. The architectural invariant it upholds: **every Platform Agent outbound call traverses LiteLLM or the Envoy egress proxy — there are no other paths to external services** (ADR 0003).

A6 is **T0, contract-owning** — ARK (A5) runs agents into A6's Sandboxes, the policy simulator (A20) consumes Envoy's egress dry-run layer, the agent SDK (B7) and PV-access system (B20) build against the sandbox runtime, and the OPA library (B16) targets the egress decision point.

## 2. Scope

### 2.1 In scope

- **agent-sandbox** operator install (reconciles `Sandbox`, `SandboxTemplate`).
- Sandbox runtime: **gVisor or Kata** kernel isolation; warm-pod pool; hibernation.
- Skill init-container → skill-volume mount path into the agent pod (skills served via LiteLLM's skill gateway; A6 provides the mount mechanics).
- **Envoy egress proxy** install via Helm, configured **per agent class**.
- FQDN allowlist enforcement derived from each agent's `EgressTarget` capabilities (composed via CapabilitySet, ADR 0032).
- OPA runtime decision call at the egress hook; mTLS support.
- **Audit emission on every egress connection** + Sandbox lifecycle audit (create/destroy/hibernate) via the audit adapter (ADR 0034).
- Envoy egress **dry-run** decision surface for the policy simulator (A20/ADR 0038).
- Runtime log-level control via `LogLevel` (ADR 0035).
- The §14.1 standard deliverable set.

### 2.2 Out of scope (and where it lives instead)

- **`EgressTarget` / `CapabilitySet` reconciliation into Envoy config** — **B13** kopf operator reconciles `EgressTarget` and emits the config A6's Envoy consumes; A6 owns the proxy that enforces it. (§6.8: `OP --> EP`.)
- **OPA engine + Rego egress policy content** — **A7** (engine), **B16** (egress Rego).
- **Agent operator / `Agent` lifecycle** — **A5** (ARK); A6 runs the pods ARK schedules.
- **Agent SDK / harness images** — **B7**; A6 provides the runtime contract harnesses comply with.
- **Persistent-volume mapping into agents** — **B20**; A6 provides the sandbox the PVs mount into.
- **LiteLLM-bound traffic** (LLM/MCP/A2A) — **A1**; never traverses Envoy.
- **Audit endpoint / adapter library** — **A18** (A6 links the adapter).
- **CapabilitySet overlay resolution semantics** — ADR 0032 + B13.
- **OpenAPI→MCP / synthetic MCP** — A12.

## 3. Context & Dependencies

**Upstream consumed:** None (W0 foundation). agent-sandbox + Envoy are OSS installs on the k8-platform baseline.

**Downstream consumers (what they consume):**
- **A5 (ARK)** — reconciles `Agent` pods into A6's `Sandbox`es; consumes `SandboxTemplate`/`Sandbox` (`sandboxTemplateRef` on `Agent`).
- **A20 (policy simulator)** — consumes Envoy egress structured dry-run decisions.
- **B16** — egress Rego targets A6's OPA hook.
- **B7 (agent SDK)** — harness images run inside A6 sandboxes and egress only through Envoy.
- **B20 (PV access)** — mounts PVs into A6 sandboxes under RBAC+OPA.

**ADR decisions honored:**
- **ADR 0003** — Envoy is the single egress chokepoint for non-LiteLLM outbound HTTP; FQDN allowlist from `EgressTarget`; OPA at the egress hook; audit on every connection; CNI-agnostic (runs on EKS/AKS/kind); config driven by CRD, not hand-edited. Defense-in-depth: even if gateway authz fails, Envoy blocks cross-tenant A2A.
- **ADR 0002 / 0018** — OPA at egress is a **restrictor** over an RBAC-bounded identity (RBAC-as-floor / OPA-as-restrictor); never grants.
- **ADR 0034** — Sandbox lifecycle + every egress connection audit via the platform audit adapter (the §6.6 hook points "agent-sandbox controller" and "Envoy egress proxy"); no direct store writes.
- **ADR 0032** — per-Agent `EgressTarget` overlays compose via CapabilitySet layering semantics.
- **ADR 0035** — Envoy log level controllable at runtime via `LogLevel` / dynamic toggle.
- **ADR 0013** — `EgressTarget` is a capability CRD; capability changes emit `platform.capability.changed` (emitted by B13).
- **ADR 0030** — `Sandbox`/`SandboxTemplate` versioning owned by **agent-sandbox install (A6)** per §6.13.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)

A6 **owns the reconciler** for the agent-sandbox CRDs (§6.12; owner = agent-sandbox install = A6; versioning owned by A6 per §6.13):

| CRD | Scope | Key fields (source-stated) |
|---|---|---|
| `Sandbox` | namespaced | `templateRef`, `runtime` (gVisor/Kata), `state` |
| `SandboxTemplate` | namespaced | `runtime`, `warmPoolSize`, `hibernationEnabled`, `resourceLimits` |

A6 **consumes** (does not own) — reconciled by B13 into Envoy config:

| CRD | Scope | Fields A6 relies on |
|---|---|---|
| `EgressTarget` | namespaced | `fqdn`, `port`, `scheme`, `allowedMethods` |
| `CapabilitySet` | namespaced | `egressTargets[]` (resolved per ADR 0032) |
| `Agent` (ARK, A5) | namespaced | `sandboxTemplateRef` |

### 4.2 APIs / SDK surfaces

- **Sandbox runtime contract** — the interface harness images (B7) and the SDK comply with: pod runs under gVisor/Kata; skill volume mounted via init container; outbound HTTP only via Envoy; LLM/MCP/A2A via LiteLLM. (§6.2 "Platform-SDK interaction surfaces".)
- **Envoy egress proxy** — per-agent-class L7 proxy enforcing the FQDN allowlist; OPA hook; mTLS. No bespoke platform HTTP API.
- **Envoy egress dry-run surface** — structured dry-run input emitting the same decision shape with `simulated: true`, no enforcement side effect, for A20 (§6.6, ADR 0038). `[PROPOSED — not in source]` exact dry-run field names — reconcile with A20.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)

Emitted (per-event-type names deferred to B12):
- `platform.audit.*` — every egress connection; Sandbox create/destroy/hibernate.
- `platform.security.*` — **sandbox-escape signal** (the §6.7 example), repeated egress-policy-bypass attempts.
- `platform.policy.*` — egress OPA decisions / policy violations (decision authored by OPA; A6 is the egress emit point).
- `platform.lifecycle.*` — Sandbox lifecycle (created/started/paused/resumed/deleted) — note Sandbox lifecycle is owned by agent-sandbox, **not** ARK (§6.6). `[PROPOSED — not in source]` whether sandbox lifecycle events go under `lifecycle.*` (the namespace covers "Sandbox … lifecycle") vs `audit.*` — both apply per their definitions; B12 fixes per-type names.

Consumed: `platform.capability.changed` indirectly (Envoy config is updated by B13 reconcile, not by A6 subscribing).

Every event carries `specversion` + `schemaVersion` (ADR 0031/0030).

### 4.4 Data schemas / connection-secret contracts

- A6 holds no system-of-record state. Sandbox state (`state` field) is reconciler-managed; warm-pool state is operational.
- Envoy mTLS material / per-agent-class config arrives from B13 reconcile of `EgressTarget` + CapabilitySet; A6 does not mint it. `[PROPOSED — not in source]` exact Envoy config secret shape — not specified; treat as B13-produced config.
- No connection-secret XRD (A6 is not substrate-asymmetric storage).

## 5. OSS-vs-Custom Decision

- **agent-sandbox:** OSS sandbox runtime operator. **Decision: config** — install and operate; reconciles `Sandbox`/`SandboxTemplate`. No fork.
- **Envoy:** OSS proxy. **Decision: config + wrap** — install via Helm, configure per agent class from `EgressTarget`-derived config (produced by B13), add the OPA hook + audit emission + mTLS. No fork.
- **ADR 0003 linkage:** Envoy over Cilium L7 NetworkPolicy because Cilium isn't available across EKS/AKS/kind — Envoy is CNI-agnostic with first-class audit + OPA hooks at egress.
- Custom code = the OPA-call + audit-emit egress filter config and the skill-mount mechanics; reconciliation lives in B13.

## 6. Functional Requirements

- **REQ-A6-01:** A6 SHALL install the agent-sandbox operator and reconcile `Sandbox` and `SandboxTemplate` CRDs.
- **REQ-A6-02:** Agent pods SHALL run under **gVisor or Kata** kernel isolation as declared by `Sandbox.runtime` / `SandboxTemplate.runtime`.
- **REQ-A6-03:** A6 SHALL support a warm-pod pool (`SandboxTemplate.warmPoolSize`) and hibernation (`hibernationEnabled`), honoring `resourceLimits`.
- **REQ-A6-04:** Skills SHALL be delivered into the agent pod via a skill init-container → skill-volume mount (served via LiteLLM's skill gateway).
- **REQ-A6-05:** A6 SHALL deploy the Envoy egress proxy via Helm as the single chokepoint for all non-LiteLLM agent outbound HTTP; no other egress path to external services SHALL exist for Platform Agents.
- **REQ-A6-06:** Envoy SHALL enforce a per-agent-class FQDN allowlist derived from the agent's resolved `EgressTarget` capabilities (`fqdn`, `port`, `scheme`, `allowedMethods`).
- **REQ-A6-07:** On each outbound connection Envoy SHALL consult OPA for a runtime decision and SHALL act as a restrictor only (never grant beyond RBAC; ADR 0018).
- **REQ-A6-08:** Envoy SHALL emit a structured audit event on **every** outbound connection via the platform audit adapter (ADR 0034); A6 SHALL NOT write audit directly to any store.
- **REQ-A6-09:** The agent-sandbox controller SHALL emit audit on Sandbox creation, destruction, and hibernation (Sandbox lifecycle owned by A6, not ARK).
- **REQ-A6-10:** Envoy egress config SHALL be driven declaratively from `EgressTarget` + CapabilitySet (reconciled by B13); A6 SHALL NOT require hand-edited Envoy config to add a destination.
- **REQ-A6-11:** The Envoy egress hook SHALL honor a structured **dry-run** input emitting the same decision shape with `simulated: true` and no enforcement side effects (A20/ADR 0038).
- **REQ-A6-12:** Envoy log level SHALL be controllable at runtime via `LogLevel` / the dynamic toggle (ADR 0035).
- **REQ-A6-13:** A6 SHALL provide CNI-agnostic egress enforcement identical on EKS, AKS, and kind (ADR 0003).
- **REQ-A6-14:** On detection of sandbox-escape or repeated egress-policy-bypass, A6 SHALL emit a `platform.security.*` event.

## 7. Non-Functional Requirements

- **Security / multi-tenancy (§6.9):** kernel isolation (gVisor/Kata) per pod; Envoy blocks cross-tenant A2A at the network layer even if gateway authz fails (defense-in-depth). RBAC-as-floor / OPA-as-restrictor at the egress hook (ADR 0018). Secrets via ESO.
- **Portability (ADR 0003/0026):** CNI-agnostic; identical enforcement across EKS/AKS/kind; independent-cluster model.
- **Performance:** warm-pod pool minimizes cold-start; hibernation conserves resources; Envoy adds bounded L7 latency on egress.
- **Observability (§6.5):** OTel from Envoy + sandbox controller; per-component Grafana dashboard via `GrafanaDashboard` XR; egress-connection + sandbox-lifecycle metrics.
- **Versioning (ADR 0030):** `Sandbox`/`SandboxTemplate` follow `v1alpha1`→`v1beta1`→`v1`, owned by A6; conversion webhooks + ≥1-release deprecation on breaking changes.

## 8. Cross-Cutting Deliverable Checklist (§14.1)

| Deliverable | Status |
|---|---|
| Helm values / manifests in Git | applicable — agent-sandbox + Envoy charts |
| Per-product docs (10.5) | applicable |
| Operator runbook (10.7) | applicable — sandbox runtime triage, egress-deny diagnosis, warm-pool sizing |
| Backup / restore | N/A — A6 holds no system-of-record state (sandboxes are ephemeral; config is in Git) |
| Alert rules | applicable — sandbox controller down, Envoy down, egress-deny spike, sandbox-escape signal, warm-pool exhaustion |
| Grafana dashboard (`GrafanaDashboard` XR) | applicable |
| Headlamp plugin | applicable — sandbox + egress-policy visibility (A6 ensures integration; EgressTarget editor is A22/ADR 0039) |
| OPA/Rego integration | applicable — egress runtime decisions + sandbox admission (Rego in B16) |
| Audit emission (ADR 0034) | applicable — every egress connection + sandbox lifecycle |
| Knative trigger flow | applicable — sandbox-escape `platform.security.*` flow; egress-deny observability flow |
| HolmesGPT toolset | applicable — egress-deny query, sandbox-state query, escape-signal investigation tools |
| 3-layer tests | applicable — Chainsaw (Sandbox/SandboxTemplate), PyTest (egress allowlist/OPA/dry-run), Playwright (Headlamp view) |
| Tutorials & how-tos | applicable — "add an egress destination", "choose a sandbox runtime/template" |

## 9. Acceptance Criteria

- **AC-A6-01** (REQ-A6-01): Creating a `SandboxTemplate` then a `Sandbox` yields a Ready sandbox reconciled by the operator. *(Chainsaw)*
- **AC-A6-02** (REQ-A6-02): A pod in a `Sandbox` with `runtime: gvisor` runs under the gVisor runtime class; same for Kata. *(Chainsaw)*
- **AC-A6-03** (REQ-A6-03): With `warmPoolSize: N`, N warm pods are pre-provisioned; a hibernation-enabled sandbox hibernates and resumes within `resourceLimits`. *(Chainsaw)*
- **AC-A6-04** (REQ-A6-04): A skill declared in the CapabilitySet is present on the skill volume inside the agent pod via the init container. *(Chainsaw + PyTest)*
- **AC-A6-05** (REQ-A6-05): An agent pod has no route to an external host except through Envoy (and LiteLLM for LLM/MCP/A2A); a direct outbound attempt is blocked. *(PyTest)*
- **AC-A6-06** (REQ-A6-06): A request to a `fqdn` in the agent's resolved `EgressTarget`s succeeds; one to a non-allowlisted FQDN is denied. *(PyTest)*
- **AC-A6-07** (REQ-A6-07): With an OPA egress-deny policy, an otherwise-allowlisted destination is denied; OPA never permits a destination RBAC would forbid. *(PyTest)*
- **AC-A6-08** (REQ-A6-08): Each egress connection produces an audit record at the audit endpoint via the adapter; no direct store write from A6. *(PyTest)*
- **AC-A6-09** (REQ-A6-09): Sandbox create/destroy/hibernate each emit an audit event. *(PyTest)*
- **AC-A6-10** (REQ-A6-10): Adding an `EgressTarget` CRD (reconciled by B13) makes the destination reachable with no hand-edit of Envoy config. *(Chainsaw)*
- **AC-A6-11** (REQ-A6-11): A dry-run egress decision returns `simulated: true` with no real allow/deny side effect and no enforcement-path audit (only the simulator-run `platform.policy.*` record). *(PyTest)*
- **AC-A6-12** (REQ-A6-12): Setting a `LogLevel` raises Envoy verbosity at runtime without redeploying callers. *(Chainsaw)*
- **AC-A6-13** (REQ-A6-13): The same egress enforcement test passes on kind and on a managed-K8s target with no CNI dependency. *(Chainsaw, two substrates)*
- **AC-A6-14** (REQ-A6-14): A simulated sandbox-escape / repeated bypass emits a `platform.security.*` event. *(PyTest)*

## 10. Risks & Open Questions

- **R1 (high):** The "no other egress path" invariant (ADR 0003) is only as strong as the pod network policy preventing direct egress; if a sandbox can reach the network without traversing Envoy, the perimeter fails. *Reconciliation:* AC-A6-05 must prove no bypass route on each substrate. Blast radius high (exfiltration).
- **R2 (med):** gVisor/Kata availability differs across EKS/AKS/kind; a runtime declared but unavailable on a substrate must fail clearly. *Open question* — is there a substrate-capability admission check? Not specified → `[PROPOSED — not in source]`.
- **R3 (med):** Envoy egress dry-run decision-shape field names are `[PROPOSED — not in source]`; reconcile with A20/ADR 0038 before either contract freezes.
- **R4 (med):** Ownership boundary — A6 enforces, B13 reconciles `EgressTarget`→Envoy config. A stale/failed reconcile yields an out-of-date allowlist. *Reconciliation:* A6 documents the config-source contract; alert on reconcile lag.
- **R5 (low):** Whether sandbox lifecycle events go under `platform.lifecycle.*` or `platform.audit.*` is `[PROPOSED]` (both namespaces' definitions apply); B12 fixes per-type names.

## 11. References

- architecture-overview.md §6.2 Agent runtime architecture (line ~247), §6.6 Security and policy architecture (~436, defense-in-depth, audit/OPA hook points, policy simulator), §6.9 (~720, multi-tenancy — cross-tenant A2A egress block), §6.5 (~370, observability), §6.13 (~981, versioning).
- ADR 0003 (Envoy egress proxy), ADR 0002 (OPA/Gatekeeper), ADR 0018 (RBAC-floor/OPA-restrictor), ADR 0034 (audit pipeline), ADR 0032 (CapabilitySet overlay semantics), ADR 0035 (dynamic log/trace toggle), ADR 0013 (capability CRDs / `platform.capability.changed`), ADR 0030 (versioning), ADR 0026 (independent-cluster install).
- Related pieces: A5 (ARK), A1 (LiteLLM), B13 (kopf operator — `EgressTarget` reconcile), A7 (OPA), A20 (policy simulator), B7 (agent SDK), B20 (PV access), A18 (audit), A22 (EgressTarget editor).
