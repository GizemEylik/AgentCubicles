# AgentCubicles

AgentCubicles is an independent Eylik Studio project building a
provider-independent, security-first workspace for coordinating coding
agents such as Claude Code and Codex from one place.

It is not another coding agent. It is intended to become the orchestration,
security, and observability layer that sits around the coding agents you
already use — so you stop manually juggling multiple agent terminals and
trying to remember which agent is doing what, in which repository, with
which permissions.

## Vision

The long-term goal is a single coordinated workspace where multiple coding
agents can work across your local projects while remaining visible and
controllable by you at all times. Rather than treating any one provider as
the definition of "an agent," AgentCubicles aims to expose a consistent
experience across providers, with explicit contracts for how each provider
plugs in.

Core commitments behind that vision:

- **Provider-independent** — Claude Code and Codex as first supported
  providers, with room for others later, none of them hard-coded as *the*
  agent.
- **Security-first** — workspace and account isolation, fail-closed
  behavior when identity or authorization is uncertain, and no pretending
  an unenforceable permission is a guaranteed boundary.
- **Human visibility and control** — agent activity, state, and permissions
  should be legible to the user, not hidden inside a terminal or asserted
  without evidence.
- **Local-first** — the core experience is meant to run on your own
  machine without requiring an AgentCubicles cloud account or Eylik
  Studio-hosted infrastructure.

## Planned capabilities

The following describes the intended v1 product direction. **None of this
is implemented yet** — see [Current status](#current-status) below.

- Registering and managing multiple local projects/workspaces, kept
  logically isolated from one another.
- Support for Claude Code and Codex as coding-agent providers, with an
  architecture designed to admit future providers.
- A first-class task system with an explicit lifecycle (queued, starting,
  working, waiting for user, reviewing, done, error, cancelled).
- An observable activity timeline that distinguishes directly observed
  facts, provider-reported information, and AgentCubicles-derived state.
- An explicit permission and approval model for sensitive actions (file
  writes, command execution, commits, pushes, network access, and access
  outside the registered workspace), with clear labeling of what
  AgentCubicles can actually enforce versus only observe.
- A Builder → Reviewer workflow, where one agent performs work and another
  reviews it, with bounded revision rounds and no uncontrolled
  agent-to-agent loops.
- Git awareness — branch, working-tree status, and diffs — with
  reviewable checkpoints rather than automatic commits or pushes.
- Concise project/task checkpoints and handoffs, so a new agent session can
  continue from a summary instead of a full conversation history.
- Visibility into provider usage and cost where a provider reliably exposes
  that information, shown as "unknown" rather than estimated when it does
  not.
- Cross-platform support for Windows, macOS, and Linux.

Further out, the project is also considering a constrained, opt-in
"Curiosity Mode" for bounded idle-agent research, and multi-agent workflows
with explicit roles — both gated behind strict time, cost, and
interruptibility limits, and both out of scope for v1.

## Current status

**AgentCubicles is at its clean-room foundation / pre-implementation
stage.** This repository currently establishes product vision, product
requirements, and licensing for an independently owned codebase.
Implementation has not yet begun, and nothing described above should be
read as available or working software today.

## Design principles

A few principles that shape how the product is meant to be built, not just
what it does:

- **Fail closed.** When identity, authorization, or provider state is
  uncertain, the product should not guess in the permissive direction.
- **No fabricated precision.** Task state, permissions, and usage/cost data
  must reflect what is actually known or enforceable — "unknown" is
  preferred over an invented number or guarantee.
- **The user stays in control.** Tasks must be stoppable, approvals must be
  deniable, and failure of AgentCubicles itself must never lock a user out
  of their own repository or provider tools.
- **Reviewable over automatic.** Automatic commits and pushes are not the
  default; changes are meant to be inspected before they leave the local
  workspace.
- **Provider-neutral core, provider-specific adapters.** Capabilities are
  expected to differ across providers, and the product should say so rather
  than claim uniform support it cannot back up.
- **Independent codebase.** The implementation and architecture are being
  built under Eylik Studio's own ownership and control, evidence-driven
  rather than by relabeling other code.

## Project documentation

For more detail than a README should carry, see:

- [`PROJECT_VISION.md`](PROJECT_VISION.md) — product identity, ownership
  and independence goals, architecture goals, and licensing direction.
- [`PRODUCT_REQUIREMENTS.md`](PRODUCT_REQUIREMENTS.md) — detailed
  functional and security requirements for the planned v1 product.

## Licensing

AgentCubicles is **public source-available software, not OSI-approved open
source.** The source is licensed under the
[PolyForm Noncommercial License 1.0.0](LICENSE).

In short: you may view, download, and use the software for personal,
educational, research, and other non-commercial purposes, and you may make
non-commercial modifications, subject to the full terms in
[`LICENSE`](LICENSE). The license does **not** grant rights to sell
AgentCubicles, resell modified versions, offer it as a paid hosted
service, or otherwise use it commercially. Commercial use requires a
separate agreement with Eylik Studio.

See [`LICENSE`](LICENSE) for the full legal text and [`NOTICE`](NOTICE)
for the required copyright notice.

## Attribution

AgentCubicles is developed by Gizem Eylik / Eylik Studio. See
[`NOTICE`](NOTICE) for the required copyright notice.
