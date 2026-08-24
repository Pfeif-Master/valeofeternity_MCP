<!-- was GitHub issue #8 (Pfeif-Master/valeofeternity_MCP) -->

# Survey MCP primitives and patterns for stateful servers

Type: research
Status: resolved

## Question

Agent-only homework feeding the MCP patterns discussion. No human needed.

Survey the current MCP specification and report:

- The full set of primitives the protocol offers, and what each is actually for — tools,
  resources, prompts, sampling, elicitation, notifications, and anything else current
- Which transports exist and what each implies for a long-lived server holding state
- How a stateful, multi-client, turn-based server is expected to work over MCP: session and
  connection lifetime, whether the server can push to a client, how subscriptions or change
  notifications work
- Error and rejection conventions — how a server tells a client "that was invalid" in a way the
  client can act on
- Prior art: existing MCP servers that hold real state across calls rather than answering
  stateless queries, and what patterns they landed on

Note the spec version surveyed, and flag anything that looks like it is still moving — the
discussion this feeds may happen weeks later.

Resolved by a `/research` subagent. Output is a document committed to the repo, not a comment.

## Answer

Spec surveyed: **MCP `2026-07-28`** (current), cross-checked against `2025-11-25` since deployed client/server support lags the newest revision.

**Output — branch `research/mcp-stateful-patterns` (not merged):** `docs/research/mcp-stateful-patterns.md`

### Verdict on the blocking-call design

The maintainer's proposal — *an agent takes its turn with one MCP tool call that blocks until the game needs that seat again, hibernating inside the pending call* — **works only with heavy caveats and is not safe as the primary mechanism.**

- The spec does not forbid an unbounded call, but explicitly tells implementations to **enforce a maximum timeout regardless of progress notifications**. Progress-resets-timeout is a MAY, not a MUST, and is inconsistently implemented across SDKs.
- **Claude Code specifically** auto-backgrounds a main-conversation MCP call after ~2 minutes — and notably does *not* do so for subagent calls or non-interactive/headless runs. "A scripted agent occupying a seat" is exactly the case where a long blocked call can be silently abandoned when the run ends.
- The current Streamable HTTP transport (`2026-07-28`) **removed sessions and stream resumability entirely** (SEP-2567). A dropped connection during a long wait is unrecoverable — the client must re-issue as a brand-new request.
- MCP's own answer to this problem is the **Tasks extension**, whose documentation literally has a section titled *"Why not just block?"* listing these same failure modes. But it is not yet in the official client-support matrix, so it cannot be relied on alone.
- **Sampling** is the wrong direction (server borrows the client's LLM) and is now deprecated. **Elicitation** is a same-round-trip mechanism, not a durable wait.

**Recommended instead:** explicit state handles (`game_id` / `seat_token`) threaded through every call, plus bounded poll-with-timeout or long-poll with progress heartbeats as the portable baseline — with `subscriptions/listen` or Tasks as an opportunistic push fast-path where the connecting client supports it, never as the only path.

**Prior art agrees:** real deployed stateful servers (Linear, Notion, GitHub, Stripe) already use explicit state handles — create → id → thread the id through subsequent calls — rather than session or connection state. SEP-2567, which removed protocol sessions, cites this pattern directly.

### What this does *not* kill

The valuable half of the idea survives, and the rules research (#2) sharpens it. Vale of Eternity is **strictly sequential** — no phase is simultaneous — so exactly one seat holds the decision at any instant. But that seat is **not always the seat whose turn it is**: an Instant effect can force an opponent to choose which of their cards to discard, mid-resolution.

So the right shape is not "block until your turn" but a **seat-as-coroutine** call whose return value carries *the next decision required of this seat*, whatever kind — a draft pick, an action-phase choice, Resolution-phase effect ordering, or a forced reactive choice during someone else's turn. What the research changes is the **transport** for that shape: the wait must be bounded and resumable via a state handle rather than held open indefinitely inside one pending call.

This is the input to *MCP patterns for a stateful turn-based server* (#7), which is where the decision gets made.

### Flagged as in flux

The `2026-07-28` revision is recent and adoption lags it; the Tasks extension's client support is still moving; SDK timeout behaviour varies. Re-check the client matrix and Claude Code's MCP docs before committing to exact numbers.

## Comments

**Pfeif-Master:**
Resolved — see the resolution comment above. Findings on branch `research/mcp-stateful-patterns` (`docs/research/mcp-stateful-patterns.md`). Feeds #7.
