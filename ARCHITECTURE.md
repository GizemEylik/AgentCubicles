# AgentCubicles Architecture v1

This document defines the first implementation architecture for
AgentCubicles. It is derived from `PROJECT_VISION.md`,
`PRODUCT_REQUIREMENTS.md`, and `CODEX_INTEGRATION_REQUIREMENTS.md`, from
first principles, independent of any prior codebase's structure.

This is a design document, not an implementation. Nothing described here
exists yet — see `README.md` for the repository's current status.

Scope discipline: this document designs enough to start small, reviewable
implementation work for the v1 scope in `PRODUCT_REQUIREMENTS.md` §15. It
does not design Curiosity Mode, multi-agent meetings, cloud sync, or any
other item in that document's §16/§17 beyond ensuring they are not
architecturally blocked (§16 below).

## 1. System Boundaries

**What AgentCubicles owns:**

- The local application (core runtime + UI) that a user runs on their own
  machine.
- Project/workspace registration and the boundaries between them.
- Work Item lifecycle, state, and history (§7).
- The execution contract and provider adapters (§3) — the code that talks
  to a coding-agent provider (process-backed or not, §3) on the product's
  behalf.
- The one authoritative Process Supervisor for every provider process it
  launches (§4).
- The permission/approval model and its enforcement, to the extent
  technically possible on the host OS and for the active provider's
  per-execution effective capabilities together with their corresponding
  live state (§3, §11).
- Any isolated environment (env vars, home/temp directories, on-disk state)
  constructed for a provider process, per the core/adapter split in §6.
- Observability: turning provider-reported facts and directly-observed
  process facts into a human-legible activity record (§10).
- Git awareness (reading repository state; never committing/pushing on the
  user's behalf by default).

**What providers own:**

- Their own internal reasoning, tool use, and whatever session/thread
  model they may or may not have (§3, §7) — the core neither assumes nor
  manufactures one.
- Their own protocol implementation and wire behavior.
- Their own authentication/account state (AgentCubicles can observe and
  gate on it where a provider has an equivalent to Codex's compliance
  check, §3, but does not own or control it).
- Whatever isolation, if any, they provide independently of AgentCubicles.
  This must never be assumed — AgentCubicles treats its own containment as
  the only boundary it can rely on, and states exactly how much that
  containment actually covers (§5) rather than overstating it.

**Trust boundaries:**

- A provider is untrusted input once active: for a process-backed
  Execution, its spawned child process's stdio and reported state; for a
  non-process Execution, whatever equivalent protocol input its execution
  driver receives (§3). In either case, its reported state and its
  requests to AgentCubicles (e.g. approval requests, where a provider has
  them at all) must all be validated. A protocol violation or malformed
  message must never be able to crash or confuse the core runtime.
- The workspace filesystem boundary AgentCubicles can actually enforce is
  exactly what §5 states: canonical launch-directory validation (Level 1)
  and, where a provider supports it, action-level mediation (Level 2).
  **This is not OS-level containment** — an already-running provider
  process is not prevented from accessing an absolute path outside the
  workspace at the OS level in v1.
- The user is the trust root for approvals. No component may synthesize an
  approval on the user's behalf, and no persisted preference may
  substitute for a live decision (§8).
- Between projects and between Executions: state, environment, and session
  ownership belonging to one project/Execution must be structurally
  unreachable from another (§6, §7). This is an internal boundary
  AgentCubicles enforces on itself, not something delegated to the OS.

**What remains outside the system (v1):**

- Any Eylik Studio cloud account, hosted infrastructure, or mandatory
  synchronization (`PRODUCT_REQUIREMENTS.md` §13).
- OS-level sandboxing/containerization beneath the provider process (§5,
  Level 3 — explicitly not provided in v1).
- Foundation models themselves — AgentCubicles calls out to existing
  coding-agent provider tools/integrations; it does not host or train
  models.

## 2. Major Components

Kept as logical responsibilities, not mandated services or even
necessarily separate source directories — see §17 for how loosely these
map onto an actual initial layout.

1. **Project/Workspace** — registers local projects, enforces that a
   project's identity/path is unambiguous, and (folded in for v1) provides
   read-only Git inspection (branch, dirty/clean, changed files, diff
   summary) for a registered project.
2. **Provider Registry & Adapters** — a lookup from provider id to adapter
   (may begin as a plain map), plus one adapter per supported provider.
   Adapters are the only place provider-specific protocol/session/auth
   knowledge lives.
3. **Process Supervisor** — the single authoritative owner of spawn/exit
   facts for every OS child process AgentCubicles launches, and the sole
   issuer of the controlled execution/process transport boundary each
   process-backed adapter uses for its own provider-protocol work (§4).
   Only process-backed Executions use this component (§3).
4. **Execution Manager** — coordinates one Execution (§7) against a
   provider adapter. For a process-backed Execution: validates the
   adapter's declarative launch requirement and requests a process from
   the Process Supervisor (§4), applies the environment/workspace
   isolation plan (§6), and is the sole caller of destructive cleanup,
   gated strictly on Process Supervisor facts (§4, §11). For a verified
   non-process Execution: applies the same core-owned authority boundary
   at whatever narrower scope that provider's driver requires (§3). In
   both cases it tracks per-execution effective capability results and
   their live-state bindings (§3), and remains the sole lifecycle
   mutation authority no adapter may bypass.
5. **Work Item Manager** — the sole authority for the provider-neutral Work
   Item and its role-scoped Executions (§7), including the v1 lifecycle
   states from `PRODUCT_REQUIREMENTS.md` §5, deterministic state
   derivation (§7), and restart reconciliation (§8).
6. **Permission & Approval Engine** — evaluates whether a requested action
   is allowed, denied, or needs a live user decision; owns live approval
   authority (§8) — never persisted beyond what §8 permits.
7. **Activity Record** — a small append-only record and subscription
   interface (§10), not a generalized event-bus framework — used by every
   other component to record and expose what happened.
8. **Persistence** — one local, file-based store shared by Project/
   Workspace, Work Item (including checkpoints — no separate Checkpoint
   Store), and Activity Record history (§12).
9. **Workflow Coordination** — the bounded Builder→Reviewer sequencing on
   top of Work Item roles, owning each workflow's current round and
   `maxRounds` bound (§9); a thin, later v1 increment (§17, §19), not a
   parallel task system.

The UI is deliberately not listed as a component here: its process
topology (embedded in the same process, or a separate host process) is an
implementation choice not yet made. The architecture only defines what it
consumes (§15).

No dedicated database service, message queue, or network service exists in
v1 — see §12 and §13.

## 3. Provider Abstraction: the Execution Contract

**Design stance.** Nothing in the authoritative inputs establishes that
Claude Code has a persistent process-backed run, a protocol
handshake/readiness phase, a native session concept, an approval
mechanism, an account/session compliance check, unit-scoped cancellation,
or the same shutdown mechanics as Codex. `CODEX_INTEGRATION_REQUIREMENTS.md`
documents all of those as *Codex-specific*, several explicitly marked
UNVERIFIED EXTERNAL ASSUMPTIONS even for Codex itself. The contract below
is the smallest thing every provider can safely be assumed to have, and
everything else is an optional, per-execution capability.

**Assumed universal (and why it is safe to assume) — genuinely true for
every provider regardless of execution mode:**

- An Execution eventually ends, in one of a small number of ways
  (completed, failed, or cancelled/interrupted). What "completed" means is
  provider-defined; the core does not require an explicit success/failure
  protocol message to exist.
- An Execution can, in principle, be asked to stop. Whether that request
  is graceful, forceful-only, or has no adapter-mediated mechanism at all
  is a per-provider capability, not a guarantee; for a process-backed
  Execution, the ultimate fallback is the Process Supervisor's forceful
  termination (§4) — a non-process Execution's equivalent fallback, if
  any, belongs to its own execution-driver boundary ("Execution lifecycle
  source," below), not this architecture's process-backed path.

**Process-backed execution: a provisional, per-provider hypothesis — never
a universal assumption.** Whether any given provider is process-backed is
never assumed; it is established independently from verified provider
evidence (§19 step 4), per provider, before that provider's adapter is
built. Both named v1 providers are *believed* to be CLI/process-backed,
but no integration fact for Claude Code — including whether it is
process-backed at all — has been verified anywhere in the authoritative
inputs (§20), and even Codex's own invocation shape is an unverified
assumption sourced only from a prior implementation's comments (§20).
**Process Supervisor participation (§4) applies only to an Execution
whose verified execution mode is process-backed.** A verified non-process
Execution instead uses the already-defined non-process lifecycle-evidence
boundary ("Execution lifecycle source," below) — it never uses the
Process Supervisor, and nothing about it is inferred from the
process-backed path.

**Explicitly NOT assumed universal — each is an optional capability,
resolved per execution, never assumed present:**

- A protocol handshake or a distinct "ready" phase before work can begin.
  Where a provider has one (Codex does), the adapter may report it; where
  it doesn't, "started" and "working" may be the same moment. For a
  process-backed Execution, "started" may include spawn/readiness/
  handshake behavior (§4); for a verified non-process Execution, its own
  validated startup/readiness mechanism (§3, "Execution lifecycle source")
  reports the equivalent moment without fabricating a spawn or process
  concept it does not have.
- A native session/thread concept. Where a provider has one, it stays
  entirely inside the adapter as an opaque handle — the core never sees a
  session id and never manufactures one for a provider that has none
  (§7).
- Approvals. A provider without an approval concept simply never offers
  one; the Permission & Approval Engine's live-authority machinery (§8) is
  never invoked for that provider.
- An account/session compliance check. Codex has one; nothing establishes
  Claude Code has an equivalent. Where a provider has no such check,
  "able to do task-capable work" is not gated on one.
- Cancellation of a specific in-progress unit by id. Some providers may
  only support ending the whole Execution.
- Identical shutdown mechanics. Each adapter defines its own "how do I ask
  this Execution to stop gracefully," expressed through its verified
  execution mode's stop authority — the Process Supervisor path (§4) for a
  process-backed Execution, or the already-defined non-process driver/
  lifecycle authority path (§3, "Execution lifecycle source") for a
  non-process Execution. The only universal part is that *some* form of
  stop request must exist for every Execution; a forceful, process-level
  fallback is never assumed universal — it is specific to the Process
  Supervisor path and has no equivalent claimed for a non-process
  Execution.

**Capability lifecycle.** Three distinct notions must never be conflated:

1. **Pre-launch/static verified integration facts** — what an adapter has
   independently verified about its provider's implementation (protocol
   shape, declared optional features) before any Execution starts. These
   are facts about the provider/adapter, not yet about any specific
   Execution.
2. **Runtime negotiation/discovery** — capability information obtained
   during one specific Execution's startup (e.g. a handshake response, a
   negotiated feature set) from validated evidence actually produced by
   that Execution's provider or runtime — not necessarily evidence from a
   provider process; a non-process Execution's execution driver (§3,
   "Execution lifecycle source") produces the equivalent evidence for a
   verified non-process provider.
3. **Effective capabilities for one immutable Execution** — the capability
   result actually bound to that Execution's identity (§7), computed from
   (1) and (2). Effective optional capabilities begin **unavailable/
   unknown** for a new Execution and become available only when
   independently verified before or during that Execution's launch — never
   inherited from a prior Execution, a static provider-level claim, or an
   adapter's self-report alone.

Optional operations require **both** halves, never one alone, and this
universal rule never names the Process Supervisor as part of it:

- the **effective capability** being present for this specific Execution,
  and
- **the authoritative live state required by this Execution's verified
  execution mode**, confirming the operation's target still exists and is
  currently owned by this Execution.

For example: `resolveApproval` requires the effective approval capability
**and** a currently registered pending approval owned by this Execution
(§8, §11 AC-CODEX-SEC-025); `interrupt` requires the effective interrupt
capability **and** a currently owned live unit of work. A graceful
provider-protocol stop requires the effective capability **and**
mode-specific live-state confirmation: for a **process-backed** Execution,
the Process Supervisor's currently supervised execution/process record for
this Execution (§4); for a **non-process** Execution, the authoritative
live state held by that Execution's verified non-process execution-driver
authority (§3, "Execution lifecycle source") — never the Process
Supervisor, which is never consulted for a non-process Execution.

**There is exactly one rule for how a claim becomes a capability: a
provider's or adapter's self-report or runtime claim is never, by itself,
sufficient to establish an effective capability — it is evidence INPUT
only.** It becomes an effective capability for this immutable Execution
only after the independent validation/verification this architecture
already requires has accepted that evidence (capability negotiation
actually completing, per item 2 above; a version/platform check the
Execution Manager performs; or an equivalent core-owned acceptance step).
The live-state half is a separate, second requirement on top of that
validated capability, always independently confirmed by the component
that actually owns that state (Process Supervisor for process/transport
facts, Permission & Approval Engine for pending approvals, Execution
Manager for active work units) — never satisfied by the same claim that
produced the effective capability.

**The execution contract** (conceptual, not a code signature):

- `capabilities(context)` — returns a **capability result**, not just
  static metadata. It is computed per execution from actually
  observed/verified evidence available at that moment (installed provider
  version, a completed protocol negotiation, platform, product
  configuration) wherever a capability can vary by any of those. A
  capability that is genuinely fixed for a provider may be reported
  statically; anything that can vary must be re-resolved per execution
  rather than trusted from a one-time static declaration. The Execution
  Manager and Permission Engine consult this before ever assuming a
  provider can enforce, report, or accept something
  (`PRODUCT_REQUIREMENTS.md` §4/§7).
- `prepareLaunch(workItemRef, role, workspace, isolationPlan) →
  LaunchRequirement` *(process-backed path step A, §4)* — the adapter's
  only pre-spawn responsibility: producing a purely declarative launch
  requirement (command/args, its isolation-plan request, and the stop
  descriptor used in §4 steps G–H). The adapter does not spawn, attach to,
  or otherwise touch a process here. For a verified non-process Execution,
  the equivalent adapter-specific preparation step feeds the narrower
  core-owned execution-driver boundary described under "Execution
  lifecycle source" below, not this process-backed path.
- `attach(boundary) → StartOutcome` *(process-backed path step E, §4)* —
  once the core has validated the launch requirement and the Process
  Supervisor has spawned the process (§4 steps B–D), the Execution Manager
  hands the adapter the resulting controlled transport boundary; the
  adapter performs all of its own provider-protocol startup and
  work-request logic exclusively through that boundary (§4 steps E–F),
  returning once the provider has either begun producing work or failed to
  start. The adapter has no path to a process except through this
  boundary — it is never given the `ProcessHandle` the Process Supervisor
  holds internally (§4). The core does not require an intermediate "ready"
  state to exist.
- **Graceful stop has exactly one defined mechanism and is not a separate
  adapter-invoked contract call**: the declarative stop descriptor the
  adapter supplied inside its `prepareLaunch` launch requirement, executed
  entirely by the Process Supervisor (§4 steps G–H). If the adapter
  supplied no descriptor, only the Process Supervisor's forceful
  termination is available. See §4 for the complete mechanics; they are
  not restated here.
- `interrupt(executionId, unitRef)` *(optional capability)* — cancel one
  in-progress unit of work without ending the whole Execution; offered
  only if the provider has a meaningful unit-scoped cancellation concept.
- `resolveApproval(executionId, approvalRef, decision)` *(optional
  capability)* — exists only for a provider whose capability result
  declares approval support for this execution, and only ever acts on a
  currently registered, pending approval owned by this Execution
  (capability lifecycle rule above).
- **Optional-operation reference model.** Every optional provider
  operation — `interrupt`, `resolveApproval`, and any provider-protocol
  operation an adapter exposes — is invoked with the immutable Execution
  identity (`executionId`, §7) already used throughout this architecture,
  never a "handle" of any kind. The adapter uses that identity to resolve
  its own internal, Execution-scoped provider context (e.g. which native
  session, if any, belongs to this Execution) entirely inside itself; the
  core never receives, stores, or passes through a provider-native session
  handle, and never receives the raw `ProcessHandle` the Process
  Supervisor holds (§4). This is the smallest model that removes the
  ambiguity: no separate `ProviderExecutionRef` type is introduced,
  because Execution identity alone is already sufficient for the adapter
  to do this resolution.
- an outbound stream of whatever verified facts the adapter can actually
  produce (§10) — not a mandatory fixed taxonomy.

**Execution lifecycle** (the only lifecycle the core imposes on every
provider): *requested → running → ended (completed / failed /
cancelled)*. "Requested" begins at the adapter's preparation step for that
Execution's verified execution mode: `prepareLaunch` for a process-backed
Execution (§4 step A); the equivalent adapter-specific preparation step
feeding the non-process execution-driver boundary (§3, "Execution
lifecycle source") for a verified non-process Execution. "Running" does
not imply "ready" or "past a handshake." Any additional phase (a
handshake, a compliance verdict, a negotiated capability set) is
provider-specific detail the adapter may track and expose through its own
capability/fact reporting, layered on top of this lifecycle — never a
precondition the core enforces identically for every provider.

**Execution lifecycle source (process-backed vs non-process).** Process
lifecycle mechanics, Process Supervisor participation, child-process
environment injection (§6), the controlled transport boundary (§4), and
process-specific concurrency partitioning (§13) apply only to
**process-backed Executions** (the provisional v1 case for both named
providers, above). For a process-backed Execution, "running" means the
Process Supervisor has confirmed the process is alive and the adapter has
not reported a start failure; authoritative running/ended evidence comes
from the Process Supervisor together with whatever authoritative
core/provider-protocol outcome is relevant (e.g. a compliance verdict, a
protocol-reported completion).

For a verified non-process Execution, its adapter-specific execution
driver would report running/ended evidence through a narrow, core-owned
authority boundary analogous to §4's transport boundary — but with no OS
process, no Process Supervisor participation, no child-process
environment injection, and no process-specific concurrency partitioning.
The Execution Manager remains the lifecycle mutation authority in either
case: a non-process adapter reports evidence, it never mutates Work Item
or Execution state directly (§10). This document does not design that
boundary's concrete mechanism now; doing so before a real non-process
provider requires it would be exactly the kind of speculative
infrastructure `PROJECT_VISION.md`'s evidence-driven principle rejects —
it is deferred (§20).

**Codex specifically:** every "External Codex/App Server Fact" in
`CODEX_INTEGRATION_REQUIREMENTS.md` §2 — including the launch invocation,
wire framing, method vocabulary, handshake order, and the Account/
`GetAccountResponse`/`PlanType` shape — is marked UNVERIFIED EXTERNAL
ASSUMPTION there and remains so here; it belongs entirely inside the Codex
adapter and must be independently re-verified before that adapter is
built (§20). The account/session compliance *mechanism* (a check must
exist and be positively evaluated, fail closed on anything unclassifiable,
be scoped to exactly one execution) is real for Codex and is expressed as
an optional capability above — never assumed for any other provider. The
current pilot's concrete eligibility rule (§5.2 of that document) is
explicitly provisional policy, not something this architecture encodes.

**Claude Code specifically:** no integration facts for Claude Code have
been verified anywhere in the authoritative inputs. Nothing here should be
read as implying Claude Code lacks sessions, approvals, a compliance
concept, or even a process-backed execution mode — only that nothing
establishes it has them either. This must be resolved by the verification
step in the implementation sequence (§19) before Claude Code's adapter is
built, exactly as Codex's facts must be independently re-verified before
its adapter is built.

**Contract stabilization.** The shared execution contract above stabilizes
through one required sequence, never a shortcut through it: provider
verification (§19 step 4) → a narrow first-provider slice (§19 step 7) →
a **PROVISIONAL** revision of this shared contract (§19 step 8) →
second-provider verification/slice against that provisional revision (§19
step 10) → revising the shared boundary again whenever the second
provider's evidence requires it → only after **both** providers have
supplied architectural evidence is the v1 shared contract considered
stable. The second provider's integration is architectural evidence about
whether the contract generalizes, never merely a conformance test against
an already-decided shape. This document does not describe the contract
above as final, and no later section may describe it as finalized after
only the first provider.

**Open question (not resolved here):** whether v1 needs to support a
provider that is not process-backed at all (a pure network API). Nothing
in `PRODUCT_REQUIREMENTS.md` requires this for Claude Code or Codex, so
the process-backed hypothesis above (per provider, never universal, see
above) is what v1 currently expects for both named providers. **The
Process Supervisor remains process-only regardless of how this question
resolves — it never grows a non-process execution path.** If a provider
breaks the process-backed hypothesis, the already-defined, Execution-
Manager-owned non-process driver/lifecycle-evidence boundary ("Execution
lifecycle source," above) is where that provider's execution/lifecycle
evidence path belongs, designed only when a real non-process provider
actually requires it (§20) — not an expansion of §4's Process Supervisor
scope, and not an invitation to design detailed non-process infrastructure
beyond that boundary now.

## 4. Process Authority

This section defines process authority for **process-backed Executions**
(§3). A verified non-process Execution does not use the Process
Supervisor, the launch requirement below, or the controlled transport
boundary — see §3's "Execution lifecycle source" rule for its (currently
undesigned, deferred) equivalent.

**Exactly one authoritative process supervisor.** The Process Supervisor is
the only component that calls the OS to spawn a process, the only
component that resolves any opaque reference to the raw OS process object,
and the only component that observes an exit. Every fact that gates a
destructive action is read only from the Process Supervisor — never
inferred from adapter state, never inferred from the *absence* of an
adapter-reported failure, and never inferred from an empty Process
Supervisor registry after a restart (§8).

**Two things stay Supervisor-private and are never handed to the
Execution Manager or the adapter: the raw OS `ProcessHandle`, and the act
of writing a graceful-stop message.** Everything the Execution Manager and
the adapter each need instead comes from two narrower, purpose-scoped
objects the Supervisor issues on spawn — an opaque **supervised
reference** (Execution-Manager-facing, used only to ask the Supervisor to
act) and a **controlled transport boundary** (adapter-facing, used only
for the adapter's own provider-protocol I/O). Neither object exposes the
raw handle or lets its holder write a stop message.

**The complete process-backed execution path (adapter → core → Process
Supervisor):**

1. **(Step A)** The adapter's `prepareLaunch()` (§3) produces a
   declarative **launch requirement**: the command/args needed to invoke
   its provider, its isolation-plan request (§6), and — where the
   provider supports one — a declarative **stop descriptor** describing
   how a graceful stop should be attempted — e.g. `{ kind: 'signal',
   value: <platform signal> }`, `{ kind: 'stdin-write', payload: <bytes>
   }`, `{ kind: 'protocol-message', ... }`, or `{ kind: 'none' }`. This is
   data, never an executable callback: the adapter cannot hand the core
   arbitrary code to run on its behalf, and it never writes this message
   itself — see step 6/G–H. This is the **only** graceful-stop mechanism
   defined anywhere in this architecture; no other adapter-invoked stop
   call exists.
2. **(Step B)** The Execution Manager validates the launch requirement
   against workspace enforcement (§5), the core-owned host-input/
   environment policy (§6), state-root policy (§6), and resource/transport
   policy (§11) before any process exists. A launch requirement that fails
   validation is rejected; nothing is spawned.
3. **(Step C)** Once validated, the Execution Manager — never the adapter
   — builds the single canonical **`ValidatedProcessLaunch`** object from
   the validated launch requirement and calls the Process Supervisor's
   `spawn()` with it. `ValidatedProcessLaunch` is the one input contract
   `spawn()` accepts; it carries: `command`, `args`, `environment`, `cwd`
   (all already validated per step B), `resourcePolicy` and
   `transportPolicy` (§11), any state/isolation inputs applicable to this
   launch (§6), and an optional `gracefulStopDescriptor` — the launch
   requirement's already-validated stop descriptor from step A, present
   only when the adapter supplied one. The Execution Manager never
   constructs or passes these fields separately from this object.
4. **(Step D)** On success, the Process Supervisor returns, to the
   Execution Manager only, exactly two things: an opaque **supervised
   execution/process reference** and a **controlled transport boundary**.
   **Neither is the raw `ProcessHandle`**, which the Supervisor keeps
   entirely to itself; the Supervisor alone can resolve the reference back
   to that internal handle. The Execution Manager keeps the reference for
   its own later Supervisor calls (step 6) and passes only the transport
   boundary to the adapter's `attach()` call — the adapter never receives
   the supervised reference.
5. **(Steps E–F)** The adapter's `attach(boundary)` call (§3) performs its
   own provider-protocol startup and work-request logic — handshake,
   session/thread management, requesting work, streaming results —
   exclusively through that boundary. **The adapter never writes a
   graceful-stop message and never receives the supervised reference**; it
   has no path to the process except the boundary's protocol I/O, and no
   role in the decision of whether or when to escalate to a forceful kill.
6. **(Steps G–H)** When a graceful stop is needed, the Execution Manager —
   never the adapter — calls the Process Supervisor's
   `requestGracefulStop(ref)` using only the opaque reference from step D.
   The Supervisor resolves that reference to its internal handle and is
   the one that executes the already-validated stop descriptor from step
   A — this is the **only** place in this architecture a graceful stop is
   actually written. It then waits a bounded grace period, escalates to
   forceful termination if the process has not exited (or immediately, if
   the provider supplied no descriptor or graceful stop is unsupported),
   and waits a second bounded window for the exit to be positively
   observed (`onExit(ref)`). **Process Supervisor remains the sole
   authority for process lifetime, the stop deadline, and escalation at
   every step; neither the adapter nor the Execution Manager owns or
   duplicates any part of this.**

**Adapters must not:** spawn a process directly; write a graceful-stop
message or otherwise independently attempt to stop the process; mutate
arbitrary environment state; request or obtain unrestricted host
filesystem/state inputs (§6); bypass resource or transport policy (§11);
or become authoritative for spawn/exit facts. Every one of these stays
exclusively with the Execution Manager and Process Supervisor.

- `spawn(launch: ValidatedProcessLaunch) → { ref, boundary } | SpawnFailure`
  — the sole spawn input contract in this architecture (step C above); no
  other `spawn()` signature exists. The OS spawn call either produced a
  live process, in which case the Supervisor privately holds its raw
  `ProcessHandle` and externally returns only the opaque `ref` and
  `boundary` described above, or it did not (`SpawnFailure`, meaning
  nothing exists to terminate). If `launch.gracefulStopDescriptor` was
  present, the Supervisor retains it internally alongside the raw process
  state for later use in step 6/G–H — it is never returned or exposed to
  the Execution Manager or adapter. This is exactly the fact
  `AC-CODEX-SEC-016`'s "pre-spawn" case requires, generalized to every
  process-backed provider.
- `requestGracefulStop(ref)` — Execution-Manager-facing only; described
  fully in step 6/G–H above and not restated here.
- `onExit(ref) → ExitFact` — a positively observed exit code/signal and
  timestamp, keyed by the same opaque reference. This is the only thing
  that can ever mark a process "confirmed terminated."
- The Supervisor holds no provider knowledge and imports nothing from any
  adapter — it is generic process management, usable identically for a
  trivial test process (§19 step 3) and for a real provider process.

**Cleanup authorization depends only on Process Supervisor facts.**
Destructive cleanup of an Execution's isolated on-disk state (§11) is
authorized in exactly two cases, both Process Supervisor facts, never
adapter state:

(a) `spawn()` returned a `SpawnFailure` for this execution — there is
nothing to terminate, so pre-spawn cleanup may proceed immediately; or

(b) `onExit()` has positively reported this execution's process as
terminated.

If an adapter's own startup logic (handshake, compliance check, capability
negotiation) fails *after* `spawn()` succeeded, the process may still be
alive — cleanup must wait for case (b), exactly as it would for any other
still-running process. The Execution Manager's response to an
adapter-side startup failure is to call `requestGracefulStop(ref)`,
escalating as needed, and only proceed with cleanup once `onExit()`
confirms it — never based on "the adapter said startup failed." An empty
Supervisor registry after an application restart is not case (b) either —
see §8.

## 5. Workspace Enforcement Levels

This section states the actual, honest workspace containment levels v1
provides. Canonical working-directory validation stops AgentCubicles
itself from launching a provider into the wrong directory (including via
a symlinked/junctioned workspace registration); it does nothing to stop
an already-running, unsandboxed provider process from opening an absolute
path outside that directory if it chooses to and the OS permits it. v1
does not provide OS-level sandboxing (§1), so neither this document nor
the product may claim a containment guarantee it does not have.

- **Level 0 — Project attribution (internal bookkeeping only).** Every
  Execution is tagged with the project/workspace it was started for. This
  is pure internal record-keeping with no enforcement effect on the
  provider process by itself.
- **Level 1 — Canonical launch-directory validation (what v1 actually
  enforces before launch).** Before `spawn()` is called, the working
  directory handed to the provider is proven, via canonical filesystem
  path resolution — not lexical string comparison — to be the registered
  workspace root or a descendant of it; a symlink/junction that would
  resolve outside the allowlisted root is rejected after resolution
  (`AC-CODEX-SEC-009`, generalized to every provider). This guarantees
  AgentCubicles never *launches* a provider into the wrong place. It says
  nothing about what the provider does once running. This level, as
  stated, governs process-backed launches (the working directory handed to
  a spawned provider process, §4 step B); a verified non-process
  provider's equivalent workspace-scoping enforcement point is defined
  when that provider's adapter is designed (§3, §19), applying the same
  canonical-validation principle.
- **Level 2 — Provider/action mediation (capability-gated, not
  guaranteed).** If, and only if, a provider adapter declares a capability
  to intercept or gate a specific action (a file write, a command) before
  it happens, the Permission & Approval Engine (§8, §11) can require
  approval or deny it. This level exists only where a provider's own
  protocol actually offers a mediation point; it is never assumed
  present. **Level 2 mediation controls only the specific action instances
  actually routed through that verified mediation point** — it is not
  proof that the provider process lacks an independent, unmediated path
  (e.g. direct filesystem access, or a subprocess it spawns on its own) to
  perform an equivalent operation outside that mediation.
- **Level 3 — OS-enforced containment. Not provided in v1.** AgentCubicles
  does not run provider processes inside a container, restricted
  filesystem namespace, or any OS-level sandbox. A provider process has
  the same filesystem and network access as the OS user account running
  it, limited only by what that account itself can access — not by
  anything AgentCubicles adds.

**Honesty requirement.** Neither this document, the product UI, nor any
user-facing text may state or imply that workspace registration prevents a
provider process from reading or writing paths outside the workspace at
the OS level, or that Level 2 mediation prevents an unmediated action a
provider process performs independently. What v1 actually provides is
Levels 0–1 always, and Level 2 only where a specific provider capability
supports it and only for the actions actually routed through it. Closing
the gap to Level 3 (OS sandboxing, a container boundary, a restricted
execution environment) is a possible future investment, tracked as a
deferred decision (§20), not a v1 claim.

## 6. Environment / State Isolation Responsibility

This section governs **process-backed** Executions — the OS child-process
environment, home/temp directories, and on-disk state described below. A
non-process Execution has no child-process environment to construct;
whatever core-supplied input surface it needs (credential material,
product-owned state) would be governed by the same default-deny classes
and validation rules in principle, with its concrete mechanism deferred
until such a provider is designed (§3, §20).

**Default-deny, scoped to what AgentCubicles itself supplies.** No
environment variable, file, or credential is **injected or disclosed by
AgentCubicles** into a provider child process unless it is explicitly
permitted by core policy and the concrete input passes validation. There
is no denylist-only escape hatch for AgentCubicles-supplied input: a
variable's mere absence from a "known-dangerous" list is never sufficient
authorization to pass it through — `CODEX_INTEGRATION_REQUIREMENTS.md` §6
requires exactly this ("an explicit, minimal allowlist ... never
everything the host process has, minus a denylist"), generalized to every
process-backed provider, not only Codex. At the same time, a fully generic
core component cannot know every provider's own configuration,
authentication, or subordinate-tool isolation semantics — that knowledge
is legitimately provider-specific, which is why the responsibility below
is split rather than assigned entirely to one side.

**This governs AgentCubicles's own grants — it is not a claim about what
an unsandboxed process can independently discover.** Without Level 3
OS-enforced containment (§5), a provider process has the same filesystem
and environment-inspection access as the OS user account running it.
AgentCubicles cannot prevent it from independently opening a credential
file elsewhere on disk, or a sibling/parent directory of one AgentCubicles
did grant, through the process's own OS-level access rather than through
anything AgentCubicles supplied. Granting one narrowly-scoped credential
file path never implies AgentCubicles will additionally grant its parent
or sibling paths, but it also never *prevents* the process's own
independent ability to read those paths if the OS otherwise permits it —
closing that gap is exactly what Level 3 (§5, not provided in v1), not
this section, would need to do. The correct, honest guarantee this section
provides is: **AgentCubicles does not itself inject, disclose, or
authorize broader host input than the explicitly supported classes
below.**

**Core owns (provider-independent, applies to every adapter):**

- **Policy: a small set of explicitly supported input classes**, defined
  narrowly:
  - an explicitly approved credential/environment variable — named,
    provider-specific, individually declared in core policy;
  - an explicitly approved credential file/path — a specific, narrowly
    scoped file location, never a directory grant;
  - product-owned state — AgentCubicles's own state-root boundary, below;
  - any other provider-specific input, only after a deliberate
    architecture/policy extension adds a new class — never granted ad hoc
    to satisfy one adapter's request.
  An adapter's isolation-plan request that does not fall into an
  explicitly supported class is rejected outright, regardless of what the
  provider process itself might request or claim to need.
- **Validation for every allowed input**, as applicable: a **verified
  provider need** (established per §19 steps 4/6/7, never assumed); **scope
  validation** (the narrowest input that satisfies that need — never "the
  whole environment" or "the whole home directory"); **canonical/path
  validation** (§5's Level 1 technique, applied identically here —
  symlink/junction indirection resolved before acceptance); **minimal
  exposure** (the smallest concrete value that satisfies the need);
  **relevant product/user authorization** (the user or product
  configuration has explicitly approved this class for this provider); and
  **no expansion to sibling/parent directories or unrelated variables in
  what AgentCubicles itself grants** (a credential file grant does not
  imply AgentCubicles will additionally grant its parent directory; an
  approved variable name does not imply approval for a similarly-named or
  related one — this bounds AgentCubicles's own grants only, per the
  Level 3 connection above, not the process's independent OS-level
  access).
- **Fail closed on missing authentication.** If a provider's verified
  authentication need cannot be supplied within an explicitly supported
  class, that provider's integration is **unavailable** for that
  execution — it fails closed rather than falling back to a broader grant.
  This architecture does not invent a universal authentication mechanism;
  each provider's authentication path is whatever its own verified
  mechanism requires, expressed narrowly through the classes above.
- **Safe process primitives** — the Process Supervisor (§4) is the only
  thing that actually calls the OS spawn function, and it is always given
  a fully explicit environment and working directory — never "inherit the
  host process's environment and then modify it."
- **An owned state-root boundary** — a per-execution directory under one
  AgentCubicles-owned base location, isolated from every other execution's
  directory and from the user's real provider home/state, is always
  available for whatever scratch/isolated state an adapter's isolation
  plan needs (`AC-CODEX-SEC-027`, generalized).
- **Validation and application of the adapter's isolation plan** (below)
  — the core is the enforcement point, not a passive pass-through.

**Each verified adapter declares (provider-specific, per adapter):**

- An **isolation plan** built only from the explicitly supported classes
  above: the minimal set of environment variables, files, or directories
  it actually needs — including, where a provider's real, independently
  verified authentication mechanism requires it, a specific, narrowly-named
  variable or credential file path that must be deliberately passed
  through (e.g. a documented API-key variable, or a specific
  credentials-file location). This is an explicit, narrow request the
  adapter must justify against that provider's actual, verified
  authentication mechanism — never a request for "the host environment."
- Any provider-specific subordinate-tool isolation requirement it needs
  met (e.g. keeping a subordinate tool like Git from silently consulting
  host-level global configuration or identity — `AC-CODEX-SEC-014`),
  stated as an outcome the core's state-root boundary and environment
  construction must satisfy, not as a specific mechanism the core must
  hard-code.

**The enforcement rule that ties them together.** The core validates every
adapter-declared isolation plan against the supported-classes policy and
the per-class validation requirements above, before applying it. A plan
requesting anything outside an explicitly supported class, or failing any
applicable validation step, is rejected — never silently honored, and
never satisfied by falling back to a broader grant. **No adapter may
inherit real user or provider state merely by requesting it**; a request
is only ever honored if it survives that validation, applied identically
to every adapter. A surviving denylist-style check alone is never
sufficient authorization. Whether a *given* provider's real authentication
mechanism can be satisfied within these classes at all is exactly the kind
of fact that must be independently verified per provider (§20, §19 step 4)
before its adapter's isolation plan is written — this is not yet known for
either Claude Code or Codex from the authoritative inputs.

## 7. Execution Ownership Model: Work Item, Execution, Session

This section defines the Work Item/Execution/Session model precisely
enough that a workflow using two providers never leaves the Work Item's
own provider association ambiguous.

- **Work Item** — a provider-neutral unit of intended work: a
  project/workspace, instructions, and permission constraints. **A Work
  Item does not itself carry a provider field.** Its lifecycle is exactly
  the v1 states in `PRODUCT_REQUIREMENTS.md` §5 (QUEUED → STARTING →
  WORKING → WAITING_FOR_USER / REVIEWING → DONE / ERROR / CANCELLED),
  deterministically derived from the state of its current Execution(s) —
  see "Work Item state derivation" below.
- **Execution** — one role-scoped unit of provider activity under a Work
  Item. Every Execution has immutable: work-item id, project/workspace id,
  provider id, **role** (e.g. `solo`, `builder`, `reviewer`), a
  **round/revision number** (§9), and its own execution id. Effective
  capability state (§3) is bound to this Execution identity and is never
  read from, or shared with, any other Execution. A plain, non-workflow
  Work Item has exactly one Execution with role `solo`. A Builder→Reviewer
  Work Item (`PRODUCT_REQUIREMENTS.md` §8) has two or more role-scoped
  Executions, each free to use a different provider — this is what makes
  the model unambiguous: the *provider* belongs to the Execution, never to
  the Work Item. A revision round or a retry is a new Execution under the
  same Work Item and role, never a mutation of the old one.
- **Session** — a provider-native unit of continuity inside one Execution
  (e.g. a Codex thread). **Sessions are adapter-owned, opaque handles, not
  a core-level entity.** The core does not model, track, or manufacture a
  session for a provider that has no native session concept — a provider
  without one simply has no session-shaped state anywhere. If a future
  product need genuinely requires reasoning about sessions across
  providers (none is established in the authoritative inputs), that would
  be a deliberate, separately justified addition, not a default. Where a
  provider does have sessions, ownership never crosses Execution
  boundaries: a session created under one Execution is never usable by, or
  reported as belonging to, a different Execution, even a later Execution
  of the same Work Item and role (generalizes `AC-CODEX-SEC-010`).

**Ownership rule threaded through all three:** authority is never
inherited by a sibling or successor. A new Execution — whether a retry of
the same role or the next role in a workflow — starts with no
carried-over compliance verdict, no carried-over session ownership, no
carried-over effective capability state, and no carried-over resource
counters (generalizes `CODEX_INTEGRATION_REQUIREMENTS.md` §3/§5.1 beyond
Codex).

**Work Item state derivation (deterministic, single authority).** The Work
Item Manager is the **sole authority** for a Work Item's lifecycle state;
no other component may set or imply it. That state is derived from the
Execution(s) belonging to the Work Item's **current round** (§9) using a
fixed precedence, evaluated top to bottom — the first matching rule wins,
so two Executions can never produce an ambiguous UI state:

1. Any current-round Execution reconciled as restart-interrupted (§8) →
   `ERROR` (carrying that reason metadata).
2. **The Workflow Coordination component (§9) has recorded round-limit
   exhaustion for this workflow → `ERROR`** (carrying `reason:
   "review-round-limit-exhausted"` metadata). Checked before rule 3: the
   reviewer Execution that hit the limit completed normally (it is not
   `failed`), so this condition would never otherwise be reached.
3. Any current-round Execution `failed` → `ERROR`.
4. Any current-round Execution `cancelled`, or the Work Item itself was
   cancelled → `CANCELLED`.
5. **Any current-round Execution — including a reviewer Execution — has a
   currently registered pending approval requiring user action (§8, §11)
   → `WAITING_FOR_USER`.** This rule is checked before, and takes
   precedence over, rule 6: a reviewer Execution that is live but
   currently blocked on a pending approval is `WAITING_FOR_USER`, never
   `REVIEWING`.
6. The reviewer-role Execution for the current round is live (and has no
   pending approval per rule 5) → `REVIEWING`.
7. Any current-round Execution is `running` → `WORKING`.
8. Any current-round Execution is `requested` but not yet `running` →
   `STARTING`.
9. **The Workflow Coordination component (§9) has recorded the current
   round's outcome as `accepted`** (Builder→Reviewer Work Items), **or**,
   for a non-workflow (`solo`) Work Item, its single Execution has reached
   `completed` → `DONE`. This is deliberately narrower than "every
   required-role Execution reached `completed`": a reviewer Execution that
   ended having produced a `MUST FIX` outcome (§9) has still reached
   `completed` as a matter of Execution lifecycle, but that outcome is
   never recorded as `accepted` — so this rule can never fire for it,
   regardless of the reviewer Execution's own lifecycle state.
10. No Execution yet exists for the Work Item → `QUEUED`.

An Execution belonging to a superseded round (§9's stale-round rule) never
participates in this derivation.

## 8. Restart Reconciliation and Permission Authority

**Universal principle: a restart loses Execution authority, not evidence
about any specific old Execution.** Live Execution state is in-memory only
(§12) and does not survive a restart, for every Execution regardless of
execution mode. What that loss means, and what (if anything) can speak to
whether the old Execution's underlying activity is still ongoing, is
mode-specific — never inferred from a universal "process state," since a
non-process Execution has none.

**Process-backed Executions.** Live Process Supervisor state is
in-memory only and does not survive a restart. **v1 does not implement a
lifetime-coupling mechanism that guarantees a provider process cannot
outlive AgentCubicles** — no cross-platform mechanism for that guarantee
has been verified (job objects, process groups, and equivalent primitives
differ enough across Windows/macOS/Linux that this architecture does not
claim the guarantee without that verification, §19/§20). v1 therefore
adopts the conservative rule: **after a restart, whether a provider
process that was live before the restart is still running is unknown, not
confirmed-terminated.** The empty in-memory Process Supervisor registry
after restart is evidence of nothing about any specific old process — it
only means the new Supervisor instance has not observed any process yet.
It must never be read as proof of termination, and it must never by
itself authorize destructive cleanup of that Execution's isolated state:
§4's cleanup rule still requires a positively observed exit or a
positively observed pre-spawn failure, neither of which a restart can
produce for an Execution that predates it.

**Non-process Executions.** The Process Supervisor holds no state for a
non-process Execution and is never consulted for one, before or after a
restart — an empty Process Supervisor registry is not authoritative, or
even relevant, evidence for it. Whatever lost-authority/recovery evidence
boundary the verified non-process execution driver defines (§3, "Execution
lifecycle source") is the only thing that could ever speak to whether such
an Execution's underlying activity is still ongoing; this architecture
does not design that boundary's concrete mechanism now (§3, §20). Absent
it, the same conservative posture applies by construction: old Execution
authority does not survive the restart (below), and this remains
fail-closed exactly as the process-backed case is — never treated as
confirmed-terminated merely because no mode-specific evidence exists yet.

**Startup reconciliation (mandatory, runs before the read-model surface in
§15 is queryable).** On every application start, before any Work Item's
status is exposed to the UI or to any command, the Work Item Manager
reconciles every persisted Work Item currently in a live status
(`STARTING`/`WORKING`/`WAITING_FOR_USER`/`REVIEWING`) as follows:

- Its live status can no longer be trusted, so it is deterministically
  transitioned to the existing terminal state **`ERROR`**
  (`PRODUCT_REQUIREMENTS.md` §5 defines no separate restart-interruption
  state, and this architecture does not add one to the public lifecycle
  vocabulary), carrying structured reason metadata — always `reason:
  "restart-interrupted"`, plus execution-mode-appropriate recovery
  metadata: `providerProcessState: "unknown"` (never `"terminated"`) when
  the affected Execution is process-backed, or the equivalent
  execution-neutral `executionRecoveryState: "unknown"` when it is a
  verified non-process Execution (§3) — so the UI and any future recovery
  flow can distinguish this from an ordinary provider-reported failure
  without implying every Execution has process state.
- Its history (activity record, checkpoints) is retained; only its *live*
  status is corrected.
- The Execution's isolated on-disk state (§6) is **preserved as
  residual**, exactly as an unconfirmed-exit case already requires (§4) —
  it is never destructively cleaned up on the basis of reconciliation
  alone.
- The reconciled Work Item is surfaced to the user as requiring attention
  — its `ERROR` status with this reason metadata is exactly the "requiring
  user attention" case `PRODUCT_REQUIREMENTS.md` §5 requires the UI to
  make legible.
- No authority carries forward from the old Execution: it cannot be
  resumed, its capability result and any live approval authority are gone
  (never resurrected — §7's ownership rule), and no command may act on it
  as if it were still live. The user may only start a **new** Execution
  against the Work Item.

If a genuinely orphaned provider process exists on disk/in the OS process
table after restart, this architecture does not attempt to detect or reap
it in v1 — doing so safely (positively identifying that a live process is
the same one an old, now-gone Execution spawned, without a false-positive
risk against an unrelated process that has since reused the same PID) is
exactly the kind of detection/reconciliation mechanism
`CODEX_INTEGRATION_REQUIREMENTS.md`'s residual-limitation guidance (§7)
says must be deliberately designed, not assumed. Closing this gap — either a verified,
cross-platform lifetime-coupling mechanism, or a positively-verifiable
orphan-detection technique — is tracked as a deferred decision (§20), not
adopted now.

**Three kinds of authority that must not be conflated:**

1. **Compliance verdicts** (e.g. a Codex-style account/session check, §3)
   — always scoped to exactly one Execution, never persisted, always
   re-derived fresh for a new Execution.
2. **User permission *preferences*** — an optional, explicitly-labeled,
   persisted record of what a user chose in the past for a given kind of
   action in a given project (e.g. "I usually allow test runs here"). If
   retained at all, a preference may only ever be used to pre-fill or
   suggest a decision for a **new** approval request in a **new**, live
   Execution — it must never itself constitute authorization, and must
   never resurrect or stand in for live approval authority after a
   restart.
3. **Live approval authority** — always in-memory, always scoped to
   exactly one currently-registered, currently-pending approval request
   belonging to one currently-live Execution (`AC-CODEX-SEC-025`,
   generalized). It never persists and never survives restart, consistent
   with the Process Supervisor's own live-only state.

**Decision: v1 does not persist approval authority.** The product presents
this option as "allow for this task" (`PRODUCT_REQUIREMENTS.md` §7);
because its actual scope is exactly one live Execution — never the Work
Item as a whole, and never a later retry, revision, or reviewer Execution
of it — this architecture defines and enforces it under the more precise
name **allow for this Execution**: a grant that means "for the remainder
of this Execution, in memory," never written to persistent storage, and
therefore unable to outlive the Execution or survive a restart. "Allow
once" and "deny" are likewise always live/in-memory decisions. A durable,
cross-restart permission *preference* (category 2 above) is not
implemented in v1; if a future version adds one, it must be built to the
constraint stated above — pre-fill only, never authority — and is tracked
as a deferred decision (§20), not shipped now.

## 9. Required v1 Flows

Required v1 flows had no end-to-end owner if left to a generic component;
each below is composed from components already defined above, and none
requires a new service.

**Usage/cost reporting.** If, and only if, a provider's per-execution
capability result (§3) declares usage/cost reporting supported, the
adapter emits provider-reported usage/cost facts as they become available.
Every such fact must preserve, never collapse:

- provider;
- Execution identity it belongs to (§7);
- metric name;
- unit/currency where applicable;
- the provider's own defined scope for that number (e.g. per-turn,
  per-session, per-account);
- period/window it covers;
- whether it is cumulative or a delta;
- availability/unknown state;
- provenance (§10).

A read model (§15) may aggregate facts across Executions **only when every
one of the following has been shown compatible, never merely assumed**:
provider compatibility where relevant (the same provider, or providers
independently shown to define the metric identically); metric-definition
compatibility (the same real-world quantity, not just the same name);
unit/currency compatibility; scope compatibility (the per-turn/
per-session/per-account distinction above); period/window compatibility;
cumulative-vs-delta compatibility; and mathematical additivity of the
resulting sum. **Identical metric names or identical units across two
different providers are never, by themselves, sufficient evidence of
compatibility** — the same word "tokens" or the same currency code can
still mean incompatible things across providers. Where compatibility is
not shown, the read model displays each scoped provider value separately,
or `unknown` where a fact is unavailable. It never invents or estimates a
number, and never derives an exact cost from a provider that exposes only
usage or partial data (`PRODUCT_REQUIREMENTS.md` §11).

Usage/cost observations are **ephemeral read-model data, not durable
product history** (§12): they are appended to the Activity Record's live/
subscribable stream (§10) so the current session's UI can display them,
but the persisted portion of Activity Record history excludes them.
Nothing in `PRODUCT_REQUIREMENTS.md` requires durable usage/cost history
for v1; if a future version adds that requirement, it is a deliberate,
separately justified persistence-policy change, not an accidental side
effect of the Activity Record's ordinary persistence.

**Builder → Reviewer sequencing.** For a coordinated Builder→Reviewer Work
Item (`PRODUCT_REQUIREMENTS.md` §8), the thin Workflow Coordination
component (§2) owns the workflow's current round and its `maxRounds`
bound, and enforces the coordinator invariant below. **The Work Item
Manager remains the sole Work Item-state mutation authority (§7) at all
times; the coordinator supplies an authoritative workflow transition/
outcome and never directly mutates public Work Item state itself** (§10's
direct-authority-delivery rule applies here: the reviewer's result is
delivered to the coordinator first, which then requests whatever Work Item
Manager transition follows).

- Every coordinated Builder→Reviewer workflow is created with an explicit
  `maxRounds` value that must be **an integer, greater than zero, and
  finite** (never fractional, zero, negative, `NaN`, or unbounded),
  validated before the workflow's first Execution starts; a workflow whose
  `maxRounds` fails this validation is never started
  (`PRODUCT_REQUIREMENTS.md` §8's required explicit round limit).
  `maxRounds` is immutable and fixed for the lifetime of one workflow
  instance — it
  cannot be silently expanded mid-workflow; only a new workflow instance
  (e.g. a user-started retry after exhaustion, below) may specify a
  different value.
- Each round is identified by an explicit revision/round number, carried
  on every Execution created for that round (§7).
- The reviewer Execution for round N may not start before the builder
  Execution for round N reaches the checkpoint handoff boundary defined
  below (a completed checkpoint record for that round).
- The builder and reviewer Executions for the **same** round N never run
  concurrently — only one of them is active at a time for that round.
- **Reviewer ACCEPT**: the coordinator atomically records round N's
  outcome as `accepted`. The Work Item may transition to `DONE` only when
  the resulting workflow completion condition is satisfied — §7's state
  derivation rule 9, which requires this recorded `accepted` outcome, not
  merely that the reviewer Execution finished running.
- **Reviewer MUST FIX, round N < `maxRounds`**: this outcome **must not**
  cause an intermediate or permanent `DONE`. The coordinator atomically
  records round N's outcome as `must-fix` and, in the same atomic step,
  creates (or marks the Work Item as needing) the builder Execution for
  round N+1 — a new Execution under the same Work Item and `builder` role,
  never a mutation of round N's Executions (§7).
- **Reviewer MUST FIX, round N == `maxRounds` (round-limit exhaustion)**:
  the coordinator **must not** create another round. It atomically records
  round N's outcome as `must-fix` and the workflow's outcome as
  `round-limit-exhausted`, and requests the Work Item Manager transition
  the Work Item to the existing **`ERROR`** state — `PRODUCT_REQUIREMENTS.md`
  §5 defines no separate "review limit reached" public state, and this
  architecture does not add one, matching the same choice already made for
  restart-interruption (§8) — carrying structured reason metadata
  (`reason: "review-round-limit-exhausted"`) and surfacing the Work Item as
  requiring human attention exactly as `PRODUCT_REQUIREMENTS.md` §5
  requires. The user may explicitly start a **new**, separately bounded
  workflow if product rules permit it; no authority (capability result,
  live approval) carries forward from the exhausted workflow's Executions
  (§7's ownership rule, §8). Because §7's `DONE` rule depends on a
  recorded `accepted` outcome, a `must-fix`-outcome reviewer Execution —
  whether or not the round limit was reached — can never, by itself,
  satisfy that rule.
- **Reviewer failure**: an ordinary Execution failure (§7's `ERROR`
  derivation rule) — no separate outcome vocabulary is introduced for it.
- **User interruptibility.** The user may cancel/stop the coordinated
  workflow at any time through the existing cancellation authority (§7's
  `CANCELLED` derivation rule) — no workflow-specific exception.
  Cancellation prevents the coordinator from creating any further round;
  it does not retroactively revoke or undo work already completed in
  prior rounds. Any Execution still live at the moment of cancellation is
  stopped through the stop authority for its own verified execution mode
  — the Process Supervisor path (§4) for a process-backed Execution, or
  the non-process driver/lifecycle authority path (§3, "Execution
  lifecycle source") for a non-process Execution — cancellation is never
  itself a second, independent stop mechanism.
- A result reported by an Execution belonging to a stale round (e.g. a
  slow reviewer response for round N arriving after round N+1 has already
  started) is discarded by the Workflow Coordination component and must
  never mutate the current round's state — round identity, not arrival
  order, decides what is current.

This invariant governs only Executions **within** one Builder→Reviewer
Work Item. It does not globally forbid, or otherwise restrict, any other,
independent Execution — a different Work Item, or an unrelated role —
which proceeds fully concurrently (§13).

**Git preflight visibility.** The Execution Manager, as part of starting
any Execution — immediately before `spawn()` for a process-backed
Execution, or the equivalent start point for a non-process Execution
(§3) — using the Project/Workspace component's Git inspection (§2)
captures the current branch,
dirty/clean state, changed files, and a diff summary for that project, and
attaches this snapshot to the Work Item's activity history
unconditionally, not as an opt-in step. This makes existing uncommitted
changes visible before automated work begins, per
`PRODUCT_REQUIREMENTS.md` §9, without introducing a separate preflight
service: it is one call the Execution Manager already needs to make at the
same point it validates the workspace path (§5, Level 1).

**Checkpoint handoff.** A checkpoint (`PRODUCT_REQUIREMENTS.md` §10) is a
small structured record — what was completed, key decisions, unresolved
issues, relevant files, a recommended next step — stored via ordinary Work
Item persistence (§12; no separate Checkpoint Store). **Every checkpoint
binds immutably to the identity that produced it**: source project id,
source workspace id, Work Item id, round/revision number (§9), producing
role, and producing Execution id (§7) — a checkpoint is never
free-floating data. A normal same-workflow handoff (builder→reviewer
within one round, or round N→N+1) requires an exact match against this
binding. "Applying" a checkpoint means composing its structured content
together with any new user instructions into the `instructions` input a
**new** Execution begins with (§3) — either a new Execution under the same
Work Item or a brand-new Work Item, at the user's choice; a brand-new Work
Item carries the checkpoint's original binding forward as provenance, it
does not inherit the checkpoint's identity as its own. **A checkpoint must
never be discovered or reused implicitly across projects.** If a future
version supports cross-project handoff, it requires an explicit,
user-selected operation and must record clear new attribution in the
destination Work Item — never presenting the checkpoint as though it
originated there. There is no separate "apply" mechanism inside the core
beyond this.

## 10. Activity and Record Model

This section defines a minimal record-and-subscription mechanism plus a
narrow, purposeful use of provenance — not a mandatory fixed event
taxonomy (`CODEX_INTEGRATION_REQUIREMENTS.md` §12 explicitly rejects
normalizing every signal into one universal taxonomy).

**Activity Record**: a small append-only record store with a subscription
interface — not a generalized "Event Bus" framework. Any component may
append a record; any component may subscribe to records relevant to it.
There is no fixed enum of required categories: a provider adapter emits
only the verified facts it actually has (§3) — a provider with no approval
concept never has to produce an "approval" record of any kind, and nothing
pretends otherwise.

**Domain authority is not routed through records — it is direct.**
Provider facts that can influence lifecycle or workflow authority —
completion, failure, a reviewer result, a security/account-invalidation
signal — are delivered **directly to the authority owner that consumes
them first** (Execution Manager, Work Item Manager, Permission & Approval
Engine, or Workflow Coordination, as applicable). That owner validates the
fact and performs, or requests, the resulting domain transition; only
**afterward** is the observational Activity Record appended.
**Activity Record subscriptions are never the source of lifecycle or
workflow authority** — a subscriber may react to what already happened, it
never decides it. This is a routing rule, not a new event taxonomy: it
does not add any record category beyond the minimal model above.

- Lifecycle transitions (a Work Item or Execution changing state) are
  performed directly by the Work Item Manager / Execution Manager as
  state-machine operations; the state itself is authoritative the instant
  it changes, not once some downstream consumer of a record gets around to
  interpreting it.
- An approval is resolved by calling the Permission & Approval Engine
  directly (§8); resolution is not something inferred later from a stream
  of notifications.
- **A reviewer result (ACCEPT/MUST FIX) is delivered directly to the
  Workflow Coordination component first** (§9), which atomically records
  the round outcome and requests whatever Work Item Manager transition
  follows — never inferred from, or awaited via, an Activity Record
  subscription.
- **Security/control-plane invalidation acts through a direct authority
  path first.** When something must invalidate a compliance verdict or
  block further task-capable action (e.g. an account-state-change
  signal), the Execution Manager is called directly and synchronously to
  enforce that block — before, and independent of, any activity/audit
  record being written. The audit record documenting that the
  invalidation happened is appended **after** the authoritative action has
  already taken effect, purely for observability — it is never the
  mechanism by which the invalidation itself happens (generalizes
  `CODEX_INTEGRATION_REQUIREMENTS.md` §4's requirement that this not be
  subordinate to ordinary activity normalization).

**Evidence provenance on authoritative transitions.** The Work Item
Manager and Execution Manager remain the sole authority that performs any
lifecycle mutation — this is never delegated to the Activity Record. But
where a mutation (e.g. transitioning an Execution to `DONE`/`ERROR`) was
decided using provider-reported or AgentCubicles-derived evidence rather
than a directly observed fact, the resulting lifecycle/audit record
retains that evidence's provenance tag, so later inspection can tell
whether a `DONE` was the provider's own claim, an inference from observed
output, or a directly observed process/protocol fact. This is a targeted
requirement on transitions where provenance affects trust or
interpretation — it does not create a new mandatory event taxonomy beyond
the minimal model above.

**Provenance is used where it is actually load-bearing, not mechanically
on every record.** A record carries a provenance tag (`directly observed`
/ `provider-reported` / `AgentCubicles-derived`) exactly where trust,
security, or interpretation depends on the distinction — a process exit
fact (directly observed), a usage/cost figure (provider-reported, §9), a
"session appears idle" inference (derived). A record kind with no such
ambiguity (e.g. "Work Item transitioned to DONE," which the Work Item
Manager itself authoritatively decided from an unambiguous signal) does
not need one.

This is deliberately not a rigid shared schema every provider must fit
forever: a provider may expose its activity through its own
explicitly-tagged payload shape rather than a shared envelope, as long as
records it produces are honestly labeled and nothing is guessed or
fabricated to fill a category it doesn't have.

## 11. Security Architecture

The properties below must hold for every provider adapter, generalized
from `PRODUCT_REQUIREMENTS.md` §7 and the technique-neutral invariants in
`CODEX_INTEGRATION_REQUIREMENTS.md` §7. Several are now defined precisely
in earlier sections; this section is the consolidated summary and covers
what isn't already fully stated elsewhere.

- **Process isolation.** Every **process-backed** Execution is backed by
  its own OS child process, spawned only through the Process Supervisor
  via the complete path in §4; a non-process Execution has no such
  process and is governed instead by §3's "Execution lifecycle source"
  rule. The core never executes provider-originated code in-process.
- **Environment/state isolation (process-backed providers, §6).** Per the
  default-deny split in §6: explicitly supported input classes,
  core-validated per-input requirements, and adapter-declared isolation
  plans. **No input is injected or disclosed by AgentCubicles outside an
  explicitly supported class** — this is a bound on AgentCubicles's own
  grants, not a claim about what an unsandboxed process can independently
  discover absent Level 3 containment (§5, §6). No adapter inherits real
  host/provider state merely by requesting it.
- **Workspace enforcement.** **Universal guarantee, every Execution
  regardless of execution mode (§5 Level 0):** every Execution has
  mode-appropriate workspace attribution/scoping enforced by the core
  architecture — no provider may silently be attributed to, or authorized
  for, a different project/workspace than the one it was started for.
  What actually enforces that scoping is mode-specific: for a
  **process-backed** Execution, canonical launch-directory validation
  (§5 Level 1) is the mechanism, guaranteed before `spawn()`; for a
  **non-process** Execution, the verified execution-mode-specific
  workspace-scoping mechanism defined when that provider's adapter is
  designed (§3, §5, §19) is the mechanism — this architecture does not
  invent a launch directory for an execution mode that has none. Level 2
  mediation, where present, covers only actions actually routed through
  it; OS-level containment is not provided in v1 (Level 3 absent).
  Neither this document nor the product may claim more than what Levels
  0–2 actually deliver for a given Execution's verified execution mode,
  and — without Level 3 OS-enforced containment — none of this proves a
  provider cannot independently access OS-visible paths outside the
  workspace through its own means (§5, §6).
- **Approval authority.** Only a currently registered, pending approval
  may receive an authoritative response; a stale, duplicate, or unknown
  approval id gains no authority (`AC-CODEX-SEC-025`). Approval capacity,
  where an adapter's transport has one, is bounded; an over-capacity
  request is never registered and never authoritative even if a fixed
  decline is sent for it. Persistence rules for approval decisions are
  exactly those in §8 — v1 persists no live approval authority, and the
  "allow for this Execution" grant (§8) never outlives the Execution it
  was granted for.
- **Capability gating.** The Permission & Approval Engine is the single
  place that decides whether a requested action is auto-allowed, denied,
  or needs a live user decision, based on the Work Item's permission
  constraints and the *current, per-execution* effective capability
  **together with its corresponding live state** (§3's capability
  lifecycle) — never a stale static declaration and never a capability
  claim alone. Destructive, externally visible, credential-sensitive, or
  difficult-to-reverse actions default to requiring approval
  (`PRODUCT_REQUIREMENTS.md` §7).
- **Cleanup ownership (process-backed Executions).** Per §4/§8: destructive
  cleanup of a process-backed Execution's isolated on-disk state is
  authorized only by Process Supervisor facts — never-spawned or
  confirmed-terminated — never by adapter state, never merely because no
  work was accepted, and never inferred from an empty Process Supervisor
  registry after a restart (§8). Ownership of an isolated-state location
  must be verifiable against the actual filesystem object at use time, not
  by pathname alone (`AC-CODEX-SEC-017`–`019`). A non-process Execution's
  isolated-state cleanup authority, if it has any isolated on-disk state at
  all, is deferred to when such a provider is designed (§3, §20) — it is
  never authorized by an absent Process Supervisor fact that was never
  applicable to it in the first place.
- **Resource bounds.** Where an adapter maintains its own transport to a
  provider (e.g. a framed stdio protocol), frame size, pending buffer
  size, outstanding requests/approvals, and tracked session count must be
  held under fixed, validated, positive caps enforced by that adapter;
  exceeding a cap fails closed rather than growing unbounded
  (`AC-CODEX-SEC-020`). This is adapter-owned because it only applies to
  adapters whose provider protocol actually has these shapes — it is not
  imposed on a provider with a fundamentally different transport.
- **Private transport.** Where a provider's integration mode is a local
  process communicating over a local-only channel (stdio or equivalent),
  the adapter must not silently open a network listener for that
  communication (`AC-CODEX-SEC-028`).
- **Fail-closed behavior.** An unclassifiable result (malformed response,
  timeout, ambiguous state) is always treated as the non-permissive
  outcome — for compliance checks where a provider has one, for cleanup
  decisions, and for permission evaluation alike. This is the single
  fail-closed principle from `PROJECT_VISION.md` and
  `PRODUCT_REQUIREMENTS.md`, applied consistently, not a Codex-only rule.
- **Diagnostics never leak raw content.** Logs and surfaced errors carry
  only fixed, pre-approved text or an exact-match allowlisted low-level
  code — never raw process, account, or protocol payload content
  (`AC-CODEX-SEC-022`).
- **Testability does not create a policy bypass.** See §14 — the
  substitution seams required there must never be reachable from
  production configuration.

**Documented residual limitation (carried forward, not solved here):**
v1's destructive-cleanup verification is necessarily implemented as
separate filesystem checks followed by a separate filesystem operation —
this cannot prove elimination of every time-of-check-to-time-of-use race,
only that a *detected* discrepancy aborts the action and ambiguity is
never reported as success (`CODEX_INTEGRATION_REQUIREMENTS.md` §7 Residual
Limitation). v1 explicitly adopts a **local/single-user/private-machine
threat model**: isolation is designed against accidental cross-run
interference, not against an actively adversarial, equally-or-more-
privileged co-resident process racing the exact cleanup window. Closing
that gap (OS-level sandboxing, a managed volume, a transactional
filesystem primitive) is out of scope for v1 and tracked as a deferred
decision (§20).

## 12. State and Persistence

Minimalism first: v1 needs enough persistence to survive an application
restart without losing what the user would reasonably expect to keep, and
no more.

**Must persist:**

- Project registry (registered projects and their paths).
- Work Item records, their Executions' history, and lifecycle history
  (for the activity timeline, for restart reconciliation in §8, and for
  resuming "what was I doing"). This includes the restart-interruption
  reason and execution-mode-appropriate recovery metadata §8 attaches when
  reconciling a Work Item to `ERROR` — it is ordinary lifecycle history,
  not a new persisted concept.
- Checkpoints (`PRODUCT_REQUIREMENTS.md` §10) — small, structured records,
  stored as part of ordinary Work Item persistence, not a separate store.

**Does not need persistence in v1:**

- Live Execution/process state (for a process-backed Execution, in-memory
  only, owned by the Process Supervisor and Execution Manager; for a
  non-process Execution, whatever equivalent live state its execution
  driver holds, §3) — a restart means those are gone, which is exactly
  what §8's reconciliation handles.
- Live approval authority or any persisted "allow for this Execution"
  decision (§8 — v1 does not persist approval authority at all).
- Full raw provider transcripts/output — only what the Activity Record
  already normalizes into activity/history needs to survive.
- Usage/cost observations (§9) — ephemeral read-model data. The persisted
  portion of Activity Record history excludes usage/cost payloads even
  though other Activity Record entries (lifecycle transitions, Git
  preflight snapshots, checkpoints) are persisted normally; this keeps
  §9's ephemerality rule and this section's Activity Record persistence
  from contradicting each other.

**Ownership:** the Work Item Manager owns Work Item and checkpoint
persistence; the Project/Workspace component owns project persistence.
Neither the Execution Manager nor any adapter persists anything directly —
an execution's isolated on-disk state (§6) is ephemeral working storage,
not product data, owned by the Execution Manager for cleanup purposes
only. A restart-interrupted Execution's preserved residual isolated state
(§8) is not deleted by ordinary persistence or cleanup; it remains on disk
until an explicit future recovery/cleanup decision (§20) addresses it.

**Mechanism:** a single local, file-based store (e.g. one embedded
database file or a small set of structured files) is sufficient for v1's
data volume and local-first requirement (`PRODUCT_REQUIREMENTS.md` §13).
No server-hosted database is needed or desired. The exact format remains a
deferred decision (§20).

## 13. Concurrency

The product must support multiple independent projects/Work Items without
needing distributed-system machinery:

- Each **process-backed** Execution is backed by its own OS process (§4);
  concurrency across those Executions is naturally provided by the OS
  process model. A non-process Execution's concurrency is whatever its own
  execution driver provides (§3), subject to the same serialized
  command-ownership invariant below. The core does not need its own
  scheduler for provider work.
- **Serialized command ownership, not a specific runtime mechanism.**
  Every authority scope — a project's registry entry, one Work Item's
  state, one set of live permission decisions — is mutated only through
  exactly one command-owning path for that scope. This is stated as an
  invariant the implementation must satisfy, independent of *how*: a
  single event loop, an actor per scope, or a worker-per-scope model can
  all satisfy it equally.
- Per-Execution work (e.g. reading stdio for a process-backed Execution,
  or handling timeouts generally) is strictly partitioned by **immutable
  execution identity** (§7) — a message, timeout, or error belonging to
  one Execution must never be able to affect a different Execution's
  pending state
  (`CODEX_INTEGRATION_REQUIREMENTS.md` §4, "stale process error cannot
  affect the current run's pending requests," generalized).
- **Executions belonging to different Work Items always proceed fully
  concurrently** — they never need to coordinate through each other's
  command-owning path. Executions within the *same* Work Item are
  likewise independent and concurrent **except** where §9's Builder→
  Reviewer sequencing invariant applies (same-round builder/reviewer
  Executions, which are sequential by construction); nothing here imposes
  coordination on unrelated roles or Work Items beyond that one, narrow
  invariant.
- The Builder→Reviewer sequencing invariant (§9) is enforced by the
  Workflow Coordination component alone; it does not require its own
  concurrency model beyond what the Work Item Manager and Execution
  Manager already provide, and it never restricts an unrelated Execution.

No message queue, external broker, or multi-process worker pool is needed
for v1. If a future capability (e.g. Curiosity Mode) needs background
scheduling, it is additive to this model, not a redesign of it (§16).

## 14. Testability

This is an explicit architectural constraint, not an implementation
detail: the following must be substitutable at safe, construction-time
dependency boundaries so they can be replaced in tests without weakening
what production actually enforces:

- process launching (the Process Supervisor's `spawn`, §4),
- time (timeouts, grace periods, bounded-wait deadlines),
- filesystem operations (path resolution, the state-root boundary,
  cleanup),
- environment input (what a test considers "the host environment"),
- product state roots (where persistence and isolated execution state
  live).

**The substitution seam must not be a production-reachable policy
bypass.** These are construction-time dependencies (what the Process
Supervisor, clock, filesystem accessor, and state-root are initialized
with when the application starts), never a runtime flag, environment
variable, or configuration value that could point a production build at
test-mode behavior. A test build is wired with test-owned substitutes at
construction; nothing in the production wiring path exposes an equivalent
switch.

**Tests must never default to real developer or provider state.** Every
test that exercises process spawning, environment construction, or
state-root usage must be given an explicitly test-owned root and an
explicitly constructed environment — never the real developer `HOME`, a
real installed provider, or the real product persistence location. This is
what makes the security invariants in §11 actually verifiable: a test can
assert "this input never reached the child" only if the child's
environment was fully test-constructed rather than inherited.

## 15. UI Boundary

The UI's process topology (embedded in the same application process, or a
separate host process) is not decided here — see §2. What it consumes
from the core is a small, stable surface, regardless of that choice:

- **Read models**: current project list (with Git preflight/status, §9),
  current Work Item list with lifecycle state (derived per §7), current
  Execution status per Work Item and role, the activity record
  (paginated/streamed, §10), the usage/cost read model where available
  (§9), current checkpoints.
- **Commands**: register/remove a project, create/cancel a Work Item,
  start an Execution (optionally composed from a checkpoint, §9), resolve
  a live approval (the Execution-scoped "allow for this Execution" grant
  defined in §8), start a Builder→Reviewer workflow.
- **A live subscription** to the Activity Record (§10), so the UI can
  update without polling raw provider output.

The UI must never need direct access to a provider process, a raw provider
fact, or the environment/isolation internals — those stay behind the
components in §2. This keeps the future visual cubicle/office presentation
a pure consumer of core state and records, never a source of product
logic.

## 16. Extension Boundaries

**Future providers:** adding a provider means writing one new adapter
against the execution contract in §3 and registering it with the Provider
Registry. Nothing in §2, §4–§14 is Codex- or Claude-Code-specific; each
reads "a provider" or "an adapter," except where explicitly scoped (§3's
Codex and Claude Code paragraphs, both adapter-internal by construction).

**Curiosity Mode (not designed here):** `PRODUCT_REQUIREMENTS.md` §17
requires explicit enablement, read-only-by-default behavior, time/cost
budgets, network policy, and no autonomous modification by default. The
architecture does not block this: it would be expressed as a Work Item
created by the system itself (still going through the same Work
Item/Execution/Permission pipeline, §7–§8, §11) rather than a parallel
execution path, with its budgets expressed as additional Permission Engine
constraints rather than a new enforcement mechanism.

**Multi-agent meetings (not designed here):** `PRODUCT_REQUIREMENTS.md`
§17 requires bounded rounds/time/cost and explicit termination — exactly
the shape the role-scoped Execution model (§7) and the thin Workflow
Coordination component (§2, §9) already establish. A future multi-role
workflow is additional coordination logic over the same Work
Item/Execution primitives, not a new core concept — nothing here assumes
exactly two roles at the data-model level, only that v1 implements two.

**Cloud sync / team features (not designed here):** because persistence is
isolated behind the Work Item Manager/Project component (§12) rather than
scattered through the core, a future sync layer can be added as another
consumer/writer of that persisted state without redesigning the Work
Item/Execution model. This is not a design of that layer — only an
acknowledgment that §12's boundary does not preclude it.

## 17. Proposed Module Structure

These are logical responsibilities (§2), not mandated physical modules.
Actual source layout may start flatter than this and split further only
when a component actually grows large enough to justify its own directory
— this is a starting point, not a diagram to satisfy for its own sake.

```
/core
  /project        Project registry + Git inspection (folded in — no
                  separate Git module until it grows large enough to
                  justify one)
  /provider       Execution contract/types; Provider Registry (may begin
                  as a plain id -> adapter map)
    /claude-code  Claude Code adapter
    /codex        Codex adapter
  /process        Process Supervisor (§4) — generic, no provider
                  knowledge; owns spawn/exit facts and issues the
                  controlled execution/process transport boundary each
                  process-backed adapter uses for its own protocol work
  /execution      Execution Manager: launch-requirement validation,
                  isolation-plan application, capability tracking,
                  cleanup authorization (§4, §6, §11)
  /workitem       Work Item Manager: lifecycle, state derivation,
                  role-scoped Executions, restart reconciliation (§7, §8);
                  checkpoints reuse this module's persistence, no separate
                  Checkpoint Store
  /permission     Permission & Approval Engine (live authority only, §8)
  /activity       Activity Record: append + subscribe (§10) — a small
                  interface, not an event-bus framework
  /persistence    Local file-based store, used by project/workitem/
                  activity
  /workflow       Builder/Reviewer coordination (§9) — thin, added once
                  the basic project/provider/workitem pipeline is stable
                  (§19); not required to exist from day one
/ui
  (topology not decided — consumes /core's read models and commands, §15)
```

Rules this layout is meant to enforce:

- `provider/claude-code` and `provider/codex` are the *only* places
  provider-specific knowledge (protocol shapes, method names, account
  quirks, session handling) may live.
- `process` owns spawn/exit facts for *every process-backed* provider
  (§4); no adapter spawns its own child process.
- `execution` applies environment/workspace isolation for *every*
  process-backed provider (§6, §5) using the core-owned policy plus each
  adapter's declared plan; no adapter constructs its own child environment
  directly.
- `ui` never imports from `provider/*` directly, only from the read-model
  and command surface `core` exposes (§15).
- No `services/`, `infrastructure/`, or generic `utils/` grab-bag module.
  If a helper is only used by one component, it lives in that component.

## 18. Dependency Policy

- Prefer the runtime's standard library for process spawning, stdio
  handling, filesystem path resolution, and JSON parsing. None of these
  require a third-party dependency to do correctly.
- A provider adapter may need a small, well-scoped library only if the
  provider's own protocol genuinely requires it (e.g. a JSON-RPC framing
  helper) — evaluated per adapter, after that provider's facts are
  actually verified (§19), not adopted product-wide speculatively.
- A local persistence mechanism (§12) is the one area where a dependency
  decision is reasonable, but is explicitly **deferred**: whether to use
  an embedded database library or a hand-rolled structured-file store
  should be decided when the Work Item Manager is actually implemented,
  based on the real data shapes at that point.
- No dependency is selected because a prior implementation used it.
  `CODEX_INTEGRATION_REQUIREMENTS.md` §12 explicitly rejects carrying
  forward implementation techniques (e.g. specific Git-isolation
  environment-variable sets, a specific cleanup-marker technique) as
  architecture; the same discipline applies to library choices.
- **Deferred to implementation time:**
  - UI framework/rendering approach and process topology (§15 only
    defines what it consumes).
  - Local persistence format/library (above).
  - Any adapter-internal protocol-framing helper, pending each provider's
    independent verification (§19).
  - Testing framework choice (any reasonably standard one satisfies §14's
    testability requirement; not an architectural decision).

## 19. Implementation Sequence

Small, independently reviewable and testable milestones. Real provider
facts are verified, and the execution contract is proven against real
providers, before either is treated as stable (§3's contract-stabilization
rule) — never frozen against a fake adapter first. **No user-capable
real-provider execution is permitted before the gates listed in step 7
are satisfied**; earlier real-provider contact is restricted to
disposable, non-user-facing verification (step 4), and **no persisted,
user-capable Execution may exist before startup reconciliation (step 6) is
implemented**. Any milestone below that touches paths, filesystem
semantics, environment, process lifetime, transport, or cleanup must
verify the relevant behavior on every platform that milestone claims to
support **before that milestone is considered closed** — not deferred to
step 13 alone; step 13 is the final gate across all of them together, not
the first time any of them is checked.

1. **Minimal project/workspace identity.** Register/list/remove a local
   project; enforce path validity; basic Git status/branch inspection
   folded in (§2, §17). No provider or Work Item concepts involved yet.
2. **Minimal Work Item/Execution identity model + Activity Record
   ownership.** Implement the Work Item/Execution ownership model (§7)
   and the append/subscribe Activity Record (§10) against a trivial
   synthetic Execution (not a real provider) — this proves the identity
   and audit-ownership model in isolation before any provider complexity
   is added.
3. **Process Supervisor, proven against a trivial real process.**
   Implement `spawn`/`requestGracefulStop`/`onExit`, the opaque supervised
   reference, and the controlled transport boundary (§4) and exercise it
   against an actual OS child process with no provider protocol involved
   yet — this proves the process-authority boundary itself, independent
   of any adapter.
4. **Verify real integration facts for both candidate providers —
   test-owned, non-user-facing.** Spike against the actual Claude Code and
   Codex CLIs to establish: real invocation shape, whether either has a
   session/handshake/compliance concept, each provider's actual execution
   mode (process-backed or not, resolving §3's provisional assumption per
   provider), and each provider's actual authentication mechanism (needed
   for §6). This step is explicitly restricted to disposable, test-owned
   workspaces and state roots, test/non-user credentials, and
   non-user-facing verification. Output is verified facts (replacing or
   confirming §3/§20's UNVERIFIED markers), never shipped code and never a
   path a real user's project or credentials pass through.
5. **Permission & Approval Engine and Git preflight flow**, built against
   the action shapes step 4 actually observed and wired into the
   Execution Manager's start path (§9) — both exist before any
   user-capable real-provider execution, so step 7's gate has something
   real to enforce against.
6. **Restart reconciliation** (§8), implemented and tested against the
   synthetic Work Item/Execution model from step 2 — **before any
   user-capable real-provider execution exists.** No persisted,
   user-capable Execution is created until this milestone is complete;
   step 7 below requires it as an explicit gate. The core reconciliation
   decision operates purely on persisted Work Item Manager state, never on
   provider specifics, so nothing about it differs once step 7's real
   provider exists — it continues to apply unchanged to real, persisted
   Executions from that point on. Mode-specific recovery evidence (§8) —
   the empty-after-restart Process Supervisor registry for a process-backed
   Execution, or the non-process execution driver's own lost-authority
   evidence boundary for a non-process Execution — is consulted only for
   attaching the mode-appropriate recovery metadata, never as part of the
   universal reconciliation decision itself.
7. **First real-provider vertical slice, chosen from evidence — the first
   user-capable execution.** Select whichever of the two providers has the
   smallest verified integration surface from step 4 — not "Claude Code
   because it was listed first." Before this slice may run against a real
   user project, gates apply in two layers: universal gates required for
   any provider, and additional gates specific to the selected provider's
   verified execution mode (step 4).

   - **Universal gates (required before any user-capable provider
     slice, regardless of execution mode):** project/workspace attribution
     (§1, §7); Git/workspace preflight required by
     `PRODUCT_REQUIREMENTS.md` §9 (step 5); the permission/approval
     boundary for the actions this slice exposes (step 5); the
     default-deny policy for AgentCubicles-supplied host inputs (§6);
     the relevant resource/capability/security policy (§11); startup
     reconciliation (step 6); and a safe authority/lifecycle stop path
     for the verified execution mode (§3, "Execution lifecycle source").
     None of these is weakened for a non-process provider merely because
     it is non-process.
   - **Additional gates for a PROCESS-BACKED execution mode:** the
     Process Supervisor path (step 3); canonical launch-directory
     validation (§5, Level 1); the process environment/state injection
     policy (§6); process transport policy (§11); and the applicable
     process resource limits and cleanup authority (§4, §11).
   - **For a NON-PROCESS execution mode:** the corresponding verified
     execution-driver boundary (§3) and the workspace/input controls that
     actually apply to it are required instead — this slice never
     requires a fabricated process environment, `spawn()`, Process
     Supervisor participation, or process transport policy for a provider
     that has none of those.

   Implement one narrow adapter, its capability declaration, and wire it
   through the gates applicable to its verified execution mode, including
   that provider's actual environment/isolation-plan needs (§6) and (for a
   process-backed provider) workspace launch-directory validation (§5,
   Level 1) — verified on every platform this milestone claims to support,
   not deferred.
8. **Revise the execution contract from what step 7 actually observed.**
   This produces a **PROVISIONAL** revision of the shared execution
   contract (§3) — informed by one real provider's real capabilities,
   lifecycle shape, and isolation needs, not written speculatively
   beforehand — never a final version. §3's contract-stabilization rule
   requires the second-provider evidence in step 10 before this contract
   may be treated as stable enough for v1.
9. **Cleanup authorization**, implemented against the Process Supervisor's
   real confirmed-exit facts from steps 3/7, including the documented
   residual TOCTOU limitation (§11).
10. **Second provider vertical slice, validating the revised contract —
    architectural evidence, not merely a conformance test.** Implement the
    remaining provider's adapter against the PROVISIONAL contract from
    step 8, subject to the same user-capable gates as step 7. Revise the
    shared boundary again if this provider's evidence requires it (§3) —
    document any capability the contract had to bend for. Only after this
    step's evidence is incorporated does §3's contract-stabilization rule
    allow the shared abstraction to be treated as stable enough for v1.
    Repeat step 7's platform verification for this provider's own
    environment/isolation needs — do not defer to a single
    end-of-project cross-platform pass.
11. **Usage/cost read model** (§9), if either provider verified in step 4
    actually exposes usage/cost facts; skipped or marked `unknown`
    otherwise.
12. **Checkpoints and bounded Builder→Reviewer workflow coordination**
    (§9), reusing Work Item persistence, once both providers and the
    basic Work Item/Execution pipeline are stable (after step 10).
13. **Final cross-platform integration gate, before any v1 release.**
    Every milestone above that touched paths, filesystem semantics,
    environment construction, process lifetime, transport, or cleanup
    (steps 1, 3, 4, 6, 7, 9, 10) is re-verified on every platform this
    release claims to support (Windows/macOS/Linux,
    `PRODUCT_REQUIREMENTS.md` §12) — not assumed from an earlier
    single-platform pass. A platform is never claimed supported without
    this gate having actually run on it.

Each milestone should ship with focused tests before the next milestone
builds on it, per `PROJECT_VISION.md`'s "focused tests before broad E2E
runs" principle, using the substitution seams required by §14. UI work
(§15) can begin against any milestone's read-model surface once step 2
exists, but is not sequenced here as it is not designed in this document.

## 20. Decisions Requiring External/Provider Verification

- **No integration facts have been verified for Claude Code anywhere in
  the authoritative inputs.** Whether it has a session concept, a
  handshake/readiness phase, an approval mechanism, a compliance-style
  check, or even a process-backed execution mode (§3) is unknown, not
  merely "assumed absent" — §19 step 4 must establish this before its
  adapter is built.
- The Codex App Server's actual wire protocol — launch invocation,
  framing, method vocabulary, handshake order, and the Account/
  `GetAccountResponse`/`PlanType` shape — is marked **UNVERIFIED EXTERNAL
  ASSUMPTION** in `CODEX_INTEGRATION_REQUIREMENTS.md` §2, sourced only
  from a prior implementation's code comments. This must be independently
  verified against Codex's own current documentation or generated
  bindings as part of §19 step 4, not assumed from this document or that
  source.
- Whether the Codex App Server ever opens a network listener as part of
  its stdio-based invocation is unverified; the private-transport
  invariant (§11) must be checked against the real, current invocation
  shape.
- Each provider's actual, real authentication mechanism — needed to
  design a correct isolation plan (§6) that neither leaks unrelated host
  state nor accidentally breaks the provider's own login — is unverified
  for both Claude Code and Codex and must be established in §19 step 4.
- Whether a Git-isolation approach (if either provider needs one, per
  `AC-CODEX-SEC-014`) fully closes every credential/identity vector, or
  only the ones a prior implementation's comments enumerate, is unverified
  and must be re-evaluated when that adapter's isolation plan (§6) is
  designed.
- The permanent account/session compliance policy for Codex (as opposed to
  the policy-independent mechanism, §3) is a product decision not yet made
  anywhere in the authoritative inputs.
- The degree to which v1's destructive-cleanup approach can be defeated by
  a sufficiently fast or privileged local actor is not established as
  excluded (§11 residual limitation) and should be revisited if the
  product's threat model changes.
- Whether v1 will ever need a non-process-backed provider (§3's open
  question) remains unestablished; nothing in the authoritative inputs
  requires it.
- No cross-platform lifetime-coupling mechanism (a way to guarantee a
  provider process cannot outlive AgentCubicles, §8) has been evaluated;
  §8's conservative unknown-after-restart rule is the adopted v1 posture.
  Whether to invest in a verified lifetime-coupling mechanism or a
  positively-verifiable orphan-detection technique instead is a deferred
  product/platform decision, not resolved here.
