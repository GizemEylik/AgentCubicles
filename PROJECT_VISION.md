# AgentCubicles Project Vision

## Product Identity

AgentCubicles is intended to become an independent Eylik Studio product,
not merely a rebrand or extended fork of Pixel Agents.

The long-term goal is a provider-independent, security-first workspace/runtime
for managing coding agents such as Claude Code, Codex, and future providers
from a common system.

## Independent Codebase Goal

The target state is an AgentCubicles codebase whose implementation and
architecture are independently owned and controlled by Eylik Studio.

Existing Pixel Agents-derived code must not simply be relabeled or cosmetically
refactored to achieve this goal.

Before migration, provenance must be established using repository history and
other evidence.

Components should be classified as:

- OWN: independently created AgentCubicles/Eylik Studio implementation
- UPSTREAM: Pixel Agents-derived implementation
- MIXED: contains both upstream-derived and new implementation
- UNCERTAIN: provenance cannot yet be established confidently

OWN components may be candidates for migration.

UPSTREAM components must not be copied into the independent implementation.

MIXED and UNCERTAIN components should not be directly migrated merely because
they contain later AgentCubicles changes. Required behavior should instead be
specified and independently implemented where appropriate.

Do not claim provenance or ownership without evidence.

## Architecture Goal

The new architecture must be designed from AgentCubicles product requirements,
not by reproducing the existing Pixel Agents directory structure under new names.

Core design goals include:

- provider-independent architecture
- explicit provider contracts
- Claude Code support
- Codex support
- extensibility for future providers
- security-first boundaries
- workspace and account isolation
- path and filesystem safety
- fail-closed provider/event routing
- testability
- cross-platform behavior
- clear separation between core runtime and provider-specific integrations

Existing implementations may inform requirements and behavioral specifications
where legally and technically appropriate, but upstream implementation should
not be mechanically translated or disguised through refactoring.

## Source Availability Goal

AgentCubicles is intended to have publicly visible source code.

Users should be able to inspect and download the source and, subject to the
eventual license terms, use and modify it for permitted non-commercial purposes.

Public visibility does not imply unrestricted commercial rights.

## Commercial Policy Goal

The intended licensing model should prohibit unauthorized commercial use.

The desired policy is broadly:

Allowed, subject to the final license:
- viewing source code
- downloading the project
- personal/non-commercial use
- non-commercial modification
- educational and research use
- contributing to the project

Not intended to be granted without a separate commercial license:
- selling AgentCubicles
- reselling modified versions
- offering AgentCubicles as a paid hosted service or SaaS
- incorporating AgentCubicles into a commercial product or service
- other unauthorized commercial exploitation

Eylik Studio should retain the ability to offer separate commercial licenses,
commercial products, hosted services, or future paid editions.

The exact legal license has NOT yet been selected.
Do not describe AgentCubicles as OSI-approved open source unless its eventual
license actually meets that definition. Until then, "source available" is the
safer description.

## Licensing and Provenance Rule

No final relicensing or ownership claim should be made until the provenance
audit and dependency/license review are complete.

Third-party dependencies and their licenses remain governed by their respective
license terms.

The project must preserve legally required notices for any third-party material
that remains.

The independent AgentCubicles implementation should avoid importing upstream
Pixel Agents implementation when the goal is exclusive control over the new
codebase's licensing.

## Development Strategy

Migration should be incremental and evidence-driven:

1. Audit provenance.
2. Extract product requirements and behavioral specifications.
3. Define independent AgentCubicles architecture.
4. Identify independently owned components that can safely migrate.
5. Independently implement required functionality for upstream/mixed areas.
6. Add focused tests during each increment.
7. Perform security and architecture review.
8. Perform final provenance and dependency/license audit.
9. Perform public-readiness and secret/history review.
10. Publish the new repository when those gates are satisfied.
11. Run strong cross-platform CI appropriate for a public source-available project.

Avoid blind repository-wide rewrites and giant migrations.

## Engineering Principles

- Security over convenience.
- Fail closed when identity or authorization is uncertain.
- Small, reviewable changes over giant rewrites.
- Evidence over assumptions.
- Focused tests before broad E2E runs.
- Provider-neutral core, provider-specific adapters.
- Preserve clean Git checkpoints.
- Separate implementation work from independent review.
- Do not sacrifice provenance clarity for migration speed.

## Current Decision

The existing repository is a transition/reference codebase.

No decision has yet been made that every existing AgentCubicles-era file is
independently owned or safe to migrate.

The immediate next step is a focused provenance audit of the newly introduced
Codex, provider-routing, security/isolation, migration, and related test layers.

The audit result will determine what can move into the independent
AgentCubicles architecture and what should be specified and reimplemented.
