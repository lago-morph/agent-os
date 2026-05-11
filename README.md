# Inspired by Claude Managed Agents

This project was born out of a desire to clone [Claude Managed Agents](https://platform.claude.com/docs/en/managed-agents/overview) as an open source project.

# Agent Runtime Architecture

This repository defines the architecture for an **agent runtime environment** with built-in guardrails and self-improvement. It is the architecture-of-record for a platform that runs AI agents as governed, observable, policy-controlled Kubernetes workloads, with a single chokepoint for LLM, MCP, A2A, and outbound HTTP traffic, and a self-management agent (HolmesGPT) that grows with the platform.

This is **not an implementation repository.** Implementation happens in independent component repositories — each component listed in the architecture has its own implementation effort. This repository is the place where architectural decisions live.

## What's in this repository

- **Architecture overview** — the canonical description of what the platform is, what it's made of, how the pieces fit together. ([architecture-overview.md](architecture-overview.md))
- **Architecture backlog and preserved ideas** — deferred design decisions, alternatives considered and rejected with rationale, evolution paths to revisit, ADR candidates, open questions to leave open. ([architecture-backlog.md](architecture-backlog.md))
- **Future enhancements** — capabilities consciously deferred past v1.0 (redundancy and degraded operations, continuous container security, operations-metrics / SLO management, etc.). Distinct from the backlog: the backlog tracks "we may revisit if X happens"; future-enhancements tracks "we commit to doing this in vNext, just not in v1.0." ([future-enhancements.md](future-enhancements.md))
- **Architecture Decision Records (ADRs)** — formal records of decisions, with context and consequences. The v1.0 set lives in [`adr/`](adr/); new ADRs are added per decision.
- **Design specifications** — detailed design for cross-component concerns (capability set semantics, multi-tenancy model, approval system shape, etc.) that the architecture defers to design-time.
- **Standards** — coding standards, naming conventions, schema conventions, that all implementation components are expected to follow.
- **Interfaces** — formal contracts between components (CRD schemas, CloudEvent schemas, API specs, SDK signatures) where the contract belongs to the architecture rather than to any one component.
- **Architecture-level testing** — tests that verify the architecture-level invariants (RBAC-floor / OPA-restrictor enforcement, audit emission completeness, every-call-through-LiteLLM, etc.). Distinct from per-component tests, which live with their components.
- **Architecture maintainer documentation** — guidance for the people who maintain this architecture. How to make architectural changes, how to write ADRs, how to update the overview, how to integrate a new component proposal.

## What's NOT in this repository

These live elsewhere, with their respective implementation components:

- Component implementations (LiteLLM install configs, ARK Helm values, kopf operator code, Headlamp plugins, etc.).
- Per-component documentation: setup guides, troubleshooting guides, operator runbooks, admin documentation.
- Per-component tests (Chainsaw / Playwright / PyTest test suites scoped to a single component).
- Vendor documentation acquisition and indexing — this is a separate companion project, referenced from the architecture but not part of v1.0 scope.

## Audience

The primary audience for this repository is people who **maintain or extend the architecture** — making decisions about how the platform evolves, evaluating proposed changes against architectural intent, writing ADRs, integrating new components into the model.

Component implementers consume this repository as input (the architecture overview is a reference document for what they're building toward), but the day-to-day implementation work happens in component repositories where their setup, troubleshooting, and admin documentation also lives.

## How to use the architecture overview

The overview is structured for two reading orders:

- **Front-to-back** for a complete picture: purpose, goals, baseline, high-level architecture, software added, architectural views, use cases, CI/CD, OSS limitations, documentation plan, dashboards, training, testing, components, glossary.
- **Reference lookup** via the table of contents and the glossary at the end — when you need the answer to one specific question without reading 1500 lines of context.

The architectural views (section 6) and the use cases (section 7) are the most reread parts. The components section (14) is the source of truth for what work needs to happen and how it's grouped.

## How to propose architectural changes

1. Read the current overview and the backlog.
2. If the change is small or contained, open a PR against `architecture-overview.md` directly with rationale in the PR description.
3. If the change is structural or contested, write an ADR (proposing the change) and open a PR adding it to the ADR directory. Discussion happens on the PR; the ADR captures both the decision and the reasoning behind it.
4. If the change touches both the overview and the backlog (resolving a deferred decision, for example), update both in the same PR.

## Repository layout (planned)

```
.
├── README.md                          (this file)
├── architecture-overview.md           (the canonical architecture document)
├── architecture-backlog.md            (deferred decisions, alternatives, evolution paths)
├── future-enhancements.md             (capabilities deferred past v1.0)
├── adr/                               (Architecture Decision Records, numbered)
├── design/                            (design specifications for cross-component concerns)
├── standards/                         (coding, naming, schema, etc.)
├── interfaces/                        (CRD schemas, CloudEvent schemas, SDK signatures)
└── tests/                             (architecture-level invariant tests)
```

## Status

v1.0 architecture in active development. Implementation will happen in independent component repositories; this repository tracks the architectural decisions that bind them together.
