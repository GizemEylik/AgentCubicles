# Provenance Decisions

Disposition record for migrating code into the new, independently-owned
AgentCubicles codebase. Based on the completed provenance audit (git
history, `git log --follow`, upstream-remote history search across all
`upstream/*` branches, and direct content/structure comparison against the
corresponding upstream-authored files) and subsequent Codex review. This
file records decisions only; see the audit for full evidence detail.

**Governing principle:** a file having no prior history outside a single
local commit ("new file in our fork commit") is not evidence of independent
authorship. Every item below was evaluated on pattern/structure/vocabulary
derivation, not on file-creation history alone.

## Decisions

| Path                                                                                                                   | Audit classification      | Decision                                                                                                                                                      |
| ---------------------------------------------------------------------------------------------------------------------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `server/src/providerEventHandler.ts`                                                                                   | MIXED                     | Rewrite from spec                                                                                                                                             |
| `server/src/providers/stream/codex/codexAppServerNormalizer.ts`                                                        | MIXED                     | Rewrite from spec                                                                                                                                             |
| `server/src/providers/stream/codex/codexAppServerClient.ts`, `codexAppServerProtocol.ts`, `codexAccountConfinement.ts` | UNCERTAIN / OWN-candidate | Do not copy directly. Extract requirements (JSON-RPC framing behavior, account-confinement gating rules, bounded-buffer limits) and reimplement independently |
| `server/src/providers/hook/codex/codex.ts`                                                                             | MIXED                     | Exclude from new product                                                                                                                                      |
| `server/src/legacyHomeMigration.ts`                                                                                    | OWN (audit)               | Exclude                                                                                                                                                       |
| `adapters/vscode/configFallback.ts`                                                                                    | OWN (audit)               | Exclude                                                                                                                                                       |

## Rationale by item

**`providerEventHandler.ts` — MIXED, rewrite from spec.**
No file-copy from upstream (no such filename anywhere in upstream history).
However it is a runtime dependent of `agentRuntime.ts`, a heavily
upstream-derived orchestration class (first appears at shared-ancestry
commit `6dfbf7e`, extended across many upstream PRs), and its broadcast
calls conform to `AgentStateStore` message shapes defined in that
upstream-derived lineage. Behavior may be retained; the implementation
should be rebuilt against independently designed state/event contracts,
not lifted as-is.

**`codexAppServerNormalizer.ts` — MIXED, rewrite from spec.**
New file, no upstream trace. Classified MIXED because it sits directly on
the `AgentEvent` union defined in `core/src/provider.ts`, whose
`StreamProvider` extension point was explicitly planned by an
upstream-authored `TODO` comment predating the fork (shared-ancestry commit
`cf3ef72`). The normalization behavior (raw Codex app-server message →
canonical event) may be retained as a requirement; the code should be
reimplemented against a newly, independently designed event contract
rather than the upstream-planned taxonomy.

**`codexAppServerClient.ts` / `codexAppServerProtocol.ts` /
`codexAccountConfinement.ts` — UNCERTAIN/OWN-candidate, do not copy
directly.**
No upstream trace and no structural borrowing from any upstream file was
found (no Claude/pixel-agents references; distinct JSON-RPC/app-server
protocol with no analog in upstream's hook/JSONL model). Despite the
absence of derivation evidence, these are large (~4,100 lines combined),
security-sensitive modules (account-confinement gating, bounded frame/
buffer limits) whose correctness cannot be fully re-verified from the audit
alone. Treat as an OWN-candidate but do not migrate verbatim: extract the
behavioral requirements (JSON-RPC framing, confinement-probe gating
semantics, byte-bounded buffering) and reimplement against the new
codebase's own contracts.

**`hook/codex/codex.ts` — MIXED, exclude from new product.**
New file, no upstream trace, but its `normalizeHookEvent` structurally
mirrors the upstream-authored `claude.ts` normalizer (same switch-on-
event-name shape, same validation idiom, same return contract) and reuses
Claude Code's own hook-event vocabulary (`PreToolUse`, `Stop`,
`SessionStart`, etc.) without independent confirmation that Codex CLI's
real hook API uses this shape. `installHooks`/`uninstallHooks` both throw
("not available in this build"), confirming this is unverified scaffolding,
not a working integration. Excluded rather than rewritten: there is no
confirmed Codex-native hook API in evidence to write a spec against yet.

**`legacyHomeMigration.ts` — exclude.**
Audited as OWN on structure/content grounds (distinct copy-based migration
strategy, no resemblance to upstream's `migrateVsCodeState.ts`). Excluded
regardless, because its entire reason for existing is migrating state from
the `.pixel-agents` legacy directory — a concern that is intrinsically
about this fork's Pixel Agents lineage and has no reason to exist in an
independently-originated codebase.

**`configFallback.ts` — exclude.**
Audited as OWN on structure/content grounds (novel VS Code
`inspect()`-based precedence utility, no upstream equivalent exists).
Excluded regardless, because it exists solely to reconcile a new
`agentcubicles.*` setting key against a legacy `pixel-agents.*` key — a
rebrand-specific concern with no counterpart in an independently-originated
codebase.

## Forward policy

The new AgentCubicles codebase will use independently designed event,
state, and provider contracts (its own `AgentEvent`-equivalent union, its
own state-store mutation/event API, its own provider interface taxonomy)
rather than carrying forward the upstream-planned `HookProvider`/
`StreamProvider` shape. Required behavior identified in this audit (event
normalization, provider routing, JSON-RPC transport, confinement gating) is
to be reimplemented from extracted specifications, not migrated by copying
files whose only qualification is having been created in this fork's
commits.
