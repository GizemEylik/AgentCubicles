# AgentCubicles Product Requirements

This document defines WHAT the independent AgentCubicles product should do:
product behavior, user-facing capabilities, functional requirements, security
expectations, and v1 scope.

It intentionally does NOT design software architecture, propose directory
structures, or prescribe classes, modules, interfaces, or schemas. See
`PROJECT_VISION.md` for ownership, provenance, source-availability, licensing,
and high-level product identity; this document stays consistent with it.

Product principle:

> AgentCubicles is designed as an orchestration, security, and observability
> layer for coding agents — not another coding agent.

## 1. Product Purpose

AgentCubicles should provide one central workspace for managing coding agents,
projects, tasks, permissions, activity, reviews, and handoffs.

Users should not need to manually coordinate multiple coding-agent terminals
and keep track of which agent is doing what in which repository.

AgentCubicles itself is not intended to replace Claude Code, Codex, or other
coding agents.

It is intended to coordinate them, observe them, constrain them, and provide
a consistent workflow around them.

## 2. Core Product Principles

The product must be:

- provider-independent
- security-first
- observable
- local-first for v1
- cross-platform
- explicit about agent state
- explicit about permissions
- conservative around destructive or external actions
- usable without requiring cloud infrastructure controlled by Eylik Studio
- extensible to future coding-agent providers

When information cannot be reliably determined, the UI must prefer "unknown"
over inventing or estimating authoritative-looking data.

## 3. Project and Workspace Management

Users must be able to register and manage multiple local development
projects.

For each project, AgentCubicles should be able to present relevant
information such as:

- project name
- local workspace path
- repository status when Git is available
- current branch when Git is available
- connected/available agent providers
- active tasks
- recent activity
- current agent states

Projects must remain logically isolated from one another.

An agent working on Project A must not silently gain access to Project B
through AgentCubicles.

Workspace boundaries must be visible, and enforceable to the extent
AgentCubicles can technically enforce them for the active platform and
provider.

## 4. Provider System

v1 must support:

- Claude Code
- Codex

The product model must not treat either provider as the definition of an
agent.

AgentCubicles should expose a consistent product experience across providers
while preserving provider-specific capabilities where necessary.

The product should be designed conceptually so additional providers can be
supported later without redefining the product.

Provider capabilities may differ.

AgentCubicles must not claim that a provider supports an action, state,
metric, or permission unless that capability can actually be determined or
enforced.

## 5. Task System

Task should be a first-class product concept.

A user should be able to create a task associated with:

- a project
- an agent/provider
- instructions
- optional role/workflow information
- applicable permission constraints

AgentCubicles must track task lifecycle explicitly.

The v1 lifecycle should include at minimum:

- QUEUED
- STARTING
- WORKING
- WAITING_FOR_USER
- REVIEWING
- DONE
- ERROR
- CANCELLED

Transitions must represent observed or reliably inferred product state rather
than fabricated precision.

The user should always be able to determine whether a task is:

- waiting
- actively working
- finished
- failed
- cancelled
- requiring user attention

## 6. Agent Status and Observability

AgentCubicles must make agent activity understandable without requiring the
user to inspect raw terminal output continuously.

The product should expose useful events such as:

- task started
- task state changed
- file read
- file modified
- command requested/executed when observable
- tests started/completed
- review requested
- review completed
- user approval requested
- task completed
- task failed
- task cancelled

The UI should provide an activity timeline/audit-oriented view.

Events must distinguish between:

- directly observed facts
- provider-reported information
- AgentCubicles-derived state

where that distinction matters for trust or security.

Do not promise observability that a provider cannot technically expose.

## 7. Permission and Security Model

Security is a core product feature, not an optional advanced setting.

AgentCubicles should support explicit permissions around agent actions.

The product should be capable of representing controls such as:

- read workspace files
- modify workspace files
- execute commands
- run tests
- create commits
- push Git changes
- delete files
- access the network
- access locations outside the registered workspace

Exact enforcement capabilities may vary by provider and operating system.

The UI must clearly distinguish:

- permissions AgentCubicles can enforce
- permissions that depend on provider behavior
- actions AgentCubicles can only observe

The product must never present an unenforceable permission as a guaranteed
security boundary.

Security-sensitive defaults should be conservative.

Access outside the active workspace should be denied by default where
AgentCubicles can enforce that boundary.

Destructive, externally visible, credential-sensitive, or difficult-to-reverse
actions should support explicit user approval where technically enforceable.

Examples include:

- pushing code
- deleting significant files
- accessing outside the workspace
- potentially destructive commands
- credential-sensitive actions

Approval concepts should include, where appropriate:

- allow once
- allow for this task
- deny

## 8. Builder → Reviewer Workflow

v1 should support a structured workflow in which one coding agent performs
work and another coding agent reviews it.

Example:

- Builder: Claude Code
- Reviewer: Codex

or the reverse.

A workflow should be able to represent:

- builder
- reviewer
- task
- review result
- bounded revision rounds
- completion/failure

Review output should support a structured convention such as:

- MUST FIX
- FOLLOW-UP
- INFORMATIONAL

The system must prevent uncontrolled agent-to-agent loops.

Review/revision workflows must have explicit limits such as maximum rounds
and must always remain interruptible by the user.

Agents must not be allowed to autonomously continue indefinitely merely
because another agent produced feedback.

## 9. Git Awareness

AgentCubicles should understand relevant Git state when a project is a Git
repository.

Useful information includes:

- current branch
- clean/dirty working tree
- changed files
- diff summary
- task-related changes when they can be determined reliably

Before significant automated work, the product should make existing
uncommitted changes visible to the user.

After a task, the user should be able to inspect resulting changes before
committing or pushing them.

Automatic commit and especially automatic push must NOT be the default
behavior.

AgentCubicles should favor reviewable checkpoints.

## 10. Context, Checkpoints, and Handoffs

AgentCubicles should reduce the need to carry extremely large conversation
histories between coding-agent sessions.

The product should support concise project/task checkpoints containing
useful continuation context such as:

- what was completed
- important decisions
- current state
- unresolved issues
- relevant files/areas
- recommended next step

A checkpoint must not pretend to contain information that was not actually
observed.

Users should be able to start a new agent session using a concise handoff
rather than requiring the entire historical conversation.

The system should preserve important decisions without encouraging unlimited
context accumulation.

## 11. Usage and Cost Visibility

Where a provider exposes reliable usage information, AgentCubicles should
make that information visible.

Possible data may include:

- usage
- tokens
- limits
- cost
- reset information

However, availability differs by provider.

AgentCubicles must not fabricate token counts, percentages, remaining quotas,
or cost estimates and present them as authoritative.

If reliable information is unavailable, show that it is unavailable or
unknown.

## 12. Cross-Platform Requirement

AgentCubicles should target:

- Windows
- macOS
- Linux

Cross-platform behavior is a product requirement, not merely a CI concern.

Platform-specific behavior must not silently weaken security guarantees.

Where a capability cannot be implemented consistently across platforms, the
difference should be explicit.

## 13. Local-First v1

v1 should primarily orchestrate agents and projects on the user's own
machine.

The core v1 experience should not require:

- an AgentCubicles cloud account
- Eylik Studio-hosted infrastructure
- mandatory cloud synchronization

This does not prohibit future cloud/team functionality.

Local credentials and provider authentication should remain under the user's
control wherever possible.

AgentCubicles should avoid becoming a credential vault unless a future
product decision explicitly introduces that responsibility.

## 14. User Control and Recovery

The user must remain in control of running work.

The product should provide appropriate ways to:

- stop/cancel a task
- deny an approval
- recover from provider failure
- understand why a task failed
- inspect resulting workspace changes
- continue work manually if orchestration fails

AgentCubicles must not create a situation where a user needs AgentCubicles
itself to recover their project.

Failure of AgentCubicles should not intentionally lock the user out of their
ordinary local repository or provider tools.

## 15. v1 Scope

v1 SHOULD include:

- multiple local projects/workspaces
- Claude Code provider
- Codex provider
- task creation and lifecycle tracking
- agent status
- observable activity timeline
- security/permission representation and enforcement where technically
  possible
- approval flow for sensitive actions where technically possible
- Git awareness and diff/status visibility
- Builder → Reviewer workflow
- bounded review rounds
- project/task checkpoints and handoffs
- reliable provider usage information when available
- Windows/macOS/Linux support

## 16. Explicitly Out of Scope for v1

The following should NOT block v1:

- Curiosity Mode
- autonomous background research
- unrestricted multi-agent meetings
- cloud synchronization
- team collaboration
- remote execution infrastructure
- agent marketplace/plugin marketplace
- mobile client
- enterprise administration
- billing/subscriptions
- AgentCubicles-hosted AI models

These may be considered for later versions.

## 17. Post-v1 Direction

Potential later capabilities include:

### Curiosity Mode

An idle agent may perform bounded research related to a project.

Any future Curiosity Mode must be constrained by controls such as:

- explicit enablement
- read-only project behavior by default
- time budget
- usage/cost budget
- network policy
- no autonomous code modification by default

### Multi-Agent Meetings

Multiple agents may collaborate using explicit roles such as:

- builder
- reviewer
- researcher

Any such workflow must have:

- bounded rounds
- bounded time
- bounded usage/cost where measurable
- explicit termination conditions
- user interruptibility

The product must never create unlimited autonomous agent conversations.

## 18. Non-Goals

AgentCubicles v1 is NOT intended to:

- build a new foundation model
- replace Claude Code or Codex
- become a general-purpose autonomous computer-use agent
- hide agent actions from the user
- automatically push every generated change
- provide fake precision around provider state or usage
- guarantee security controls that it cannot actually enforce
- reproduce Pixel Agents behavior merely for compatibility
- preserve old architecture for its own sake

## 19. v1 Success Criteria

AgentCubicles v1 should be considered successful when a user can:

1. Register multiple local projects.
2. Connect/use Claude Code and Codex within the supported provider model.
3. Start a bounded task for a selected project and agent.
4. Understand whether the agent is working, waiting, reviewing, done, or
   failed.
5. Observe meaningful activity without continuously watching a terminal.
6. Keep agent activity constrained to the intended project as far as the
   product can technically enforce.
7. Review sensitive actions before they occur where enforcement is
   available.
8. Inspect Git changes produced by a task.
9. Have one agent implement work and another review it in a bounded
   workflow.
10. End that work with a concise checkpoint that another session can
    continue from.
11. Stop the workflow and continue using the underlying repository/provider
    tools normally if AgentCubicles fails.

The product should accomplish this without requiring Eylik Studio cloud
infrastructure.

## 20. Requirement Integrity

This document defines product requirements, not implementation.

Do not use this document later as evidence that a security property is
already implemented.

Each security or observability guarantee must eventually be mapped to actual
technical enforcement and tests.

If implementation reality conflicts with a requirement, record the gap
rather than silently weakening the wording or pretending the requirement is
satisfied.
