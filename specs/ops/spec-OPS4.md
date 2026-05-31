# SPEC OPS4 — Multi-Tenant Compute Isolation

> kind: COMPONENT (cross-cutting NFR layer) · workstream: — (operational/NFR) · tier: T1
> upstream: [A6, A7, ADR-0016, A21, B20, B16] · downstream: [] · adrs: [0016, 0018, 0002, 0003, 0028, 0029, 0037, 0034, 0030, 0025, 0026, 0041] · views: [6.9, 6.2, 6.6]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

OPS4 is the cross-cutting **compute-isolation** layer of the operational/NFR architecture. The functional architecture fixes *who is a tenant* (a Kubernetes namespace, ADR 0016) and *what enforces a policy decision* (RBAC floor + Gatekeeper admission + OPA at LiteLLM + Envoy egress, ADR 0016/0018/0003). What it does **not** state as a coherent layer is the **workload-isolation** posture *between tenant agent runtimes*: how two tenants' `Sandbox` pods are kept apart on the **compute substrate** — namespace/node/runtime-class boundaries, network policy at the pod layer, noisy-neighbor and resource-quota interplay, and the blast radius of a sandbox escape. Tenant-scoped quotas are explicitly deferred (architecture-backlog §1.2; ADR 0037; ADR-0016 OQ-1) and the sandbox runtime depth (gVisor/Kata) is a per-`SandboxTemplate` field with no platform-wide selection policy. This is a missing *layer*, not a component: it constrains A6 (the sandbox + egress runtime), A7/B16 (admission + OPA), A21 (tenant provisioning), and B20 (PV mounts), and is realized by them.

The problem OPS4 solves is to state, as testable invariants, the **data-plane isolation guarantees** the platform makes between tenants — and, just as importantly, to be explicit about the **threat model**: what is guaranteed, what is best-effort, and what is *not* guaranteed at the v1.0 isolation tier. It does **not** re-decide ADR 0016 (namespace = tenant); it specifies the compute-isolation obligations that honoring it imposes and surfaces the genuinely-open decisions (soft vs hard multi-tenancy; sandbox sandbox-runtime selection policy) as proposed ADRs rather than deciding them.

## 2. Scope

### 2.1 In scope

- **Isolation-boundary model**: the ordered set of boundaries that separate two tenants' agent runtimes — namespace (ADR 0016), Kubernetes `NetworkPolicy` at the pod layer, node/node-pool placement, `RuntimeClass` (the gVisor/Kata kernel boundary realized through `Sandbox.runtime`).
- **Threat model & guarantee tiers**: explicit statement of the adversary (a compromised or adversarial Platform Agent inside a `Sandbox`), and per-boundary what is **guaranteed**, **best-effort**, or **out-of-scope** at v1.0.
- **Pod-layer network isolation**: a default-deny posture between tenant namespaces at the network layer, complementing (not replacing) the Envoy egress chokepoint (ADR 0003) and OPA-at-LiteLLM (ADR 0016). Envoy governs *egress to external FQDNs*; OPS4 governs *intra-cluster pod-to-pod* reachability across tenants.
- **Noisy-neighbor / resource-quota interplay**: the obligation that one tenant's resource consumption (CPU/memory/sandbox count/warm-pool size) cannot starve another, expressed against `SandboxTemplate.resourceLimits`, the `AgentEnvironment` XR `quotas` field, and Kubernetes `ResourceQuota`/`LimitRange`.
- **Escape-blast-radius bounding**: the requirement that a sandbox escape (the `platform.security.*` sandbox-escape signal A6 already emits) is contained to a stated radius, and the detection/signal obligation on the runtime.
- **Conformance posture**: which boundary is enforced by which K8s primitive / OPA policy / network policy, stated so the PLAN can map each to a test layer.
- The cross-cutting obligations OPS4 pushes **back into** existing specs (A6, A21, B16) — see §6 NFR-absorption notes.

### 2.2 Out of scope (and where it lives instead — name the owning piece)

- **The namespace-as-tenant decision itself** — ADR 0016 / spec-ADR-0016 (settled; OPS4 honors, does not re-argue).
- **The RBAC-floor / OPA-restrictor composition rule** — ADR 0018; OPS4 consumes it, does not redefine.
- **Egress to external FQDNs + the OPA egress hook** — A6 (Envoy egress proxy, ADR 0003). OPS4 covers *intra-cluster* pod reachability; egress is A6's.
- **OPA engine + Rego content** — A7 (engine), B16/B3 (Rego). OPS4 names the policy *targets*; the Rego is authored in B16.
- **Tenant onboarding flow + the `TenantOnboarding` XRD** — A21 / ADR 0037. OPS4 states which isolation primitives onboarding must provision; A21 owns the composition.
- **Identity / JWT claim schema** — ADR 0028 / 0029. OPS4 consumes the tenancy boundary, not the identity chain.
- **PV-mount governance** — B20. OPS4 states the isolation invariant a mounted volume must not violate; B20 owns the mount mechanism.
- **Per-tenant cost attribution & autoscaling/capacity** — OPS1 (scale & cost). OPS4 covers *isolation* of resource consumption (noisy-neighbor), not the cost model.
- **Memory-store access modes** (`private`/`namespace-shared`/RBAC-OPA) — ADR 0025 / B4. A distinct data-isolation mechanism; OPS4 references it as an adjacent boundary, does not own it.
- **Multi-cluster / cross-cluster isolation** — out of scope per ADR 0026 (independent-cluster model). v1.0 isolation is *within one cluster*.
- **The concrete tenant-quota field set** — deferred (architecture-backlog §1.2; ADR 0037). OPS4 states the invariant; the field schema is `[PROPOSED — not in source]` and surfaced as a proposed ADR (§10).

## 3. Context & Dependencies

**Upstream pieces consumed (and exactly what is consumed):**

- **A6 (agent-sandbox + Envoy egress proxy)** — the `Sandbox`/`SandboxTemplate` CRDs (`runtime` = gVisor/Kata, `warmPoolSize`, `hibernationEnabled`, `resourceLimits`), the kernel-isolation runtime, and the `platform.security.*` sandbox-escape signal. OPS4 binds its guarantees to these fields.
- **A7 (OPA / Gatekeeper)** — the admission + runtime decision engine that OPS4's isolation admission policies target.
- **ADR-0016 (namespace multi-tenancy)** — the four-layer enforcement model and the namespace-as-boundary invariant OPS4 extends to the compute/data plane.
- **A21 (tenant onboarding reconciler)** — provisions the namespace(s), tenancy-metadata labels, and default ServiceAccounts OPS4's isolation primitives attach to.
- **B20 (PV access)** — the PV-mount mechanism whose isolation invariant OPS4 states.
- **B16 (OPA policy library content)** — where the isolation admission/runtime Rego is authored.

**Downstream consumers:** none as a build piece (this is a constraint layer). At runtime every tenant `Sandbox` is governed by OPS4's invariants; the obligations land in A6/A21/B16 as NFR-absorption notes (§6).

**ADR decisions honored (constraint each imposes):**

- **ADR 0016** — namespace = tenancy boundary; isolation enforced across independent layers; isolation must remain even if one layer is misconfigured (the independence invariant).
- **ADR 0018** — RBAC-as-floor / OPA-as-restrictor; OPS4's OPA isolation policies may only *restrict*, never grant cross-tenant reach.
- **ADR 0002** — Gatekeeper admission + OPA runtime authorization; OPS4 isolation admission is a Gatekeeper policy.
- **ADR 0003** — Envoy is the single *egress* chokepoint; OPS4's pod-network policy is the *intra-cluster* complement, independent of egress.
- **ADR 0026** — independent-cluster model; isolation guarantees are scoped to a single cluster, no cross-cluster federation.
- **ADR 0030 / 0041** — any CRD/XRD field OPS4 implies (e.g. a quota field set) is versioned by its owning component with conversion webhooks; OPS4 introduces no new owner.
- **ADR 0034** — isolation-relevant security/audit events flow through the audit adapter and the closed CloudEvent taxonomy; OPS4 mints no new namespace.
- **ADR 0025** — `MemoryStore` access modes are the data-isolation mechanism for memory; OPS4 does not overload them.

## 4. Interfaces & Contracts

Names below are Canon (`_meta/interface-contract.md` / `_meta/glossary.md`); anything else is tagged `[PROPOSED — not in source]`.

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)

OPS4 **owns no CRD**. It binds invariants to existing Canon kinds:

| Kind (owner) | Fields OPS4 relies on | Isolation role |
|---|---|---|
| `Sandbox` (A6) | `runtime` (gVisor/Kata), `state` | Kernel-boundary (`RuntimeClass`) per pod. |
| `SandboxTemplate` (A6) | `runtime`, `warmPoolSize`, `hibernationEnabled`, `resourceLimits` | Kernel class + resource ceiling per template (noisy-neighbor input). |
| `AgentEnvironment` (XR, B4) | `region`, `quotas`, `defaultCapabilitySetRef` | The Canon-stated `quotas` field — the per-environment quota carrier OPS4 anchors tenant-quota interplay to. |
| `TenantOnboarding` (XRD, A21/B4) | `tenantId`, `namespaces[]`, `defaultServiceAccounts[]`, `clusterOIDCClaimMapping` | Provisions the namespace boundary + labels the isolation primitives attach to. |

Kubernetes built-ins OPS4 requires (not platform CRDs): `NetworkPolicy`, `ResourceQuota`, `LimitRange`, `RuntimeClass`, node labels/taints/tolerations, `PriorityClass`. These are native primitives, not Canon CRDs.

- **`[PROPOSED — not in source]`** — A **tenant-quota field set** on `TenantOnboarding` and/or `AgentEnvironment.quotas` (max agents, max sandboxes, CPU/memory ceiling, warm-pool cap, virtual-key count). The Canon names the carrier (`AgentEnvironment.quotas`) but **not** the field list; deferred per architecture-backlog §1.2 / ADR 0037. Surfaced as PROPOSED-ADR-A (§10).
- **`[PROPOSED — not in source]`** — any namespace-level `NetworkPolicy` default-deny template, node-pool/taint scheme, or `RuntimeClass` selection policy. The substrate primitives exist; the *platform-wide policy over them* is not in source.

### 4.2 APIs / SDK surfaces

N/A — OPS4 introduces no API/SDK surface. It is a constraint + policy-target layer realized through Kubernetes primitives, Gatekeeper/OPA admission, and existing component reconcilers.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)

OPS4 mints **no new namespace** (the ten-namespace set is closed). It binds to existing ones:

- **`platform.security.*`** — sandbox-escape signal (already A6-emitted), cross-tenant-reachability-attempt detection, repeated isolation-policy-bypass. OPS4 states the *signal obligation*; per-event-type names deferred to B12.
- **`platform.policy.*`** — OPA/Gatekeeper isolation admission decisions (deny of a cross-tenant `NetworkPolicy`, a misplaced node selector, a quota-violating sandbox).
- **`platform.tenant.*`** — namespace-association changes that alter the isolation topology (consumed from A21).
- **`platform.audit.*`** — every isolation-relevant admission decision audited via the adapter (ADR 0034).
- **`platform.observability.*`** — noisy-neighbor threshold crossings (a tenant approaching its `ResourceQuota`/`AgentEnvironment.quotas` ceiling).

Every event carries `specversion` + `schemaVersion` (ADR 0031/0030).

### 4.4 Data schemas / connection-secret contracts

N/A — OPS4 provisions no substrate primitive and writes no connection secret (the ADR 0041 connection-secret contract applies only to substrate XRDs). Isolation state lives in native K8s objects (`NetworkPolicy`, `ResourceQuota`, node labels) reconciled from Git like every other declaration.

## 5. OSS-vs-Custom Decision

**Compose + config; no new build.** OPS4 is realized entirely over upstream primitives — Kubernetes `NetworkPolicy`/`ResourceQuota`/`LimitRange`/`RuntimeClass`/node-pools (the k8-platform baseline), agent-sandbox's gVisor/Kata runtimes (A6), and OPA/Gatekeeper (A7) for admission conformance. No fork; no new controller binary. The only *new* artifacts are declarative isolation manifests + Gatekeeper/OPA Rego (authored in B16) and the conformance tests. The runtime-sandbox depth (gVisor vs Kata vs runc) and soft-vs-hard tenancy stance are **decisions deferred to proposed ADRs** (§10), not made here.

## 6. Functional Requirements

Numbered, testable. Several carry an **NFR-absorption note** — the existing spec that must absorb the obligation.

- **REQ-OPS4-01:** OPS4 SHALL define the tenant compute-isolation boundary as an **ordered, independent set**: (1) namespace (ADR 0016), (2) pod-layer `NetworkPolicy`, (3) node/node-pool placement, (4) `RuntimeClass` kernel boundary (gVisor/Kata via `Sandbox.runtime`). Each boundary SHALL be independent such that a misconfiguration in one is still bounded by the others (the ADR 0016 independence invariant extended to the compute plane).
- **REQ-OPS4-02:** Each tenant namespace SHALL carry a **default-deny** pod-layer `NetworkPolicy`: a pod in tenant A's namespace SHALL NOT reach a pod in tenant B's namespace except via an explicitly-declared, OPA-checked allow (the §6.9 cross-tenant publication path). *NFR-absorption: ADR-0016 §6 layer (4) "network policy" is realized at the Envoy egress layer for external traffic; OPS4 adds the intra-cluster pod-to-pod policy A6 does not own — A6's spec §7 should note this complement.*
- **REQ-OPS4-03:** The intra-cluster network policy of REQ-OPS4-02 SHALL be **independent of** the Envoy egress chokepoint (ADR 0003): a gateway/Envoy bypass SHALL NOT confer cross-tenant pod reachability, and vice-versa.
- **REQ-OPS4-04:** Every tenant agent runtime SHALL run under a `RuntimeClass`-backed kernel boundary (gVisor or Kata) as declared by `Sandbox.runtime` / `SandboxTemplate.runtime`; a `Sandbox` SHALL NOT run a tenant workload under the host `runc` runtime unless an explicit, audited, policy-gated exception is declared. *NFR-absorption: the platform-wide rule "which runtime is the default / minimum for a tenant sandbox" is not in A6; surfaced as PROPOSED-ADR-B.*
- **REQ-OPS4-05:** A `Sandbox` declaring a `runtime` unavailable on the cluster substrate (gVisor/Kata availability differs across EKS/AKS/kind, A6 R2) SHALL be **rejected at admission** with a clear error, NOT silently downgraded to a weaker runtime. *NFR-absorption: resolves A6 R2 (substrate-capability admission check) — A6's spec should adopt this as a Gatekeeper check; Rego in B16.*
- **REQ-OPS4-06:** Each tenant SHALL be subject to a **resource ceiling** — a Kubernetes `ResourceQuota` + `LimitRange` per tenant namespace, bounded by the `AgentEnvironment.quotas` field where present — such that one tenant's CPU/memory/sandbox-count/warm-pool consumption CANNOT prevent another tenant from scheduling its declared workloads (noisy-neighbor containment). The concrete quota field set is `[PROPOSED — not in source]` (PROPOSED-ADR-A).
- **REQ-OPS4-07:** A request to create a `Sandbox` (or raise `warmPoolSize`/`resourceLimits`) that would exceed the tenant's quota SHALL be denied at admission and SHALL emit a `platform.policy.*` decision and `platform.observability.*` threshold event; it SHALL NOT degrade an unrelated tenant's running workloads.
- **REQ-OPS4-08:** OPS4 SHALL state a **threat model** (§7) and, for each isolation boundary, classify the guarantee as **guaranteed**, **best-effort**, or **out-of-scope** at v1.0; the platform documentation SHALL NOT claim a stronger guarantee than the boundary's classification.
- **REQ-OPS4-09:** A **sandbox escape** SHALL be contained to a stated blast radius: at minimum the escaping workload SHALL NOT obtain cross-tenant pod reachability (REQ-OPS4-02) or cross-tenant node co-tenancy beyond the declared node-pool boundary, and the `platform.security.*` sandbox-escape signal (A6 REQ-A6-14) SHALL fire. The **node-sharing** blast radius (whether two tenants may share a node) is governed by the soft-vs-hard tenancy decision (PROPOSED-ADR-C).
- **REQ-OPS4-10:** Node-pool / taint placement, where used to separate tenants or tiers, SHALL be enforced by admission (Gatekeeper) so a tenant SHALL NOT schedule onto a node-pool it is not entitled to (e.g. by setting a rogue `nodeSelector`/`toleration`). The node-pool entitlement scheme is `[PROPOSED — not in source]` (PROPOSED-ADR-C).
- **REQ-OPS4-11:** A PV mounted into a tenant `Sandbox` (B20) SHALL NOT cross the tenancy boundary except via the §6.9 OPA-checked publication path; OPS4's isolation invariant SHALL be a conformance target B20's admission policy satisfies. *NFR-absorption: B20 already states this (REQ-B20-02); OPS4 names it as a cross-cutting invariant for conformance.*
- **REQ-OPS4-12:** Every isolation-relevant admission decision (network-policy, runtime-class, node-pool, quota) SHALL be audited via the platform audit adapter (ADR 0034) and SHALL NOT write to any store directly.
- **REQ-OPS4-13:** All isolation primitives (network policies, quotas, runtime-class bindings, node-pool labels) SHALL reconcile from Git (GitOps), provisioned at tenant onboarding (A21) — no out-of-band `kubectl` path SHALL be the sanctioned way to establish a tenant's isolation. *NFR-absorption: A21's composition (REQ-A21-02) labels the namespace; OPS4 requires it also provision the default-deny NetworkPolicy + ResourceQuota baseline.*
- **REQ-OPS4-14:** OPS4 SHALL be **single-cluster** in scope (ADR 0026): it SHALL make no cross-cluster isolation claim; multi-cluster federation is explicitly out of guarantee.

## 7. Non-Functional Requirements

**Threat model (the load-bearing NFR).** Adversary = a **compromised or adversarial Platform Agent executing inside a tenant `Sandbox`** (e.g. via prompt injection or a malicious skill). Not in the v1.0 adversary model: a malicious cluster administrator, a compromised control plane, a hypervisor/kernel 0-day defeating gVisor/Kata, or side-channel/timing attacks. Per-boundary guarantee classification:

| Boundary | Primitive | Guarantee tier (v1.0) | Note |
|---|---|---|---|
| Namespace (RBAC/Gatekeeper/OPA scope) | K8s namespace + ADR 0016 four layers | **Guaranteed** | The settled tenancy boundary. |
| Intra-cluster pod reachability | `NetworkPolicy` default-deny | **Guaranteed** *iff* the CNI enforces NetworkPolicy on the substrate | Substrate caveat: enforcement depends on the k8-platform CNI; `[PROPOSED — not in source]` whether kind's default CNI enforces it — must be verified per substrate. |
| Kernel boundary | `RuntimeClass` gVisor/Kata | **Best-effort / strong** | Strong syscall-surface reduction, but NOT a guarantee against a kernel/hypervisor 0-day; classified best-effort because runtime availability varies by substrate (REQ-OPS4-05). |
| Node co-tenancy | node-pool / taints | **Open (PROPOSED-ADR-C)** | Whether tenants may share a node = the soft-vs-hard tenancy decision; until decided, classified **not-guaranteed** (soft default assumed). |
| Resource fairness (noisy-neighbor) | `ResourceQuota`/`LimitRange`/`quotas` | **Best-effort** | Quotas bound *aggregate* consumption; sub-quota contention (CPU throttling, IO) is best-effort, not a hard latency SLO (that is OPS1). |
| Data plane (PV / memory) | B20 admission + ADR 0025 access modes | **Guaranteed (boundary), best-effort (concurrency)** | Cross-tenant mount blocked; concurrent-writer semantics deferred (B20 R4). |

**What is explicitly NOT guaranteed at v1.0:** hard (VM-per-tenant) isolation by default; protection against a kernel/hypervisor escape; node-exclusivity per tenant (pending PROPOSED-ADR-C); side-channel resistance; cross-cluster isolation (ADR 0026); a hard per-tenant latency/throughput SLO under contention (OPS1).

- **Security / multi-tenancy (§6.9, ADR 0016/0018):** defense-in-depth; boundaries independent; OPA only restricts. The compute-isolation layer is *additive* to the four ADR-0016 layers, not a replacement.
- **Portability (ADR 0003/0026):** isolation must hold identically on EKS/AKS/kind *to the extent the substrate CNI/runtime supports it*; substrate capability gaps are surfaced (REQ-OPS4-05), not silently tolerated.
- **Observability (§6.5):** noisy-neighbor approach-to-quota and isolation-deny rates surfaced via `platform.observability.*` + a Grafana dashboard; sandbox-escape via `platform.security.*`.
- **Versioning (ADR 0030/0041):** any quota field set composes additively onto `AgentEnvironment`/`TenantOnboarding` without a breaking bump (per A21 OQ-5); OPS4 introduces no new versioned owner.

## 8. Cross-Cutting Deliverable Checklist

OPS4 is a cross-cutting NFR layer (not a Workstream-A component); the §14.1 standard set is owned by the enforcing components. OPS4's own deliverables:

- Helm/manifests in Git — **applicable**: the default-deny `NetworkPolicy`, `ResourceQuota`/`LimitRange`, `RuntimeClass` bindings, node-pool label baseline (as a tenant-baseline overlay reconciled by A21).
- Per-product docs (10.5) — **applicable**: the isolation model + the threat-model/guarantee-tier table (§7) as published platform documentation (REQ-OPS4-08).
- Runbook (10.7) — **applicable**: diagnosing a cross-tenant-reachability alert, a quota-exhaustion event, a sandbox-escape signal.
- Alerts — **applicable**: cross-tenant reachability attempt, quota approach/exhaustion, sandbox-escape, runtime-class admission failure.
- Grafana dashboard (`GrafanaDashboard` XR) — **applicable**: per-tenant quota utilisation + isolation-deny rates (rides A7/A6/audit metrics; no new exporter).
- Headlamp plugin — **N/A** — isolation state visible via standard CRD/NetworkPolicy/ResourceQuota views; no dedicated plugin in v1.0.
- OPA/Rego integration — **applicable**: Gatekeeper admission Rego for network-policy/runtime-class/node-pool/quota conformance, authored in **B16/B3**.
- Audit emission (ADR 0034) — **applicable**: every isolation admission decision (REQ-OPS4-12).
- Knative trigger flow — **applicable**: sandbox-escape `platform.security.*` flow (shared with A6); quota-threshold `platform.observability.*` flow.
- HolmesGPT toolset — **applicable**: "show a tenant's quota utilisation / isolation posture / recent isolation denials" read tools.
- 3-layer tests — **applicable**: Chainsaw (admission + reconcile of isolation primitives across two substrates), PyTest (OPA isolation decision logic), Playwright **N/A** (no dedicated UI).
- Tutorials & how-tos — **applicable**: "understand the tenant isolation guarantees", "raise a tenant's quota".

## 9. Acceptance Criteria

- **AC-OPS4-01** (REQ-OPS4-01/03): With namespace + NetworkPolicy + runtime-class all in place, disabling any *one* boundary in test still leaves a cross-tenant reach blocked by the remaining boundaries. *(Chainsaw, fault-injection)*
- **AC-OPS4-02** (REQ-OPS4-02): A pod in tenant A's namespace cannot open a connection to a pod in tenant B's namespace; an explicitly-published, OPA-checked path succeeds. *(Chainsaw + PyTest)*
- **AC-OPS4-03** (REQ-OPS4-03): A simulated Envoy/gateway-authz bypass yields no cross-tenant pod reachability (the network policy holds independently). *(PyTest)*
- **AC-OPS4-04** (REQ-OPS4-04): A `Sandbox` with `runtime: gvisor`/`kata` runs under the corresponding `RuntimeClass`; a sandbox attempting host `runc` without the audited exception is rejected. *(Chainsaw)*
- **AC-OPS4-05** (REQ-OPS4-05): A `Sandbox` declaring a runtime unavailable on the substrate is rejected at admission with a clear error, not downgraded. *(Chainsaw, kind vs managed-K8s)*
- **AC-OPS4-06** (REQ-OPS4-06): With a tenant `ResourceQuota` set, tenant A consuming to its ceiling does not prevent tenant B from scheduling its declared sandboxes. *(Chainsaw)*
- **AC-OPS4-07** (REQ-OPS4-07): Creating a sandbox that would exceed the tenant quota is denied at admission and emits `platform.policy.*` + `platform.observability.*` events; no unrelated tenant is affected. *(Chainsaw + PyTest)*
- **AC-OPS4-08** (REQ-OPS4-08): The published isolation docs contain the per-boundary guarantee-tier table and make no claim exceeding a boundary's classification. *(doc-lint / review check)*
- **AC-OPS4-09** (REQ-OPS4-09): A simulated sandbox escape fires `platform.security.*` and the escaped workload still cannot reach a cross-tenant pod. *(PyTest)*
- **AC-OPS4-10** (REQ-OPS4-10): A pod setting a `nodeSelector`/`toleration` for a node-pool the tenant is not entitled to is rejected at admission. *(Chainsaw)*
- **AC-OPS4-11** (REQ-OPS4-11): A B20 PV mapping crossing the tenancy boundary is rejected; OPS4's invariant is asserted in the shared conformance suite. *(Chainsaw, reuse B20 AC-B20-02)*
- **AC-OPS4-12** (REQ-OPS4-12): Each isolation admission decision produces an audit record via the adapter; no direct store write. *(PyTest)*
- **AC-OPS4-13** (REQ-OPS4-13): Onboarding a tenant via `TenantOnboarding` provisions the default-deny NetworkPolicy + ResourceQuota baseline from Git, with no manual `kubectl`. *(Chainsaw, extends A21 AC-A21-02)*
- **AC-OPS4-14** (REQ-OPS4-14): The isolation conformance suite asserts no cross-cluster claim is made (single-cluster scope documented and tested as a guard). *(review check)*

## 10. Risks & Open Questions

Open decisions are surfaced as **PROPOSED-ADR TITLES** (titles + one-line rationale) — NOT decided here.

- **PROPOSED-ADR-A — "Tenant-scoped resource-quota field schema (max agents / sandboxes / CPU-memory / warm-pool / virtual-keys) on `TenantOnboarding` + `AgentEnvironment.quotas`."** *Rationale:* the Canon names the quota carrier but defers the field list (backlog §1.2 / ADR 0037 / ADR-0016 OQ-1); the noisy-neighbor invariant (REQ-OPS4-06/07) needs a concrete, versioned schema. *(blast radius: high — every tenant.)*
- **PROPOSED-ADR-B — "Default and minimum sandbox `RuntimeClass` policy per tenant tier (gVisor vs Kata vs runc) and substrate-availability fallback."** *Rationale:* `Sandbox.runtime` is per-template with no platform-wide default/minimum; REQ-OPS4-04/05 need a policy for what a tenant gets by default and how substrate gaps are handled. *(blast radius: high — kernel boundary.)*
- **PROPOSED-ADR-C — "Soft vs hard multi-tenancy: node-sharing policy and node-pool entitlement scheme between tenants."** *Rationale:* whether two tenants may co-tenant a node sets the escape blast radius (REQ-OPS4-09/10); v1.0 currently has no stated stance — soft (shared-node) is the implicit default, hard (node/VM-per-tenant) is unbuilt. *(blast radius: high — escape containment.)*
- **PROPOSED-ADR-D — "Pod-layer NetworkPolicy enforcement contract across substrates (CNI capability + a deny-by-default tenant baseline)."** *Rationale:* REQ-OPS4-02 assumes the CNI enforces NetworkPolicy; ADR 0003 chose Envoy precisely because Cilium L7 isn't universal — the intra-cluster pod-policy enforcement contract across EKS/AKS/kind is unstated. *(blast radius: med-high — cross-tenant reach.)*

Risks:

- **R1 (high):** The pod-layer NetworkPolicy guarantee (REQ-OPS4-02) is only as strong as the substrate CNI's NetworkPolicy enforcement; on a CNI that ignores `NetworkPolicy`, cross-tenant reachability is open despite the manifest. *Reconciliation:* AC-OPS4-02 must run per substrate; gap surfaced as PROPOSED-ADR-D. `[PROPOSED — not in source]` whether kind's default CNI enforces NetworkPolicy.
- **R2 (high):** Soft multi-tenancy (shared-node default) means a kernel/hypervisor escape defeating gVisor/Kata reaches a co-tenant — outside the v1.0 threat model but a real residual risk. *Reconciliation:* explicitly classified out-of-scope in §7; PROPOSED-ADR-C decides whether to harden.
- **R3 (med):** Quotas bound aggregate consumption but not sub-quota CPU/IO contention; a hard per-tenant latency SLO is OPS1's, not OPS4's — risk of over-claiming "isolation". *Reconciliation:* §7 classifies resource fairness best-effort; REQ-OPS4-08 forbids over-claiming.
- **R4 (med):** Ownership of the new isolation manifests — A21 (onboarding) vs A6 (sandbox) vs a new OPS-owned overlay. *Reconciliation:* PLAN places them as an A21-reconciled tenant-baseline overlay with Rego in B16; confirm with A21/A6 owners.
- **R5 (low):** Node-pool/taint entitlement (REQ-OPS4-10) presumes a node-pool scheme the source does not define. `[PROPOSED — not in source]`; tied to PROPOSED-ADR-C.

## 11. References

- architecture-overview.md §6.9 (multi-tenancy & namespacing — namespace boundary, cross-tenant publication), §6.2 (agent runtime — Sandbox, gVisor/Kata), §6.6 (security & policy — defense-in-depth, sandbox-escape signal, OPA/audit hook points).
- architecture-backlog.md §1.2 (deferred tenant-scoped quotas).
- ADR 0016 (multi-tenancy via namespaces), ADR 0018 (RBAC-floor / OPA-restrictor), ADR 0002 (OPA/Gatekeeper), ADR 0003 (Envoy egress — single egress chokepoint, CNI-agnostic), ADR 0026 (independent-cluster model), ADR 0025 (memory access modes), ADR 0034 (audit pipeline), ADR 0030/0041 (versioning), ADR 0028/0029/0037 (tenant identity & onboarding).
- Related pieces: A6 (agent-sandbox + Envoy), A7 (OPA/Gatekeeper), A21 (tenant onboarding), B16/B3 (OPA content), B20 (PV access), spec-ADR-0016 (namespace tenancy).
- Sibling OPS pieces: OPS1 (scale & cost — resource fairness SLOs), OPS3 (key/secret), _meta/pending-operational-nfr-layer.md.
- _meta/interface-contract.md §1.3 (`Sandbox`/`SandboxTemplate`), §1.6 (`AgentEnvironment`, `TenantOnboarding`), §2 (`platform.security.*`/`platform.policy.*`/`platform.tenant.*`/`platform.observability.*`), §5 (audit adapter).
- _meta/glossary.md (Tenant, Approved namespace, Platform Agent, Sandbox, RBAC-as-floor / OPA-as-restrictor, substrate).
