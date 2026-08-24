<!-- was GitHub issue #7 (Pfeif-Master/valeofeternity_MCP) -->

# MCP patterns for a stateful turn-based server

Type: grilling
Blocked by: 07

## Question

How is this game modelled over MCP?

- Which MCP primitives carry what — tools, resources, prompts?
- How does an agent bind to a seat in a running game?
- How does an agent learn its turn has come — polling, or a server-pushed notification?
- What do rejections look like, so an agent can recover rather than flail?

Comes into the session with the homework already done — see the MCP primitives survey — so the
discussion starts from what the protocol actually offers rather than from memory.

**Sharpened by the survey (#8):** indefinite tool-call blocking is out — implementations are
told to enforce a maximum timeout regardless of progress notifications, Claude Code
auto-backgrounds main-conversation calls at ~2 minutes, and Streamable HTTP `2026-07-28`
removed sessions and stream resumability. So the live question is not *whether* a seat can
hibernate inside a pending call, but what replaces it: explicit `game_id`/`seat_token` handles
with a bounded wait, and whether the wait is a short poll, a long poll with progress
heartbeats, or an opportunistic push where the client supports it.

Note the shape the rules force: play is strictly sequential, so exactly one seat holds the
decision at any instant — but that seat is not always the seat whose turn it is, because an
Instant effect can require an opponent to choose which of their cards to discard mid-resolution.
So whatever a seat waits on should deliver *the next decision required of it*, of any kind, not
"your turn has started".

Unblocked from #3 and #5 on 2026-08-23: the wake-and-session question turns on transport
lifecycle, which #8 answered, plus two facts the rules already settle. The primitive mapping
may need revisiting if the action model lands somewhere exotic.

## Comments

**Pfeif-Master:**
Research input ready: *Survey MCP primitives and patterns for stateful servers* (#8) is resolved — see its resolution comment, and `docs/research/mcp-stateful-patterns.md` on branch `research/mcp-stateful-patterns`.

Headline for this discussion: the **blocking-call hibernation pattern does not hold up** as the primary mechanism (spec-mandated max timeouts, Claude Code auto-backgrounds main-conversation calls at ~2 min and does *not* for headless/subagent runs, and Streamable HTTP `2026-07-28` removed sessions and stream resumability outright). The recommended shape is explicit state handles (`game_id`/`seat_token`) plus bounded poll or long-poll with heartbeats, with push as an opportunistic fast-path.

The *seat-as-coroutine* half of the idea survives and is sharpened by the rules research (#2): the game is strictly sequential, so exactly one seat holds the decision at any instant — but that seat is **not always the seat whose turn it is**, because an Instant effect can force an opponent to choose which of their cards to discard mid-resolution. So the call should return *the next decision required of this seat*, of whatever kind, rather than "it is your turn".
