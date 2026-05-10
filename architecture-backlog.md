# Architecture Backlog and Preserved Ideas

This document captures everything raised during architecture discussions that did not fully land in the architecture overview. The intent is to preserve context so future work (ADRs, design specifications, implementation tasks, revisits) has a single place to draw from rather than re-deriving from chat transcripts.

Sections are organized by what to do with the content: deferred design decisions, alternatives considered and rejected with rationale, evolution paths to revisit if circumstances change, topics that need design before implementation, open questions to leave open until the right moment, architecture invariants worth formalizing, and ADR candidates.

## 1. Deferred design decisions

These are decisions explicitly punted to design-time. Each will need its own design specification or ADR before implementation.

### 1.1 CapabilitySet layering semantics

The architecture now commits to add-if-not-there / replace-if-there field-level overlay, processed in declared order, with per-Agent overrides applied last (see overview section 6.8). Detailed semantics still deferred:

- Whether a CapabilitySet can include other CapabilitySets, and whether inclusion is recursive or one-level deep.
- Concrete validation rules: enforce that referenced CapabilitySets exist at admission time. What happens if a referenced set is deleted.
- Whether namespace boundaries apply to CapabilitySet references — can an Agent in namespace A reference a CapabilitySet in namespace B (with cross-namespace policy approval).
- Whether per-Agent overrides can grant capabilities not present in any referenced set, or whether Agent capabilities must be a subset of what the referenced sets provide.

### 1.2 Multi-tenancy details

The architecture model is established (tenancy = namespace, RBAC + OPA + network enforcement, dynamic agent registration, three visibility modes, **tenant carried as Keycloak claims with required JWT schema specified in section 6.9**). Details deferred to design:

- Tenant onboarding flow: who creates a tenant, what gets provisioned automatically.
- Tenant-scoped resource quotas (max agents, max sandboxes, max virtual keys, cost budget, rate limit) — defaults and how they're customized.
- Cross-tenant collaboration patterns when explicitly published.
- Concrete `platform_roles` catalog (the architecture commits to the claim's presence and shape; the role names are design-time).

### 1.3 Self-service virtual key request flow

Virtual keys are CRDs reconciled by the kopf operator and issuance is self-serve via Headlamp under RBAC + OPA. To be designed:

- Specific request UX in Headlamp.
- How the secret value (the actual key string) is returned to the requestor securely.
- Rotation policy — schedule, on-demand, both.
- Specific scope grammar — what can a key be scoped to.

### 1.4 Cost budget enforcement details

Architecture commits to OPA as the policy layer + LiteLLM as the spend tracker, with Headlamp as the editing surface. Details deferred:

- Whether a `BudgetPolicy` CRD is its own type or budget data is attached to CapabilitySet — design-time call.
- Specific enforcement actions on threshold crossing: rate-limit, suspend, alert, OPA-deny.
- Verification of which LiteLLM budget features are OSS vs enterprise; if features we need are enterprise, where the gap-filling callbacks live.

### 1.5 Approval CRD details

Architecture establishes the generalized approval system (`Approval` CRD with action attributes, default level, OPA elevation only, Argo Workflow mechanics, Headlamp plugin). Details deferred to design:

- Exact `Approval` CRD schema: what attributes are first-class, what goes in a generic metadata blob.
- Timeout / escalation behavior beyond the basic ask-and-decide flow.
- Whether the Argo Workflow template is shared across approval types or per-type.
- Audit detail level for approve vs reject decisions.

### 1.6 Skill versioning and promotion

Skills are managed by LiteLLM's skill gateway, stored in Git. Open questions deferred:

- How skill versions are tracked (Git tags, semantic version embedded in artifact, content hash) — likely whatever LiteLLM does natively.
- How a skill change moves from dev → staging → prod, given v1.0's GitHub Actions-only CI scope.
- Skill compatibility tracking: how an Agent CRD pins a specific skill version, how breakage is detected.

### 1.7 Knowledge base access pattern from agent pods

Architecture says agent pods can reach the knowledge base "through SDK API or volume mount, decided in design." Both have implications:

- SDK API: each query is a network call, observable per-call, supports semantic ranking and freshness checks, adds latency.
- Filesystem mount: read-only filesystem mount of indexed content, lower latency for grep-style access, harder to instrument and update.

A reasonable hybrid: SDK API for semantic / RAG queries, filesystem mount for structured reference data the agent reads frequently. Design needs to specify per-content-type behavior and cache invalidation.

### 1.8 LLM provider failover details

Architecture is clear on behavior (zero providers invalid; one provider, fail visibly; multiple providers, silent failover). Details deferred:

- Failure classification (transient vs permanent) and back-off behavior.
- Cost vs reliability trade-off — failing back to a more expensive provider may exceed budget; how OPA budget enforcement and failover interact.
- Whether per-call provider override is supported (an agent declares a preferred provider for a specific call vs the default order).

### 1.9 MCP server health-check details

Architecture commits to using only what MCP itself supports (no synthetic health-check layer). Details deferred:

- Specific signals from the MCP protocol that LiteLLM watches.
- How LiteLLM surfaces unhealthy state to Platform Agents (error codes, structured error responses, transparent failure).
- Whether to circuit-break on sustained unhealth, and the exact thresholds.

### 1.10 PV access for Platform Agents (B20)

Architecture introduces the concept (declarative mapping of pre-defined Kubernetes PVs into Platform Agents, with namespace + RBAC + OPA-driven access). Details deferred:

- Specific CRD shape for PV access declaration.
- How a pre-defined PV gets registered as available.
- Read-only vs read-write modes.
- Multi-agent shared access semantics.

### 1.11 Initial MCP services details (A17)

Architecture lists the initial set: GitHub, Google Drive, Firecrawl, Context7. Per-service details deferred:

- **GitHub**: exact mechanism for system-credential vs user-credential modes; how LiteLLM brokers OAuth on behalf of the user; what scopes get requested; how revocation flows.
- **Firecrawl**: secret storage shape (probably ESO + a `Secret` referenced by the `MCPServer` CRD), per-agent access restriction model, budget enforcement specifically for system-authenticated services where one account serves many agents.
- **Google Drive**: probably similar to GitHub (system + user credential modes), but specifics depend on the MCP server implementation.
- **Context7**: less complex; install + register.

### 1.12 Coach Component mechanism details

Architecture establishes Coach as a Platform Agent on schedule, dependent on Langfuse + ARK + LiteLLM + observability. Details deferred:

- What Coach queries from Langfuse / Tempo / audit at each scheduled run.
- What kinds of changes Coach proposes (skill changes are confirmed; what else).
- Coach's specific LLM-driven analysis approach.

### 1.13 Audit retention policy (deferred to Workstream F)

The architecture establishes the dual-write path (OpenSearch + S3 archive). The actual retention policy — durations, lifecycle rules, redaction strategy, regulatory compliance review — is part of Workstream F and gets defined when the platform is ready to ship.

### 1.14 Agent development environment (B21)

Decision between local kind-based development and shared dev cluster development is per-developer / per-team, not platform-wide. The component ships support for both paths. Specific tooling, IDE integration, and conventions are design-time.

### 1.15 Documentation portal vs Headlamp linking strategy

Headlamp deep-links into other UIs (Langfuse, OpenSearch Dashboards, Argo Workflows, ArgoCD, LiteLLM admin). Specifics of linking — context preservation, authentication handoff, deep-link URL schemas — are design-time.

### 1.16 OpenAI ↔ A2A translation source

Architecture says "expected to be handled by LiteLLM directly; if not in OSS, build a small adapter as part of LibreChat install." Confirmation needed during implementation, and adapter shape if required.

### 1.17 Identity federation details

Architecture commits to the federation chain (cluster OIDC → IRSA / Workload Identity for cloud, cluster OIDC → Keycloak federation for platform JWT) in section 6.11. Details deferred:

- Specific token-exchange flow between cluster OIDC and Keycloak (RFC 8693 token exchange shape, subject-token vs actor-token, audience handling).
- Caching and refresh strategy for platform JWTs in long-running Platform Agents.
- Azure Workload Identity bootstrap specifics — TBD until v1.0 lands and Azure becomes a real target.
- Service-account → CapabilitySet binding mechanism (annotation, label, separate CRD).

### 1.18 Versioning specifics per surface

Architecture commits to the versioning policy in section 6.13 (CRDs via Kubernetes API versioning; CloudEvents via per-event schemaVersion; semver for SDK and CLI; URL-path versioning for HTTP APIs). Per-surface specifics deferred:

- Concrete deprecation windows (the architecture says "at least one minor platform release"; the calendar definition of "platform release" is design-time).
- Compatibility matrix maintenance — where it lives, how it's verified in CI.
- Conversion webhook patterns for CRD breaking changes.

## 2. Alternatives considered and rejected

These were evaluated and not chosen. Captured here so the rationale is preserved and so future revisits can challenge the original decision with full context.

### 2.1 Agent operator: kagent vs ARK
**Chose ARK.** Broader CRD set, CLI + SDKs + dashboard, better fit for declarative governance.

### 2.2 Policy engine: Kyverno vs OPA Gatekeeper
**Chose OPA + Gatekeeper.** OPA's runtime decision capability is needed for LiteLLM callbacks regardless; adding Kyverno would mean two policy languages. Cost: lose Kyverno's image signature verification path.

### 2.3 Egress control: Cilium L7 NetworkPolicy vs Envoy proxy
**Chose Envoy egress proxy.** Cilium isn't available across EKS, AKS, kind. Envoy is CNI-agnostic and gives us audit + OPA hooks at the egress point.

### 2.4 Tracing strategy: Tempo + Langfuse correlated, vs Langfuse-only
**Chose both correlated by trace_id.** Langfuse for LLM-grade traces; Tempo for general OTel. Correlation by trace_id lets developers move between them.

### 2.5 LiteLLM provider: Go Crossplane provider vs Python kopf operator
**Chose Python kopf.** Team has no Go experience. kopf achieves the same outcome and keeps Python as the default language. Trade-off: lose Crossplane's connection secret handling, package model, composition integration for the LiteLLM-specific parts. Crossplane stays for cloud-resource Compositions.

### 2.6 Documentation portal: Backstage vs Material for MkDocs vs Confluence
**Chose Material for MkDocs.** Backstage is heavy and only worth it if also using its catalog. Confluence breaks docs-as-code / GitOps.

### 2.7 Eventing: Argo Events vs Knative Eventing
**Chose Knative Eventing.** CloudEvents-native, strong source/broker/trigger model, broader ecosystem.

### 2.8 Knative broker backend: Kafka vs NATS JetStream vs RabbitMQ vs SNS+SQS
**Chose NATS JetStream everywhere (dev and prod).** SNS+SQS doesn't fit Knative's pub/sub semantics. NATS chosen for being lighter, runs anywhere, JetStream durability, native Knative channel.

### 2.9 Memory backend: Mem0 vs Letta vs Zep / Graphiti vs Cognee vs LangMem
**Chose Letta.** Fully OSS, OS-style tiered memory model.

### 2.10 Chat UI: LibreChat vs Open WebUI vs custom
**Chose LibreChat (locked down).** Lowest-friction path; lockdown via `librechat.yaml` config is real.

### 2.11 Auth proxy: oauth2-proxy vs Pomerium vs Authelia
**Chose oauth2-proxy.** Most widely deployed, simplest model.

### 2.12 Test reporting: Allure vs ReportPortal vs none initially
**Chose none initially.** CI output + Grafana dashboards from test metrics covers MVP.

### 2.13 Load testing: custom harness vs Chainsaw / Playwright concurrent invocation
**Chose Chainsaw / Playwright concurrent invocation.** Goal is finding gross race conditions at low concurrency, not benchmarking.

### 2.14 Test execution model: declarative test CRDs vs CLI orchestration
**Chose CLI orchestration.** Add CRDs only if declarative test execution becomes a felt need.

### 2.15 Search store: Elasticsearch vs OpenSearch
**Chose OpenSearch.** OSS Security plugin includes OIDC and SAML; vector and hybrid search at parity.

### 2.16 Initial agent SDK choice
**Chose Langchain Deep Agents** (single SDK in v1.0). Multi-SDK harness shape preserved for future additions.

### 2.17 CI system support: GitHub Actions only for v1.0 vs multi-CI
**Chose GitHub Actions only for v1.0.** Jenkins and GitLab CI explicitly out of scope. CLI-based design means adding them later is mechanical.

### 2.18 Generalized approval system: defer vs do now
**Reversed during this design pass to do now.** Originally limited to two specific flows with explicit "do not generalize" language. Moved to a generic `Approval` CRD + Argo Workflows + OPA elevation. Initial use cases remain the two original ones, but the mechanism is reusable.

### 2.19 Identity: SPIFFE vs Kubernetes ServiceAccounts
**Chose Kubernetes ServiceAccounts.** Revisit if multi-cluster federation with strong cross-cluster identity is needed.

### 2.20 Day-2 operations: Crossplane Operation type vs ad-hoc
**Chose ad-hoc.** Revisit if day-2 patterns accumulate enough to justify unification.

### 2.21 Agent fanout: Argo Workflows only vs Argo Workflows + Temporal
**Chose Argo Workflows only.** Add Temporal only when an actual workflow is awkward in Argo.

### 2.22 Approval mechanism rebuild on need vs generalize once
**Chose generalize once.** Originally planned to refactor as a third use case appeared; reversed during this design pass.

## 3. Evolution paths to revisit

Decisions that are right for the current scope but worth revisiting under specific triggers.

### 3.1 LiteLLM at high RPS
LiteLLM has known Python overhead at 5,000+ RPS. v1.0 target is well below that. **Trigger to revisit**: gateway throughput or latency due to overhead becoming a measurable concern. Bifrost is the Go alternative.

### 3.2 Adding Temporal alongside Argo Workflows
**Trigger**: when a workflow is awkward to express in Argo Workflows.

### 3.3 More approval flow features
Already generalized at the basic level. **Trigger**: M-of-N approvers, delegation, escalation timers, complex routing — added when concrete use cases reveal gaps in OPA-elevation-only.

### 3.4 Allure or ReportPortal
**Trigger**: cross-run trend analysis or per-run inspection beyond CI output / Grafana.

### 3.5 SPIFFE identity
**Trigger**: multi-cluster federation with cross-cluster A2A or shared resources.

### 3.6 Crossplane Operation type for day-2
**Trigger**: enough day-2 patterns to benefit from a common pattern.

### 3.7 Test CRDs
**Trigger**: CLI-driven model fights us — scheduled execution, complex composition.

### 3.8 Custom load testing harness
**Trigger**: a real production race condition that should have been caught by stress probes.

### 3.9 Image signature verification
Lost when we chose OPA over Kyverno. **Trigger**: supply chain security requirements demanding image signature verification. Add Kyverno specifically, or use Sigstore policy-controller, or Connaisseur.

### 3.10 Knowledge base interactive UI beyond LibreChat
**Trigger**: richer search-and-browse experience for platform builders becomes a felt need.

### 3.11 HolmesGPT autonomous remediation expansion
Starts read-only with auto-PR for narrow safe changes. **Trigger**: a remediation pattern we trust enough to gate behind OPA but execute autonomously.

### 3.12 Power-user personal agents in LibreChat
Currently locked down. **Trigger**: power users have a real felt need for personal lightweight agents.

### 3.13 Multi-cluster federation
Currently independent installs. **Trigger**: a use case that requires cross-cluster awareness or shared resources.

### 3.14 Multi-CI support beyond GitHub Actions
**Trigger**: a deployment environment requires Jenkins or GitLab CI.

### 3.15 Adding SDKs beyond Langchain Deep Agents
v1.0 ships single-SDK with multi-SDK harness shape preserved. **Trigger**: a use case that fits another SDK better than Langchain Deep Agents.

## 4. Topics that need further design before implementation

These are not "alternatives" — they're concrete pieces of the architecture that need design specifications before code.

- **Adversarial threat model (B22)** — the highest-priority design specification, called out in section 6.6 of the overview. Its output sets security standards for every component.
- Specific agent profiles to ship in the profile library (B17 scope).
- Specific recommended agent compositions (B18 scope).
- Specific OPA policy library content — concrete Rego bundles for v1.0 (B16 scope).
- Specific Crossplane Compositions to build first — `AgentEnvironment`, `MemoryStore`, `SyntheticMCPServer`, `GrafanaDashboard` (B4 scope).
- Specific cross-cutting Headlamp plugins owned by B5 (capability inspector, approval queue UI, virtual key admin).
- Specific tutorials and how-to guides to write first.
- Memory namespace and sharing model details (3-mode access already specified).
- Test manifest schema for the `agent-platform test` CLI.
- GitHub Actions reference pipeline implementation (security-first per overview section 8).
- Glossary maintenance — keeping terminology consistent as the platform evolves.
- Knative Trigger flow design for each component (the "what events does this emit and consume" part of standard component deliverables).
- Concrete `platform_roles` catalog — names and semantics of each platform role used in Keycloak claims.
- Cluster OIDC → Keycloak federation token-exchange flow (section 1.17).
- Per-event-type CloudEvent schemas under each top-level namespace (section 6.7).

## 5. Open questions to leave open

These questions don't need answers yet but should be revisited at the right moment.

- What's the right granularity for virtual keys — per agent, per team, per workload class, or some combination?
- How do we handle dev environments (kind) without cloud-specific event sources — webhook receivers for everything, synthetic event generators?
- Does the platform itself run more agents to manage itself beyond Coach and HolmesGPT (release management agent, upgrade agent, backup agent)?
- How do we share tested CapabilitySet profiles across deployments (curated catalog, internal, vendor-provided, community)?
- What is the human review burden expected for Coach and HolmesGPT auto-PRs, and at what ratio of auto-PR to manual approval do we tune the system?
- Platform self-upgrade strategy — staged rollout, blue-green, canary; same as agent upgrades or different?
- Whether B5 (cross-cutting Headlamp framework + cross-component plugins) ends up overlapping with per-component Headlamp plugins enough to merge or split.
- Agent topology for the AI agents that will implement the platform — what kinds of agents, how they coordinate, what human-in-the-loop patterns apply. Held for a separate conversation after architecture is solid.

## 6. Architecture-level invariants worth documenting as ADRs

Invariants that are implicit in the architecture but should become explicit (these are good ADR candidates).

- All custom code is Python (with the explicit exception of Headlamp plugins — TypeScript/React because Headlamp is).
- All controllers either use Crossplane (for cloud-shaped resources) or kopf (for application-API reconciliation). No bespoke controller frameworks.
- All configuration is in Git.
- Anything in OpenSearch must be reproducible from a primary source (Postgres, object storage, or external system).
- Every audit-relevant action emits a structured event to the audit index, regardless of which component performed it.
- Every Platform Agent invocation traverses LiteLLM. The Envoy egress proxy handles non-LiteLLM HTTP. There are no other paths to external services.
- Every component contributes a HolmesGPT toolset, an OPA policy contribution, audit emission, observability, Knative trigger flow design, tests, and tutorial / how-to documentation.
- SSO is exclusively Keycloak. No tool maintains its own user store. **All privileged platform identity ultimately resolves to a Keycloak-issued JWT carrying the section 6.9 claim schema.**
- **RBAC is the floor; OPA may raise the floor on a per-decision basis.** OPA never grants permissions RBAC didn't already give.
- All dashboards are namespaced and access-controlled (Crossplane-composed `GrafanaDashboard` XRs).
- Approvals flow through one mechanism (`Approval` CRD + Argo Workflows + Headlamp plugin). New approval types reuse the mechanism rather than building parallel ones.
- Memory access modes are per-store: private, namespace-shared, or RBAC/OPA-controlled.
- Each cluster is an independent install of the platform. Multi-cluster federation is not a v1.0 requirement.
- **All platform CloudEvents fall under one of the committed top-level namespaces** (section 6.7).
- **Every API surface (CRDs, CloudEvents, SDKs, CLIs, HTTP APIs, agent A2A/MCP interfaces) carries an explicit version, owned by the component that exposes it** (section 6.13).
- **Mapper authoring (translating upstream IdP attributes into platform claims) is out of scope for this architecture**; the platform commits to the claim schema, not the mappers (section 6.9).
