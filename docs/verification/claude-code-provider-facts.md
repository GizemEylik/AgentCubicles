# Claude Code Provider — Verified Integration Facts

Status: **T01 (Claude Code provider verification) is CLOSED —
REQUIRED-NOW = 0.** Every fact T01's gate required is VERIFIED (see
"Remaining requirements before §19 step 4 is complete" for the full
checklist). The broader `ARCHITECTURE.md` §19 step 4 milestone remains
**INCOMPLETE** only because of explicitly deferrable, non-blocking
UNKNOWNs recorded below and Codex's separate, not-yet-addressed facts
(T02, out of scope for this document) — neither reopens T01 or represents
a hidden blocker.

Task 1C compliantly re-verified the existing process/output/session
findings in §§1-5 and §7 as far as a credential-free probe can establish
them. The probes used a disposable workspace, an isolated state root,
and an explicitly constructed child environment containing no ambient
provider credential. No real project, real `~/.claude` state, or real
user credential was used.

The following are now **VERIFIED** under those constraints: the CLI and
version are present on Windows; `-p` is the non-interactive print-and-exit
surface; that surface is process-backed and returns an OS exit code;
`json` and line-delimited `stream-json` output are available; logged-out
model calls and an invalid resume have the concrete error behavior in
§4; and the CLI creates and recognizes an Execution-independent,
disk-backed session identifier in its isolated state root when persistence
is enabled.

The following remain **UNKNOWN for the credential-free `ANTHROPIC_API_KEY`
path specifically**, because a credential-free run cannot reach
authenticated model work: successful-result shape and exit status,
successful-turn readiness/handshake behavior, mid-turn API failures,
cancellation of active model work, and semantic conversation continuity
after resume. Help-visible but unexercised flags are labeled as such.
Mid-turn API failures, cancellation, and semantic conversation continuity
are UNKNOWN but **deferrable/non-blocking for T01**.

A later task (T01-E4) established **successful-result shape/exit status**
and **handshake/readiness behavior** using a genuine subscription login
instead (finding 6) — since T01's gate only required these facts on at
least one supported auth path, this closes both as **VERIFIED**, and
neither remains a T01 blocker. A subsequent task (T01-E5) established the
**approval/permission-prompt mechanism** the same way (finding 9):
**VERIFIED EXISTS**, machine-observable, and now closed. See findings 6
and 9 for the evidence and the "Remaining requirements" section — no
REQUIRED-NOW T01 fact remains open.

Sections 6 and 8 concern auth/OAuth/compliance. Task 1C did not investigate
or alter their prior evidence classification. The original ambient-state,
ambient-credential probes are retained below only as explicitly
**NON-COMPLIANT historical evidence** and are not used to satisfy §19
step 4. A separate documentation-only review of official Anthropic/Claude
Code sources (see §8 and its dedicated evidence-sources entry below) later
added VERIFIED request-level gating/error-behavior evidence to §8 for the
v1-supported `ANTHROPIC_API_KEY` path, but did **not** resolve §8's
underlying architecture-level compliance-concept question, which remains
**UNKNOWN** — never ABSENT. Per the narrow compliance-style-check exception
recorded in `ARCHITECTURE.md` §20 (added after all reasonably available
compliant verification means were exhausted for that sub-question alone),
this factual UNKNOWN itself no longer requires further verification
before Claude Code adapter implementation begins: conditions 1 and 2 of
that exception (UNKNOWN recorded, never ABSENT; verified request-level
eligibility-failure evidence documented) are satisfied by this document
today. **The exception as a whole is not yet fully satisfied**, because
its condition 3 — the Claude Code adapter actually implementing §3's
fail-closed handling for any ambiguous or unclassifiable outcome — is a
future implementation gate, not yet built. Proceeding under the exception
therefore still requires that implementation gate to be met before a
user-capable Claude Code Execution may rely on the exception. See §8 for
the exception's exact conditions and status. Every other Claude Code
UNKNOWN in this document (§§2-5) is unaffected by this exception. Of
`ARCHITECTURE.md` §20's Claude Code bullet items, the session concept and
process-backed execution mode are already **VERIFIED** (findings 5 and 2),
and as of T01-E4 the handshake/readiness phase is also **VERIFIED**
(finding 6, via the subscription path). As of T01-E5, the approval
mechanism is also **VERIFIED EXISTS** (finding 9 is authoritative for the
observed surface); none of this bullet's items remain open, and no
REQUIRED-NOW T01 verification fact remains.

A later architecture decision (`PRODUCT_REQUIREMENTS.md` §15/§16,
`ARCHITECTURE.md` §5/§6) added a **second** v1 Claude Code authentication
path — the user's own subscription login through an isolated, opaque,
per-project provider-auth profile that AgentCubicles never reads, copies,
parses, or relays. T01-E4 exercised this design end to end with a real
subscription login (see finding 6's dedicated evidence-sources entry) and
established the genuine `authMethod` value, successful-task result
shape/exit status, absence of a distinct handshake phase, and
`--safe-mode`/`--no-session-persistence` behavior under authentication —
all now **VERIFIED**. The one-time login step's own output/result shape
remains UNKNOWN but was already classified as not required for the T01
gate. OAuth/subscription authentication is therefore no longer
categorically deferred from v1, and its T01-required-now facts are now
verified per finding 6.

## Evidence sources

### Evidence sources — Task 1C process/output/session re-verification

- `claude --version` and `claude --help`, executed with the same isolated
  child environment as the operational probes.
- Logged-out `json`, `stream-json`, invalid-resume, persisted-session, and
  resume probes described under "Probes run".
- A fresh temporary root for each probe group. Each root contained both
  the working directory and all home/config/cache/state/temp locations.
- For every child process, the parent cleared the inherited environment
  and added only required operating-system variables. It redirected
  `HOME`, `USERPROFILE`, `APPDATA`, `LOCALAPPDATA`, `TEMP`, `TMP`, all
  relevant `XDG_*` roots, and `CLAUDE_CONFIG_DIR` into the temporary
  root. It supplied no `ANTHROPIC_API_KEY` or other provider credential.
- `claude auth status` was used only as the initial safety check for this
  probe environment. It reported `loggedIn:false`, `authMethod:"none"`,
  and config/project directories inside the temporary root. Task 1C did
  not investigate OAuth or compliance.

### Evidence sources — original process/output/session probes (NON-COMPLIANT)

The first probe batch used a scratch working directory but inherited the
host's real Claude Code authentication and real shared `~/.claude` state.
It violated §19 step 4 and is not authority for any VERIFIED Task 1C fact.
Any stronger successful-turn or semantic-resume conclusions from that
batch remain historical signal only and are classified UNKNOWN here.

### Evidence sources — auth/compliance re-run (compliant)
A separate, later batch of probes was run specifically to close the auth
and compliance-concept UNKNOWNs left by the batch above, this time
satisfying ARCHITECTURE.md §19 step 4:
- **Test-owned/disposable state root:** every command was run with the
  `HOME` and `USERPROFILE` environment variables both overridden to a
  fresh, empty directory under this session's own temp scratchpad
  (`.../scratchpad/auth-probe/home`), never the real `~/.claude`. `claude
  auth status` itself reported back `configDirectory`/`projectsDirectory`
  paths rooted under that scratch directory, confirming the override was
  honored (Claude Code resolves its config root from `HOME`/`USERPROFILE`,
  not a fixed path) and that the real `~/.claude` state root was not
  touched by these probes.
- **Throwaway/test credential:** one probe additionally set
  `ANTHROPIC_API_KEY=sk-ant-test-throwaway-not-a-real-key-000000`, an
  obviously fake value, never a real account credential.
- **No real project workspace:** commands were run from a scratch working
  directory under the same isolated scratchpad tree, never a registered
  AgentCubicles project.
- The isolated scratch state root (`.../scratchpad/auth-probe/`) was
  deleted after the probes. Deletion of one subdirectory
  (`auth-probe/proj`, the scratch working directory used for the timed-out
  fake-key `-p` run below) failed with a "device or resource busy" error,
  most likely because the OS process spawned for that run was still
  exiting when cleanup ran; the directory was confirmed empty (no files)
  before the failed removal. It contains no credential material and is
  not under any real `~/.claude` or project path.

### Evidence sources — genuine subscription-mode probe (compliant, real login, T01-E4)

A separate task exercised the subscription-mode design end to end using a
real Claude Pro subscription, under the isolation and non-observation
constraints `ARCHITECTURE.md` §5/§6 require:

- **Isolated, empty `CLAUDE_CONFIG_DIR`/`HOME`/`USERPROFILE`**, created
  fresh under this session's own temp scratchpad
  (`.../t01e4-sub-probe/{config,home,proj}`), never the real `~/.claude`.
- **Real interactive login, performed by the user, not AgentCubicles.**
  The user ran the unmodified `claude` CLI themselves against the isolated
  `CLAUDE_CONFIG_DIR` and completed Anthropic's own browser OAuth flow with
  their own subscription. No credential exchange was read, copied, parsed,
  or observed by this task at any point.
- **Only credential-free commands were run afterward**: `claude auth
  status`, and two `claude -p` calls (`--output-format json` and
  `--output-format stream-json --verbose`) with `--safe-mode`,
  `--no-session-persistence`, `--tools ""`, from the isolated scratch
  working directory, never a registered AgentCubicles project.
- **Cleanup:** the isolated `config` and `proj` directories (including the
  linked credential) were deleted after the probe. One leftover file
  (`home/AppData/Local/NVIDIA/DXCache/*.nvph`, an unrelated GPU
  shader-cache artifact created as an OS/driver side effect, not Claude
  Code state) could not be removed immediately (file in use); it contains
  no credential or session material.

### Evidence sources — approval/permission-prompt probe (compliant, real login, T01-E5)

A separate task exercised the smallest safe tool-enabled probe needed to
classify the approval mechanism, reusing the T01-E4 subscription-auth
approach with a fresh, disposable profile/workspace:

- **Fresh isolated `CLAUDE_CONFIG_DIR`/`HOME`/`USERPROFILE`**, created
  under a new temp scratchpad root (`.../t01e5-approval-probe/
  {config,home,proj}`), never the real `~/.claude`; the prior T01-E4
  profile had already been deleted.
- **Real interactive login, performed by the user, not AgentCubicles**,
  exactly as in T01-E4 — no credential exchange read, copied, parsed, or
  observed.
- **Only credential-free `claude auth status`, then narrowly tool-scoped
  `claude -p` calls**: `--tools "Bash"` with a harmless `echo` command, and
  `--tools "Write"` with a request to create one small file inside the
  disposable working directory only — never a real project, never
  broader/default tool access, never `--dangerously-skip-permissions`.
  `--safe-mode` and `--no-session-persistence` were set throughout.
- **Cleanup:** the isolated `config`, `proj`, and `home` directories
  (including the linked credential) were fully deleted after the probe,
  along with local temp files holding raw probe output.

### Evidence sources — documentation-only request-error review for §8 (compliant, no probes)

A separate task (no live probes, no account/credential exercised) reviewed
current official Anthropic documentation for documented request-level
error/gating behavior on the v1-supported `ANTHROPIC_API_KEY` path
specifically. This review established VERIFIED ordinary request-error
facts (see §8); it did not exercise a real account and does not establish
or rule out the separate architecture-level compliance-concept question
§8 also covers, which remains UNKNOWN/blocked:

- [Claude API errors](https://platform.claude.com/docs/en/api/errors)
  (official Anthropic API reference), read directly, quoted verbatim below.
- [Error reference — Claude Code Docs](https://code.claude.com/docs/en/errors)
  (official Claude Code CLI reference), read directly; the named error
  strings cited below were confirmed present in that page's own error
  index table. The full causal/detail paragraph text for each entry could
  not be retrieved (page content was truncated on fetch), so only the
  verified index-table strings are cited, not their full detail sections.
- No GitHub issues, third-party blogs, or other non-official sources were
  used as evidence for this finding; any such sources encountered during
  research were discarded as non-authoritative.

## Findings

### 1. Non-interactive launch method
**VERIFIED under compliant isolation.** `claude --help` identifies the
default invocation as interactive and `-p` / `--print` as the
non-interactive mode that prints a response and exits. It also states
that workspace trust is skipped with `--print` or when standard input is
not a TTY. The help surface exposes `text`, `json`, and `stream-json`
output formats and `text` and `stream-json` input formats for print mode.
Credential-free `json` and `stream-json` print invocations both ran as
bounded one-shot processes and exited after reporting the authentication
failure described in §4.

### 2. Process/execution model
**VERIFIED for the probed `claude -p` surface:** it is process-backed. The
probe harness launched an ordinary OS child process, observed it run to
completion, and received exit code `1` for the logged-out calls. The
streaming call emitted a `system` initialization event before its
authentication-failure assistant/result events.

**VERIFIED for the probed `claude -p` surface, authenticated via
subscription login (T01-E4, finding 6):** no additional readiness or
handshake phase exists before an authenticated turn begins. No separate
handshake was visible on the logged-out path either; T01-E4 confirmed this
holds for a genuine authenticated turn too, on the subscription-login
path. "Started" and "working" are the same moment for that tested surface.

**Scope of this VERIFIED conclusion — not generalized to
`ANTHROPIC_API_KEY`.** This finding covers only the subscription-login
path actually tested. Whether an authenticated turn over the
`ANTHROPIC_API_KEY` path has any additional readiness/handshake behavior
remains **UNKNOWN** — no credential-bearing `ANTHROPIC_API_KEY` probe has
been run (the earlier attempt below found no throwaway/test key
available). This is **not** a T01 blocker: T01's gate required this fact
on at least one supported auth path, and the subscription path satisfies
it; a duplicate `ANTHROPIC_API_KEY`-specific verification is not required
unless future evidence shows auth mode changes this behavior. Other
possible Claude integration modes outside the probed CLI print surface
were not investigated.

**Attempted, BLOCKED — no probe run (historical, `ANTHROPIC_API_KEY`
specifically):** a follow-up task attempted to resolve the
`ANTHROPIC_API_KEY`-path handshake UNKNOWN (and the successful-result-shape
UNKNOWN in finding 3/4) using a throwaway/test `ANTHROPIC_API_KEY`, per
that task's explicit requirement not to fall back to a real or ambient
credential if none is available. No throwaway/test key was available in
that task's environment (`ANTHROPIC_API_KEY` was unset, and provisioning a
real Anthropic API key requires an out-of-band Console account/billing
action outside that task's scope). No probe was run, no credential was
fabricated or substituted, and no existing finding in this document was
changed as a result. This `ANTHROPIC_API_KEY`-specific UNKNOWN remains open
pending a genuinely available throwaway/test API key, but is not required
for T01, per above.

### 3. Machine-readable / streaming output
**VERIFIED under compliant isolation, for logged-out behavior:**

- `--output-format json` emitted one JSON result object. Its observed
  authentication-failure fields included `type:"result"`,
  `subtype:"success"`, `is_error:true`, `result:"Not logged in · Please
  run /login"`, `session_id`, `terminal_reason:"api_error"`, zero usage,
  and `total_cost_usd:0`. Consumers must not treat `subtype:"success"`
  alone as successful work; `is_error`, terminal fields, and process
  outcome matter.
- `--output-format stream-json --verbose` emitted three line-delimited
  JSON objects: a `system`/`init` event, an `assistant` event containing
  an authentication-failed message, and a final `result` event with
  `is_error:true` and `terminal_reason:"api_error"`. All three carried
  the same `session_id`. The init event identified the isolated scratch
  working directory, no tools, no MCP servers, `apiKeySource:"none"`, and
  CLI version `2.1.270`.

**UNKNOWN:** the successful authenticated result/event shapes, their
completion fields, real token/cost accounting, partial-message behavior,
and streaming-input behavior. The flags for `--include-partial-messages`
and `--input-format stream-json` are help-confirmed but were not exercised.

### 4. Completion / error / cancellation behavior
- **Successful completion: UNKNOWN.** Without a credential, the probes
  could not reach completed model work or verify its exit code and
  terminal fields.
- **Authentication error: VERIFIED.** Both `json` and `stream-json`
  logged-out calls exited `1`. Their final result objects used
  `subtype:"success"` together with `is_error:true`, the message "Not
  logged in · Please run /login", and `terminal_reason:"api_error"`.
- **Invalid resume: VERIFIED.** Resuming a nonexistent, syntactically
  valid UUID exited `1`, emitted no standard output, and wrote plain text
  to standard error: "No conversation found with session ID: <uuid>".
  Requesting JSON therefore does not make every preflight error JSON.
- **Mid-turn error and cancellation: UNKNOWN.** Credential-free work
  cannot create a long-running model turn suitable for either probe.
  Task 1C makes no claim that a particular signal or stop mechanism was
  verified.

### 5. Session/resume behavior
**VERIFIED structurally under compliant isolation:**

- Each probed print call returned a UUID `session_id` in its JSON output.
- With persistence enabled, the first logged-out call created
  `<isolated-home>/.claude/projects/<encoded-scratch-cwd>/<session_id>.jsonl`.
  All persistence remained inside the disposable state root.
- A second process using `--resume <session_id>` recognized the stored
  session rather than returning "No conversation found", returned the
  same `session_id`, and appended/used the same isolated session context.
- The `--no-session-persistence` probes produced no project transcript in
  their isolated state root.
- Help confirms `--resume`, `--continue`, `--fork-session`,
  `--session-id`, and `--no-session-persistence` as native CLI surfaces.

**UNKNOWN:** semantic conversation continuity across processes. Both
persisted and resumed calls stopped at authentication failure, so the
credential-free probe could not prove that model-visible conversational
content is restored. Behavior of `--continue`, `--fork-session`, and a
caller-assigned `--session-id` is help-confirmed but not live-probed.

### 6. Required auth/config/state inputs
Confirmed for the mechanism/state-location questions in scope; some
sub-questions remain UNKNOWN as noted below. Verified compliantly (see
"Evidence sources — auth/compliance re-run"):
- **State root resolution:** Claude Code resolves its config/state root
  from the `HOME` (and, on Windows, `USERPROFILE`) environment variable,
  not a hardcoded path — overriding both to an isolated scratch directory
  caused `claude auth status` to report `configDirectory` and
  `projectsDirectory` rooted entirely under that scratch directory, and
  to create `.claude/` (with `projects/`, `sessions/`, `backups/`,
  `.last-cleanup`) plus a `.claude.json` file there instead of touching
  the real `~/.claude`. This is itself the concrete mechanism an
  isolation plan (§6 of ARCHITECTURE.md) can rely on to sandbox a Claude
  Code invocation's state.
- **`claude auth status` as a direct auth-mechanism probe:** run with a
  clean, never-logged-in isolated state root, it printed machine-readable
  JSON: `{"loggedIn": false, "authMethod": "none", "apiProvider":
  "firstParty", "analyticsDisabled": false, "projectsDirectory": "...",
  "configDirectory": "..."}`. Re-run in the same isolated state root with
  only `ANTHROPIC_API_KEY` set to an obviously fake/throwaway value, it
  reported `{"loggedIn": true, "authMethod": "api_key", "apiProvider":
  "firstParty", ..., "apiKeySource": "ANTHROPIC_API_KEY"}` — confirming
  the env var alone flips `loggedIn`/`authMethod` without any network
  validation of the key's actual validity (this call does not appear to
  contact the API). `apiProvider: "firstParty"` (vs. presumably
  `bedrock`/`vertex`/`foundry`, not directly observed) and `authMethod`
  (`"none"` vs. `"api_key"`, other values such as an OAuth-token variant
  not directly observed) are therefore confirmed as real, distinct,
  machine-readable states an adapter can check without parsing prose.
- **`claude auth` subcommand family** (`login`, `logout`, `status`) is
  confirmed to exist, with `login` supporting `--claudeai` (Claude
  subscription, default), `--console` (Anthropic Console/API billing),
  `--sso`, and `--email <email>` to prefill the login page — all of which
  are interactive/browser-driven flows per `--help` text; **not** probed
  live, since doing so would require either a real account or would open
  an interactive browser flow, neither compliant with step 4's
  throwaway-credential requirement. A separate `claude setup-token`
  command exists to mint "a long-lived authentication token" but
  explicitly "requires Claude subscription" — not probed for the same
  reason.
- **Behavior of an actual model call with a fake API key:** a `-p` run
  (`claude -p "hi" --output-format json --tools "" --no-session-persistence`)
  in the same isolated state root with the fake `ANTHROPIC_API_KEY` set
  produced **no output and no exit within 25 seconds** (bounded via
  `timeout 25`, which killed it) — i.e. an invalid API key does not fail
  fast with a clean error message; it hangs (silently, on stdout/stderr)
  for at least 25s, unlike the fast plain-text failure observed for the
  invalid-session-id case in finding 4. **UNKNOWN:** what happens if the
  process is left running longer — whether it eventually times out with
  an error, retries indefinitely, or something else — not probed further
  (bounding the probe at 25s was judged sufficient to establish the
  "does not fail fast" fact without an open-ended non-interactive hang).
- `--help` text for `--bare` mode states that outside of bare mode,
  "Anthropic auth" can come from `ANTHROPIC_API_KEY`, an `apiKeyHelper`
  (via `--settings`), or OAuth/keychain reads. The isolated re-run only
  exercised the `ANTHROPIC_API_KEY` path directly; the OAuth/keychain
  path was not exercised (doing so would require a real account login,
  which is not a throwaway/test credential and is out of scope for a
  compliant probe).
- Third-party providers (Bedrock/Vertex/Foundry) are called out in
  `--help` as using their own separate credential paths — not probed.
- **UNKNOWN — the exact OAuth/keychain token storage format/location is no
  longer needed and is not being tracked as a blocker.** `ARCHITECTURE.md`
  §5/§6 now define a subscription-mode design (v1-supported, per
  `PRODUCT_REQUIREMENTS.md` §15/§16) that never requires AgentCubicles to
  read, copy, parse, or relay this format at all: the user signs in
  directly through the unmodified `claude` CLI into an isolated,
  AgentCubicles-owned, opaque, per-project provider-auth profile
  (`CLAUDE_CONFIG_DIR`), and AgentCubicles checks login state only via
  `claude auth status`'s machine-readable fields — never by inspecting the
  profile's contents. This design was originally an **architecture
  decision, not a verified provider fact**; **T01-E4 has since exercised it
  end-to-end** with a genuine Claude Pro subscription login, an isolated
  AgentCubicles-created profile (never the real `~/.claude`), `--safe-mode`,
  `--no-session-persistence`, and one successful authenticated `claude -p`
  call — see the VERIFIED bullets immediately below.
- **VERIFIED — genuine subscription `authMethod`.** A real Claude Pro
  subscription was signed into an isolated, empty, AgentCubicles-created
  `CLAUDE_CONFIG_DIR` (never the real `~/.claude`) through the unmodified
  `claude` CLI's own interactive login, with the user completing the
  browser OAuth flow directly. AgentCubicles did not read, copy, parse, or
  observe the credential exchange itself — only the credential-free
  `claude auth status` output afterward. That output: exit `0`,
  `{"loggedIn": true, "authMethod": "claude.ai", "apiProvider":
  "firstParty", "analyticsDisabled": false, "projectsDirectory": "<isolated
  path>", "configDirectory": "<isolated path>", "email": "[redacted —
  real account email, not recorded]", "orgId": "[redacted]", "orgName":
  "[redacted]", "subscriptionType": "pro"}`. `authMethod: "claude.ai"` is
  now a confirmed, distinct value from `"none"` and `"api_key"`.
  `email`/`orgId`/`orgName`/`subscriptionType` are new fields present only
  for a genuine logged-in session. `subscriptionType`'s actual value
  (`"pro"`) **is** recorded above — it is a non-personal account
  classification fact, not credential or identity material. Only
  `email`/`orgId`/`orgName`'s actual values are intentionally not
  recorded (personal account data, not needed by the fail-closed gate,
  which only depends on `loggedIn`/`authMethod`); their presence/shape is
  recorded, not their values.
- **VERIFIED — successful authenticated task result shape and exit
  status.** One trivial authenticated `claude -p "Reply exactly OK"` call
  against the same isolated profile, with `--safe-mode`,
  `--no-session-persistence`, `--tools ""`, no project modification:
  exit `0`. `--output-format json` returned one result object with
  `is_error:false`, `subtype:"success"`, `terminal_reason:"completed"`,
  `stop_reason:"end_turn"`, `result:"OK"`, plus real, non-zero token/cost
  usage accounting (`total_cost_usd`, `usage`, `modelUsage` per-model
  breakdown), `permission_denials:[]`, and `session_id`. This is
  materially richer than the logged-out authentication-failure shape in
  finding 3/4 — consumers must branch on `is_error`/`subtype`/
  `terminal_reason`, not assume the logged-out shape generalizes.
- **VERIFIED — no distinct handshake/readiness phase.** The same call
  repeated with `--output-format stream-json --verbose` emitted, in order:
  one `system`/`init` event, one `rate_limit_event` (a new event type not
  seen in any logged-out probe — informational usage/quota data, not a
  gate), the `assistant` message, a second `rate_limit_event`, and the
  final `result` event — all under the same `session_id`. No separate
  pre-turn readiness/handshake step exists beyond the same `system`/`init`
  event already observed logged-out; "started" and "working" are the same
  moment for this authenticated path too.
- **VERIFIED — `--safe-mode` and `--no-session-persistence` under
  authentication.** Both calls above ran with both flags set and completed
  normally (exit `0`, no crash or incompatibility). After both runs, no
  `projects/` directory was created at all under the isolated
  `CLAUDE_CONFIG_DIR`, and no file anywhere under it matched either run's
  `session_id` — confirming `--no-session-persistence` leaves no
  transcript for an authenticated subscription-mode run, matching the
  logged-out finding in finding 5.
- **UNKNOWN, unaffected by this probe:** the login step's own
  output/result shape (not required for the T01 gate, per above) and
  whether auth mode changes the process/output/completion contract beyond
  what's shown here (the `ANTHROPIC_API_KEY`-specific live-success fact
  remains unverified and non-blocking, per its existing classification).

### 7. Version/platform caveats
- **VERIFIED under compliant isolation** on Windows/win32 with `claude`
  CLI `2.1.270`. The executable was resolvable on this probe host and the
  version command exited `0`.
- **UNKNOWN** on macOS, Linux, standalone installations, and CI hosts.
  Nothing in Task 1C establishes cross-platform behavior or install-path
  discovery.
- All CLI flags and output shapes recorded here are a versioned snapshot,
  not a stability guarantee.

### 8. Account/session compliance concept
**UNKNOWN — not verified, and never asserted as ABSENT — for the
ARCHITECTURE.md §3/§19 account/session compliance-concept question.**
Official documentation gives **VERIFIED request-level gating/error
behavior** for the v1-supported non-interactive `ANTHROPIC_API_KEY` path
(recorded below), but that evidence does not establish, and must not be
read as establishing, the existence of the specific architecture-level
concept §3/§19 ask about: a separate, positively-evaluated,
Execution-scoped compliance check comparable to Codex's
`GetAccountResponse`/`PlanType` (a check that "must exist and be
positively evaluated, fail closed on anything unclassifiable, be scoped
to exactly one execution," per §3). No such separate check or verdict has
been verified to exist for Claude Code, and none is verified absent
either — the question stays genuinely open.

**§8's factual UNKNOWN no longer requires further verification before
adapter implementation, per the narrow exception recorded in
`ARCHITECTURE.md` §20 — but that exception is only PARTIALLY satisfied
today, not fully closed.** The exception applies to this sub-question
because all reasonably available compliant verification means for it have
been exhausted (official provider documentation, reviewed below;
first-party source inspection, ruled out because Claude Code's core CLI
source is not publicly available; live-account behavioral probing of a
real credential, excluded as non-compliant). Its three conditions stand
as follows:

1. **Satisfied today.** The compliance-concept existence question remains
   recorded here as UNKNOWN, never ABSENT.
2. **Satisfied today.** Verified request-level policy/eligibility failure
   behavior for Claude Code is documented immediately below.
3. **NOT yet satisfied — mandatory implementation gate, not a verification
   task, and not a T01 blocker.** The exception requires the Claude Code
   adapter to actually apply §3's fail-closed handling to any ambiguous or
   unclassifiable provider outcome, exactly as it would for any other
   provider. No Claude Code adapter has been implemented yet, so this
   condition is unmet. **Proceeding under this exception for a
   user-capable Claude Code Execution requires condition 3 to be satisfied
   by the adapter itself** — it is not discharged by this document's
   evidence and does not become true simply because conditions 1 and 2 are
   met. This condition belongs to adapter implementation, not to T01's
   verification gate (see "Remaining requirements" below).

This exception is scoped to the compliance-style-check sub-question
alone; it does not extend to, and does not close, any other Claude Code
UNKNOWN in this document. Of `ARCHITECTURE.md` §20's Claude Code bullet
items, the native session concept (finding 5: a disk-backed session
identifier is created and recognized on resume), the probed `-p` surface's
process-backed execution mode (finding 2), and — as of T01-E4, on the
subscription-login path — the handshake/readiness phase (finding 2) are
already VERIFIED and are not open facts (the handshake/readiness question
remains separately UNKNOWN, and non-blocking, for the untested
`ANTHROPIC_API_KEY` path specifically). As of T01-E5, the approval
mechanism is also **VERIFIED EXISTS** (finding 9) and is no longer open.
Every T01 REQUIRED-NOW fact in `ARCHITECTURE.md` §20's Claude Code bullet
is now closed — but not every fact in that bullet: the account/session
**compliance concept** in the same bullet remains genuinely **UNKNOWN and
unresolved**. It is non-blocking for T01 solely because of the existing
§20 narrow exception (above), not because the underlying question has been
answered.

**VERIFIED request-level gating/error behavior** (documentation-sourced,
no probe run, no real account exercised) — this is ordinary API/CLI
request-failure behavior, not a compliance concept:
- [Claude API errors](https://platform.claude.com/docs/en/api/errors)
  (official Anthropic API reference — the API surface Claude Code's
  `ANTHROPIC_API_KEY` path calls via the `X-Api-Key` header) documents a
  `403 permission_error` type, separate from `401 authentication_error`:
  "Your API key does not have permission to use the specified resource.
  Check your organization's access and workspace settings in the Claude
  Console." `401 authentication_error` is defined separately, only for a
  malformed, revoked, or expired key. The same page documents `402
  billing_error` ("There's an issue with your billing or payment
  information") and notes `429 rate_limit_error` also fires when an
  organization or the Claude Code workspace has reached a spend cap.
- [Error reference — Claude Code Docs](https://code.claude.com/docs/en/errors)
  (official Claude Code CLI reference) lists, in its own error index
  table, organization-level error strings distinct from a plain
  invalid-key message, including `This organization has been disabled`,
  `Your organization has disabled API key authentication`, and `Model ...
  is restricted by your organization's settings`. These entries were
  confirmed present in the live page's index table; the full detail
  paragraph for each (exact trigger conditions and wording) could not be
  retrieved in this task due to page-content truncation on fetch, so only
  the confirmed index-table strings are cited as evidence here, not their
  full detail sections.

Why this evidence does not resolve the underlying factual question (it
only satisfies the §20 exception's evidentiary condition, above — it does
not prove EXISTS or ABSENT):
- All of the above are **ordinary per-request error responses** returned
  when a specific API/CLI request is attempted (an HTTP status/error
  `type`, or a CLI error string surfaced from one) — the same category of
  fact as any other documented authentication or rate-limit failure
  already covered by §§2-5. They are ARCHITECTURE.md §3's "ordinary
  authentication/API request errors" category, not evidence of a distinct
  compliance mechanism layered on top of that lifecycle.
- §3/§19's compliance-concept question is about a **separate, positively
  evaluated, Execution-scoped verdict** — something an adapter would need
  to check and act on independent of, or prior to, an ordinary request
  outcome (comparable to Codex's account/session check). No official
  documentation reviewed describes such a distinct compliance-check
  endpoint, handshake, or pre-flight verdict for the `ANTHROPIC_API_KEY`
  path. Its absence from the documentation reviewed is, per §3's own
  framing, negative evidence only from a narrow documentation surface —
  it does not rule the concept in or out.
- Whether a real (non-throwaway) account can be gated in a way that
  requires the adapter to model a separate compliance verdict (rather
  than simply surfacing the request-level error, as it already must for
  §§2-5's ordinary failures) remains **UNKNOWN**. This has not been tested
  against a real account, and no official documentation reviewed answers
  it either way.
- This finding does not claim every possible account/organization state
  or every error string has been enumerated; the full detail sections of
  the Claude Code CLI error reference for the three named entries above
  were not retrieved verbatim.
- Scope: this UNKNOWN concerns the v1-supported non-interactive
  `ANTHROPIC_API_KEY` path only. Claude Code v1 also supports a
  subscription-login path (`PRODUCT_REQUIREMENTS.md` §15/§16,
  `ARCHITECTURE.md` §5/§6), but that path's own request-level/compliance
  behavior has not been reviewed or probed at all and is not covered by
  this finding — see finding 6's new subscription-mode UNKNOWNs.

### 9. Approval/permission-prompt mechanism (T01-E5)

**VERIFIED EXISTS.** Claude Code's `-p` surface has a real, enforced,
machine-observable approval/permission-prompt mechanism distinct from
ordinary tool success/failure. Verified via the subscription-login path
(same isolation/non-observation constraints as T01-E4: fresh isolated
`CLAUDE_CONFIG_DIR`, real user-driven login, `--safe-mode`,
`--no-session-persistence`, disposable working directory, no real
project/user files touched):

- **Credential-free discovery:** `--help` documents `--permission-mode
  <mode>` (`acceptEdits`, `auto`, `bypassPermissions`, `manual`, `dontAsk`,
  `plan`) and `--permission-prompts <target>` for `-p`: `"host"` (the SDK
  host or `--permission-prompt-tool` answers) or `"none"` — *"nobody:
  anything that would prompt is denied automatically; the permission mode
  still decides everything else."*
- **A harmless, read-only-style `Bash` tool call (`echo <string>`) was NOT
  gated** under `--permission-mode default`, with and without
  `--permission-prompts none`: it executed normally, `permission_denials`
  stayed `[]`, and no distinct permission-request event appeared in the
  `stream-json --verbose` event sequence (`system/init` →
  `rate_limit_event` → `assistant`(thinking) → `assistant`(`tool_use`) →
  `user`(`tool_result`, `is_error:false`) → `assistant`(text) → `result`).
  Not every tool action is gated; simple/benign commands were not.
- **A `Write` tool call (creating a new file inside the disposable working
  directory) WAS gated and denied** under `--permission-mode default
  --permission-prompts none`. The write did **not** occur (confirmed: no
  file was created). Observed surface:
  - Top-level `result` object gained a populated **`permission_denials`**
    array: `[{"tool_name":"Write","tool_use_id":"<id>","tool_input":
    {"file_path":"<path>","content":"<content>"}}]` — structured,
    machine-parseable, naming the specific tool and its input.
  - The individual `tool_result` for that call carried `"is_error":true`
    and a fixed, model-facing denial message: *"Permission for this tool
    use was denied. It requires approval, and this session has no
    approval surface... it was denied automatically... this action, and
    anything else that requires approval, will be denied the same way for
    the rest of this session."*
  - That `tool_result`'s `tool_use_result` carried a
    `"tool_result_meta":[{"id":"<id>","non_execution_kind":
    "permission-rule"}]` field — a distinct, structured classification an
    adapter could key on without parsing prose.
  - **The overall process/turn still completed as an ordinary success**:
    exit `0`, top-level `is_error:false`, `subtype:"success"`,
    `terminal_reason:"completed"`. A denied permission does **not** fail
    the Execution or change its exit code — an adapter must inspect
    `permission_denials` (and/or individual `tool_result`
    `is_error`/`tool_result_meta.non_execution_kind`) to detect that
    something was blocked; it cannot rely on top-level `is_error` or exit
    status alone.
- **Exit/result behavior when approval is unavailable:** confirmed
  fail-closed at the tool-action level (the specific action is blocked,
  not silently allowed) but **not** fail-closed at the process level (the
  overall turn reports success). This is a real behavioral asymmetry an
  adapter's fail-closed handling (`ARCHITECTURE.md` §3) must account for
  explicitly, rather than assuming a blocked action surfaces as a failed
  Execution.
- **Not tested:** `--permission-prompts host` (would require a live
  approval-answering surface, out of this probe's scope), other
  `--permission-mode` values, and whether other tool types/actions beyond
  `Bash`(benign) and `Write` are gated differently.

## Probes run

### Task 1C compliant process/output/session probes

Each command ran from a new disposable workspace with an isolated home,
config, cache, state, and temp root. The child environment was cleared
before the required operating-system variables and isolated path
variables were added. No ambient provider credential was passed.

1. `claude auth status` as an isolation safety check: exit `1`;
   `loggedIn:false`, `authMethod:"none"`; reported config and projects
   paths were under the disposable root.
2. `claude --version`: exit `0`; `2.1.270 (Claude Code)`.
3. `claude --help`: exit `0`; captured the invocation, format,
   persistence, and session flag descriptions used in §§1, 3, and 5.
4. `claude -p "Reply exactly OK" --output-format json --tools ""
   --no-session-persistence`: exit `1`; one JSON authentication-error
   result; no project transcript was observed.
5. `claude -p "Reply exactly OK" --output-format stream-json --verbose
   --tools "" --no-session-persistence`: exit `1`; three NDJSON events
   (`system`, `assistant`, `result`) with one session ID; no project
   transcript was observed.
6. `claude -p "hi" --output-format json --tools ""
   --no-session-persistence --resume
   00000000-0000-0000-0000-000000000000`: exit `1`; no stdout; plain
   invalid-session error on stderr.
7. `claude -p "Remember the number 42." --output-format json --tools ""`
   followed by `claude -p "What number?" --resume <first-session-id>
   --output-format json --tools ""`: both exited `1` with the logged-out
   result, returned the same session ID, and used the transcript created
   under the isolated state root. This verifies structural persistence
   and resume recognition, not semantic recall.

### Earlier evidence outside Task 1C

- The original process/output/session probes were **NON-COMPLIANT**
  because they inherited real ambient authentication and shared user
  state. They do not support the VERIFIED classifications above.
- The later §6/§8 auth/compliance probes and their classifications remain
  recorded in those sections. Task 1C did not repeat or extend them.

## Cleanup result

- Every Task 1C child process exited before cleanup.
- Both completed Task 1C temporary probe roots were removed successfully.
- One initial session-probe setup attempt failed before launching Claude
  because its local PowerShell variable conflicted with PowerShell's
  read-only `HOME` alias. Its empty, test-owned temporary root was
  enumerated only by the task-specific name prefix, checked to be under
  the resolved system temp directory, and removed before the corrected
  probe ran.
- Final cleanup checks reported the Task 1C roots absent. No real user
  project, real `~/.claude` path, or ambient credential was enumerated,
  read, written, or passed to a child process by Task 1C.

## Remaining requirements before §19 step 4 is complete

**Scope note:** this document's Claude Code facts are tracked under T01.
`ARCHITECTURE.md` §19 step 4 also requires Codex's corresponding facts,
but those belong to a separate track (T02) and must not be treated as
blocking T01's own closure — T01 closes on the Claude Code facts alone.

Task 1C closes the compliant-isolation gap for the limited credential-free
facts now marked VERIFIED in §§1-5 and §7. It cannot close facts that need
authenticated model work. For the `ANTHROPIC_API_KEY` path specifically:
successful completion/output/accounting and successful-turn
readiness/handshake behavior remain UNKNOWN pending a compliant
test/non-user credential — but neither is independently required for T01's
gate any longer, because **T01-E4 established both facts on the
subscription-login path instead** (finding 6), and T01's gate only ever
required them on at least one supported auth path. Semantic session
resume, mid-turn errors, and cancellation are also UNKNOWN, but — as
previously classified — remain **deferrable**, not required for T01's own
gate.

Subscription authentication is no longer an "OAuth sub-question... UNKNOWN/
blocked" item: `PRODUCT_REQUIREMENTS.md` §15/§16 and `ARCHITECTURE.md` §5/§6
make it an accepted v1 architecture path, not a deferred or blocked one.
T01-E4 verified its `authMethod` value, successful-task result shape/exit
status, absence of a distinct handshake phase, and `--safe-mode`/
`--no-session-persistence` behavior under a genuine authenticated
Execution (finding 6) — all now **VERIFIED**, closing REQUIRED-NOW items
1-4 below. The one-time interactive login step's own output/result shape
remains UNKNOWN but was already, independently of this probe, classified
as **not required as a parsed contract**: `ARCHITECTURE.md` §6 determines
linking success solely through the credential-free `claude auth status`
gate, never by parsing the login step's output. A separate
documentation-only task added VERIFIED request-level gating/error-behavior
evidence to §8 for the v1-supported `ANTHROPIC_API_KEY` path; that
evidence does not resolve, and is not claimed to resolve, §8's
architecture-level compliance-concept question — the underlying fact
remains genuinely **UNKNOWN**, never ABSENT (see §8 above). Per the narrow
compliance-style-check exception subsequently added to `ARCHITECTURE.md`
§20, §8's factual UNKNOWN **no longer requires further verification**
before Claude Code adapter implementation begins — conditions 1 and 2 of
that exception are satisfied by §8 as it now stands. Condition 3 (the
adapter actually implementing §3's fail-closed handling) is an **adapter
implementation gate, not a T01 verification blocker** — it applies once a
Claude Code adapter is built, and does not stand between T01 and its own
closure.

**Closed by T01-E4 (previously REQUIRED NOW, now VERIFIED — finding 6):**
1. The genuine subscription `authMethod` value (`"claude.ai"`).
2. Successful authenticated `claude -p` execution semantics (result shape
   and exit status), established on the subscription-login path. The
   `ANTHROPIC_API_KEY`-specific equivalent (findings 3/4) remains
   unverified but is **not** a separate T01 blocker, since T01-E4's
   evidence shows no sign that auth mode changes the underlying
   process/output/completion contract (both share the same
   `system`/`init` → turn → `result` shape and JSON result fields; only
   the specific error vs. success payload differs, as expected).
3. Successful-turn handshake/readiness classification: no distinct
   handshake phase exists, verified on the subscription-login path. The
   `ANTHROPIC_API_KEY` path was not separately tested and remains
   UNKNOWN, but is not a duplicate T01 blocker per item 2's reasoning
   above.
4. `--safe-mode` and `--no-session-persistence` behavior during a genuine
   authenticated subscription-mode Execution.

**Closed by T01-E5 (previously REQUIRED NOW, now VERIFIED EXISTS —
finding 9):**
5. The approval mechanism question — whether Claude Code has any
   approval/permission-prompt concept (`ARCHITECTURE.md` §20's Claude Code
   bullet). **VERIFIED EXISTS**: a `Write` tool call was denied and
   surfaced through a structured `permission_denials` array, a
   `tool_result` with `is_error:true` and `tool_result_meta.
   non_execution_kind:"permission-rule"` — machine-observable without
   parsing prose. The overall process/turn still completes as a top-level
   success (exit `0`, `is_error:false`) regardless of the denial, which an
   adapter's fail-closed handling must account for explicitly.

**REQUIRED NOW before the T01 full gate: none remaining.** All five items
above are now VERIFIED. T01's Claude Code facts are closed; only §8
condition 3 (an adapter-implementation gate, not a T01 verification item)
and the explicitly deferrable items below remain outstanding, neither of
which blocks T01's own closure.

**Not required now for T01 (deferrable, previously so classified; the
UNKNOWNs themselves remain recorded, not resolved):**
- The `ANTHROPIC_API_KEY`-specific live-success result shape/exit status
  (findings 3/4) — still factually UNKNOWN (no credential-bearing
  `ANTHROPIC_API_KEY` probe has ever run), but no longer a standalone T01
  blocker once item 2 above is satisfied via the subscription path, unless
  that verification surfaces evidence that the two auth modes produce a
  different process/output/completion contract.
- The `ANTHROPIC_API_KEY`-specific handshake/readiness behavior (finding
  2) — still factually UNKNOWN for that path specifically, not generalized
  from the subscription-path VERIFIED finding, and not a standalone T01
  blocker for the same reason as item 3 above.
- Mid-turn failure behavior (finding 4).
- Cancellation of active model work (finding 4).
- Semantic conversation continuity across resume (finding 5).
- The interactive login step's own output/result shape (finding 6) — not
  needed as a parsed contract, per the auth-status-gate design above.
- §8 condition 3 (adapter fail-closed implementation) — an implementation
  gate that applies once an adapter exists, not a T01 verification item.
- Codex's §19 step 4 facts — belong to T02, tracked separately, and never
  block T01.

Of `ARCHITECTURE.md` §20's Claude Code bullet items, the session concept,
process-backed execution mode, handshake/readiness phase (T01-E4), and
approval mechanism (T01-E5) are all **VERIFIED** (findings 5, 2, 6, and 9)
and are not open facts. **T01 REQUIRED-NOW provider-verification facts:
0 / CLOSED** for these four items. This does **not** mean every fact in
that bullet is resolved: the account/session **compliance concept**
remains genuinely **UNKNOWN and unresolved** — it is non-blocking for T01
only because of the existing §20 narrow exception, not because it has been
answered.

Subscription-mode authentication is no longer categorically deferred from
v1 (see the top-of-document status and findings 6 and 9); it is a
supported v1 path whose T01-required-now facts are all now verified.
