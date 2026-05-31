# Canon Glossary (FROZEN)

This is the authoritative vocabulary for the Agentic Execution Platform. Downstream
spec authors MUST use these exact spellings and casings. Do not coin variants. Where a
term is a Kubernetes kind / CRD / XRD, the casing is the API kind casing. This file is
derived from `architecture-overview.md` §15 (glossary), §6.12 (CRD inventory), §6.7,
§6.8, §14, and the ADRs. It is FROZEN — change only by explicit Canon revision.

> Convention: backtick-cased identifiers (e.g. `MCPServer`) are exact API kinds /
> field names / event namespaces and must be reproduced verbatim. Product names follow
> upstream-vendor casing (e.g. LiteLLM, NATS JetStream).

## Core platform terms (from §15)

| Term | Canonical spelling | Meaning |
|---|---|---|
| Platform Agent | **Platform Agent** | An agent run and controlled by this platform: declared as an `Agent` CRD, reconciled by ARK, runs in a `Sandbox`, has a `CapabilitySet`, calls out only through LiteLLM and the Envoy egress proxy. Use this term for any agent in our system; reserve bare "agent" for external/generic contexts. |
| A2A | **A2A** | Agent-to-Agent protocol — the inter-agent calling convention brokered by LiteLLM, for Platform Agents calling each other and approved external A2A peers. Native inter-agent transport; OpenAI-compatible HTTP is translated to A2A at the gateway. |
| MCP | **MCP** | Model Context Protocol — the tool-calling protocol Platform Agents use to invoke external services through the gateway. Approved MCP servers are `MCPServer` CRDs. |
| CapabilitySet | **`CapabilitySet`** | A Kubernetes CRD bundling references to MCP servers, A2A peers, RAG stores, egress targets, skills, OPA policy snippets, and LLM providers. Layerable in a Helm-style overlay with add-if-not-there / replace-if-there semantics. |
| Approved capability | **Approved capability** | An MCP server, A2A peer, RAG store, or egress target registered as a CRD. Approval implies governance (observability, OPA gating, Headlamp visibility). Unapproved capabilities are unreachable from a Platform Agent. |
| Capability | **Capability** | A unit of access an agent has — an MCP server, A2A peer, RAG store, egress target, skill, or LLM provider. Always declared via a CRD; never configured directly on the gateway. |
| Knowledge Base | **Knowledge Base** | The `platform-knowledge-base` RAGStore — a single approved RAG store of platform docs, runbooks, pinned vendor docs. Included via CapabilitySet; not built into any agent. |
| HolmesGPT | **HolmesGPT** | The platform self-management agent (a Platform Agent). Broad read access, policy-gated write access for remediation. Lands very early (ADR 0012). |
| Coach Agent | **Coach Agent** | The platform self-improvement agent (a Platform Agent). Runs on schedule, observes traces/audit, proposes prompt/skill changes via PR or suggestion card. (Component B10; also called "Coach Component".) |
| Interactive Access Agent | **Interactive Access Agent** | General-purpose Platform Agent that LibreChat connects to as the default chat endpoint. (Component A16.) |
| Workstream | **Workstream** | A coherent grouping of components: A (install + operate), B (custom dev), C (docs), D (dashboards), E (training), F (production readiness). |
| CRD | **CRD** | Custom Resource Definition. All capability defs, agent specs, sandboxes, memory configs are CRDs reconciled from Git. |
| Approved namespace | **Approved namespace** | A Kubernetes namespace where Platform Agents may run. Tenant / capability-scope / RBAC boundary. |
| Tenant | **Tenant** | A logical isolation boundary, implemented as one or more Kubernetes namespaces, established via Keycloak claims (§6.9). |
| Platform JWT | **Platform JWT** | A Keycloak-issued JWT carrying the platform claim schema: `platform_tenants`, `platform_namespaces`, `platform_roles`, `tenant_roles`, `capability_set_refs`. Consumed by LiteLLM, OPA, Headlamp, LibreChat. |
| IRSA / Workload Identity | **IRSA** / **Workload Identity** | OIDC-based federation of Kubernetes service accounts to cloud identity (AWS IAM via IRSA, Azure AD via Workload Identity). See §6.11. |
| k8-platform | **k8-platform** | The companion repo documenting the assumed Kubernetes baseline (CNI, ingress, DNS, OIDC issuer, CSI). |

## Additional Canon terms (not in §15 but load-bearing)

| Term | Canonical spelling | Meaning |
|---|---|---|
| Warehouse | **Warehouse** | Kargo construct (ADR 0040) where a candidate commit lands before promotion through Stages. |
| Stage | **Stage** | Kargo promotion target (ADR 0040). v1.0 starts with a **single Stage**; staging/prod Stages come online later per the phased rollout. |
| Approval gate | **`Approval`-CRD-backed human gate** | A Kargo Stage gate backed by the `Approval` CRD (B19). |
| RBAC-as-floor / OPA-as-restrictor | **RBAC-as-floor / OPA-as-restrictor** | Platform-wide enforcement model (ADR 0018): RBAC grants the permission floor; OPA may further restrict per-decision. |
| Substrate | **substrate** | A deployment target (`kind` or `aws`) abstracted by Crossplane Compositions (ADR 0044). Selection is label-driven via `platform.io/environment=kind|aws`. |
| Connection secret | **connection secret** | The uniform secret shape (host, port, user, password, dbname or equivalent) every substrate XRD Composition writes per the XRD contract (ADR 0044). |
| ESO | **ESO** | External Secrets Operator — handles MCP-service secrets (§A17). |

## CRDs and XRDs (exact kinds — from §6.12)

ARK-reconciled CRDs:
- `Agent` — Platform Agent declaration.
- `AgentRun` — a single execution of an `Agent`.
- `Team` — coordinated multi-agent grouping.
- `Tool` — ARK-native tool definition.
- `Memory` — agent-scoped memory binding (access mode lives on `MemoryStore`, ADR 0025).
- `Evaluation` — agent evaluation specification.
- `Query` — ARK-native query primitive.

agent-sandbox-reconciled CRDs:
- `Sandbox` — sandbox instance running agent pods.
- `SandboxTemplate` — reusable sandbox class definition.

kopf-operator (B13) reconciled CRDs:
- `MCPServer` — approved MCP server registered with the gateway.
- `A2APeer` — approved A2A peer (internal or external).
- `RAGStore` — RAG-capable store (vector + hybrid).
- `EgressTarget` — approved outbound HTTP destination.
- `Skill` — skill artifact reference (managed via LiteLLM's skill gateway).
- `CapabilitySet` — bundled, layerable set of capabilities.
- `VirtualKey` — LiteLLM virtual key bound to a CapabilitySet and identity.
- `BudgetPolicy` — budget specification consumed by OPA + LiteLLM.

Other reconcilers:
- `Approval` — human-approval request (reconciled by Argo Workflow + B19).
- `LogLevel` — declarative log-level / trace-granularity toggle (ADR 0035), per-component reconcile.

Crossplane XRs / XRDs (reconciled by Crossplane, B4):
- `MemoryStore` (XR) — composed memory backend.
- `AgentEnvironment` (XR) — composed environment for a class of agents.
- `SyntheticMCPServer` (XR) — MCP server synthesized from an OpenAPI spec.
- `GrafanaDashboard` (XR) — namespaced dashboard (ADR 0021 / 0044).
- `AuditLog` (XR) — composes the audit pipeline (ADR 0034 / 0044). Composes `Postgres` + `ObjectStore`.
- `TenantOnboarding` (XRD) — provisions a tenant (ADR 0028 / 0037).
- `AgentDatabase` (XR) — per-agent/tenant/user Postgres or MongoDB databases (ADR 0020 / 0044).
- `Postgres` (XR) — substrate-abstracted Postgres (ADR 0044).
- `SearchIndex` (XR) — substrate-abstracted search/index backend (ADR 0044).
- `ObjectStore` (XR) — substrate-abstracted object storage (ADR 0044).
- `MongoDocStore` (XR) — substrate-abstracted Mongo-compatible document store (ADR 0044).

> All v1.0 platform CRDs are **namespaced**. There are no cluster-scoped platform CRDs in v1.0.

> Crossplane v2 naming (ADR 0044): All substrate XRs are namespace-scoped directly — users
> create them in their namespace. The `X`-prefix convention (ADR 0041) is retired with claims.
> Primitive XRs: `Postgres`, `SearchIndex`, `ObjectStore`, `MongoDocStore`. Higher-level XRs:
> `AgentDatabase` (composes `Postgres` or `MongoDocStore`), `AuditLog` (composes `Postgres` +
> `ObjectStore`), `GrafanaDashboard`. All reference ADR 0044.

## Products and vendor components (exact casing)

| Canonical | Notes |
|---|---|
| **LiteLLM** | The gateway (component A1). |
| **Langfuse** | LLM observability / tracing (A2). |
| **ARK** | The agent operator (A5; ADR 0001). Reconciles `Agent`, `AgentRun`, `Team`, `Tool`, `Memory`, `Evaluation`, `Query`. |
| **Letta** | Memory backend (A10; ADR 0005). |
| **OpenSearch** | Search + vector store; advisory audit fanout (A11; ADR 0009). |
| **Knative** | Knative Eventing for the event mesh (A4). |
| **NATS JetStream** | Broker backend for Knative (A4; ADR 0004). |
| **Kargo** | Promotion fabric across environments (A23; ADR 0040). |
| **Headlamp** | Kubernetes UI + plugin framework (A9). |
| **HolmesGPT** | Self-management agent (A14; ADR 0012). |
| **Crossplane** | Composition / substrate abstraction (Crossplane v2; B4; ADR 0044). |
| **OPA / Gatekeeper** | Policy engine (A7; ADR 0002). "OPA" for the engine, "Gatekeeper" for admission. |
| **Envoy** | Egress proxy (A6; ADR 0003). "Envoy egress proxy". |
| **Keycloak** | Identity provider / JWT issuer (ADR 0028, 0029). |
| **Mattermost** | Chat integration, Team Edition (A19; ADR 0036). |
| **oauth2-proxy** | SSO proxy in front of platform UIs (A15 install, B1 config). |
| **Reloader** | Config/secret rolling-restart (A15). |
| **Tempo** | Distributed tracing backend (A13; ADR 0015). |
| **Mimir** | Metrics backend (A13). |
| **agent-sandbox** | Sandbox runtime operator (A6). Reconciles `Sandbox`, `SandboxTemplate`. |
| **LibreChat** | Locked-down chat frontend (A8; ADR 0007). |
| **Argo Workflows** | Workflow engine (A3). |
| **ArgoCD** | GitOps sync (referenced throughout; within-environment deploy). |
| **Argo Workflow** | A single workflow run (used in `Approval` reconciliation). |
| **Material for MkDocs** | Documentation portal (C1; ADR 0008). |
| **CloudNativePG** | In-cluster Postgres on kind (per `Postgres`, ADR 0044). |
| **MinIO** | MinIO-compatible object store on kind (per `ObjectStore`). |
| **Bitnami MongoDB** | In-cluster MongoDB on kind (per `MongoDocStore`). |
| **Context7** | An initial MCP service (A17; ADR 0020). |
| **Firecrawl** | Explicitly **NOT in v1.0** (A17 / ADR 0020) — replaced by generic web-search + web-scrape services. |
| **LangGraph** | Supported v1.0 agent SDK (B7; ADR 0019). `Agent.sdk` value `langgraph`. |
| **Langchain Deep Agents** | Opinionated default agent SDK, built on LangGraph (B7; ADR 0019). `Agent.sdk` value `deep-agents`. |
| **AlertManager** | Source of the v1.0 AlertManager → HolmesGPT trigger flow (§6.7). |

## Repos / identifiers
- `platform-knowledge-base` — the single Knowledge Base RAGStore name.
- `agent-platform` — the platform CLI (B9) and test framework (B14) name prefix.
- `platform.io/environment` — the cluster substrate-selection label (`kind` | `aws`).
- CloudEvent namespaces: `platform.lifecycle.*`, `platform.audit.*`, `platform.gateway.*`, `platform.policy.*`, `platform.capability.*`, `platform.evaluation.*`, `platform.approval.*`, `platform.observability.*`, `platform.tenant.*`, `platform.security.*` (see interface-contract.md).
- `platform.capability.changed` — the specific capability-change CloudEvent (ADR 0013 update).
