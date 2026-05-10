# ADR 0010: GitHub Actions only for v1.0 CI/CD

## Status

Accepted

## Context

The platform exposes itself to CI as a **vendor-neutral CLI** (`agent-platform`)
distributed as **container images called from native CI pipelines**. The CLI
provides subcommands like `validate`, `eval`, `redteam`, `package`, `test`,
`deploy-preview`, `promote`, `scan`, and `update-base`. Pipeline definitions
remain native to the chosen CI system; they orchestrate the same CLI calls.
The CLI is the contract; pipeline files are syntax around it.

Given this design, the question is which CI systems v1.0 ships reference
pipelines for. Earlier drafts envisaged three reference pipelines — GitHub
Actions, GitLab CI, and Jenkins — but each one carries non-trivial
maintenance cost: a how-to guide, a maintained example repo, secrets and
runner conventions documented per system, and end-to-end tests that exercise
the reference pipeline against a live platform.

Backlog §2.17 records the decision to limit v1.0 to GitHub Actions.
Backlog §3.14 records the trigger for revisiting: a deployment environment
that requires Jenkins or GitLab CI.

## Decision

**v1.0 supports GitHub Actions only.** Jenkins and GitLab CI are explicitly
out of scope for v1.0.

What ships in v1.0:

- The `agent-platform` CLI as a container image, usable from any CI system.
- One reference pipeline, in GitHub Actions, published as a how-to guide in
  the doc portal.
- A documented schema for required environment variables, secrets, and
  reachable network endpoints (Langfuse, Git, container registry, ArgoCD,
  OPA bundle store, OpenSearch).
- A scheduled GitHub Actions workflow for the agent base image
  rebuild-scan-eval-bump cycle.

What is deliberately not built in v1.0:

- A GitLab CI reference pipeline.
- A Jenkins (Jenkinsfile or shared library) reference pipeline.
- Tooling that abstracts pipeline definitions across CI systems.

## Consequences

Positive:

- Tighter scope and fewer reference artefacts to maintain through v1.0.
- The team builds deep familiarity with one CI surface, reducing the chance
  of inconsistencies between three half-finished references.
- Because the **agent-platform CLI** is the integration surface — not native
  pipeline templates — adding Jenkins or GitLab CI later is mechanical: write
  a pipeline file in the target syntax that calls the same CLI image. No
  platform-side code change is required.

Negative / accepted:

- Organizations standardized on Jenkins or GitLab CI must hand-write their
  pipeline against the CLI contract during v1.0. They get the CLI image and
  the env/secret schema, but no copy-pasteable starter.
- Trigger to add another CI is recorded in backlog §3.14: a deployment
  environment that requires Jenkins or GitLab CI. When that trigger fires,
  the work is mostly authoring a reference pipeline and a how-to guide,
  not changing the platform.

## Alternatives considered

- **Three reference pipelines from day one (GitHub Actions, GitLab CI,
  Jenkins).** Rejected for v1.0 scope. The CLI-based design makes deferral
  cheap; carrying three references through v1.0 is not.
- **No reference pipeline, ship only the CLI.** Rejected: a single concrete,
  end-to-end reference is the fastest way for a team to get from "we have
  the CLI" to "our PRs run validate, eval, redteam, package, deploy-preview".
- **Platform-owned cross-CI abstraction layer.** Rejected: re-introduces the
  pipeline-template coupling we explicitly avoided when we chose option 2
  (CLI tasks called from native pipelines) over option 1 (platform-owned
  pipeline templates).

## Related

- ADR 0011 — Three-layer testing with CLI orchestration (uses the same
  `agent-platform` CLI surface in CI).
- Architecture overview §5 (CI/CD integration kit summary), §8 (CI/CD
  integration requirements), §14 (B15 — GitHub Actions reference pipeline).
- Architecture backlog §2.17 (this decision), §3.14 (revisit trigger).
