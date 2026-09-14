# Codex Integration Requirements

*Revision note: this document has been revised following an independent
specification review. Material changes: the account-eligibility rule in §5 is
now explicitly marked provisional rather than a permanent product
requirement; several test/security requirements that had implicitly baked in
the current implementation's technique have been rewritten to describe
outcomes only; the destructive-cleanup guarantees have been narrowed to match
what pure-Node/pathname-based verification can actually prove, with the
residual gap documented explicitly; the Account/`GetAccountResponse`/
`PlanType` shape has been reclassified as unverified; five new
implementation-independent security invariants and three new test
requirements have been added; and legacy-architecture rejections have been
extended to cover the cleanup technique and the private test-seam pattern.
Exact ID renumbering is documented in the mapping notes at the end of §7 and
§10 — no ID was changed purely for cosmetic reasons. A further follow-up
review found nine test requirements (`TEST-007`–`TEST-009`, `TEST-022`,
`TEST-045`, `TEST-049`, `TEST-054`, `TEST-067`, `TEST-068`) that still
prescribed a specific technique or an unsupported absolute claim; these were
rewritten in place (same IDs — see the addendum at the end of §10) to
require only the underlying outcome. A subsequent specification-consistency
pass (this revision) fixed six additional cross-reference inconsistencies
(cleanup-authorization wording in SEC-016/TEST-059/TEST-060/§8, resource-limit
failure semantics in §4/TEST-031/§8, revocation-timing claims in TEST-074/§8,
policy-independence of §5.1, remaining technique-mandates in §4/§11/TEST-050/
TEST-056/TEST-082, and missing direct test coverage for SEC-027/SEC-028 as new
TEST-085/TEST-086), scoped §3's non-overlap rule to a single managed
integration instance, corrected this note's own arithmetic, and reworded
TEST-030 to state its byte-oriented property directly. No requirement was
weakened; see the end of §10 for the full accounting. A final
specification-consistency pass resolved four MUST FIX findings (pre-spawn
cleanup wording across SEC-016/§8/TEST-060; approval-capacity semantics in
§4/TEST-031, preserving SEC-025; forward-looking revocation wording in
§8/TEST-078, alongside TEST-074's send-before/send-after distinction;
TEST-071's "block the wire" scoped to task-capable traffic only) plus four
follow-ups (§4 notifications no longer require normalizing security signals
first; TEST-044 restated as a one-directional safety property rather than
symmetric disjointness; TEST-069 made outcome-oriented; and the mapping
arithmetic corrected from "three merges removing four old IDs" to the
accurate three) — eight findings in total. No IDs were added or removed in
this pass; see the end of §10 for the corrected accounting.*

## 1. Purpose and Scope

This document preserves the **engineering knowledge** learned while prototyping
and hardening Codex CLI integration in the transition repository, so that
support can be **reimplemented independently** in the new AgentCubicles
architecture — against these requirements, not against the current code.

It intentionally does **not** preserve implementation expression: no algorithms,
no class/function structure, no internal variable names (internal review
labels such as "M2", "M6.1", "M7.2" are cited only as evidence pointers, never
as requirement names or an architecture to keep). Where a behavior appears to
exist only because of the current, upstream-derived `StreamProvider`/
`AgentEvent` architecture, it is explicitly marked **LEGACY-ARCHITECTURE
COUPLING** and excluded from the requirement set.

Inspected sources (per the audit's scope restriction — no other part of the
repository was read to produce this document, and this revision inspected no
additional source beyond what is cited here):

- `server/src/providers/stream/codex/codexAccountConfinement.ts`
- `server/src/providers/stream/codex/codexAppServerClient.ts`
- `server/src/providers/stream/codex/codexAppServerNormalizer.ts`
- `server/src/providers/stream/codex/codexAppServerProtocol.ts`
- `server/__tests__/codexAccountConfinement.test.ts`
- `server/__tests__/codexAppServerClient.test.ts`
- `server/__tests__/codexAppServerIsolation.test.ts`
- `server/__tests__/codexAppServerNormalizer.test.ts`
- `core/src/provider.ts` (read only narrowly, to understand the `StreamProvider`/
  `AgentEvent` contract these files implement against)

Out of scope for this document (not inspected): `server/src/providers/hook/codex/`
(a separate, non-functional hook-based Codex placeholder — see
`PROVENANCE_DECISIONS.md`), and every other part of the codebase.

No existing behavior is treated as automatically correct. This document
records what the current prototype *does* and *claims*, not what a new
implementation *must* copy. Where the prototype's own policy choices are
provisional rather than settled product decisions, that is called out
explicitly rather than presented as a requirement.

## 2. External Codex/App Server Facts

The current implementation targets the Codex CLI's **App Server** mode,
invoked as a subprocess speaking newline-delimited JSON-RPC-shaped messages
over stdio, rather than the transcript/hook mechanisms used elsewhere in this
codebase for other CLIs.

Every row below is classified **UNVERIFIED EXTERNAL ASSUMPTION**: all of them
are sourced only from the current implementation's own code comments, which
are not authoritative external documentation, however confidently worded.
This review did not independently inspect Codex's published documentation or
generated protocol bindings for any of them.

| Fact | Status | Evidence |
| --- | --- | --- |
| Launch shape: `<codex-executable> app-server --listen stdio:// --strict-config`, stdio fully piped (`['pipe','pipe','pipe']`) | UNVERIFIED EXTERNAL ASSUMPTION | `codexAppServerClient.ts:2107-2115` |
| Wire framing: one JSON object per line (`\n`-terminated); the App Server "uses JSON-RPC 2.0 semantics but intentionally omits the `jsonrpc` header" | UNVERIFIED EXTERNAL ASSUMPTION | `codexAppServerProtocol.ts:54-57` |
| Frame shapes: `notification {method, params}`, `request {id, method, params}`, `response {id, result?, error?}`; `id` is a number or a string | UNVERIFIED EXTERNAL ASSUMPTION | `codexAppServerProtocol.ts:1-52` |
| The App Server itself can send **requests** to the client that require a reply (approval requests), i.e. the protocol is bidirectional, not client-request/server-response only | UNVERIFIED EXTERNAL ASSUMPTION | `codexAppServerClient.ts:2517-2560`, `codexAppServerNormalizer.ts:98-110` |
| Confirmed startup handshake order: `initialize` (request/response) → `initialized` (notification) → `account/read` (probe request) | UNVERIFIED EXTERNAL ASSUMPTION (documented in-file as "confirmed" but with no citation to an external spec) | `codexAppServerClient.ts:2210-2224` |
| `GetAccountResponse = { account: Account \| null; requiresOpenaiAuth: boolean }`; `Account` is a discriminated union of `apiKey`, `chatgpt { email, planType }`, `amazonBedrock { usesCodexManagedCredentials }` | **UNVERIFIED EXTERNAL ASSUMPTION** — the implementation cites "independent review" of Codex CLI 0.153.4's generated TypeScript bindings, but this review did not itself inspect that generated binding or any other Codex documentation; the implementation's own citation is not treated as authoritative here | `codexAccountConfinement.ts:1-33` |
| `PlanType` has exactly 17 values (`free, go, plus, pro, prolite, team, self_serve_business_prolite, self_serve_business_usage_based, business, ent26, enterprise_cbp_automation, enterprise_cbp_usage_based, enterprise, edu, edu_plus, edu_pro, unknown`), where `unknown` is itself a real declared variant | UNVERIFIED EXTERNAL ASSUMPTION — same basis and same caveat as the row above | `codexAccountConfinement.ts:21-33`, `61-79` |
| Observed/assumed method vocabulary: `thread/started`, `turn/started`, `item/started`, `item/completed`, `turn/completed`, `thread/closed`, `serverRequest/resolved`, `item/commandExecution/requestApproval`, `item/fileChange/requestApproval`, `account/updated`, `account/login/start` | UNVERIFIED EXTERNAL ASSUMPTION | `codexAppServerNormalizer.ts:53-112` |
| Tool "item" types observed: `commandExecution`, `fileChange`, `mcpToolCall`, `dynamicToolCall`, `collabAgentToolCall`, `webSearch`, `imageView` | UNVERIFIED EXTERNAL ASSUMPTION | `codexAppServerNormalizer.ts:17-25` |
| `account/updated` is a **threadless** notification (carries no thread/session id) | UNVERIFIED EXTERNAL ASSUMPTION | `codexAppServerClient.ts:2661-2668` |
| A minimum Git version of **2.31.0** is required for the environment-variable mechanism the current prototype's isolation policy depends on | UNVERIFIED EXTERNAL ASSUMPTION about Git itself (a Git behavior claim, not a Codex one) | `codexAppServerClient.ts:390-397` |

**Read as a whole:** every fact above should be treated as "what the prototype
assumed while being built against a specific Codex CLI version," not as a
stable, externally-published contract, and not as something this review has
confirmed. A reimplementation must independently verify the current App
Server protocol (method names, payload shapes, handshake order, and the
Account/PlanType shape) against Codex's own current documentation or
generated bindings before relying on any of it — a comment in the prior
implementation asserting something was "confirmed" or "independently
reviewed" is not a substitute for that verification.

## 3. Process Lifecycle Requirements

- **Startup** must be atomic and non-overlapping **within the same managed
  integration/run instance** — i.e., the same logical agent's Codex lifecycle
  authority: a new run for that instance must not begin while that same
  instance's previous run's startup is still in flight, and must not begin
  while that same instance's previous run's termination has not yet been
  confirmed. This is a per-instance ordering guarantee, not a global one — it
  does not prohibit independent, concurrently-running Codex integrations
  belonging to different agents/instances, each with its own lifecycle
  authority and its own generation sequence. (`codexAppServerClient.ts:1619-1657`, `1828-1837`; test group "M2 HIGH 2 — cleanup lifecycle authorization" / "cross-generation lifecycle authorization", `codexAppServerClient.test.ts:6482,6690`)
- **Readiness** is not "process spawned" — a run only becomes usable for real
  work after (a) a successful protocol handshake and (b) a passed
  account/session compliance check (see §5). No task-capable request may
  reach the child before both hold. (`codexAppServerClient.ts:2210-2252`, §5 below)
- **Shutdown** must be graceful-first: signal the child to close normally, wait
  a bounded grace period, then force-terminate, then wait a second bounded
  window for the termination to actually be observed before treating it as
  confirmed. (`codexAppServerClient.ts:2569-2634`)
- **Unexpected exit** (the child dies without an explicit shutdown request)
  must be handled identically to a deliberate shutdown for the purpose of
  rejecting pending work and reporting session end — the only distinction
  carried forward is *why* it ended (deliberate shutdown vs. unexpected
  exit), for observability only; the exact vocabulary used to represent that
  distinction is an implementation choice, not a requirement (see §9).
  (`codexAppServerClient.ts:2975-3001`)
- **Restart/recovery**: a failed or terminated run does not auto-restart. A
  fresh run is a new, independent lifecycle with no carried-over
  authorization, no carried-over owned sessions, and no carried-over resource
  counters. (`codexAppServerClient.ts:1841-1852`, test "generation B starts with NO probe verdict of its own", `codexAppServerClient.test.ts:9157`)
- **Cancellation**: the protocol distinguishes ending a whole run (shutdown)
  from interrupting a single in-progress unit of work scoped to one session
  (a turn-interrupt style request). Both must be rejectable/no-ops safely when
  the target no longer exists or is not owned by the current run.
  (`codexAppServerClient.ts:2454-2460`)
- If a graceful shutdown's forced-termination step cannot get a confirmed exit
  within its bounded window, cleanup of that run's isolated state must **not**
  proceed — the state is left in place as a safe residual, retried only if a
  later, real exit is eventually observed. (`codexAppServerClient.ts:2984-3000`)

## 4. Transport / Message Requirements

- **Request/response correlation**: every outbound request must carry a
  unique id; a response must resolve or reject exactly the pending call that
  id belongs to, exactly once, and clear its timeout. A response naming an
  unknown id is ignored. (`codexAppServerClient.ts:2484-2515`, `2648-2658`)
- **Notifications** (no id) are dispatched for handling without any reply
  expected. This does not require that every notification first pass through
  normalized-activity-event handling: a security/control-plane notification
  (e.g. an unsolicited account-state-change signal, see §5, §7
  AC-CODEX-SEC-007) may be, and must be able to be, acted on directly and
  independently of — and no later than — whatever activity-normalization
  path ordinary lifecycle notifications go through, since its safety
  properties (immediate verdict invalidation) cannot wait on, or be
  contingent upon, the activity-normalization pipeline. (`codexAppServerProtocol.ts:37-41`)
- **Malformed messages** (invalid JSON, wrong `jsonrpc` version, missing
  required fields, an oversized id) must be rejected/ignored without ever
  throwing out of the parser and without ever being partially trusted.
  (`codexAppServerProtocol.ts:28-52`; test group "rejects malformed and non-JSON frames", `codexAppServerNormalizer.test.ts:7`)
- **Unknown messages** (unrecognized method name) must be ignored, never
  guessed at or partially normalized. (`codexAppServerNormalizer.ts:98-101`; test "rejects unknown JSON-RPC methods", `codexAppServerNormalizer.test.ts:18`)
- **Message ordering**: two logically related signals arriving in the same
  physical chunk of bytes (e.g. an authorization-invalidating signal and a
  would-be-authorizing response, in either order) must resolve to the same
  fail-closed outcome regardless of which one is textually first. (`codexAppServerClient.test.ts:9821` "delivered in the SAME stdout chunk... regardless of line order")
- **Bounded resource usage** — all of the following must be enforced with
  fixed, configurable-but-capped limits, independent of each other:
  - size of one inbound frame (line),
  - size of the pending not-yet-delimited inbound buffer,
  - size of one outbound frame,
  - number of outstanding (unanswered) outbound requests,
  - number of outstanding (unanswered) inbound approval requests,
  - number of concurrently tracked owned sessions.
  Every limit must be validated at configuration time (positive, finite,
  integral, at or below a fixed hard ceiling) and every size check must
  count true bytes (not string/UTF-16 length), so multi-byte content cannot
  evade a limit that is actually exceeded, nor be wrongly rejected when it
  isn't. (`codexAppServerClient.ts:80-196`, `638-655`, `2765-2826`)

  These caps fall into two distinct failure categories with different
  consequences, and the two must not be conflated:
  - **Untrustworthy transport** — an inbound frame or the pending
    undelimited buffer exceeding its byte-size bound means byte boundaries in
    the stream can no longer be trusted to resynchronize. This poisons the
    whole connection for the rest of that run (see the framing-untrustworthy
    bullet below) and rejects every request/approval currently pending for
    that run.
  - **Capacity reached** — the outbound-frame size cap, the outstanding-
    request/approval counts, and the tracked-session count are *counting*
    limits on new operations, not signals that the byte stream is
    unparseable. Reaching one of these means the specific new
    request/approval/session-start that would exceed it is refused
    immediately and is never written to the wire (or never accepted); it does
    **not** invalidate the transport, does not affect any request or
    approval already accepted before the cap was reached, and does not by
    itself trigger shutdown.

  **Approval capacity — one contract, stated precisely**: "every inbound
  approval request" (§4's bounded-resource list, above, and the
  bounded-waiting bullet below) means every approval request that has been
  *accepted and registered* as pending — the cap bounds how many such
  registered, pending approvals may exist at once, not an unlimited intake
  buffer ahead of registration.
  - (A) Only an approval request that has actually been accepted/registered
    as pending acquires approval authority and becomes subject to the normal
    bounded-waiting / authoritative-response lifecycle described below and
    in AC-CODEX-SEC-025.
  - (B) If approval capacity is already full when a new approval request
    arrives, that request must **not** be registered as pending and must
    **not** acquire any authority — it can never later be answered as if it
    had been accepted.
  - (C) For that specific over-capacity request: if an independently
    verified (per §2) protocol mechanism safely supports sending a fixed,
    non-authoritative capacity-decline response, that fixed decline *may* be
    sent; otherwise the request must fail closed **without sending any
    approval response at all**. Neither option may be read as requiring a
    reply mechanism the protocol has not been verified to actually provide
    (see §2, §7 outbound-capability-boundary invariant) — the decline path is
    conditional on independent protocol verification, never assumed to
    exist.
  - (D) A fixed capacity-decline sent under (C) is **not** an
    authorization-bearing "approval response" for the purposes of
    AC-CODEX-SEC-025, and sending it must **not** create or register a
    pending approval id of any kind. It is a bounded, non-authoritative
    rejection of an unregistered request, not an answer to a tracked one.
- **Bounded waiting**: every outbound request, and every inbound approval
  request that has actually been accepted/registered as pending (per (A)
  above — this does not extend to a raw inbound approval message that was
  refused at the capacity gate before registration), must be subject to its
  own independently enforced deadline, so that an unanswered *registered*
  approval resolves to a safe default decision (decline) rather than hanging
  forever, and an unanswered request is eventually rejected rather than left
  pending indefinitely. The specific mechanism (a per-request timer, a
  scheduled sweep, or any equivalent) is an implementation choice — what is
  required is that the *waiting* is bounded and independent per
  request/registered-approval, not that a timer object exists. (`codexAppServerClient.ts:2484-2498`, `2695-2699`)
- **Pending requests during failure**: if the transport or process fails
  (process error, framing violation, confirmed exit) every currently
  outstanding request and approval belonging to that run must be resolved
  (rejected, or auto-declined) immediately rather than left to time out
  individually, and this must never touch a *different* run's pending work.
  (`codexAppServerClient.ts:2848-2880`, `2968-2981`)
- Once a connection's framing has been judged untrustworthy (an oversized
  frame or buffer), further bytes for that same run must be silently
  discarded rather than re-parsed, and the connection must proceed to a
  controlled shutdown — resynchronizing is not attempted. (`codexAppServerClient.ts:2796-2860`)
- Outbound writes that would exceed the resource cap must fail immediately
  and visibly (clearing any reserved state right away), which must be kept
  distinct from ordinary backpressure (data accepted by the OS but not yet
  flushed) — the latter is not a failure and must not be treated as one.
  (`codexAppServerClient.ts:2494-2514`, `2882-2918`)
- **Outbound capability boundary**: outbound requests are limited to a small,
  explicitly enumerated set of approved methods. The integration must not
  rely on, enable, or invoke any optional/experimental protocol capability
  that has not been independently verified (per §2) to exist and to be
  required by the product — whatever declaration/negotiation mechanism the
  actual protocol provides for this (a handshake field, an implicit absence
  of use, or something else) is an implementation detail, not a requirement.
  The current prototype's technique — an `initialize` request explicitly
  declaring `experimentalApi: false` — is evidence that this outcome is
  achievable, not a mandate that a future implementation's protocol expose,
  or that the integration use, that same field (see §7, invariant on the
  outbound RPC capability boundary). (`codexAppServerClient.ts:32-51`, `2194-2201`)

## 5. Account/Session Compliance Requirements

This section is the most safety-critical part of the prototype. It separates
two things that the prior version of this document conflated: (a) the
**mechanism** by which an account/session compliance check is enforced —
which is a genuine, permanent requirement regardless of what the check
actually verifies — and (b) the **concrete rule** the current prototype
enforces, which is a provisional pilot-stage security posture, not an
approved product decision.

### 5.1 Policy-independent enforcement requirements (permanent)

Regardless of what concrete account/session policy the product ultimately
adopts, the following must hold:

- **A compliance check must exist and be positively evaluated** before the
  integration does anything with a live Codex session. Its result must derive
  from Codex-reported information — the App Server's account/session
  self-report is the only such evidence this review found, but a future
  independently-verified protocol capability may supplement or replace it —
  and must never be treated as an OS-level guarantee or independent proof of
  credential origin, regardless of which Codex-reported evidence a future
  policy consults. (`codexAccountConfinement.ts:1-53`)
- **An unclassifiable result is never treated as compliant.** If the check
  cannot confidently classify the account/session state (malformed response,
  RPC error, timeout, or any input the check cannot confidently interpret),
  that must be treated exactly like a non-compliant result — "unknown" is
  never "probably fine." (`codexAccountConfinement.ts:129-154`, `codexAppServerClient.ts:2225-2251`)
- **Verification failure surfaces as one fixed, non-sensitive message** —
  never the raw RPC error, response content, or account details. (`codexAppServerClient.ts:60-64`, `2225-2251`)
- **No task-capable operation before verification completes.** The only
  outbound calls allowed before a compliant verdict are the protocol
  handshake itself and the compliance check call. Every other operation that
  would create or advance real work must be refused with a fixed gate error
  until a compliant verdict is recorded, and that gate must be re-checked
  after *every* asynchronous suspension point inside such an operation, not
  only at entry — a call authorized under an old/terminated run must never
  resume and act once a different run becomes current. (`codexAppServerClient.ts:1690-1826`, `2312-2481`)
- **A verdict is scoped to exactly one run** and is never inherited by a new
  run; it is invalidated the instant that run begins shutting down or is
  confirmed to have exited, and it is invalidated proactively by (a) this
  integration itself initiating an action that can change the authenticated
  account (such as a login flow) and (b) an unsolicited account-change signal
  from the child. In every case the previous verdict must never be
  resurrected — a fresh run must earn its own verdict from a fresh check.
  (`codexAppServerClient.ts:1711-1826`, `2461-2481`, `2704-2762`)
- **Verification attempts are bounded, never silently retried without
  limit** — the current prototype performs exactly one check per run with no
  automatic retry; whatever bound the future implementation chooses, it must
  be an explicit, deliberate decision, not an accident of retry logic.
  (`codexAppServerClient.ts:2222-2224`, in-file comment "no retry in the MVP")
- **Data minimization bounds output, not the policy's legitimate input.** No
  account-identifying field (email, plan type, credential-adjacent fields, or
  any other Codex-reported value) may ever be inspected beyond what the
  *currently adopted policy's own compliance rule* actually requires to
  evaluate compliance — this does not permanently cap inspection at mere
  shape/presence validation, since a future approved policy may legitimately
  need to examine a field's actual value (e.g. a specific plan type) to reach
  its verdict. Whatever is inspected must never be returned, logged, or
  persisted in any form, in a public-facing surface or otherwise — only the
  check's own fixed verdict may ever leave it. (`codexAccountConfinement.ts:50-53`, `140-154`)

### 5.2 PROVISIONAL PILOT POLICY — current concrete rule (not a permanent requirement)

> **PROVISIONAL PILOT POLICY.** The current prototype's concrete
> account-eligibility rule is: the App Server reporting **no authenticated
> account at all** (`account: null`) is the only state classified compliant;
> the presence of *any* recognized, non-null account (of any known type or
> plan) is classified non-compliant, regardless of whether that specific
> account would actually be problematic for a given task. (`codexAccountConfinement.ts:129-138`, `codexAppServerClient.ts:2248-2251`)

This rule:

- describes the **current prototype/pilot security posture** — a
  conservative, bring-up-stage restriction, not a considered product policy;
- is **not yet an approved permanent product policy**; nothing in the
  inspected code states it is intended to ship as the final behavior beyond
  the word "pilot"/"MVP" in its own comments;
- **must not be carried forward by default.** The future independent
  implementation must derive its actual account/session policy from
  `PRODUCT_REQUIREMENTS.md` together with independently verified Codex
  capabilities (what account states Codex actually reports, what isolation
  Codex itself provides, and what the product actually needs to allow) —
  not from this prototype's placeholder rule;
- does not change §5.1: **whatever concrete policy is ultimately adopted,
  fail-closed behavior remains a requirement** whenever that policy's
  compliance state cannot be established or enforced (an unclassifiable
  result, a verification failure, or a state the policy does not explicitly
  permit must all be treated as non-compliant).

## 6. Workspace / Session Isolation Requirements

**What AgentCubicles must enforce (its own responsibility, not Codex's):**

- A caller-supplied working directory must be proven, via canonical
  filesystem path resolution (not lexical string comparison), to be the
  configured workspace root or a descendant of it before it is used for any
  session; a symlink/junction that would resolve outside the allowlisted root
  must be rejected after resolution, not merely checked against its
  unresolved spelling. (`codexAppServerClient.ts:667-690`; test group "H1 fail-first — child environment isolation contract", `codexAppServerIsolation.test.ts:104-120`)
- A session/thread created under one run must never be usable, or reported as
  owned, by a different run — ownership is scoped strictly to the run that
  created it. (`codexAppServerClient.ts:2444`, test "a session owned by generation A cannot be used by generation B", `codexAppServerClient.test.ts:9192`)
- The set of environment variables handed to the child process must be an
  explicit, minimal allowlist assembled by AgentCubicles — never "everything
  the host process has, minus a denylist." Everything identity-, credential-,
  proxy-, or toolchain-hijack-relevant (see §7) must be excluded regardless of
  the allowlist, matched case-insensitively so a differently-cased variant
  cannot slip through. (`codexAppServerClient.ts:197-310`, `833-865`)
- The child's notion of its own home directory, temp directories, and (where
  applicable) any tool-specific identity/config resolution must always be
  values AgentCubicles itself generates for the isolated run, never values
  copied from the host process — even variables that would otherwise be
  allowlisted. (`codexAppServerClient.ts:236-310`, `865-958`)
- Activity signals for a session/thread id that is not currently recognized
  as belonging to the active run must be discarded and must never be
  attributed to, or update the state of, a different session or a different
  integration/consumer — cross-integration or cross-session leakage must be
  structurally impossible, not merely discouraged. (`codexAppServerClient.ts:2674-2675`; test "fails closed for unknown sessions and never crosses provider boundaries", `codexAppServerIsolation.test.ts:54`)
- Isolated per-run state that AgentCubicles creates on disk for this
  integration must remain under an AgentCubicles-owned base location, with
  each run's own generated files confined beneath one instance-specific
  location under that base, and kept separated from the user's real Codex
  home/state where the product claims that separation (see §7, State Root
  Containment). (`codexAppServerClient.ts:350-365`, `3040-3063`)

**What remains Codex-dependent or unknown (Codex's own guarantees, not
independently established by this audit):**

- Whether the Codex App Server itself enforces any workspace/account
  isolation independently of what AgentCubicles imposes around it is not
  established from source inspection — the prototype treats AgentCubicles's
  own containment as the *only* trusted boundary and never assumes anything
  from Codex's side. This stance should be kept: **never turn "we did not
  observe Codex breaking isolation" into "Codex guarantees isolation."**
- Whether Codex's own account/session model has any server-side (not
  locally-observable) isolation guarantees is unknown and out of scope for
  what this audit can confirm.

## 7. Security Invariants

Each invariant states what must remain true; none prescribe *how* the old
code achieved it. The concrete account-eligibility rule from §5.2 is
deliberately **not** encoded here as an invariant — it is a provisional
policy choice, not a permanent property. The invariants below hold regardless
of which concrete account policy the product ultimately adopts.

- **AC-CODEX-SEC-001**: An account/session compliance check that cannot
  confidently classify its input must be treated as non-compliant, never as
  compliant. (`codexAccountConfinement.ts:129-138`)
- **AC-CODEX-SEC-002**: The compliance check itself must never throw; any
  internal fault (malformed input, a hostile object, a throwing accessor)
  must resolve to the non-compliant/unclassifiable outcome. (`codexAccountConfinement.ts:140-154`)
- **AC-CODEX-SEC-003**: The compliance check must never reflect raw input
  content (field values, error messages) into logs, thrown errors, or its
  own return value. (`codexAccountConfinement.ts:50-53`)
- **AC-CODEX-SEC-004**: Validation of an untrusted structured result must
  depend only on properties genuinely present on the object, never on
  properties satisfiable purely through inheritance/prototype-based access.
  (`codexAccountConfinement.ts:85-93`)
- **AC-CODEX-SEC-005**: Verification of account/session compliance must not
  be silently retried without bound; the number of attempts per run is an
  explicit, deliberate, policy-owned decision. (`codexAppServerClient.ts:2222-2224`)
- **AC-CODEX-SEC-006**: A compliance verdict is valid only for the exact run
  that produced it and must never authorize a different run, including a
  later run that happens to reuse the same process object or session id.
  (`codexAppServerClient.ts:1703-1709`, `1739-1748`)
- **AC-CODEX-SEC-007**: Any signal that the authenticated account may have
  changed (an explicit action this integration itself initiates, or an
  unsolicited notification) must invalidate the current run's verdict,
  synchronously, before any further task-capable action is taken. (`codexAppServerClient.ts:2461-2481`, `2748-2762`)
- **AC-CODEX-SEC-008**: No task-capable operation may reach the child process
  before its run's compliance verdict is compliant, and this gate must be
  re-checked after every asynchronous suspension point inside such an
  operation, not only at entry. (`codexAppServerClient.ts:2312-2481`, `1750-1826`)
- **AC-CODEX-SEC-009**: A working directory must be proven, by canonical path
  resolution, to be the configured workspace root or a descendant of it;
  symlink/junction indirection must not defeat this proof. (`codexAppServerClient.ts:667-690`)
- **AC-CODEX-SEC-010**: A session/thread is usable only by the run that
  created it; cross-run reuse must be rejected. (`codexAppServerClient.ts:2444`)
- **AC-CODEX-SEC-011**: The child process receives an explicit allowlist of
  environment variables, matched case-insensitively; nothing else reaches it
  regardless of what the host process's environment contains. (`codexAppServerClient.ts:197-234`, `846-863`)
- **AC-CODEX-SEC-012**: Credential-, proxy-, and runtime-hijack-relevant
  environment variables are never inherited by the child, under any casing,
  even if a future allowlist change would otherwise include them. (`codexAppServerClient.ts:246-310`)
- **AC-CODEX-SEC-013**: The child's home/temp directories and any
  tool-specific identity/config-resolution variables are always freshly
  generated for the isolated run, never copied from the host process.
  (`codexAppServerClient.ts:865-958`)
- **AC-CODEX-SEC-014**: Where isolation is promised for a subordinate tool
  invoked inside the isolated environment (e.g. Git), that tool must not
  silently consult or be steered by host-level system/global configuration
  or identity, must fail closed (refuse to guess) rather than fabricate an
  identity on the run's behalf, and setup for it must not unexpectedly
  trigger an interactive prompt the run has no way to answer. (`codexAppServerClient.ts:890-938`) — the current prototype's exact mechanism for this is a Git-specific, MVP-stage set of environment variables; see §10's Git-isolation group and §12.
- **AC-CODEX-SEC-015**: If the chosen isolation approach depends on a
  specific capability of an external tool (e.g. a minimum version), that
  capability must be positively verified, and the run must fail closed
  before any real work starts, rather than assumed present. (`codexAppServerClient.ts:390-397`, `507-537`)
- **AC-CODEX-SEC-016**: Destructive cleanup of a run's isolated state is
  authorized in exactly two cases: (a) the run's process was **never
  successfully spawned** (a pre-spawn failure — there is nothing to
  terminate), or (b) the run's process **has been positively confirmed
  terminated** (an observed exit). It is never authorized merely because the
  run's process was spawned but never accepted any task/work — a spawned,
  live process that has not yet done real work is still active, may still be
  alive, and its isolated state must not be destructively cleaned up while
  that possibility holds; "no work accepted" is not a substitute for
  "confirmed terminated." It must never be authorized while a run is still
  starting or otherwise active, and a stale run's recorded ownership must
  never authorize a different run's cleanup. (`codexAppServerClient.ts:1383-1392`, `3016-3038`)
- **AC-CODEX-SEC-017**: Ownership of an isolated-state location must be
  established through a mechanism that can be verified against the actual
  filesystem object at use time, not by pathname alone. (`codexAppServerClient.ts:1216-1266`)
- **AC-CODEX-SEC-018**: Before a destructive delete is authorized, the target
  must be checked against the identity/ownership established at claim time;
  a detected mismatch, an unreliable identity, or recognized symlink/junction
  indirection must abort the delete. **This is a detection requirement, not a
  proof of eliminating every race — see the residual limitation below.** (`codexAppServerClient.ts:1099-1151`)
- **AC-CODEX-SEC-019**: Cleanup must never (a) authorize deletion of an
  object it cannot verify as the one it owns, (b) report an ambiguous or
  partially-completed cleanup as successful, or (c) silently transfer
  ownership/authorization to an object that was not the one originally
  claimed. (`codexAppServerClient.ts:3056-3074`)
- **AC-CODEX-SEC-020**: All resource-bearing transport state (outstanding
  requests/approvals, tracked sessions, buffered/frame byte sizes) must be
  held under fixed, validated, positive caps; exceeding a cap must fail
  closed, never grow unbounded. (`codexAppServerClient.ts:80-137`, `638-655`)
- **AC-CODEX-SEC-021**: Once a transport's framing is judged untrustworthy,
  no further bytes on that connection may be parsed or trusted again for the
  life of that run. (`codexAppServerClient.ts:2796-2826`, `2848-2860`)
- **AC-CODEX-SEC-022**: Diagnostic output (logs, thrown messages) must never
  include raw process, account, or protocol payload content — only fixed,
  pre-approved text or an exact-match allowlisted low-level error code.
  (`codexAppServerClient.ts:983-1035`, `2921-2938`)
- **AC-CODEX-SEC-023**: An error or unexpected event on the child process or
  its stdio must never crash the host process, and must never, by itself,
  authorize state cleanup — only a positively observed process exit may do
  that. (`codexAppServerClient.ts:2116-2166`, `2953-2955`)
- **AC-CODEX-SEC-024 (Outbound RPC capability boundary)**: Outbound requests
  to the Codex process must be restricted to an explicit, approved set of
  methods appropriate to the product's own policy; unsupported or dangerous
  surfaces (e.g. arbitrary process-control methods, or optional/experimental
  protocol capabilities the prototype demonstrably excluded) must not become
  implicitly available just because the underlying protocol might expose
  them. (`codexAppServerClient.ts:32-51`, `2194-2201` — initialize declares `experimentalApi: false`)
- **AC-CODEX-SEC-025 (Approval authority)**: Only a currently registered,
  pending approval request may receive an *authoritative* approval response,
  and only according to the permitted response semantics; a stale, duplicate,
  or unknown approval id must not gain any authority — it must not be acted
  on, and must not be re-registered if it arrives again while already
  tracked. This includes the over-capacity case in §4: an approval request
  refused at the capacity gate before it was ever registered is not a
  "currently registered, pending approval," so any fixed, non-authoritative
  capacity-decline sent for it (§4) is not an approval response under this
  invariant and must not itself create or register a pending approval id.
  (`codexAppServerClient.ts:2517-2527`, `2682-2694`)
- **AC-CODEX-SEC-026 (Public event data minimization)**: Whatever
  normalized/publicly exposed activity representation AgentCubicles produces
  from Codex signals, it must not unintentionally expose raw sensitive
  payload content the prototype's own evidence demonstrates as
  sensitive — including raw prompts, commands, patches/diffs, tokens, or
  authentication URLs. This is a **product-facing data-minimization
  boundary**, not a requirement about any particular event schema. (`codexAppServerNormalizer.ts:41-45`, `codexAppServerClient.ts:2185-2187`)
- **AC-CODEX-SEC-027 (State root containment)**: Any provider isolation/state
  AgentCubicles creates for a Codex run must remain under an
  AgentCubicles-owned state base, with each run's generated files confined
  beneath one instance-specific location under that base, and — where the
  product claims this separation — proven not to overlap the user's real
  Codex home/state. This does not prescribe the base location's name or
  directory layout. (`codexAppServerClient.ts:350-365`, `3040-3063`)
- **AC-CODEX-SEC-028 (Private transport)**: Where the supported Codex
  integration mode is a local App Server process communicating only over
  stdio, the integration must not silently expose a network listener for
  that communication. (The exact current invocation shape, including whether
  it truly never opens a network port, remains an UNVERIFIED EXTERNAL
  ASSUMPTION per §2 — this invariant states the required property
  independent of confirming the current invocation.) (`codexAppServerClient.ts:2107-2115`)

### Residual Limitation (explicitly NOT guaranteed)

The current prototype's destructive-cleanup verification (SEC-016–019) is
implemented entirely in pure Node.js against filesystem pathnames and
metadata — there is no OS-level sandboxing, access-control boundary, or
transactional filesystem primitive underneath it. Because verification and
the actual destructive filesystem operation are necessarily two separate
steps, **this approach cannot prove elimination of every possible
time-of-check-to-time-of-use (TOCTOU) race between them.** It can only
guarantee that a *detected* discrepancy (an identity mismatch, an unreliable
identity, a recognized symlink/junction) aborts the action, and that
ambiguity is never reported as success. Whether a sufficiently fast or
sufficiently privileged co-resident actor could win a race inside the narrow
window between one of these checks and the operation it guards is **not
established as excluded** by the inspected evidence.

This corresponds to an implicit **local/single-user/private-machine threat
model**: the isolation is designed against accidental cross-run interference
and against the concrete race shapes its own extensive test suite exercises,
not against a co-resident, actively adversarial, equally-or-more-privileged
local process specifically racing the exact cleanup window.

**The future architecture must do one of:**
- explicitly accept and document this residual under a stated threat model
  (e.g., "this integration assumes a trusted local machine; no other actor
  has write access to the isolation base directory"), or
- introduce a stronger, independently designed mechanism (e.g., OS-level
  sandboxing, a dedicated managed volume/container boundary, or a
  transactional filesystem primitive) if the product's actual threat model
  requires closing this gap.

### Mapping note (SEC invariants)

Old `AC-CODEX-SEC-002` (the concrete "any non-null account is unsafe" rule)
is **removed** from this list — it is now documented only as the PROVISIONAL
PILOT POLICY in §5.2, not as a permanent invariant. Old `SEC-003`–`SEC-024`
shift down by one to become new `SEC-002`–`SEC-023` (wording generalized
where noted inline; no substantive property was dropped). Old `SEC-017`'s
containment clause was split out into new `SEC-027`. New `SEC-024`–`SEC-028`
are additions with no prior numbering. Total: 24 (old, including the removed
one) → **28** (new).

## 8. Failure Model

| Failure case | Required safe outcome |
| --- | --- |
| Process fails to spawn (missing/invalid executable) | Startup fails; any in-flight startup request is rejected promptly rather than waiting out its own timeout. Because it is positively established here that no process was ever successfully spawned, this is exactly the pre-spawn case AC-CODEX-SEC-016(a) authorizes: safe cleanup of that run's (never-used) isolated state is permitted and may proceed immediately, provided the cleanup's ownership/authorization is itself valid for this run — never because "no real work began," which is not by itself a cleanup-authorization condition (see AC-CODEX-SEC-016). (`codexAppServerClient.ts:2137-2166`, `2280-2298`) |
| Startup protocol handshake fails or times out | Startup fails closed with a fixed message. The applicable cleanup rule is exactly AC-CODEX-SEC-016's two cases, nothing else: if no child was ever spawned, safe pre-spawn cleanup runs immediately. If a child *was* spawned but the handshake (or the compliance check that follows it) never succeeded, destructive cleanup of that run's isolated state is **not** performed on that basis — the process may still be alive, so cleanup still waits for its termination to be positively confirmed. Whether the run ever reached a task-capable state is not itself a cleanup-authorization condition; only "never spawned" or "confirmed terminated" authorize destructive cleanup. (`codexAppServerClient.ts:2194-2277`) |
| Malformed / non-JSON protocol input | The offending line is dropped; parsing never throws; the connection is not torn down for a single bad line unless it also violates a size bound. (`codexAppServerProtocol.ts:28-52`) |
| A single frame or the undelimited buffer exceeds its byte-size bound | Transport is permanently poisoned for that run; further bytes discarded; pending requests for that run are rejected; a controlled shutdown begins. (`codexAppServerClient.ts:2796-2860`) |
| Outbound request/approval times out | The specific pending call is rejected/auto-declined; no other pending call for the run is affected. (`codexAppServerClient.ts:2494-2498`, `2695-2699`) |
| The account/session compliance check classifies the account as non-compliant with the currently adopted policy, or cannot classify it at all | This is the **startup verification failure** case: the verdict is decided before any task-capable operation is ever allowed to start, so no task-capable operation is ever allowed to have run at all — there is no already-started work to reason about here (contrast with the post-start invalidation case below, where work may already have been transmitted). Startup fails closed with one fixed message; cleanup deferred until exit is confirmed. Which concrete states are "non-compliant" is a policy decision (§5.2) — this row is about the enforcement outcome, not the current pilot rule specifically. (`codexAppServerClient.ts:2244-2277`) |
| The compliance check's own round trip errors or times out | Treated identically to a non-compliant verdict — one fixed message, no raw RPC detail surfaced. (`codexAppServerClient.ts:2225-2232`) |
| Unexpected process termination (no shutdown requested) | Same pending-work rejection and session-end reporting as a deliberate shutdown; cleanup is authorized because exit was actually observed. (`codexAppServerClient.ts:2968-3000`) |
| Outstanding request/approval at the moment of termination | Rejected/declined immediately as part of termination handling, never left to its own individual timeout. (`codexAppServerClient.ts:2968-2974`) |
| Cancellation of an in-progress unit of work | Rejected/no-op safely if the target session isn't owned by the current run; otherwise forwarded as its own request, independent of full-run shutdown. (`codexAppServerClient.ts:2454-2460`) |
| A *counting* capacity cap is reached (outstanding requests, tracked sessions, or a single outbound frame's size cap) | This is distinct from an inbound frame/buffer byte-size **overflow** (the row above, which poisons the transport). Reaching a counting cap refuses the specific new request/session-start/outbound write that would exceed it — it is never queued unboundedly and never written to the wire; it does not invalidate the connection, and communication already accepted before the cap was reached is unaffected. (`codexAppServerClient.ts:2490-2514`, `2691-2694`) |
| The outstanding-approvals capacity cap is reached | Handled by the approval-specific contract in §4/AC-CODEX-SEC-025, not the blanket "never written to the wire" rule above: the new, over-capacity approval request is never registered as pending and never acquires authority, but — unlike the other counting caps — a fixed, non-authoritative capacity-decline response *may* be written to the wire for it if an independently verified (§2) protocol mechanism safely supports doing so; otherwise it fails closed with no approval response sent. Either way, no pending approval id is created or registered for it, and communication already accepted before the cap was reached is unaffected. (`codexAppServerClient.ts:2490-2514`, `2691-2694`) |
| Authorization is revoked (§5) after a task-capable RPC has already been transmitted, or while its response is still pending | The already-sent request cannot be un-sent and any response already in flight cannot be un-received; the requirement is forward-only: no *further* task-capable action is taken from the moment of revocation, the eventual response (if any) is not treated as authorizing anything beyond itself, and the run proceeds toward safe shutdown/containment (§3, §9). This does not claim the already-transmitted request or any work it already caused is retroactively prevented or undone — only that no new authority or work follows it. (`codexAppServerClient.ts:2461-2481`, `2748-2762`) |
| Forced termination does not yield a confirmed exit within its bounded window | Isolated state is left as a safe residual; cleanup deferred until a real exit is eventually observed for that exact run. (`codexAppServerClient.ts:2984-3000`) |
| A destructive-cleanup target is found replaced, symlinked, or already gone at one of the implementation's verification points | Whatever check catches the discrepancy aborts that specific delete step. The guarantee is: an unverified replacement/victim is **not** intentionally authorized for deletion; an ambiguous or partial cleanup is **not** reported as successful; ownership/authorization is **not** silently transferred to the replacement. **This does not claim protection at every conceivable point in time between verification and deletion — see the Residual Limitation in §7.** (`codexAppServerClient.ts:1099-1151`, `3040-3074`) |
| An action that could change the authenticated account is in progress when that same action changes it | This is a **post-start invalidation** case (see the general revocation-timing row above): only that one already-in-flight action finishes under the verdict it started with — that action's own already-transmitted request is not, and cannot be, retroactively undone — and every other operation, including any new authority, continuation, task-capable action, or session authorization, is blocked immediately from the invalidation point onward, except whatever is required for safe termination/containment. (`codexAppServerClient.ts:2461-2481`) |
| Unsolicited account-state-change notification | Also a **post-start invalidation** case: the current run's verdict is invalidated immediately and a safe shutdown of that run begins; from that point on, no new authority, continuation, task-capable action, or session authorization may proceed for this run except what safe termination/containment itself requires. If this races an in-flight verification round trip, the invalidation wins regardless of arrival order. This does not, and cannot, undo any request already transmitted or any work already executed before the invalidation point — only the general revocation-timing row above states what happens to that already-in-flight work. (`codexAppServerClient.ts:2748-2762`) |

## 9. Observability Requirements

- **Directly observed** (locally, by AgentCubicles's own process management —
  independent of anything Codex reports): process spawn/exit and exit
  code/signal, raw byte volume and framing health of stdout, whether a
  request/approval is still pending or has been resolved, and wall-clock
  timeouts firing. (`codexAppServerClient.ts:2190-2192`, `2637-2646`)
- **Codex-reported** (only as good as §2's unverified protocol facts):
  session/thread start and close, turn start/completion (including whether a
  turn was interrupted), individual tool-like "item" start/completion, an
  approval request and its resolution, and an account-state-change signal.
  (`codexAppServerNormalizer.ts:53-112`)
- **AgentCubicles-derived** (computed by this integration, not stated by
  Codex): some coarse indication of whether a session is actively working or
  idle, derived from turn boundaries; human-readable status text for a given
  tool/item type; and a distinguishable indication of *why* a session ended
  (an explicit end signal from Codex itself, versus AgentCubicles inferring
  termination from process exit, further distinguishing a deliberate
  shutdown from an unexpected exit where that distinction matters for safe
  recovery or diagnostics). The exact status/reason vocabulary is an
  implementation choice, not a requirement. (`codexAppServerNormalizer.ts:119-132`, `codexAppServerClient.ts:2975-2979`)
- The new AgentCubicles event/observability contract should be designed
  independently (see §12) — this section records *what facts must be
  derivable*, not the shape they must be delivered in.

## 10. Test Requirements

IDs are assigned independently of the existing test suite's own structure;
they group the individual cases found across the four inspected test files
into the behaviors they protect. No test implementation is reproduced — only
the scenario and the behavior it protects. Several groups have been
consolidated and generalized relative to the first version of this document;
see the mapping note at the end of this section.

**A. Account/session compliance check — provisional-policy coverage plus
policy-agnostic hardening** (`codexAccountConfinement.test.ts`):

- **AC-CODEX-TEST-001**: Under the current PROVISIONAL PILOT POLICY (§5.2),
  both shapes of a genuinely absent account (`account: null`, either boolean
  for the auth-required flag) classify compliant. This documents the current
  policy's own test coverage, not a mandate that this specific rule persist
  — whatever concrete policy the product ultimately adopts must have
  equivalent coverage for its own defined positive case(s).
- **AC-CODEX-TEST-002**: Under the current PROVISIONAL PILOT POLICY, every
  recognized non-null account variant (all known account types, and for the
  type with a plan field, all 17 known plan values) classifies non-compliant.
  Security relevance: demonstrates the fail-closed default was checked
  against the full known account surface, not one example — whatever policy
  replaces this one must be exercised with the same breadth against its own
  negative surface.
- **AC-CODEX-TEST-003**: Additive/unknown fields on an otherwise-valid
  compliant or non-compliant shape never change the verdict (forward
  compatibility without weakening the check).
- **AC-CODEX-TEST-004**: Structurally invalid top-level input (null, array,
  primitives, `undefined`, missing/mistyped required fields) always
  classifies unclassifiable, never compliant. Security relevance: an
  unrecognized/malformed response must never be treated as the compliant
  case by default.
- **AC-CODEX-TEST-005**: Malformed or unrecognized account sub-shapes
  (unknown/missing/mistyped discriminator, missing/mistyped variant fields)
  classify unclassifiable.
- **AC-CODEX-TEST-006**: A hypothetical future/unrecognized top-level schema
  shape fails closed to unclassifiable rather than being assumed compliant.
  Security relevance: protects against a future Codex protocol change being
  silently treated as compliant.
- **AC-CODEX-TEST-007**: Validating an untrusted structured result must not
  be satisfiable through a value that is not a genuine, directly-held
  property of the object being checked. *Runtime-specific example (current
  implementation, JavaScript/Node):* a required field satisfied only through
  the prototype chain (not as the object's own property) does not count as
  present. Security relevance: closes a spoofing vector for structural
  validation; whatever runtime the future implementation uses, it must have
  an equivalent case for that runtime's own analogous indirect-property-
  access mechanism, if one exists — this is not a JavaScript-only
  requirement.
- **AC-CODEX-TEST-008**: An input that fails or throws during ordinary
  property access anywhere while being validated (top level or nested) is
  caught and classified unclassifiable, never allowed to propagate as an
  uncaught exception out of the check. *Runtime-specific example (current
  implementation, JavaScript/Node):* a throwing property getter. Whatever
  runtime is used, an equivalent case for that runtime's own way of making
  structured-data access fail unexpectedly must be covered.
- **AC-CODEX-TEST-009**: A structured-data input that can misrepresent its
  own shape to a naive validator — reporting one shape to an enumeration
  check and a different one to a direct-access check, or throwing from a
  shape-introspection operation — never causes a throw and never produces an
  incorrect verdict. *Runtime-specific example (current implementation,
  JavaScript/Node):* adversarial `Proxy`-based inputs (a throwing `get` or
  `getOwnPropertyDescriptor` trap, a manipulated `ownKeys` trap, a `has`/`get`
  disagreement fabricating an apparent shape, or a revoked `Proxy` at any
  level). Security relevance: this is the check's core untrusted-input
  hardening; whatever runtime is used, an equivalent adversarial-shape case
  relevant to that runtime must be covered.
- **AC-CODEX-TEST-010**: The returned verdict is always exactly one of the
  fixed literal outcome values, never a value derived from the input.
- **AC-CODEX-TEST-011**: The compliance check exposes no logging or
  persistence side channel — it is provably a pure function.

**B. Protocol framing & normalization** (`codexAppServerProtocol.ts` /
`codexAppServerNormalizer.test.ts`):

- **AC-CODEX-TEST-012**: Malformed or non-JSON lines are dropped without
  throwing.
- **AC-CODEX-TEST-013**: Frames naming an unrecognized method are ignored
  rather than guessed at.
- **AC-CODEX-TEST-014**: Every recognized Codex lifecycle signal (session/
  thread start, turn start/completion, tool/item start/completion, approval
  request/resolution, session/thread end) is translated into whatever
  activity representation the integration exposes, without retaining raw
  prompt, command, patch/diff, token, or authentication-URL content in that
  representation.
- **AC-CODEX-TEST-015**: Approval requests, approval resolution, turn
  completion (including the interrupted case), and session end all translate
  correctly, including whatever identifying/method information is needed to
  answer an approval later.
- **AC-CODEX-TEST-016**: A malformed payload for an otherwise-recognized
  method fails closed (produces no activity signal) rather than normalizing
  partial or guessed data.

**C. Codex signal isolation & cross-run/session boundary**
(`codexAppServerIsolation.test.ts`):

- **AC-CODEX-TEST-017**: Activity signals for a session/thread id that is not
  currently recognized as belonging to the active run are discarded and must
  never be attributed to, or update the state of, a different session or a
  different integration/consumer. Security relevance: the core cross-tenant
  isolation guarantee.
- **AC-CODEX-TEST-018**: A representative full session lifecycle — from
  session start, through one or more units of work with intervening idle
  periods, to session end and cleanup — is tracked correctly end-to-end,
  without asserting any particular fixed status-label vocabulary.
- **AC-CODEX-TEST-019**: A workspace path outside the allowlisted canonical
  root is rejected.
- **AC-CODEX-TEST-020**: A symlink/junction that would resolve outside the
  allowlisted root is rejected after canonicalization, not merely by its
  unresolved spelling.
- **AC-CODEX-TEST-021**: Credential-adjacent values are confirmed scrubbed
  from the observed child environment.
- **AC-CODEX-TEST-022** *(conditional on architecture choice)*: **If** the
  new architecture chooses to expose a shared, integration-neutral
  representation for cross-integration activity (as the current prototype's
  `AgentEvent`/`StreamProvider` design does), then interpreting a
  Codex-sourced entry in that representation must not require
  Codex-specific knowledge, and no Codex-internal identifier or shape may
  leak through a field meant to be integration-neutral. This test does not
  apply, and no equivalent is mandated, if the product architecture instead
  exposes Codex activity through a Codex-specific, explicitly tagged payload
  shape — that remains a valid design if `PRODUCT_REQUIREMENTS.md` permits
  it. Nothing in this document requires a shared cross-integration
  representation to exist. *(Supersedes former AC-CODEX-TEST-022, which
  asserted keeping the upstream-derived `StreamProvider` contract
  "provider-agnostic" — that contract itself is rejected as architecture in
  §12; only the underlying neutrality property is retained here, and now
  only conditionally on that kind of representation being chosen at all.)*

**D. Process/transport resilience** (`codexAppServerClient.test.ts`,
child-process and bounded-transport groups):

- **AC-CODEX-TEST-023**: A spawn-time process error promptly rejects the
  in-flight startup request instead of waiting out its own timeout.
- **AC-CODEX-TEST-024**: Errors on the process object and on each stdio
  stream are caught, logged only as a bounded/allowlisted code, and never by
  themselves authorize termination.
- **AC-CODEX-TEST-025**: Error-code values that are control characters,
  secret-shaped, oversized, or merely regex-plausible are never logged
  verbatim; only an exact allowlisted code is ever logged as-is. Security
  relevance: prevents credential/secret leakage through error-code log lines.
- **AC-CODEX-TEST-026**: A misbehaving logger callback cannot crash the host
  process or block the handling logic that runs after it.
- **AC-CODEX-TEST-027**: A stale (superseded-run) process error cannot affect
  the current run's pending requests.
- **AC-CODEX-TEST-028**: Well-formed small frames are unaffected by the
  size-bounding logic.
- **AC-CODEX-TEST-029**: An oversized frame (with or without a terminating
  delimiter) fails the transport closed without logging raw content and
  rejects any pending request tied to it.
- **AC-CODEX-TEST-030**: Frame/buffer size limits are measured in true
  encoded (UTF-8) bytes, never in character count, in both directions. This
  is exercised with (a) a multi-byte-character input whose encoded byte
  length sits exactly at the configured boundary, which must be accepted,
  and (b) a multi-byte-character input whose encoded byte length exceeds the
  boundary even though it would appear to fit under a character/code-unit
  count, which must be rejected. *(Reworded to state the byte-oriented
  property directly and to require both a valid boundary case and an
  over-limit multi-byte case, without assuming any particular runtime's
  internal string representation — the prior wording's "code-unit count"
  language named a JavaScript/UTF-16-specific detail that is not itself a
  requirement.)*
- **AC-CODEX-TEST-031**: Fixed caps on outstanding requests and outstanding
  approvals are enforced independently. Once the outstanding-requests cap is
  reached, the specific *new* request that would exceed it is refused
  without ever being written to the wire. Once the outstanding-approvals cap
  is reached, the specific *new* approval request is never registered as
  pending and never acquires authority — but this is **not** an unconditional
  "nothing is ever written to the wire" rule for approvals: per the §4/
  AC-CODEX-SEC-025 approval-capacity contract, a fixed, non-authoritative
  capacity-decline response *may* be written to the wire for it if an
  independently verified (§2) protocol mechanism safely supports doing so,
  and must otherwise fail closed with no approval response sent — either way
  without registering a pending approval id. Both refusals are scoped to the
  over-cap new operation only — neither must be read as prohibiting protocol
  writes in general, and in particular neither may block communication
  needed for work already accepted before the cap was reached, nor does
  reaching a counting cap by itself invalidate the transport (that outcome is
  reserved for a framing/byte-size overflow — see AC-CODEX-SEC-020/021 and
  the §4/§8 distinction between capacity-reached and untrustworthy-transport
  failures).
- **AC-CODEX-TEST-032**: An oversized protocol id is rejected outright,
  never tracked.
- **AC-CODEX-TEST-033**: Data belonging to a stale (superseded) run cannot
  poison or otherwise affect the current run's transport state.
- **AC-CODEX-TEST-034**: Approval-response writes pause once real
  backpressure is detected and resume only once the stream actually drains
  for that same run.
- **AC-CODEX-TEST-035**: Every bounded numeric configuration option is
  validated at construction time — invalid values are rejected outright,
  and a value above the fixed hard ceiling is rejected even if otherwise
  well-formed; valid custom values are honored.
- **AC-CODEX-TEST-036**: The first transport framing failure stops consuming
  further input and begins a controlled shutdown without prematurely
  marking the run as terminated.
- **AC-CODEX-TEST-037**: Owned-session bookkeeping is pruned the moment the
  far end confirms a session closed; both new registration and new
  session-start are refused once the owned-session cap is reached.
- **AC-CODEX-TEST-038**: Buffer bounding is exact at its configured boundary
  (accept at the limit, reject one unit over) and enforced across multiple
  chunks, never allocating past the limit to find out.
- **AC-CODEX-TEST-039**: Concurrent attempts to claim a capacity-limited slot
  (owned session registration/start) can never oversubscribe the cap, and a
  reserved-but-failed attempt releases its slot for a later caller.
- **AC-CODEX-TEST-040**: Outbound frames are bounded the same exact way as
  inbound ones; a size-based refusal clears pending state immediately rather
  than waiting out the request's timeout, and a pathologically large payload
  is rejected without ever being fully serialized.

**E. Environment isolation** (`codexAppServerClient.test.ts`, "M1"–"M6.6"
groups):

- **AC-CODEX-TEST-041**: The isolated home directory never overlaps the real
  Codex home directory.
- **AC-CODEX-TEST-042**: The spawned child's environment contains exactly the
  allowlisted baseline plus the generated isolation variables — no known
  credential/proxy/runtime-hijack variable reaches it, matched
  case-insensitively.
- **AC-CODEX-TEST-043**: POSIX temp-directory resolution is isolated
  identically to the Windows-style temp variables.
- **AC-CODEX-TEST-044**: The isolated user-home directory is provisioned with
  a proven safety boundary against the real OS user home: it must not equal
  the real home, and must not contain, alias, or otherwise resolve into the
  real home in a way that would expose real user state to the isolated run.
  This is checked case-insensitively and includes a check that specifically
  catches a post-creation rebind (not just a pre-creation one), using
  canonical/symlink-aware path resolution throughout. *(Reworded from a
  symmetric "disjointness" requirement: the actual property is one-directional
  safety — the isolated home must never expose or reach into the real home —
  not that the two paths can never be nested at all. A product-owned isolated
  directory that happens to live beneath a broader real-user path, e.g. under
  the real home's own tree, is not automatically invalid provided it remains
  safely contained, separate, and does not itself expose real home content.)*
- **AC-CODEX-TEST-045**: No production-reachable configuration surface can
  override a security-relevant verification check (e.g., an
  isolation-boundary comparison, or a required-capability/version check). A
  test suite is not required to provide any bypass/override mechanism for
  such a check at all — a check with no override path of any kind trivially
  satisfies this. **If** a test-only bypass mechanism does exist for a given
  check, it must be demonstrably unreachable from any supported production
  configuration path (attempting to supply the override through a supported
  production path must be rejected or must not be possible). Security
  relevance: proves these boundaries can't be silently widened by ordinary
  application code, whether or not a test seam happens to exist. *(Merges
  former AC-CODEX-TEST-045 and AC-CODEX-TEST-052. Revised again to make the
  existence of a test-only override conditional — the requirement is that
  IF one exists it must be production-unreachable, not that one must exist
  at all.)*
- **AC-CODEX-TEST-046**: Windows-specific home/AppData path derivations are
  pure functions of the isolated path that exactly reconstruct it, and
  reject any path shape they cannot safely represent (relative, UNC,
  non-drive-rooted).
- **AC-CODEX-TEST-047**: Windows-only environment variables are never
  generated on non-Windows platforms, and non-Windows behavior is otherwise
  unaffected.
- **AC-CODEX-TEST-048**: A full end-to-end run (start through spawn) applies
  the derived environment correctly on a real host, exercised for both the
  Windows-specific and POSIX-specific code paths.
- **AC-CODEX-TEST-049** *(prototype evidence only — not a mandatory future
  test)*: The current prototype's own test suite confirms that, when its
  specific Git-isolation mechanism (a particular set of environment
  variables) is in effect, the host's system/global Git configuration does
  not apply inside the isolated execution, while an explicit repository-local
  Git identity for the task's own workspace continues to work normally. This
  is recorded as evidence that the outcome is achievable, not as a
  requirement that the future architecture reproduce this specific test. The
  permanent, outcome-oriented requirement lives at **AC-CODEX-SEC-014** and
  applies regardless of mechanism. *If* a future implementation independently
  chooses an environment-variable-based Git-isolation mechanism similar to
  this one, it should have an equivalent test for that outcome; if it chooses
  a different mechanism entirely (or Git isolation is out of scope for a
  given deployment), this specific test does not apply and no equivalent is
  mandated beyond satisfying SEC-014 by whatever means the chosen mechanism
  provides.
- **AC-CODEX-TEST-050** *(prototype evidence only — not a mandatory future
  test, paired with TEST-049)*: The current prototype's own test suite
  verifies its specific Git-isolation mechanism's effect against a real Git
  installation operating on a real temporary repository, not only against
  simulated behavior. This is recorded as evidence that verifying against a
  real external tool (rather than only a simulation) is achievable and was
  done, not as a requirement that the future architecture reproduce this
  exact test. The permanent, mechanism-agnostic requirement — that whatever
  approach is chosen for AC-CODEX-SEC-014 be verified against real behavior
  of the actual subordinate tool it isolates, not only simulated behavior —
  applies regardless of mechanism; if a future implementation chooses a
  different Git-isolation mechanism, or a different subordinate tool
  entirely, or no Git isolation at all, this specific test does not apply and
  no equivalent is mandated beyond satisfying that outcome by whatever means
  fit the chosen mechanism.
- **AC-CODEX-TEST-051**: If the isolation approach depends on a minimum
  capability of an external tool, the check for that capability fails closed
  (and the task is never launched) for every way the check itself can fail
  (the tool missing, its version output unreadable, or the reported version
  below the required minimum), and correctly accepts realistic real-world
  version-output formatting variations for a tool that does meet the
  requirement. *(Generalized from former AC-CODEX-TEST-051, which named a
  specific version number and parsing algorithm as requirements; those are
  prototype-specific — see §12.)*
- **AC-CODEX-TEST-052**: Environment-variable name patterns that could
  reintroduce a blocked credential/configuration-override vector (not just a
  single fixed literal name) are blocked at every matching form, not merely
  the specific ones the integration itself currently happens to use.
- **AC-CODEX-TEST-053**: Where a subordinate tool invoked inside the isolated
  execution could otherwise block waiting for interactive input it can never
  receive, the isolated environment is configured so it fails immediately
  instead of hanging, on every platform.
- **AC-CODEX-TEST-054**: One integrated scenario exercises every
  environment-isolation invariant together against a single, intentionally
  hostile, fully populated simulated host environment, and demonstrates that
  building the isolated child's environment does not unintentionally mutate
  whatever the integration treats as "the host environment," for any
  variable, under any casing. Tests must be able to prove this outcome — by
  whatever technique fits the runtime and test framework — without that
  technique itself being a requirement: nothing here mandates mutating and
  restoring the real process-wide environment (e.g. Node's `process.env`),
  nor checking the result from a separate, later test. Security relevance:
  this is the regression backstop for the whole environment-isolation
  surface acting together, not in isolation. *(Absorbs former
  AC-CODEX-TEST-056. Revised again to drop the requirement that the proof
  technique mutate-and-restore the real global process environment or be
  verified from a subsequent test — that was the current prototype's own
  test technique, not part of the required outcome.)*
- **AC-CODEX-TEST-055**: Real on-disk isolation paths are proven not to
  overlap real OS credential/config locations, without reading any file's
  contents.
- **AC-CODEX-TEST-056**: The requirement is not that the isolation layer may
  never write any configuration artifact of its own to disk — a
  product-owned, minimal isolation configuration file (e.g. a scoped Git
  config the isolation mechanism itself generates) is a legitimate technique.
  Negative-absence checks instead confirm that whatever the isolation layer
  generates on disk contains only product-generated, minimal content: it
  never copies or inherits forbidden host/user configuration, and never
  contains a fabricated, copied, or otherwise synthesized credential/identity
  value. *(Reworded from a blanket "never synthesizes ... an artifact" ban,
  which would have prohibited the isolation mechanism from writing its own
  minimal config at all; the actual security property is that generated
  artifacts stay product-owned and never smuggle in forbidden host state or
  credentials.)*

**F. Destructive-cleanup race protection (technique-neutral)**
(`codexAppServerClient.test.ts`, "M2 HIGH 1/2/3" groups):

- **AC-CODEX-TEST-057**: Establishing exclusive ownership of a new
  isolated-state location succeeds only when that location's identity/
  ownership can be consistently and reliably confirmed across every check
  the claim process performs; any detected inconsistency — including an
  unreliable/non-distinguishing identity, or a genuine filesystem race
  between checks, verified against a real filesystem where practical —
  aborts the claim before any destructive action and without granting
  ownership. *(Merges former AC-CODEX-TEST-059 and AC-CODEX-TEST-061.)*
- **AC-CODEX-TEST-058**: A failed or aborted ownership claim never
  destructively removes anything beyond an empty, freshly-created,
  not-yet-claimed location; a location that already has unexpected content,
  or whose identity cannot be verified, is left untouched.
- **AC-CODEX-TEST-059**: Destructive cleanup of a run's isolated state is
  authorized in exactly two cases — (a) the run's process was **never
  successfully spawned**, or (b) the run's own execution has been positively
  confirmed to have ended — and in no other case. It is refused while the
  run is still active or starting, **including the case where the run's
  process was successfully spawned but failed readiness/handshake, or was
  spawned and never accepted any task/work**: neither "readiness/handshake
  failed" nor "no work was ever accepted" is, by itself, grounds to authorize
  destructive cleanup of a process that has not been confirmed terminated,
  since it may still be alive. A different (stale) run's recorded ownership
  can never authorize the current run's cleanup. *(Reworded to remove the
  standalone "never began real work at all" clause, which could be misread as
  authorizing cleanup of a still-live, merely idle process; case (a) is now
  stated as "never successfully spawned," using the exact same boundary as
  AC-CODEX-SEC-016 and §8, rather than the looser "never successfully
  started," which could be misread as covering a spawned-but-not-yet-ready
  process.)*
- **AC-CODEX-TEST-060**: A failure occurring before any ownership was ever
  established and **before the run's process was ever successfully spawned**
  leaves the integration retryable from a clean state via safe, non-
  destructive pre-spawn cleanup. A failure occurring after the process was
  successfully spawned — regardless of whether it ever went on to accept any
  task/work — is left in a well-defined terminal state, not silently reset:
  destructive cleanup of that run's state remains gated on its termination
  being positively confirmed (AC-CODEX-SEC-016), never authorized merely
  because it never accepted work. *(Reworded to replace "before any real
  work began" with "before the process was ever successfully spawned" as the
  dividing line, so a spawned-but-idle process is not treated as
  clean-state-retryable while it may still be alive.)*
- **AC-CODEX-TEST-061**: In the ordinary case, removing a run's isolated
  state deletes exactly the location that run owns and nothing else,
  verified against a real filesystem.
- **AC-CODEX-TEST-062**: If a test substitutes a different real object, or a
  symlink/junction, at the target location after ownership was established
  but before the implementation's own verification step(s) run, that
  replacement/link must not be authorized for deletion by those checks —
  reproducing a copyable value (such as ownership-marker content) on the
  replacement must not be sufficient to grant it deletion authority, since
  authority must derive from the object's own established, non-copyable
  identity. **This does not claim protection against a race occurring inside
  a window the implementation's checks cannot observe — see the Residual
  Limitation in §7.** *(Narrowed from former AC-CODEX-TEST-065, which
  claimed protection "at any point between verification and deletion.")*
- **AC-CODEX-TEST-063**: If the target location no longer exists at the
  moment deletion is attempted, nothing is deleted and the cleanup remains
  retryable rather than being treated as already successful.
- **AC-CODEX-TEST-064**: Tests can deterministically inject realistic
  failure conditions into the cleanup path (e.g., a filesystem operation
  failing partway through, or an object being replaced at a specific point)
  without weakening or bypassing the production verification/authorization
  logic itself; no test-only capability exists in the production build that
  could fabricate a filesystem result or substitute for a genuine filesystem
  check. *(Generalized from former AC-CODEX-TEST-067, which named the
  current "data-only fault plan" API specifically — that API is one
  implementation's technique, not a requirement; see §12.)*
- **AC-CODEX-TEST-065**: Any name obtained while enumerating the contents of
  a location being destructively removed is validated as a safe, contained,
  in-place entry before it is used to build a further path; names that would
  escape the location being cleaned up are rejected.
- **AC-CODEX-TEST-066**: If removal fails at its final step, this is not
  reported as a successful cleanup, and ownership/authorization state
  remains intact so a later attempt can retry safely.
- **AC-CODEX-TEST-067**: A later cleanup attempt may perform a recursive/
  destructive removal of a location left behind by an earlier failed attempt
  only if ownership/authorization over that exact location is freshly and
  reliably re-established at the time of the later attempt — never assumed
  from the earlier attempt's now-stale result. Requiring the residual to be
  empty before allowing a (non-recursive) removal is one way the current
  prototype achieves reliable re-establishment when it has no stronger
  identity proof available; a future implementation may instead use a
  stronger object/handle/transaction identity mechanism (e.g. a held file
  handle, a transactional filesystem operation, or an OS-level ownership
  primitive) to re-establish authorization for a full recursive removal,
  provided that mechanism is at least as reliable. In every case: any
  rebinding, content change, or ownership/staleness ambiguity discovered
  during that fresh re-establishment blocks the removal. *(Revised to drop
  "the residual must be empty" as a permanent precondition — that was one
  mitigation tied to a specific, weaker identity mechanism, not a
  requirement of the underlying property.)*
- **AC-CODEX-TEST-068**: At each verification/substitution point the
  implementation's own removal logic actually observes or controls during a
  multi-step or recursive removal, an object substituted after it was
  selected for removal but before that specific check runs does not inherit
  the authorization already granted to the original object at that point;
  when a transient, non-adversarial failure causes a retry, the retry
  re-establishes authorization at that point rather than reusing an earlier
  check's result. **This is bounded by what the implementation can actually
  observe or control — it is not a claim that every conceivable substitution
  at every possible instant in time is detected. It explicitly does not
  contradict, and must be read together with, the Residual Limitation in
  §7: a race occurring strictly inside a window none of the implementation's
  checks can observe is not proven excluded.** *(Generalized from former
  AC-CODEX-TEST-071, with "marker"/nested-traversal specifics removed.
  Revised again to drop the "at every step" absolute framing in favor of "at
  each point the implementation actually checks.")*
- **AC-CODEX-TEST-069**: A symlink/junction encountered while removing a
  location's contents must never have its target traversed or modified.
  Given that, the implementation may either (a) safely remove the link
  object itself, or (b) fail closed and leave the link (and any verified
  residual around it) in place for a later attempt — both are acceptable
  outcomes, and this does not mandate one specific deletion technique for
  the link object itself, only that its target is never the thing acted on.
  *(Reworded from mandating removal of the link as the sole outcome, to
  state the outcome — target never traversed/modified — and allow either a
  safe removal or a fail-closed-with-residual response.)*

**G. Account/session compliance runtime gate (policy-agnostic gating
mechanism)** (`codexAppServerClient.test.ts`, "M7.2"/"M7.3" groups):

- **AC-CODEX-TEST-070**: A compliant verdict lets startup complete and is
  recorded only against the exact run that produced it.
- **AC-CODEX-TEST-071**: A non-compliant, unclassifiable, error, or
  timed-out verdict all fail startup closed with one fixed message, block
  every *task-capable* outbound call from that point on, and defer cleanup
  until exit is confirmed. This does not block the protocol handshake or the
  compliance check's own round trip — those necessarily occur, and are
  permitted to occur, before a compliant verdict exists at all (§5.1); the
  fail-closed guarantee this test protects applies to task-capable traffic
  specifically, not to the handshake/verification exchange that produces the
  verdict in the first place. *(Reworded to remove "block the wire
  entirely," which read as contradicting the fact that handshake and
  compliance-check traffic must occur before any verdict — compliant or
  not — can exist.)*
- **AC-CODEX-TEST-072**: Verification is attempted a bounded number of times
  per run (the current prototype uses exactly one, with no automatic
  retry); this bound is never silently exceeded.
- **AC-CODEX-TEST-073**: Every task-capable operation is rejected with a
  fixed gate error before its run's verification has passed, and proceeds
  (subject to its own other preconditions) once it has.
- **AC-CODEX-TEST-074**: A task-capable call's authorization is re-checked at
  every asynchronous suspension point during its own execution, not only at
  entry. Two distinct cases are covered: (a) if invalidation occurs **before**
  the call's RPC has been transmitted, the call is blocked at that
  suspension point and never reaches the wire; (b) if invalidation occurs
  **after** the RPC has already been transmitted (including while its
  response is still pending), the already-sent request cannot be retroactively
  un-sent — the requirement instead is that no *further* task-capable action
  is taken from that point on, and the eventual response (if any) is not
  treated as authorizing any additional action beyond itself. *(Reworded to
  drop the absolute claim that an already-transmitted call "never reaches the
  wire" — that is only true for the before-transmission case; the
  after-transmission case is bounded to "no further authority is granted,"
  which is the guarantee that is actually achievable.)*
- **AC-CODEX-TEST-075**: A new run begins with no compliance verdict of its
  own regardless of the previous run's outcome, and a late/forced result
  destined for a superseded run can never mark a newer, live run as
  verified — this holds even when the late result and the newer run's own
  genuine success are separated only by which one a check happens to read
  first.
- **AC-CODEX-TEST-076**: Once a run is torn down (explicit shutdown or an
  unprompted exit), every gated operation immediately rejects again and no
  further session state is recorded for it; a stale lifecycle signal naming
  an older run cannot affect a newer run's state.
- **AC-CODEX-TEST-077**: An action that can change the authenticated account
  (such as a login flow this integration itself initiates) invalidates the
  current run's verdict synchronously and strictly before that action's own
  request is sent; only that one in-flight action finishes under the old
  verdict, and every other operation is blocked from that instant on.
- **AC-CODEX-TEST-078**: An unsolicited signal that the account may have
  changed invalidates the current run's verdict and begins a safe shutdown.
  From that invalidation point onward, no new authority, continuation,
  task-capable action, or session authorization may proceed for this run
  except what safe termination/containment itself requires — this holds both
  for operations that had not yet started and for a step of an operation
  already in progress that has not yet reached the wire. This does not claim
  that a request the operation already transmitted, or work it already
  caused, before the invalidation point is retroactively prevented or undone
  — only that nothing further is authorized from that point on. The signal's
  content never appears in logs or thrown errors.
- **AC-CODEX-TEST-079**: If an account-change signal races an in-flight
  verification round trip (in either arrival order, including within the
  same physical chunk of data), the resulting shutdown always wins — a late
  compliant response for that exact round trip can never resurrect
  authorization — and the concurrent shutdown/failure paths never perform
  duplicate teardown work or produce an unhandled failure.
- **AC-CODEX-TEST-080**: Ordinary successful-startup behavior with no
  account-change activity is unaffected by the invalidation machinery.
- **AC-CODEX-TEST-081**: A shutdown already in progress for the exact same
  run is awaited by a second concurrent trigger rather than duplicated.

**H. New coverage added by this revision** (implementation-independent;
supported by evidence already gathered, no additional source inspected):

- **AC-CODEX-TEST-082 (Outbound capability boundary)**: Outbound requests to
  the Codex process are limited to an explicit, reviewed set of methods, and
  no optional/experimental protocol capability is used unless it has been
  independently verified (per §2) to exist and to be required by the
  product. The current prototype demonstrates one way to achieve this (an
  `initialize` request explicitly declaring `experimentalApi: false`), which
  is evidence the outcome is achievable, not a mandate that a future
  implementation's protocol necessarily exposes, or that the integration
  necessarily uses, that same declaration field — the permanent requirement
  is the absence of unverified-capability reliance, not this specific
  handshake mechanism. (`codexAppServerClient.ts:32-51`, `2194-2201`)
- **AC-CODEX-TEST-083 (Approval authority)**: A response to an approval id
  that is not currently a tracked, pending approval (unknown,
  already-answered/duplicate, or expired) has no effect and grants no
  authority; an incoming approval request for an id already being tracked is
  not re-registered or re-armed. (`codexAppServerClient.ts:2517-2527`, `2682-2694`)
- **AC-CODEX-TEST-084 (Diagnostic data minimization)**: Raw diagnostic output
  from the Codex process (its stderr stream) is consumed so it cannot block
  the process, but its content is never forwarded into any public-facing
  log, event, or error message — since it may carry prompts, commands,
  patches, or authentication material. (`codexAppServerClient.ts:2185-2187`)
- **AC-CODEX-TEST-085 (State root containment)** *(new — added in this
  revision to give AC-CODEX-SEC-027 direct acceptance-test coverage, which
  was previously only implied by other groups)*: A derived per-run isolated-
  state path is accepted only when it is canonically verified to remain
  under the AgentCubicles-owned state base, confined beneath that run's own
  instance-specific location. A path that would escape the owned base or
  the run's own instance location — by construction, or by symlink/junction
  indirection that resolves outside it — is rejected/fails closed rather
  than being used for any isolated-state operation.
- **AC-CODEX-TEST-086 (Private transport)** *(new — added in this revision
  to give AC-CODEX-SEC-028 direct acceptance-test coverage, which was
  previously only implied by §2's launch-shape evidence row)*: For the
  supported local App Server integration mode, verification confirms that no
  network listener (TCP or UDP, loopback-bound or otherwise) is opened by the
  integration for App Server communication, and that all such communication
  occurs only over the spawned child process's own stdio pipes.

### Mapping note (test requirements)

- `TEST-001`–`TEST-002`: kept, reframed as documenting the current
  PROVISIONAL PILOT POLICY specifically rather than a permanent rule.
- `TEST-003`–`TEST-021`: kept, `TEST-014`/`TEST-017`/`TEST-018` reworded to
  remove `AgentEvent`/provider-boundary/fixed-status-label language.
- `TEST-022`: **removed** as a StreamProvider-contract requirement; a
  generalized replacement occupies the same ID (contract-neutrality only,
  no `StreamProvider` reference).
- `TEST-023`–`TEST-044`: unchanged.
- `TEST-045` + `TEST-052` (old numbering): **merged** into new `TEST-045`.
- `TEST-046`–`TEST-051` (old): unchanged content, `TEST-051` (old) reworded
  to drop the exact version number/parsing-algorithm mandate → new
  `TEST-051`.
- `TEST-053`, `TEST-054` (old): reworded to drop exact variable names → new
  `TEST-052`, `TEST-053`.
- `TEST-055` + `TEST-056` (old): **merged** into new `TEST-054`.
- `TEST-057`, `TEST-058` (old): unchanged → new `TEST-055`, `TEST-056`.
- `TEST-059` + `TEST-061` (old): **merged** into new `TEST-057`.
- `TEST-060`, `TEST-062`–`TEST-064` (old): unchanged/lightly generalized →
  new `TEST-058`–`TEST-061`.
- `TEST-065` (old): **narrowed** (dropped "at any point" claim) → new
  `TEST-062`.
- `TEST-066` (old): unchanged → new `TEST-063`.
- `TEST-067` (old): **generalized** (dropped the "data-only fault plan" API
  reference) → new `TEST-064`.
- `TEST-068`–`TEST-070` (old): unchanged/lightly generalized → new
  `TEST-065`–`TEST-067`.
- `TEST-071` (old): **generalized** (dropped marker/nested-traversal
  specifics) → new `TEST-068`.
- `TEST-072` (old): unchanged → new `TEST-069`.
- `TEST-073`–`TEST-084` (old): unchanged in substance, "safe"/"unsafe"
  reworded to "compliant"/"non-compliant" throughout → new `TEST-070`–`TEST-081`.
- `TEST-082`–`TEST-084`: **new**, no prior numbering.

Net count (prior revision): 84 (old) → **84** (new) — three pairwise merges
(`TEST-045`+`TEST-052`, `TEST-055`+`TEST-056`, `TEST-059`+`TEST-061`, each
two old IDs collapsing into one new ID) remove three old IDs; one
removal-and-replacement (`TEST-022`) keeps the same ID and changes net count
by zero; three additions (`TEST-082`–`TEST-084`) restore the three removed.
84 − 3 + 3 = 84 — the count was unchanged only by coincidence, not because
nothing substantive changed.

**Later revision, same IDs:** a subsequent review found nine of the items
above still smuggled in implementation technique or an absolute claim the
evidence didn't support. `TEST-007`–`TEST-009`, `TEST-022`, `TEST-045`,
`TEST-049`, `TEST-054`, `TEST-067`, and `TEST-068` were rewritten in place —
no ID was added, removed, merged, or renumbered for this pass — to: mark the
JavaScript/Node adversarial-input cases as runtime-specific examples of a
generalized requirement (007–009); make a shared cross-integration
representation conditional on the architecture choosing one at all (022);
make a test-only override conditional on existing at all, rather than
required (045); demote the exact Git mechanism to non-mandatory prototype
evidence, cross-referencing the real (outcome-only) requirement at
`AC-CODEX-SEC-014` (049); drop the requirement to mutate/restore the real
process-wide environment or verify it from a later test (054); drop "the
residual must be empty" as a permanent precondition, allowing a stronger
identity/handle/transaction mechanism instead (067); and narrow "at every
step" to "at each point the implementation actually observes or controls,"
explicitly tying the result back to the §7 Residual Limitation (068).

**This revision (specification-consistency fix):** in addition to further
rewording `TEST-030`, `TEST-031`, `TEST-050`, `TEST-056`, `TEST-059`,
`TEST-060`, `TEST-074`, and `TEST-082` in place for the reasons described
alongside each (byte-oriented boundary framing; capacity-vs-untrustworthy-
transport scoping; conditional Git-mechanism evidence; artifact-generation
vs. host-config-copying scope; the "never began real work" cleanup-
authorization ambiguity; the revocation-timing absolute claim; and the
handshake-mechanism assumption, respectively), two genuinely new IDs were
added: `AC-CODEX-TEST-085` (state root containment, direct coverage for
`AC-CODEX-SEC-027`) and `AC-CODEX-TEST-086` (private transport, direct
coverage for `AC-CODEX-SEC-028`). Neither existing requirement was merged or
renumbered to accommodate them, per this revision's own instruction not to
preserve a prior total for its own sake.

Net count (this revision): 84 → **86** (two additions, `TEST-085`–`TEST-086`;
no removals, no merges).

**Final specification-consistency pass (this pass), same IDs, no count
change:** a second full-file review found eight findings, none requiring a
new or removed ID (four MUST FIX plus four follow-ups). MUST FIX: the
`AC-CODEX-SEC-016`/§8/`TEST-060` pre-spawn cleanup wording; the §4/
`AC-CODEX-SEC-025`/§8/`TEST-031` approval-capacity contract; the §8
compliance-failure/revocation rows and `TEST-078` (made forward-looking — no
new authority/continuation/task-capable action/session authorization
proceeds after invalidation except what safe termination/containment
requires, without claiming already-transmitted work is undone); and
`TEST-071` (scoped "block the wire entirely" to task-capable traffic only,
since handshake/compliance-check traffic must occur before any verdict
exists). Follow-ups: the §4 notification bullet (security/control-plane
notifications need not first become normalized activity events); `TEST-044`
(replaced a symmetric "disjoint from the real home" requirement with the
actual one-directional safety property — must not equal or expose/alias the
real home, not "never nested under it at all"); `TEST-069` (made
outcome-oriented — the symlink/junction target must never be traversed or
modified, but either safely removing the link or failing closed with a
verified residual is acceptable, not one mandated deletion technique); and
the "three merges removing four old IDs" arithmetic above, corrected to
"three pairwise merges remove three old IDs" (`84 − 3 + 3 = 84`). Net count
unchanged: **86**.

## 11. Cross-Platform Requirements

Supported only to the extent the inspected evidence actually establishes it —
platform-gated tests only prove the behavior on the platform(s) that actually
ran them.

- **Windows**: environment-variable name matching must be case-insensitive
  (Windows itself reports variable names inconsistently, e.g. `PATH` vs.
  `Path`, `SystemRoot`); `USERPROFILE`/`HOMEDRIVE`/`HOMEPATH` and
  `APPDATA`/`LOCALAPPDATA` must be derived from the isolated user-home path
  using genuine Windows path semantics (drive-letter-rooted absolute paths
  only — a UNC path is explicitly unsupported and must fail closed rather
  than degrade). (`codexAppServerClient.ts:710-810`, `833-863`; test groups "M6.2", "M6.3")
- **POSIX (macOS/Linux)**: `HOME` must be an absolute POSIX path or the run
  fails closed; `TMPDIR` must be set identically to `TEMP`/`TMP` since POSIX
  tools commonly consult it instead. (`codexAppServerClient.ts:812-831`, `869-873`; test group "M6.4", explicitly noted as "skipped on Windows" for its real-host variant, `codexAppServerClient.test.ts:3904`)
- **All platforms**: whatever concrete allowlist-construction technique and
  whatever Git-specific (or alternative) isolation mechanism a future
  implementation chooses, it must achieve **equivalent isolation outcomes on
  every supported platform** — the credential/proxy/runtime-hijack exclusions
  (AC-CODEX-SEC-012) and the subordinate-tool isolation outcome
  (AC-CODEX-SEC-014) must hold regardless of platform. This does not mandate
  that Windows and POSIX share one construction technique (e.g. a single
  "union of a Windows baseline and a POSIX baseline" allowlist shape) or that
  the same concrete Git mechanism be used on every platform — only that
  whichever mechanism each platform uses closes the same gap. The current
  prototype's own choice (baseline union; one Git environment-variable
  mechanism reused unchanged across platforms — see §10's Git-isolation group
  and §12) is evidence that a single shared technique is *one* way to reach
  that outcome, not a requirement that it be the way. (`codexAppServerClient.ts:206-234`, `890-938`)
- **Not established by this audit**: whether every environment-isolation and
  destructive-cleanup invariant has actually been exercised in CI on all
  three of Windows, macOS, and Linux — some real-filesystem/real-`git`
  test groups are explicitly platform-gated in source (e.g. the POSIX
  real-host group is skipped on Windows), so cross-platform assurance should
  be re-established for the new implementation rather than assumed from this
  document.

## 12. Legacy Architecture Couplings to Reject

These exist in the current code because of the pre-existing (upstream-derived)
provider architecture, or because of one implementation's specific technique
choices, not because Codex or sound security engineering requires them. The
new codebase should design its own contracts and mechanisms and derive a
Codex mapping from Codex's own protocol shape and the security *properties*
in §7, not from the shapes below.

- The requirement to normalize every Codex signal into the existing
  `AgentEvent` kind taxonomy (`toolStart`/`toolEnd`/`turnStart`/`turnEnd`/
  `sessionStart`/`sessionEnd`/`permissionRequest`/`approvalResolved`/etc.)
  and the `StreamProvider` interface shape (`kind`, `id`, `displayName`,
  `protocolVersion`, `formatToolStatus`, three fixed tool-name-classification
  sets) is a fitting exercise into a taxonomy that was planned for a
  different provider style before Codex support existed at all. It should
  not be treated as a requirement for the new architecture. (`codexAppServerNormalizer.ts:1-16`, `114-136`; provenance: see `PROVENANCE_DECISIONS.md`)
- The broader `kind: 'hook' | 'stream'` provider-taxonomy split and the
  associated `RuntimeProvider = HookProvider | StreamProvider` registration
  model are likewise pre-existing, upstream-planned constructs (see
  `PROVENANCE_DECISIONS.md`) — the new architecture's own integration
  registration model should be designed independently, not assumed to need
  the same split.
- The three-bucket tool classification (`permissionExemptTools`,
  `subagentToolNames`, `readingTools`) is a category scheme shaped around a
  different CLI's tool vocabulary; Codex's own "item" types
  (`commandExecution`, `fileChange`, `mcpToolCall`, `dynamicToolCall`,
  `collabAgentToolCall`, `webSearch`, `imageView`) are only loosely mapped
  onto it (e.g. a single `collabAgentToolCall` standing in for "subagent").
  A fresh design should classify Codex's own item types on their own terms.
  (`codexAppServerNormalizer.ts:17-25`, `133-135`)
- Baking human-readable, UI-facing status strings (`formatToolStatus`)
  directly into the same module responsible for protocol normalization
  couples presentation concerns to protocol parsing. A new design can keep
  these separate.
- The `onEvent?: (providerId, sessionId, event) => void` callback shape is a
  direct extension of the existing runtime's broadcast wiring convention; it
  is not implied by anything about Codex itself and should not constrain a
  new event-delivery design. (`codexAppServerClient.ts:629`)
- The generation-counter/lifecycle-state-machine bookkeeping pattern used
  throughout `codexAppServerClient.ts` (numeric monotonic "generation"
  counters, a hand-rolled state machine of
  `idle/starting/running/failed-pre-spawn/terminated`) is one specific
  implementation technique for the *underlying requirements* in §3 and §7 —
  the requirements (no overlapping runs, no cleanup before confirmed exit,
  no stale-run authorization) must be kept; the specific mechanism does not
  need to be.
- The specific destructive-cleanup technique — an on-disk ownership-marker
  file, a rename-to-quarantine step, paired lstat/fstat (or equivalent
  double-check) identity comparison, a "marker-last" ordering, and a
  separate "markerless residual reclaim" recovery path — is one
  implementation's way of satisfying the ownership/authorization/fail-safe
  requirements in §7 (SEC-016–019), including their documented residual
  TOCTOU limitation. The new architecture may use a completely different
  mechanism (e.g., OS-level sandboxing, a different filesystem primitive, a
  managed volume, or a stronger transactional guarantee) as long as it
  satisfies those requirements — and, if it cannot close the residual gap
  either, documents that limitation just as explicitly as this document
  does.
- The "private class field with no constructor/options-surface path,
  reached only through a test-local `as`-cast harness" pattern used to make
  certain security-relevant checks unit-testable without a production
  override surface is one implementation's testability technique. The new
  architecture may design its own approach to testability, as long as
  production configuration genuinely cannot override the security-relevant
  checks it protects (see AC-CODEX-TEST-045).
- The exact Git-isolation mechanism (the specific set of `GIT_CONFIG_*`
  environment variables, the specific minimum-version threshold, and the
  specific `git --version` output-parsing algorithm) is a prototype/MVP-
  stage choice, not a permanent architecture requirement. It should not be
  carried forward unless the new architecture independently selects Git
  environment-variable isolation (as opposed to some other mechanism) as its
  own solution to AC-CODEX-SEC-014.
- The concrete PROVISIONAL PILOT POLICY account-eligibility rule (§5.2) is
  itself a legacy-of-the-pilot coupling in spirit, even though it is not an
  architectural pattern: it must not be assumed as the permanent account
  policy without a product decision (see §5.2 and the Open Questions below).

## 13. Open Questions / Unverified Assumptions

- Every "External Codex/App Server Fact" in §2, **including the Account/
  GetAccountResponse/PlanType shape**, is sourced only from the current
  implementation's own code comments — this review did not independently
  inspect Codex's published documentation or generated protocol bindings for
  any of them, including the version-pinned citation (Codex CLI 0.153.4) the
  implementation itself gives for the Account shape. An implementation
  comment claiming something was "confirmed" or "independently reviewed" is
  not a substitute for this document (or the future implementation)
  independently verifying it.
- The full App Server method vocabulary the normalizer expects
  (`thread/*`, `turn/*`, `item/*`, `serverRequest/resolved`, `account/*`) has
  no version-pinned citation in-code at all and should be treated as at
  least as unverified as every other row in §2.
- The concrete account/session policy to adopt permanently (§5.2) is a
  **product decision that has not been made in the inspected evidence** —
  it must be derived from `PRODUCT_REQUIREMENTS.md` together with
  independently verified Codex account/session capabilities, not carried
  forward from the prototype's provisional rule by default.
- Whether the current prototype's Git-isolation mechanism (§10, §12) fully
  closes every credential/identity vector across all Git distributions and
  configurations, or only the ones its own comments enumerate as covered, is
  not independently established — the code itself documents specific
  *known-open* limitations (command-local environment overrides a
  Codex-run subprocess could set for itself) as accepted, not fixed. This is
  independent of whether the new architecture even keeps a Git-specific,
  environment-variable-based mechanism at all (see §12).
- The degree to which the current pure-Node/pathname-based destructive-
  cleanup verification (§7's Residual Limitation) can actually be defeated
  by a sufficiently fast or sufficiently privileged local actor is not fully
  characterized by the inspected evidence — only that it is not proven
  excluded. Whether the product's threat model requires closing this gap
  (e.g. via OS-level sandboxing or a different primitive) or can accept it
  under a local/single-user trust assumption is a product decision, not
  something this document can resolve.
- Whether the relationship between this stream/app-server integration and the
  separate, non-functional hook-based Codex provider (out of scope for this
  document; see `PROVENANCE_DECISIONS.md`) reflects two competing designs or
  a planned migration from one to the other is not established from the
  inspected sources.
- Real-`git`- and real-filesystem-backed test coverage is explicitly
  platform-gated in places (e.g., skipped on Windows for one POSIX-specific
  group); the degree of actual cross-platform proof behind §11 should be
  re-verified against current CI results rather than assumed from source
  alone.
