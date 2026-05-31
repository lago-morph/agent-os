# Decisions Needed — Plain-Language Executive Summary

**What this document is.** The engineering team has produced a technical decision register listing open issues that need a ruling before (or while) building the agentic execution platform. This document translates that register into plain English. Nothing here is a new analysis — it is a faithful re-statement of the engineers' findings written for a reader who will not look anything up.

**How to read it.** Part 1 covers eight contradictions or blockers that must be resolved before build can proceed safely. Part 2 covers twenty-eight design choices that nobody has made yet — they are not crises, but leaving them open will slow teams down. Part 3 covers smaller housekeeping items.

---

## Cast of characters

The platform is built on Kubernetes (the industry-standard system for running software in containers at scale) and Crossplane (a tool that extends Kubernetes so teams can self-service infrastructure by writing a configuration file). The following pieces are mentioned repeatedly. Codes in parentheses are the engineering shorthand; the plain name is what matters.

- **The AI gateway / LiteLLM (A1)** — the central traffic cop that all AI agents talk through. Every request an agent makes to a language model, a tool, or an external service passes through here. It enforces budgets and keeps a virtual "key" per user.
- **The policy engine / OPA + Gatekeeper (A7)** — "OPA" stands for Open Policy Agent; "Gatekeeper" is its Kubernetes plug-in. Together they act as the rule-checker: every action an agent tries is checked here against written rules (called "policies") before it is allowed.
- **The policy library framework (B3)** — the scaffolding that holds and organises the rules the policy engine reads. Think of it as the rulebook binder; A7 is the judge who reads it.
- **The policy library content (B16)** — the actual rules inside that binder (the initial set of allow/deny decisions for agents).
- **The agent operator / ARK (A5)** — the controller that watches agent declarations and makes them real: starts them, restarts them if they crash, shuts them down when done.
- **The sandbox + egress proxy / agent-sandbox + Envoy (A6)** — every agent runs in an isolated "sandbox" (a locked-down container environment). Envoy is the network gatekeeper that controls which external websites or services the agent may contact.
- **The Crossplane compositions / infrastructure abstraction layer (B4)** — the layer that translates a high-level request ("give this agent a database") into the actual cloud or on-premises resource. It abstracts away whether you're running on AWS or a local test cluster.
- **The memory backend adapter (B11)** — the code that lets agents store and retrieve memory (conversation history, learned facts). It talks to the Letta memory backend (A10), which in turn stores data in OpenSearch (A11, a search and indexing engine).
- **The audit endpoint + adapter library (A18)** — receives audit events from every component and writes them to the permanent audit record. Every component that emits an event relies on the contract this piece defines.
- **The CloudEvent schema registry (B12)** — a catalogue of every type of structured event (called a "CloudEvent") the platform emits. A "CloudEvent" is a standardised message format; this registry is the official list of all message types.
- **The approval system (B19)** — handles the workflow for a human signing off before a risky action runs. Backed by Argo Workflows (A3, a workflow automation engine) and the policy engine.
- **The Kargo promotion fabric (A23)** — manages the pipeline that moves a software change from development through staging to production (called "promotion"). Kargo is the product name.
- **The Headlamp UI framework (A9/A22)** — Headlamp is the web dashboard for the platform. A22 is the framework for building graphical forms ("editors") inside it so operators can configure platform objects without writing YAML.
- **The tenant onboarding reconciler (A21)** — the code that sets up a new team ("tenant") on the platform: creates their namespace, applies their permissions, provisions their infrastructure.
- **The kopf operator for LiteLLM (B13)** — a custom Kubernetes controller (written in Python using a framework called "kopf") that manages AI gateway configuration objects as Kubernetes resources.
- **The MCP services integration (A17)** — "MCP" stands for Model Context Protocol, the standard protocol agents use to call external tools (GitHub, Google Drive, web search, databases, etc.). A17 is the initial set of approved tool connections.
- **The policy simulator service (A20)** — lets engineers test "what would happen if this agent tried to do X?" against the policy engine without actually running the agent.
- **The security threat model (B22)** — the document that formally identifies security risks and the platform's v1.0 stance on each.
- **ESO (External Secrets Operator)** — a Kubernetes add-on that fetches secrets (passwords, API keys) from a secure vault and makes them available to services, so secrets are never stored directly in configuration files.

---

## Part 1 — Contradictions and blockers that must be resolved

### D-01 — The AI tool-connection authentication modes are defined in two places that now disagree

**Severity: High** — this makes a key specification untestable and leaves a gap in the frozen official contract.

**What's going on.** The platform lets agents call external tools (GitHub, databases, etc.) through a standard protocol called MCP (Model Context Protocol). Each approved tool connection (called an `MCPServer`) has a field called `authMode` that says how the connection authenticates — for example, does the system hold the credential ("system" mode) or does the individual user supply their own ("user-cred" mode)?

There are two documents that are both marked frozen and authoritative: the interface contract (the canonical list of all fields) and an architectural decision record called ADR-0020 (a record of a design choice, in this case about which MCP tool services to include initially). The interface contract lists exactly two authentication modes: `system` and `user-cred`. ADR-0020 silently adds a third mode, `system-mediated`, without updating the interface contract.

**What caused the contradiction.** ADR-0020 was written or updated after the interface contract was frozen, and the third mode was added to the decision record without triggering a corresponding update to the official contract.

**If we don't fix it.** The initial MCP services integration (A17) has acceptance criteria (testable checkboxes) that depend on the `system-mediated` mode. Because the mode is not in the official contract, those tests cannot be written or passed. Two frozen documents conflict, so any team reading them gets different answers about how authentication works.

**Recommended fix.** Choose one source of truth and update it. The most sensible path is to formally add `system-mediated` as a third valid value in the interface contract (requiring a controlled Canon revision), and ensure ADR-0020 references that updated contract. If `system-mediated` was a mistake, remove it from ADR-0020 instead.

- **Pros of adding to the interface contract:** Captures a real authentication pattern that the MCP integration team needs; unblocks testable acceptance criteria for A17; aligns both documents.
- **Cons:** Requires a formal Canon revision process, which adds overhead; need to confirm the `system-mediated` semantics are fully specified before freezing.

---

### D-02 — Four approval event types exist in the system but nobody owns them

**Severity: Blocker** — events that no team is responsible for authoring will never be built, silently breaking every component that listens for them.

**What's going on.** The platform uses structured events (CloudEvents) to communicate between components. Every event type must be registered in the CloudEvent schema registry (B12) and must have a team or component responsible for authoring (writing) it. The approval system (B19) uses four event types that live in the `platform.approval.*` namespace (meaning they are approval-related events).

The approval system (B19) attributes those event types to the CloudEvent schema registry (B12), saying "B12 owns them." But B12 explicitly says it only maintains the registry — each component is responsible for its own event types. B12 does not author them. So the four approval events are registered nowhere, authored by nobody.

Four other components — the Kargo promotion fabric (A23), the approval system itself (B19), the platform CLI tool (B5, the cross-cutting Headlamp plugins), and HolmesGPT (A14, the self-management agent) — all subscribe to these approval events. If the events are never defined and authored, those subscriptions silently receive nothing.

**What caused the contradiction.** The approval system (B19) incorrectly delegated event ownership to the registry rather than claiming ownership itself, and no review caught this before the specification was frozen.

**If we don't fix it.** The four approval event types will have no authoritative schema and no build task assigned. Components that depend on them will break at integration time, likely late in the project.

**Recommended fix.** Assign ownership of the four `platform.approval.*` event types to the approval system (B19). Add an explicit build task to B19's plan to author these event schemas and contribute them into B12's registry. This is the engineers' recommendation and it is clearly right.

- **Pros:** Unblocks four dependent components; correctly places ownership with the component that generates the events; simple task addition.
- **Cons:** Slightly broadens B19's scope of work (minor).

---

### D-03 — The policy engine's decision output format has four claimed owners but can only have one

**Severity: High** — without a single owner, the format can drift, and any component that reads policy decisions may break silently.

**What's going on.** OPA (Open Policy Agent, the policy engine, A7) produces a structured response for every policy check — this is called the "decision document." Its shape (which fields it contains, what they mean) must be defined by exactly one authoritative owner.

Four components each behave as if they own or co-define this shape: the policy engine (A7), the policy library framework (B3), the agent-sandbox + Envoy egress proxy (A6), and the AI gateway (A1). Two of them (A20, the policy simulator, and B3, the policy library framework) agree that B3 should own the shape. A7 explicitly claims ownership. A6 and A1 also bind to it without deferring to anyone.

**What caused the contradiction.** Ownership was not clearly assigned when the decision document was first designed, and multiple components independently wrote specifications that assumed they were in charge.

**If we don't fix it.** The decision document format can be changed by any of the four components without coordinating with the others. A change to the format by one component silently breaks the others. Integration testing may catch this, but very late.

**Recommended fix.** Designate B3 (the policy library framework) as the sole owner of the OPA decision document shape. A7, A6, and A1 become consumers — they read and depend on the format but do not define it. This is the engineers' recommendation, supported by agreement between A20 and B3.

- **Pros:** Follows the principle of single ownership; supported by two independent sources; B3 is the natural home (it is the policy library, not the gateway).
- **Cons:** Requires A7, A6, and A1 to update their specifications to remove their ownership claims (documentation work, not code changes).

---

### D-04 — The memory infrastructure promises a connection password that nobody has specified

**Severity: High** — components expecting a defined credential format will fail at runtime if it is never defined.

**What's going on.** The platform uses a "MemoryStore" — a Crossplane-managed infrastructure object (part of B4, the infrastructure abstraction layer) that provisions the backend storage for agent memory. When Crossplane provisions infrastructure, it produces a "connection secret" — a Kubernetes secret containing the credentials (hostname, port, username, password, etc.) that other services use to connect.

The memory backend adapter (B11) has a firm requirement that it will be handed a connection secret from the MemoryStore — its setup explicitly depends on receiving those credentials. However, the MemoryStore definition in B4 only specifies two configuration fields (`accessMode` and `backendType`) and makes no promise about what connection secret it will produce. The credential keys needed by the audit endpoint (A18) and the memory backend adapter (B11) beyond the five standard canonical keys are all marked as `[PROPOSED]` — meaning proposed but not yet defined.

**What caused the contradiction.** B11's requirements were written assuming B4 would fully define its connection secret, but B4's MemoryStore specification was left incomplete.

**If we don't fix it.** B11 and A18 will be built against a credential format that does not yet exist. At integration time, there will be no agreed shape for the connection secret, and both components will likely fail.

**Recommended fix.** The B4 specification must be updated to concretely define the full set of connection secret keys for the MemoryStore (what fields the secret contains, their names, and their meanings). This is also connected to the edge-direction question in D-09 (Part 3 of this document).

- **Pros:** Unblocks B11 and A18; follows the same pattern every other substrate XRD already uses for connection secrets.
- **Cons:** Requires B4 specification authors to commit to a concrete secret schema — this may require consulting the memory backend (Letta/A10) to confirm what fields it actually needs.

---

### D-05 — The audit system's core interface is unspecified, yet every component must emit to it

**Severity: High** — this is a platform-wide release blocker; every component that sends audit events is blocked until the interface is frozen.

**What's going on.** The audit endpoint and adapter library (A18) is the platform's central audit pipeline. Every component on the platform that performs a significant action emits an audit event through this adapter. The adapter has a defined interface (how components call it) and an `audit_events` schema (the structure of the events themselves).

Both the adapter interface and the `audit_events` schema are currently marked `[PROPOSED]` — meaning designed but not frozen. There is also no "freeze gate" — no acceptance criterion or process step that forces these to be locked before downstream components start emitting events.

**What caused the contradiction.** The source specifications deferred defining the audit adapter contract, likely intending to come back to it, but no mechanism was put in place to prevent downstream components from building against an unfinalised interface.

**If we don't fix it.** Teams building components that emit audit events (which is nearly every component on the platform) have nothing stable to build against. If some teams proceed and others wait, the platform will have inconsistent audit implementations. At the extreme, the audit trail — a compliance and security requirement — will be incomplete or unreliable at launch.

**Recommended fix.** Freeze and publish the audit adapter interface and the `audit_events` schema before any downstream component begins implementing audit emission. Add a formal freeze-gate acceptance criterion to A18 that must be satisfied before downstream work can start.

- **Pros:** Unblocks the entire platform; ensures a consistent, compliant audit trail; the freeze-gate pattern is already used elsewhere in the design.
- **Cons:** Adds a sequencing constraint — A18 must deliver the frozen spec before other teams can start their audit-emission work. This requires prioritising A18 specification work immediately.

---

### D-06 — Two components both claim to own the tenant onboarding editor

**Severity: High** — both components have this as a hard requirement; if not resolved, both teams may build it independently, or neither will build it believing the other is doing it.

**What's going on.** The platform allows operators to onboard new tenants (teams that will use the platform) through a graphical form in the Headlamp web dashboard. This form is called the "TenantOnboarding editor."

Two components both claim to own building it:
- The Headlamp editor framework (A22) — whose specification says it "ships the initial editor set including TenantOnboarding."
- The tenant onboarding reconciler (A21) — whose specification says it "owns its CRD's editor" (a CRD, or Custom Resource Definition, is a Kubernetes object type; the editor is the graphical form for creating/editing that object type).

Both specifications mark TenantOnboarding as `[PROPOSED]` and both hard-mandate building it. There is also a three-way overlap with B4 (the infrastructure abstraction layer) in terms of what the onboarding process does.

**What caused the contradiction.** The design places both the UI framework and the specific feature in ambiguous ownership territory — a common tension when a framework component and a feature component both have a stake in the same deliverable.

**If we don't fix it.** Two teams may build duplicate implementations, wasting effort. Or each team assumes the other is doing it, and it gets built by neither. Either outcome delays or breaks tenant onboarding.

**Recommended fix.** A21 (the tenant onboarding reconciler) owns its own TenantOnboarding editor. A22 (the Headlamp editor framework) owns the framework and tooling that makes it possible to build such editors, plus any editors for components that don't have their own team. This also resolves the three-way overlap with B4. The engineers recommend this split and it follows the principle of placing ownership with the team closest to the feature.

- **Pros:** Clear ownership; consistent with the principle that each component owns its own CRD editor; removes ambiguity.
- **Cons:** A22 must be careful about what "framework" means — there is a risk of A21 building something that diverges from the A22 framework patterns if the interface between them is not defined.

---

### D-07 — Three infrastructure types are used throughout the design but never defined

**Severity: High** — using undefined types in specifications creates silent inconsistency; teams building against them have no authoritative definition to rely on.

**What's going on.** The Crossplane infrastructure abstraction layer (B4) manages infrastructure by defining object types (called XRDs — Crossplane Composite Resource Definitions, which are like templates for infrastructure). The authoritative glossary lists all defined types. Three types — `SearchIndex`, `MongoDocStore`, and `ObjectStore` — are referenced across multiple specifications but do not appear in the glossary. The engineers believe these are "claim forms" (the user-facing names) for the XRDs `XSearchIndex`, `XMongoDocStore`, and `XObjectStore` respectively, but this has not been formally confirmed.

Note: in Crossplane, an XRD typically has two names — the cluster-level composite (prefixed with `X`) and the user-facing namespaced claim (without the prefix). So `XSearchIndex` and `SearchIndex` should be the same thing, just used in different contexts. But without an explicit definition, this assumption is unverified.

**What caused the contradiction.** The glossary was not updated when these types were referenced in specifications, leaving a gap between what is officially defined and what is actually used.

**If we don't fix it.** Teams building specifications or code that references these types have no authoritative definition. Implementations may diverge, and the glossary — which is supposed to be the single source of truth — will be wrong.

**Recommended fix.** Add `SearchIndex`, `MongoDocStore`, and `ObjectStore` to the authoritative glossary, explicitly noting them as the claim forms of their corresponding `X`-prefixed XRDs. Alternatively, if these names are incorrect, rename every reference in the specifications to the correct canonical form.

- **Pros:** Closes a documentation gap with minimal effort; restores the glossary as a reliable source of truth.
- **Cons:** Requires a Canon revision to the frozen glossary (controlled change process).

---

### D-08 — Two critical wiring contracts are unspecified on both sides simultaneously

**Severity: High** — having both sides of a contract undefined at the same time means neither team can make progress; and the audit system's outbound network access is also unresolved.

**What's going on.** There are two important contracts that connect components, and both have the same problem: neither the component producing the contract nor the component consuming it has defined it yet. Both sides are marked `[PROPOSED]`.

**Contract 1: How egress targets are enforced.** The agent-sandbox + Envoy egress proxy (A6) and the kopf operator for LiteLLM (B13) together control which external URLs an agent may contact. A6 provides the Envoy proxy; B13 manages the `EgressTarget` objects (approved outbound destinations). The wiring mechanism that connects an `EgressTarget` declaration to actual Envoy configuration is undefined on both sides. B13 flags this as high blast-radius — get it wrong and agents either can't reach anything or can reach everything.

**Contract 2: The budget policy data shape.** The policy engine (A7) and the kopf operator (B13) share a data document that tells the policy engine what budget limits apply to each agent. The exact structure of this document is undefined on both sides.

**There is also a third gap:** the audit endpoint (A18) needs to send events to an audit receiver. This outbound connection is an `EgressTarget` (an approved outbound destination). But no `EgressTarget` for the audit endpoint has been declared anywhere.

**What caused the contradiction.** These contracts span component boundaries, making ownership less obvious. Both sides deferred the definition, each perhaps assuming the other would define it.

**If we don't fix it.** Teams building A6, A7, and B13 will build independently and discover incompatibilities at integration. The egress control (a security boundary) may be non-functional. Budgets may not be enforced. Audit events may not flow.

**Recommended fix.** A6, A7, and B13 must jointly define and freeze both contracts before implementing them. Additionally, declare the audit endpoint's `EgressTarget` (its approved outbound network destination) explicitly in the B13 or A18 specifications.

- **Pros:** Resolves three gaps in one coordinated effort; the joint-freeze approach is already used for other cross-component contracts.
- **Cons:** Requires coordination across three teams; cannot be done unilaterally, so scheduling alignment is needed.

---

## Part 2 — Open design decisions (proposed architectural choices)

These are not contradictions — they are design choices where the engineering team has laid out the options but no decision has been made. They are grouped by theme. A "recommended default" is suggested for each; where the choice has major consequences, pros and cons are included.

---

### Secrets and key management

**1. Which key management system to use for encrypting secrets at rest.**
The question: Which cloud or on-premises service (e.g., AWS KMS, HashiCorp Vault) should encrypt connection secrets before storing them? Every connection secret on the platform is gated on this choice.
Why it matters: Without a decision, no team can implement secret encryption. This is a security baseline.
Recommended default: Match whichever key management system the organisation already uses in production (AWS KMS for AWS-hosted deployments). Formalise as an ADR once chosen.

**2. How often should secrets and signing keys be rotated, and what is the overlap window?**
The question: Define a rotation cadence (e.g., every 90 days) per class of secret (database passwords, signing keys, OIDC root keys), plus how long the old key stays valid after the new one is issued.
Why it matters: OIDC root keys (used to verify AI agent identity tokens) have the most risk if this is left undefined — a stale or leaked key could allow impersonation.
Recommended default: 90-day rotation for database/service credentials; 1-year for OIDC signing keys with a 7-day overlap window. Document per-class in the secrets management runbook.

**3. Where should rotation metadata live — in the operator's configuration, on the CRD field, or in an external policy?**
The question: When a secret needs to be rotated, where is the "rotation schedule" recorded so the system knows when to act?
Why it matters: The location determines who can change the schedule and how it is audited.
Recommended default: Store rotation metadata on the CRD field (alongside the resource it governs), so it is version-controlled and visible in the same place as the resource.

**4. How should user-supplied OAuth tokens (for user-credential MCP connections) be managed in the AI gateway?**
The question: When a user supplies their own OAuth token to connect to an external service (e.g., their own Google Drive credentials), how does LiteLLM (the AI gateway, A1) store, refresh, and revoke it?
Why it matters: Affects A1 (the AI gateway) and A17 (the MCP services integration). OAuth tokens expire; without a refresh strategy, user connections will silently fail.
Recommended default: Store tokens encrypted via ESO (External Secrets Operator); implement a background refresh before expiry; revoke on user logout or credential change.

---

### Tenancy and isolation

**5. How should compute quotas for tenants be implemented?**
The question: The platform defines per-tenant resource limits (CPU, memory, number of agents) in an `AgentEnvironment` configuration object. Should these limits be enforced by: (a) the `AgentEnvironment` object itself, (b) native Kubernetes quota objects (`ResourceQuota` and `LimitRange`), or (c) both? And which system fires the error when a tenant exceeds their quota?
Why it matters: This is flagged as the same decision appearing in two separate operational specifications — a sign it is genuinely unresolved and affects onboarding, the sandbox (A6), and every tenant. Gates B4, A5, and A6.
Recommended default: Use native Kubernetes `ResourceQuota` and `LimitRange` as the enforcement layer (they are the reliable, auditable Kubernetes primitive), with the `AgentEnvironment` field as a declarative input that generates those objects. Admission denial fires at the Kubernetes API layer.

**Pros:** Enforcement happens at the Kubernetes layer, which is authoritative and hard to bypass; observable via standard tooling.
**Cons:** Adds a translation step between the `AgentEnvironment` declaration and Kubernetes objects; requires the Crossplane composition (B4) to generate `ResourceQuota`/`LimitRange` objects correctly.

**6. What is the default container isolation level per tenant tier?**
The question: Kubernetes can run containers with different degrees of isolation: `runc` (standard, lightweight), gVisor (sandboxed kernel), or Kata Containers (full VM-level isolation). Which should be the default per tier of tenant, and what is the fallback if the chosen runtime is not available on a given cluster?
Why it matters: Higher isolation = better security but higher resource cost and latency. This sets the security posture for every agent sandbox.
Recommended default: gVisor for standard tenants; Kata for tenants with elevated security requirements; `runc` only for dev/test environments. Document the fallback policy.

**7. How strictly should tenants be isolated at the node level?**
The question: Should agents from different tenants be allowed to run on the same physical or virtual machine (soft multi-tenancy), or should each tenant get dedicated nodes (hard multi-tenancy)?
Why it matters: Flagged as the biggest isolation question in the operational specs. Node-sharing is cheaper; dedicated nodes reduce the blast radius of a container escape.
Recommended default: Soft multi-tenancy (shared nodes) with strict namespace and network policy isolation for v1.0. Reserve dedicated node pools for tenants with contractual isolation requirements.

**Pros:** Lower infrastructure cost for v1.0; operational simplicity.
**Cons:** A container escape attack (where malicious code breaks out of its container) could potentially reach another tenant's workload on the same node.

**8. How should network isolation between tenants be enforced?**
The question: Kubernetes network policies ("deny by default, allow explicitly") must be applied consistently across all cluster network plug-ins (called CNI — Container Network Interface). Not all CNIs support the same features.
Why it matters: Without a confirmed baseline, a tenant's agents may be able to make network calls to another tenant's services.
Recommended default: Establish a documented minimum CNI capability requirement; enforce deny-by-default network policy at the namespace level for all tenants.

**9. What are the default quota values for a new tenant?**
The question: When a new tenant is onboarded, what limits apply by default (maximum number of agents, sandboxes, virtual API keys, cost budget, rate limit)? And how can a tenant request more?
Why it matters: Without defaults, onboarding is undefined and operators must manually configure limits every time.
Recommended default: Define a tiered default table (e.g., starter / standard / enterprise) with explicit values, and document the override path (ticket, approval workflow).

---

### Scale and cost

**10. How should the platform auto-scale its own components?**
The question: Kubernetes offers several auto-scaling mechanisms: HPA (Horizontal Pod Autoscaler, scales by CPU/memory), KEDA (scales by custom metrics, e.g., queue depth), or single-instance per a previous architectural decision (ADR-0035). Which applies to which components?
Why it matters: Under-scaling causes performance failures; over-scaling wastes money.
Recommended default: Single-instance for low-throughput control-plane components (per ADR-0035); HPA for the AI gateway (A1); KEDA for event-driven components (e.g., the Knative event adapters, B8) where queue depth is the right signal.

**11. How should per-tenant costs be calculated and displayed?**
The question: The platform tracks token usage, compute, and storage per tenant. Should cost attribution be calculated at query time in the dashboard (flexible but slow) or stored as pre-computed rollups (fast but requires a batch pipeline)?
Why it matters: Affects the operator dashboard and whether finance/business users can get timely cost data.
Recommended default: Store pre-computed daily rollups for the dashboard; keep raw events in the audit store for on-demand queries.

**12. Should the v1.0 capacity baselines become formal Service Level Indicators?**
The question: The platform has defined capacity baselines (e.g., "supports X concurrent agents"). Should these be promoted to formal SLIs (Service Level Indicators — measurable signals that feed SLO management)?
Why it matters: Ties into the SLO (Service Level Objective) decision below. SLIs are the prerequisite.
Recommended default: Yes, promote the baselines to candidate SLIs immediately so the SLO conversation has concrete numbers to work with.

---

### Disaster recovery

**13. How should backup and DR policies be attached to infrastructure?**
The question: When an operator wants to configure backup for a database or store, should the backup policy be set in the Crossplane XRD (the infrastructure definition object) as a first-class field, or configured separately by a platform operator out of band?
Why it matters: Out-of-band configuration is error-prone and hard to audit. First-class fields make backup policy version-controlled and visible.
Recommended default: First-class XRD field, so backup policy is declared alongside the resource and tracked in Git.

**14. What are the per-data-class RPO and RTO targets for v1.0?**
The question: RPO (Recovery Point Objective) is how much data can be lost in a disaster (e.g., "no more than 1 hour of data lost"). RTO (Recovery Time Objective) is how long recovery may take (e.g., "system back online within 4 hours"). These numbers must be set per type of data (agent state, audit logs, configuration).
Why it matters: Flagged as the biggest open question in the disaster recovery spec — every DR (disaster recovery) drill and backup frequency is gated on these numbers. Without them, you cannot know if your backup strategy is adequate.
Recommended default: The engineering team cannot set these — they are a business decision based on compliance requirements and tolerance for data loss. **This needs an explicit ruling from the business before the DR architecture can be finalised.**

**Pros of deciding now:** Unblocks all DR implementation and testing work.
**Cons of delaying:** DR drills and backup cadence remain undefined; the platform may not meet compliance obligations at launch.

**15. Should Kubernetes control-plane state be recovered from GitOps re-apply or from etcd snapshots?**
The question: "etcd" is the database Kubernetes uses to store its own state (which pods exist, what their configuration is, etc.). In a disaster, you can restore Kubernetes by re-running GitOps (replaying all the configuration from Git) or by restoring an etcd snapshot. The two approaches have different trade-offs.
Why it matters: GitOps re-apply is simpler but may miss in-cluster state that was never in Git. etcd snapshots are more complete but operationally heavier.
Recommended default: GitOps re-apply as the primary path for v1.0 (simpler, already required for the GitOps architecture); etcd snapshots as a backup for break-glass scenarios.

**16. What is the v1.0 cross-region failover model?**
The question: If the primary region goes down, what happens? Options range from backup-restore only (restart from a backup in a new region, accepting hours of downtime) to active-passive (a warm standby ready to take over in minutes) to active-active (two live regions, near-zero downtime).
Why it matters: The cost and complexity of each option varies enormously. This is a business decision about the acceptable downtime.
Recommended default: Backup-restore only for v1.0 (simpler, lower cost). Active-passive as a roadmap item.

**17. How granular should tenant restore be?**
The question: If a single tenant's data is corrupted, can the platform restore just that tenant's data (per-tenant point-in-time restore), or does a restore affect the whole cluster?
Why it matters: Per-tenant restore is much harder to implement but avoids disrupting other tenants during an incident.
Recommended default: Whole-cluster restore for v1.0; per-tenant restore as a roadmap item. Document the limitation explicitly for tenants.

---

### Day-to-day operations and upgrades

**18. How should CRD and infrastructure schema upgrades be managed in production?**
The question: When a CRD (a Kubernetes object type) changes its schema, migrating existing objects to the new schema is complex and risky. Decisions needed: How often are stored versions migrated? How are the conversion webhooks (code that translates between schema versions) kept highly available during a migration? What is the deprecation calendar (how long does an old version stay supported before removal)?
Why it matters: Flagged as the highest blast-radius Day-2 (post-launch operations) question. Getting this wrong can corrupt data or take the platform offline during an upgrade.
Recommended default: Define a deprecation calendar in terms of platform releases (at least 2 releases of support before removal). Run conversion webhooks with at least 2 replicas for high availability. Automate stored-version migration tests in the CI/CD pipeline.

**19. What are the platform's SLO and SLI targets and who owns them?**
The question: An SLO (Service Level Objective) is a promise about platform availability and performance (e.g., "99.5% uptime per month"). An SLI (Service Level Indicator) is the measurement that tells you whether you're meeting it. Who sets these, and who is responsible if they are missed?
Why it matters: Without SLOs, there is no shared definition of "the platform is working well enough."
Recommended default: Platform team owns SLOs for the core components; tenant teams own SLOs for their agents. Define a starter SLO of 99.5% availability for v1.0 and revisit after 3 months of production data.

**20. Should the platform adopt the Crossplane "Operation" type for Day-2 imperative tasks?**
The question: Crossplane v2 introduces a new object type called `Operation` for declarative one-off or recurring maintenance tasks (e.g., "run this database migration"). Should the platform adopt this pattern for Day-2 operations instead of ad-hoc scripts?
Why it matters: Adopting it now is cleaner long-term; deferring it is simpler for v1.0.
Recommended default: Adopt `Operation` for new Day-2 tasks where it fits; do not retrofit existing scripts for v1.0.

**21. How should recurring health checks and capacity tests be declared?**
The question: The platform needs scheduled synthetic tests (e.g., "every 5 minutes, try to create an agent and verify it starts"). Should these be declared as custom Kubernetes objects (`ScheduledTest` or `HealthCheck` XRD), and if so, what is the mechanism?
Why it matters: Without a declared mechanism, health checks are ad hoc and not version-controlled.
Recommended default: Define a lightweight `ScheduledTest` XRD and use it for all platform-level synthetic probes.

**22. Should ArgoCD automatically fix configuration drift, or just alert?**
The question: ArgoCD is the GitOps tool that keeps the cluster state matching what is in Git. When it detects a drift (someone changed something directly on the cluster without going through Git), should it automatically revert the change, or just alert the team? The answer may vary by resource type.
Why it matters: Auto-heal is safer for most resources but dangerous for stateful ones (databases, active workflows) where an automatic revert could cause data loss.
Recommended default: Auto-heal and auto-prune for stateless platform components; detect-and-alert only for stateful resources and active agent workloads.

---

### Policy management

**23. Who can change policy, and what is the change-control process?**
The question: The platform's OPA (Open Policy Agent) policies control what every agent is allowed to do. Who is authorised to change a policy? What review and approval steps are required? Can a team change their own tenant policy, or only platform operators?
Why it matters: Policy changes can either block legitimate work or allow dangerous actions. The change-control model defines the safety culture of the platform.
Recommended default: Platform operators own global policy; tenants may propose changes via a pull request; changes require at least one platform security team approval before merge.

**24. Who owns the global OPA policy bundle, and how should the risk be managed?**
The question: All OPA policies are assembled into a "bundle" (a packaged set of rules) that the policy engine loads. Currently there is no declared owner for the global/cross-tenant bundle. This is also a security and availability single point of failure — if the bundle is misconfigured, all policy checks platform-wide could fail or become permissive.
Why it matters: Flagged as the biggest open question in the policy operational spec, and a combined security and availability risk.
Recommended default: The platform security team owns the global bundle. Implement bundle validation tests in the CI/CD pipeline. Run the bundle distributor with multiple replicas.

**Pros:** Clear ownership; prevents split-brain scenarios where two teams make conflicting changes.
**Cons:** Centralisation means the platform security team becomes a bottleneck for policy changes; mitigate with a clear delegation and review-and-merge SLA.

**25. Should policy bundles be signed, and should load-time verification be enforced?**
The question: A policy bundle can be signed cryptographically so that the policy engine refuses to load a bundle that has been tampered with or that did not come from an authorised source.
Why it matters: Without signing, a compromised CI/CD pipeline could push a malicious policy bundle that grants all agents unlimited permissions.
Recommended default: Yes, sign all policy bundles and enforce load-time verification. This is a security baseline.

**26. How should policy bundle rollouts be staged?**
The question: When a new or updated policy bundle is ready, should it first run in "audit mode" (log what it would block but don't actually block it), then be promoted to "enforce mode" after review? And should the Kargo promotion fabric (A23) gate policy bundle promotions the same way it gates code promotions?
Why it matters: A policy change that accidentally blocks legitimate agent actions is a platform-wide incident. Staging reduces the blast radius.
Recommended default: Require an audit-mode soak period before any policy change goes to enforce mode. Use Kargo Stage gates for policy bundle promotions alongside code.

**27. How should the platform recover if a bad policy blocks all access (anti-lockout)?**
The question: If a bad policy update is pushed that blocks all agents — or even blocks operators from fixing the policy — how does the team recover? There needs to be a "break-glass" procedure.
Why it matters: Without a defined recovery procedure, a policy mistake could take the platform offline with no recovery path.
Recommended default: Define a break-glass recovery procedure: a designated operator with direct Kubernetes API access (bypassing OPA) can remove or replace the policy bundle. Document it in the runbook; test it in DR drills; require multi-person authorisation.

---

## Part 3 — Smaller cleanups

**D-09 — Which direction does data flow between the infrastructure layer (B4) and the memory backend adapter (B11)?**
The engineering dependency graph currently says B4 depends on B11 in one place, but B11's own dependency list omits B4 entirely. Until this is clarified, the build order for these two components is ambiguous. This is also linked to D-04 (the MemoryStore connection secret question) — the direction of the dependency determines which component should define the secret. A ruling is needed from the B4 and B11 component owners.

**QN-01 — The operational spec for Day-2/upgrades (OPS5) may have acceptance criteria gaps.**
The specification has 22 requirements but only 16 acceptance criteria. Approximately 6 requirements may have no testable pass/fail criterion. Either add missing acceptance criteria, or document which requirements are covered by shared criteria.

**QN-02 — Metadata formatting is inconsistent across specification files.**
Some specification files use 8-character hash identifiers, others use 12, 16, or 40 characters; the "FROZEN" status marker is formatted differently across files. This is purely cosmetic but should be normalised in a single cleanup sweep to reduce confusion.

**QN-03 — Ten event namespaces are defined in the CloudEvent registry; only one has a build task assigned.**
The CloudEvent schema registry (B12) defines 10 namespaces of structured events (e.g., `platform.lifecycle.*`, `platform.approval.*`, `platform.gateway.*`, etc.). As noted in D-02, the approval event types have no owner. The other eight namespaces similarly need authoring tasks assigned before any component that emits those events can start building. This is a planning gap, not a code defect.

**QN-04 — Several acceptance criteria are untestable as written.**
Three specific cases: (1) The initial OPA policy library content (B16) tests a security invariant by "planting a canary" — an artificial test case that may not reflect real attack patterns. (2) B16 uses a self-referential test (a test that checks the test itself). (3) Several conformance acceptance criteria in the security/policy architecture view (V6-06) depend on enumerations that are not yet frozen. These should be rewritten as concrete, independently verifiable checks.

**QN-05 — "Platform release" is used as a unit of time in versioning policies but is never defined.**
Several acceptance criteria (approximately 5) state things like "deprecated features must remain supported for N platform releases." If "platform release" has no agreed definition (is it a calendar cadence? a semantic version increment?), these criteria are untestable. Defining the unit is also a prerequisite for the CRD/XRD migration ADR (item 18 in Part 2 above).

**Three object types are referenced but never defined.**
`SearchIndex`, `MongoDocStore`, and `ObjectStore` appear in specifications platform-wide but are not in the authoritative glossary. They are almost certainly the user-facing "claim forms" of the Crossplane XRDs `XSearchIndex`, `XMongoDocStore`, and `XObjectStore` respectively (in Crossplane, the cluster-level composite type has an `X` prefix; the user-facing version drops it). We need to either formally define them in the glossary as such, or rename every reference in the specifications to the `X`-prefixed canonical form. This duplicates D-07 above but is worth noting here as a housekeeping item, since the fix is a glossary edit.
