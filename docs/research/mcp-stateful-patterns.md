# MCP Primitives and Patterns for Stateful, Turn-Based Servers

Research for issue [#8](https://github.com/Pfeif-Master/valeofeternity_MCP/issues/8) and the
blocking-tool-call design candidate for the Vale of Eternity game server + MCP agent seats.

**Spec revision surveyed:** [`2026-07-28`](https://modelcontextprotocol.io/specification/2026-07-28/),
the revision marked **Current** at
[modelcontextprotocol.io/specification/versioning](https://modelcontextprotocol.io/specification/versioning)
as of this survey (2026-08-10). This revision is very fresh — it superseded `2025-11-25` only
around three weeks before this survey — and it is a large, mostly-breaking rewrite (stateless
core, MRTR, Tasks moved out of core). See **"What's still in flux"** at the end: most deployed
clients and servers today almost certainly still speak `2025-11-25` or older, so this document
covers both and flags every place they disagree.

---

## Bottom line

**The blocking-call pattern works only with heavy caveats, and I would not build the seat
protocol around it as the primary mechanism.**

- The spec does not forbid a long-running tool call. Nothing in the spec sets a maximum request
  duration, and cancellation is opt-in (`notifications/cancelled` on stdio, stream-close on HTTP).
  So *protocol-legally*, a tool call can sit unanswered indefinitely.
- But every layer above the wire actively works against it. Implementations **SHOULD** enforce a
  maximum timeout regardless of progress notifications
  ([Cancellation §Timeouts](https://modelcontextprotocol.io/specification/2026-07-28/basic/utilities/cancellation#timeouts)).
  The reference TypeScript SDK ships a 60s default. Claude Code — the concrete harness this repo
  will attach agents through — auto-backgrounds any main-conversation MCP call that runs past two
  minutes, and explicitly does **not** do this for subagent calls or non-interactive/headless runs,
  meaning a long blocking call in exactly the deployment shape this repo cares about (a scripted
  agent occupying a seat) risks the run ending before the result ever arrives (see
  [Claude Code MCP docs](https://code.claude.com/docs/en/mcp#automatic-backgrounding-of-long-tool-calls)).
- The new Streamable HTTP transport (`2026-07-28`) removed stream resumability and session IDs
  entirely — a dropped connection during a multi-hour blocked call is not recoverable at the
  protocol level; the client must re-issue the whole request. The older transport
  (`2025-11-25` and earlier) at least offered `Last-Event-ID` resumption, but even that only
  covers reconnect-and-replay, not surviving a full client/process restart.
- Sampling and elicitation are not designed for this either. Both are now "answer inline, in the
  same round-trip" mechanisms ([MRTR](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/mrtr))
  — the client is expected to gather the input and immediately retry, not to leave a request open
  for a real-world span of time. Sampling itself is now **deprecated**
  ([deprecated features registry](https://modelcontextprotocol.io/specification/2026-07-28/deprecated)).
- The MCP project's own answer to "a tool call might take minutes or hours" is the **Tasks
  extension** — and its own rationale document is titled, pointedly, **"Why not just block?"**
  ([ext-tasks docs](https://modelcontextprotocol.io/docs/extensions/tasks#why-not-just-block)). It
  lists exactly the failure modes this ticket is worried about: no long-lived connections, no
  crash resilience, no progress visibility, no mid-flight interaction without a second channel.

**Recommendation:** don't hold the seat's turn inside one blocked tool call. Model the seat as an
**explicit state handle** (`game_id` / `seat_token`, per
[SEP-2567](https://modelcontextprotocol.io/seps/2567-sessionless-mcp)) that the agent threads
through every call, and resolve "wait for my turn" with a **short-poll or task-poll loop** driven
from outside the blocked-request model: either the MCP Tasks extension where the client supports
it, `subscriptions/listen` for a push-when-ready signal, or — most portable today, since Tasks
client support is not yet in the tracked extension matrix — a cheap `wait_for_turn(seat_token,
timeout_s=25)` tool the harness calls in a loop, each call bounded well under every timeout in the
stack. That is functionally "the server does not return until it needs your input again," just
implemented as bounded round-trips instead of one arbitrarily long one — which is exactly the
distinction the spec authors themselves draw.

---

## 1. MCP primitives and what each is for

| Primitive | Status (2026-07-28) | What it's for |
|---|---|---|
| **Tools** (`tools/list`, `tools/call`) | Active | Model-invokable actions with a JSON Schema. The primary vehicle for "the agent does something." |
| **Resources** (`resources/list`, `resources/read`, `resources/templates/list`) | Active | Addressable, read-only(-ish) data the client/model can pull in — files, DB rows, API responses. |
| **Prompts** (`prompts/list`, `prompts/get`) | Active | Reusable, server-defined prompt templates the client can surface to the user/model. |
| **Sampling** (`sampling/createMessage`) | **Deprecated** in this revision | Let a server ask the client's LLM to generate a completion, so the server doesn't need its own model credentials. Migration path: "integrate directly with LLM provider APIs" ([deprecated registry](https://modelcontextprotocol.io/specification/2026-07-28/deprecated)). |
| **Elicitation** (`elicitation/create`) | Active | Structured mid-call requests for more input from the user, in `form` mode (schema'd data) or `url` mode (out-of-band web flow for sensitive data / third-party auth). Now delivered via MRTR, not a standalone server-initiated request ([Elicitation spec](https://modelcontextprotocol.io/specification/2026-07-28/client/elicitation)). |
| **Roots** | **Deprecated** in this revision | Let the client tell the server which filesystem directories are in scope. Migration path: pass paths as ordinary tool parameters. |
| **Logging** (`logging/setLevel`, `notifications/message`) | **Deprecated** in this revision | Server → client log stream. Migration path: `stderr` on stdio, or OpenTelemetry. |
| **Notifications** — `notifications/progress`, `notifications/cancelled` | Active | Request-scoped, in-band signals tied to one in-flight request (see §2). |
| **Notifications** — list-changed / resource-updated | Active, re-plumbed | Delivered over a long-lived `subscriptions/listen` stream instead of a standalone `resources/subscribe` + GET stream (see §5.3). |
| **Tasks** (`io.modelcontextprotocol/tasks` extension: `tasks/get`, `tasks/update`, `tasks/cancel`) | **Extension, not core** — moved out of the base spec in this revision | Durable handle for long-running work; polling-based. See §5.1. |
| **`server/discover`** | Active, new/mandatory | Single RPC returning a server's supported protocol versions, capabilities, and identity — replaces the `initialize` handshake for version/capability negotiation. |

Source for the primitive taxonomy and the overview text: [Base Protocol overview](https://modelcontextprotocol.io/specification/2025-06-18/basic/index) (component list); per-primitive status and migration guidance from the [2026-07-28 changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog) and [deprecated features registry](https://modelcontextprotocol.io/specification/2026-07-28/deprecated).

**Implication for this repo:** Sampling was never the right primitive for "let the agent seat make a move" — it's for the *server* borrowing the *client's* model, the inverse of what a game server needs. Tools are the right primitive for moves; the open question is only how the *server* tells the agent "it's your turn."

---

## 2. Timeouts, cancellation, and progress — what the spec actually says

### Cancellation (`notifications/cancelled`)

- Optional, best-effort: a client **SHOULD** send it; a server **MAY** ignore it if the request is
  unknown or already done.
- **Transport-specific delivery mechanism, not a uniform message:**
  - **stdio**: client **MUST** send `notifications/cancelled` referencing the request ID — there's
    no per-request channel to close.
  - **Streamable HTTP**: closing the SSE response stream *is* the cancellation signal. The server
    **MUST** treat a client disconnect as cancellation. No `notifications/cancelled` message is
    expected or required on this transport in the current revision.
- Servers **MUST NOT** send `notifications/cancelled` for anything except tearing down a
  `subscriptions/listen` stream.

Source: [Cancellation](https://modelcontextprotocol.io/specification/2026-07-28/basic/utilities/cancellation).

### Timeouts

> "Implementations **SHOULD** establish timeouts for all sent requests, to prevent hung
> connections and resource exhaustion. […] Implementations **MAY** choose to reset the timeout
> clock when receiving a progress notification corresponding to the request, as this implies that
> work is actually happening. However, implementations **SHOULD** always enforce a maximum
> timeout, regardless of progress notifications, to limit the impact of a misbehaving client or
> server."
> — [Cancellation §Timeouts](https://modelcontextprotocol.io/specification/2026-07-28/basic/utilities/cancellation#timeouts)

Two load-bearing facts for the blocking-call design:

1. **Progress notifications resetting the timeout is a MAY, not a MUST**, and even where
   implemented, a hard ceiling is still supposed to apply on top. A well-behaved client is allowed
   to kill your "held open with heartbeats" call regardless of how diligently the server emits
   progress.
2. This is spec guidance to *implementations*, not a protocol-enforced limit. The spec sets no
   number. Real numbers come from client SDKs and hosts (§4).

### Progress notifications (`notifications/progress`)

- Client opts in per-request by attaching a `progressToken` in `_meta`.
- Server **MAY** send zero or more `notifications/progress` (monotonically increasing `progress`,
  optional `total`, optional human-readable `message`) before the final response.
- Entirely best-effort: a server that never sends one is compliant.
- Progress notifications **MUST stop after completion** and both sides **SHOULD** rate-limit them.

Source: [Progress](https://modelcontextprotocol.io/specification/2026-07-28/basic/utilities/progress).

**Net effect for the proposed pattern:** progress notifications can, at best, convince a
*generously configured* client not to time out a request early. They cannot make the wait durable
across a disconnect, cannot survive a client/process restart, and are explicitly not something the
spec lets you rely on for the actual hold time.

---

## 3. Transports and what each implies for holding a request open

### stdio

- Client spawns the server as a subprocess; single shared byte stream (`stdin`/`stdout`), one
  message per line, no per-request channel.
- **No built-in timeout or resumability concept at the protocol level** — the process either keeps
  running or it doesn't. If the server process dies mid-call, the client loses the request outright;
  since the protocol is now stateless, the client's only recourse is to restart the server and
  retry ([stdio: Unexpected Termination](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/stdio#unexpected-termination)).
- Shutdown is cooperative (close stdin → wait → SIGTERM/SIGKILL escalation) — nothing here is
  designed around one request running for minutes to hours, but nothing forbids it either, since
  there's no HTTP intermediary to time it out. The practical ceiling is entirely the *client's*
  own timeout logic (§4), not the transport.
- A held-open stdio call also **blocks the single shared channel for every other in-flight
  exchange with that server** in practice, because stdio has no per-request stream — though the
  spec allows interleaving notifications for other subscriptions, a naive implementation that just
  waits on one response can starve everything else on the same process.

Source: [stdio transport](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/stdio).

### Streamable HTTP (current transport, `2026-07-28`)

This is the transport that matters most for a long-lived, network-accessible game server.

- One HTTP endpoint, POST-only. Each JSON-RPC request/notification is its own POST. The server
  answers with either a single JSON object or an SSE stream scoped to *that one request*.
- **Sessions and `Mcp-Session-Id` are gone entirely** in this revision
  ([changelog #1](https://modelcontextprotocol.io/specification/2026-07-28/changelog),
  [SEP-2567](https://modelcontextprotocol.io/seps/2567-sessionless-mcp)). There is no
  connection-level state to hang a seat off of; every request must carry whatever context it needs
  (see §6).
- **No stream resumability.** `Last-Event-ID` and SSE event IDs are removed. "A broken response
  stream loses the in-flight request; clients **MUST** re-issue it as a new request with a new
  request ID" ([changelog #9](https://modelcontextprotocol.io/specification/2026-07-28/changelog)).
  A held-open request that survives for the duration of an opponent's turn has **zero
  fault-tolerance** on this transport: any network blip, proxy idle-kill, or load-balancer
  reshuffle silently discards the wait and the agent gets nothing back — not even an error, unless
  it separately notices the connection dropped.
- No redelivery/queueing story either — this is a direct consequence of statelessness. There's
  nowhere for the server to park an in-flight response if the client isn't listening for it at the
  moment it's ready.
- Long-lived server → client streams are *explicitly repurposed* to only carry change
  notifications via `subscriptions/listen`, not arbitrary request results:
  > "Request-scoped notifications such as `notifications/progress` and `notifications/message`
  > continue to flow on the response stream of the request they relate to, not the
  > `subscriptions/listen` stream." — [changelog #4](https://modelcontextprotocol.io/specification/2026-07-28/changelog)
- Infrastructure-facing detail that matters for a self-hosted server: `X-Accel-Buffering: no` and
  periodic SSE comment-line keep-alives are the documented mitigations for proxies/load balancers
  buffering or idle-killing a long response stream
  ([Streamable HTTP §Receiving Messages](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http#receiving-messages)) —
  i.e., the spec itself acknowledges that intermediaries will kill a quiet long-held stream unless
  you actively fight them.

Source: [Streamable HTTP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http).

### Streamable HTTP, older revisions (`2025-03-26` through `2025-11-25`) — the practical baseline today

Because `2026-07-28` is only weeks old, the deployed reality (Claude Code, most third-party
servers) is almost certainly still this shape as of the survey date:

- **`Mcp-Session-Id`** assigned at `initialize`, sent on every subsequent request — genuine
  connection-scoped state exists.
- **Resumable via `Last-Event-ID`**: a disconnected stream can be reconnected and replayed from the
  last delivered event, *if* the server implements the resumability store. This is meaningfully
  better fault-tolerance than the current revision for a long-held request, but is opt-in and
  still doesn't survive the *client* itself restarting without persisted event-ID bookkeeping.
- **Server-initiated requests permitted on the SSE stream** — a server could open a GET stream and
  push a `sampling/createMessage` or (informally) a custom notification whenever it wanted, without
  waiting for the client to poll. This is the one respect in which the old transport is *more*
  hospitable to "the server pushes to me when my turn comes" than the new one — see §5.4.
- The server **MAY close the underlying TCP connection while keeping the logical SSE stream open**
  for polling reconnects (`retry` field + reconnect), which is a built-in acknowledgment that
  holding one connection open for a long time is expected to be unreliable in practice.

Source: [2025-11-25 Transports](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports) (session management, resumability, and multi-connection sections).

### HTTP+SSE (`2024-11-05`, deprecated)

Deprecated since `2025-03-26`, and reclassified formally **Deprecated** under the new feature
lifecycle policy in this revision, eligible for removal
[three months after SEP-2596 reaches Final](https://modelcontextprotocol.io/specification/2026-07-28/deprecated).
Two separate endpoints (GET for the SSE stream, POST for messages), no `Mcp-Session-Id` concept at
all — state was whatever the server could tie to the SSE connection object itself. Not worth
designing against; new implementations **SHOULD NOT** adopt it.

---

## 4. Can the client harness itself sustain a blocked tool call?

This is where the proposal runs into the sharpest wall, and it's not a protocol question at all —
it's a **host/harness** question, and the spec explicitly leaves it up to the client.

### Reference SDK defaults

- TypeScript SDK: 60-second default request timeout
  ([typescript-sdk#2250](https://github.com/modelcontextprotocol/typescript-sdk/issues/2250),
  [typescript-sdk#245](https://github.com/modelcontextprotocol/typescript-sdk/issues/245)). Whether
  progress notifications reset that timeout was, for a long stretch, **inconsistent between the TS
  and Python SDKs** — TS defaulted `resetTimeoutOnProgress` to `false` while Python honored
  progress resets — until a fix aligned TS to match Python's behavior. This is exactly the kind of
  cross-SDK inconsistency the spec's "MAY reset on progress" wording (§2) permits, and it bit real
  users.
- Python SDK (FastMCP): `tool_timeout_seconds`, default 300s (5 minutes), configurable via
  constructor or `FASTMCP_TOOL_TIMEOUT_SECONDS`, can be disabled by setting `None`.

### Claude Code specifically (the concrete harness for this repo's agent seats)

Claude Code's own docs are the most directly useful primary source here, because they describe
exactly the "agent occupies a seat via a long MCP call" shape:

- **Per-server hard wall-clock limit**: the `timeout` field in `.mcp.json` (or `MCP_TOOL_TIMEOUT`
  env var) — progress notifications **do not extend it**: "The per-server `timeout` is a hard
  wall-clock limit per tool call, and progress notifications from the server don't extend it."
  Default when unset: **~28 hours**.
- **Separate per-request timer** (HTTP/SSE/connector servers only): 60 seconds to the *first
  response byte*, unless the per-server `timeout`/`MCP_TOOL_TIMEOUT` is raised to 60s or more.
  Stdio/WebSocket servers don't have this timer.
- **Idle timeout**: if a server sends *no* response and *no* progress notification for the idle
  window, the call is aborted regardless of the wall-clock limit. Default: 5 minutes for
  HTTP/SSE/connector servers, 30 minutes for stdio. Configurable via
  `CLAUDE_CODE_MCP_TOOL_IDLE_TIMEOUT`.
- **Automatic backgrounding — the critical detail:**
  > "An MCP tool call in the main conversation that is still running after two minutes moves to a
  > background task instead of blocking the session. Claude receives the task ID immediately and
  > keeps working, and the result arrives as a task notification when the call settles."
  — [Claude Code MCP docs](https://code.claude.com/docs/en/mcp#automatic-backgrounding-of-long-tool-calls)

  And, directly on point for a scripted/headless agent occupying a game seat:

  > "Some calls never move to the background: Calls from subagents […] Calls in non-interactive
  > mode, unless `CLAUDE_AUTO_BACKGROUND_TASKS` is set to `1`, **since a one-shot run can end before
  > the result arrives**."

  In other words: in the exact deployment shape this repo cares about most (a non-interactive/
  headless agent process occupying a seat, or a subagent), Claude Code does **not** protect you
  from the pathology the "blocking call" design is trying to exploit as a feature. A held-open call
  either doesn't get auto-backgrounded (headless, without an opt-in flag) and the whole run can
  terminate with the tool call simply abandoned, or (main conversation, interactive) it silently
  stops being "the agent hibernating inside the pending call" after two minutes and becomes a
  background poll loop that the harness manages — which is functionally the poll/task pattern this
  document recommends, just implemented for you, invisibly, with its own separate `/tasks`
  bookkeeping you don't control.

Source: [Claude Code — Connect to MCP](https://code.claude.com/docs/en/mcp) (§ "Timeouts and idle
detection" and § "Automatic backgrounding of long tool calls"),
[Claude Code environment variables](https://code.claude.com/docs/en/env-vars).

### What kills the "no tokens consumed while blocked" premise, independent of all of the above

Even setting timeouts aside: "the agent is hibernating inside the pending tool call, consuming no
context and no tokens" is true only for the *wall-clock* dimension. It is not true the moment the
harness auto-backgrounds the call (Claude Code, main conversation, >2 min) — at that point the
harness is actively scheduling the agent to keep working on other things, which is good for
utilization but means the "one seat, one blocked call, nothing else happens" mental model doesn't
hold in the concrete harness this repo will run against.

---

## 5. Server-initiated interaction: is sampling/elicitation/tasks/subscriptions a better fit?

### 5.1 Tasks extension — purpose-built for this, but not core, and adoption is unclear

The Tasks extension (`io.modelcontextprotocol/tasks`) was **moved out of the core protocol and
into an official extension** in this revision, replacing an earlier "experimental core" version
that had a blocking `tasks/result` method
([changelog #6](https://modelcontextprotocol.io/specification/2026-07-28/changelog)). Its own
rationale doc answers this ticket's question almost verbatim:

> **"Why not just block?"** You could hold the connection open until the work finishes. Tasks solve
> problems that blocking cannot:
> - **No long-lived connections.** Blocking ties up a connection for the duration of the operation.
>   Many clients and transport intermediaries impose timeouts that make this impractical beyond a
>   few seconds.
> - **Crash resilience.** A task ID is a durable handle. If the client disconnects or restarts, it
>   can resume polling with the same ID.
> - **Progress visibility.** Tasks carry status metadata (`working`, `input_required`, `completed`,
>   `failed`, `cancelled`) and optional status messages.
> - **Mid-flight interaction.** When a task needs input, it moves to `input_required` and surfaces
>   the request. The client responds via `tasks/update` — no second connection or unsolicited
>   server-to-client messages required.
> - **Server-directed.** The server decides per-request whether to create a task.
>
> — [ext-tasks: Why not just block?](https://modelcontextprotocol.io/docs/extensions/tasks#why-not-just-block)

Mechanics: a supported request (server's choice, per-request) returns `CreateTaskResult`
(`resultType: "task"`) with a `taskId`, initial status, `ttlMs`, and a suggested `pollIntervalMs`
instead of the final result. Client polls `tasks/get`; terminal states are `completed` (with
`result`), `failed` (with `error`), `cancelled`. If the task needs input mid-flight, it moves to
`input_required` and the client answers via `tasks/update` — this is the mechanism that would
carry "your opponent moved, here's the board state, what do you play" if you wired the Tasks
extension end-to-end. Servers **MAY** also push status changes via `notifications/tasks` over
`subscriptions/listen`, which lets a well-behaved client skip polling entirely.

**Caveat on adoption:** Tasks is conspicuously **absent** from the tracked
[Extension Support Matrix](https://modelcontextprotocol.io/extensions/client-matrix) as of this
survey — only MCP Apps, OAuth Client Credentials, and Enterprise-Managed Authorization are listed
there, and none of the popular hosts (Claude Desktop, Claude web, VS Code Copilot, ChatGPT, Cursor,
Goose) show Tasks support in that table. "MCP Tasks is an extension to the core MCP specification.
Host support varies by client" is the extension's own framing
([ext-tasks](https://modelcontextprotocol.io/docs/extensions/tasks#client-support)). Treat Tasks as
directionally correct but **not yet something you can assume the harness on the other end
understands** — build your own equivalent (§6) that degrades to plain polling if the client never
declares the `io.modelcontextprotocol/tasks` capability.

### 5.2 Sampling — wrong direction, and deprecated

Sampling lets a **server** ask the **client's** LLM to generate a completion. It is not a channel
for "wake the agent up" — if anything it's the inverse relationship (server borrows client's model
credentials). It is also formally deprecated in this revision, migration path "integrate directly
with LLM provider APIs"
([deprecated registry](https://modelcontextprotocol.io/specification/2026-07-28/deprecated)). Not
relevant to this design question at all except to rule it out.

### 5.3 Elicitation — real-time nudge, not a durable wait mechanism

Elicitation is for the server saying "I need one more piece of structured input (or an out-of-band
web interaction) before I can finish *this specific request*." Under MRTR, the server returns
`InputRequiredResult` and the client is expected to gather the answer and **immediately retry** the
original call — this is designed as a synchronous, same-conversation-turn round-trip, not a
mechanism for waiting an indeterminate real-world span (an opponent's turn, which could be seconds
or hours depending on whether the opponent is a human or another agent). URL-mode elicitation is
the closest thing to "wait for an external, possibly slow event" — the client is told to poll the
*original* request again once the user finishes the out-of-band flow — but it's scoped to
user-facing web interactions (auth, payment, sensitive data), not "wait for another player."
Source: [Elicitation §Implementation Considerations, §URL Mode Flow](https://modelcontextprotocol.io/specification/2026-07-28/client/elicitation).

Practically: elicitation is the right shape for "the seat needs a decision from the human co-owner
of an agent+human hybrid seat," not for "wait until it's this seat's turn again."

### 5.4 Resource subscriptions / `subscriptions/listen` — the actual push mechanism, transport-dependent

This is the direct answer to "can the server push to the client, and how do subscriptions work":

- **Current revision (`2026-07-28`)**: `subscriptions/listen` replaced both the old GET-stream
  mechanism and `resources/subscribe`/`resources/unsubscribe`. Client opts into specific
  notification types (`toolsListChanged`, `promptsListChanged`, `resourcesListChanged`,
  `resourceSubscriptions: [uri, ...]`) in one request; server acknowledges with
  `notifications/subscriptions/acknowledged` carrying a `subscriptionId`; matching notifications
  flow on that single long-lived stream, tagged with the ID, until either side closes it or the
  transport drops. Source: [Subscriptions](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/subscriptions).
- This is a genuine, spec-sanctioned way to get server-to-client push **for change notifications**
  — and `notifications/tasks` rides this same channel, so a client that supports both `tasks` and
  `subscriptions/listen` gets a real "wake me up" signal with no polling at all.
- The catch for this repo's design: it's built for *resource* changes (`resources/updated`) and
  *list* changes, not for arbitrary application events like "it's your turn." You could shoehorn
  "it's your turn" into a resource URI per seat (`game://<id>/seat/<n>/to-act`) and subscribe to
  it, which is a legitimate pattern, but you're relying on this mechanism's general-purpose
  plumbing rather than a first-class primitive for it.
- Same fault-tolerance caveats as any Streamable HTTP stream apply: on `2026-07-28`, if the stream
  drops, there is no resumption — "On stdio, if the connection is terminated and then
  re-established, the client MUST re-send `subscriptions/listen`… the server holds no subscription
  state across reconnections." On HTTP the equivalent is: reconnect and re-subscribe from scratch,
  and hope you didn't miss a notification in between (no redelivery in this revision).

---

## 6. Concrete alternatives and their trade-offs

| Approach | How it works | Pros | Cons |
|---|---|---|---|
| **Blocking call (as proposed)** | One `tools/call` stays open until the server has something to return. | Simplest mental model; genuinely zero polling overhead if it worked as described. | Fights every timeout layer (SDK, harness, transport, intermediary); zero fault tolerance on the current transport (no resumability); Claude Code specifically does *not* protect a headless/subagent seat from silent abandonment; violates the spec's own "SHOULD enforce a maximum timeout" guidance if you try to defeat it with aggressive progress pings. |
| **Poll-with-short-timeout** (`wait_for_turn(seat_token, timeout_s≈20-25)`, called in a loop by the harness) | Ordinary bounded tool call; server returns "not yet, still waiting" or the new state, whichever comes first, within the bound. | Portable *today* — works on every client/transport/spec-revision combination surveyed, including stdio and both HTTP eras, no extension capability negotiation required; trivially fault-tolerant (a dropped connection just means the next poll starts fresh); each call is cheap and well inside every observed timeout (60s SDK default, Claude Code's 60s-to-first-byte and 2-minute backgrounding threshold). | Genuinely more requests on the wire; a naive implementation without server-side long-poll semantics wastes a full round-trip when nothing changed; needs a sane poll interval so a slow opponent doesn't generate a stream of empty polls. |
| **Long-poll with progress heartbeats** (hold the request up to N seconds, emit `notifications/progress` periodically, return early if the turn arrives) | Same tool call as the blocking design, but *bounded* — the server itself enforces a ceiling (e.g. 25–30s) and returns "still waiting" instead of relying on the client to eventually give up. | Cuts request volume vs. plain short-poll while staying inside every client-side timeout observed (well under Claude Code's 60s per-request timer and 2-minute backgrounding threshold, and under the reference SDK's 60s default); progress notifications give a live "still your opponent's turn" signal in supporting clients. | Still needs a poll/retry loop on top for the "still waiting" case — it reduces round-trips, it doesn't eliminate them; progress-resets-timeout behavior is inconsistent across client SDKs (§4), so you can't rely on it to justify holding *longer* than your own self-imposed ceiling. |
| **Resource subscriptions + `notifications/resources/updated`** (or `notifications/tasks`) | Client opens `subscriptions/listen` once, subscribes to a per-seat "to-act" resource URI (or task ID); server pushes when it changes. | True push, no polling once subscribed; multiplexes over one stream for many seats/games. | Requires the client to actually implement `subscriptions/listen` (or Tasks) — not confirmed for the mainstream hosts as of this survey (§5.1); inherits all the transport fault-tolerance caveats above (no resumability, no redelivery on the current revision); shoehorning turn-notification into "resource changed" is a workable but slightly off-label use of the primitive. |
| **Elicitation** | Not a fit — see §5.3. Retained here only to rule it out explicitly, since the ticket asked about it. | — | Wrong shape: synchronous, same-round-trip, and aimed at "I need a little more input from *this* caller," not "wait for a separate real-world event." |
| **Tasks extension** (`tasks/get` polling, or `notifications/tasks` push if the client supports `subscriptions/listen`) | `tools/call` (or any of the three MRTR-eligible methods) returns `CreateTaskResult`; harness polls `tasks/get` on its own schedule, or gets pushed via `subscriptions/listen` if both sides support it. | Purpose-built for exactly this ("Why not just block?", §5.1); durable across client crash/restart via the task ID; carries mid-flight `input_required` state cleanly. | Extension, not core — capability negotiation required both ways; **client-side support is not established** for the hosts most likely to run this repo's agents (absent from the official client matrix as of this survey); if the client never declares the `tasks` capability, the server must have a fallback (which ends up being poll-with-short-timeout anyway). |
| **Out-of-band wakeup outside MCP entirely** (webhook, queue, or the harness's own scheduler pings the agent process directly when it's their turn) | MCP tool calls stay short and stateless; turn-notification happens through a side channel the game server owns (e.g. it calls back into whatever invoked the agent). | Sidesteps every MCP timeout/transport constraint outlined above entirely, since it isn't MCP traffic; matches how Claude Code's own scheduled/webhook-driven agent patterns work in adjacent product surfaces. | Only works if the orchestrator around the agent (not the MCP server) can be woken by something outside MCP — for a plain "agent occupies a seat via an MCP client" deployment with no such orchestrator, this isn't available; adds an integration surface outside the otherwise-clean MCP boundary. |

**For Vale of Eternity specifically**, given the game is turn-based with potentially long
human-timescale gaps between an agent's turns (waiting on other players, human or agent), and given
this repo explicitly wants all-agent games as a first-class case (so you cannot assume a human is
present to babysit a stuck connection): the pragmatic recommendation is **poll-with-bounded-timeout
as the baseline**, with **long-poll heartbeats layered on top** to cut chattiness during genuinely
short waits (agent-vs-agent, sub-second-to-seconds turns), and **`subscriptions/listen` /
`notifications/tasks` as an opportunistic fast-path** wherever the connecting client happens to
support it — never as the only path, since support isn't guaranteed. Do not build the seat contract
around a single unbounded blocked call.

---

## 7. Prior art: servers holding real state across calls

Directly germane to this repo's design, because it recommends the exact shape a per-seat game
handle should take:

- [SEP-2567](https://modelcontextprotocol.io/seps/2567-sessionless-mcp) — the SEP that removed
  sessions — surveys **official, remote, currently-deployed MCP servers** that already use the
  explicit-handle pattern instead of relying on session scope:

  | Server | Create tool → returned ID | Operate tools taking that ID |
  |---|---|---|
  | [Linear](https://linear.app/docs/mcp) | `create_issue` → issue id | `get_issue`, `update_issue`, `create_comment` |
  | [Notion](https://developers.notion.com/docs/mcp) | `notion-create-pages` → page id | `notion-update-page`, `notion-move-pages` |
  | [GitHub](https://github.com/github/github-mcp-server#tools) | `create_pull_request` → PR number | `pull_request_read`, `update_pull_request`, `merge_pull_request` |
  | [Stripe](https://docs.stripe.com/mcp) | `create_customer` → customer id | `create_invoice`, `list_subscriptions` |

  This is precisely the "seat" shape: `create_game()` / `join_game(game_id)` → `game_id` +
  `seat_token`, then every subsequent tool call (`play_card`, `wait_for_turn`, `get_board_state`)
  takes that handle as an ordinary argument. None of these servers hold the durable state in the
  MCP session or connection — the state lives in the server's own database, addressed by the ID the
  model carries.

- The same SEP's own survey data ([§Anticipated objections → Breaking change](https://modelcontextprotocol.io/seps/2567-sessionless-mcp#anticipated-objections))
  found that **90% of a 1000-repo sample of open-source MCP servers made no application-level use
  of the session ID at all**, and only 2.5% used session-keyed *application* state — i.e., relying
  on session/connection scope for game state was already a minority pattern even before this
  revision made it impossible.

- The SEP explicitly documents the stdio special case that's most relevant to a
  process-per-agent-seat deployment: **"Stdio servers using process-lifetime state… are the most
  common stateful servers today"** and are not mechanically broken by the sessionless rewrite
  (there's still a real process backing each connection) — but the SEP explicitly recommends
  against relying on it even there, because "such servers SHOULD NOT rely on process-lifetime
  state and SHOULD migrate to explicit handles," precisely because process lifetime has the same
  undefined-scope problem sessions had, and a stdio-only design can't offer equivalent behavior if
  this repo ever wants a remote/HTTP-connected seat (e.g. a cloud-hosted agent joining a game the
  human host is running locally).

- The **Tasks extension** itself is the closest thing to sanctioned prior art for "long-running,
  resumable, poll-or-push work" — see §5.1 — and its own doc frames the whole feature as the
  learned answer to "people tried blocking and it didn't hold up," rather than a hypothetical.

- One more relevant, spec-referenced real-world failure mode: the footnoted report that **both
  ChatGPT's and Claude.ai's connectors were observed opening a fresh MCP session on every single
  tool call**, destroying any session-scoped state a server tried to keep
  ([microsoft/playwright-mcp#1045](https://github.com/microsoft/playwright-mcp/issues/1045),
  cited in [SEP-2567](https://modelcontextprotocol.io/seps/2567-sessionless-mcp#session-lifetime-is-undefined-and-servers-cant-design-around-it)).
  This is strong independent corroboration, from a widely-used real server (Playwright MCP), that
  "the connection/session will reliably stay alive long enough to carry my application state" was
  already a bad assumption against real hosts *before* the spec formally removed sessions — the
  same skepticism applies a fortiori to "the connection will reliably stay open long enough to
  carry a multi-minute-to-multi-hour blocked wait."

---

## What's still in flux — flag before acting on this

1. **The whole stateless/MRTR/Tasks-as-extension rewrite (`2026-07-28`) is very new** — it went
   Current only weeks before this survey, after a release-candidate period
   ([2026-07-28 Release Candidate post](https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/)).
   Secondary sources checked during this survey describe `2025-11-25` as "the version to target if
   shipping today," with `2026-07-28` framed as the release candidate — that framing is now stale
   per the spec site itself, but it tells you ecosystem *adoption* lags the *spec* by a wide margin.
   **Assume the concrete MCP client/host you build and test against (Claude Code, most third-party
   servers) speaks `2025-11-25` or older until you've verified otherwise.** That means: real
   session IDs and stream resumability may actually be available to you in practice even though the
   latest spec removed them — but design against the new revision's constraints anyway, since
   anything you build today needs a migration path once hosts catch up, and the new revision is
   strictly more restrictive (no sessions, no resumability) than the old one.
2. **Tasks extension client support is unconfirmed for the hosts this repo cares about.** Absent
   from the official [client matrix](https://modelcontextprotocol.io/extensions/client-matrix) as
   of this survey; treat any Tasks-based design as needing a fallback path, not as the guaranteed
   mechanism.
3. **SEP-1539 (Timeout Coordination)** — a proposal for servers to *advertise* their expected
   per-capability timeouts to clients, directly addressing the "clients guess and guess wrong"
   problem documented in §4 — is **dormant, not merged**
   ([modelcontextprotocol/modelcontextprotocol#1539](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1539)).
   If it lands later, it would meaningfully change the calculus for long-poll durations (a server
   could legitimately tell clients "expect up to 30s for this tool"), but don't design against it
   existing yet.
4. **Progress-resets-timeout behavior is inconsistent across client SDKs today**, and was only
   recently reconciled between the TypeScript and Python reference SDKs
   ([typescript-sdk#245](https://github.com/modelcontextprotocol/typescript-sdk/issues/245)). Don't
   assume any given third-party client honors it correctly even where the spec allows it.
5. **Claude Code's auto-backgrounding thresholds (2 min) and idle timeouts are product behavior,
   not spec text**, and are versioned/gated behind specific Claude Code releases
   (`v2.1.212+`, `v2.1.203+`, `v2.1.187+` for different pieces of this). They will keep changing
   independent of the MCP spec; re-check
   [code.claude.com/docs/en/mcp](https://code.claude.com/docs/en/mcp) before relying on exact
   numbers.
6. **URL-mode elicitation** is itself flagged in-spec as new and unstable: "Its design and
   implementation may change in future protocol revisions"
   ([Elicitation §URL Mode Elicitation Requests](https://modelcontextprotocol.io/specification/2026-07-28/client/elicitation#url-mode-elicitation-requests)).

## Update — 2026-08-24

- Claude Code now defaults (v2.1.232+) to a "v2" client runtime built on MCP TypeScript SDK 2.0,
  which auto-negotiates protocol revision `2026-07-28` with HTTP and claude.ai-connector servers
  (stdio servers stay on the older handshake unless `MCP_PROTOCOL_NEGOTIATION=auto` is set). The
  guidance to assume this repo's concrete host speaks `2025-11-25` or older no longer holds by
  default for remote servers — verify which runtime a given session is on before relying on either
  assumption ([code.claude.com/docs/en/mcp](https://code.claude.com/docs/en/mcp)).
