<!-- was GitHub issue #1 (Pfeif-Master/valeofeternity_MCP), label wayfinder:map -->

# Vale of Eternity game server

## Destination

A Vale of Eternity game server playable end to end: 2–4 seats, any seat drivable by an
AI agent over MCP or by a human through a UI the maintainer is happy with, with the repo
in a state he can share with friends.

## Notes

**Domain** — Vale of Eternity, base game only: 70 cards across five families
(Fire/Water/Earth/Wind/Dragon, 15/15/15/15/10). Multiple rounds of snake drafting from a
shared pool. Each drafted card is a fork — play it into a point engine, or sell it for the
currency that plays other cards. The strategy is emergent: what to draft, and what to play
versus sell.

**Settled — do not reopen without cause**

- Core is a **server**. The MCP service and the Electron app attach to it as clients.
- Core is **seat-agnostic** — human or agent is a controller detail the core does not model.
- An **all-agent game is a valid use case**, not a test mode.
- Core is **authoritative**: it rejects invalid actions from players.
- **Base game only.** Expansions must be addable later without re-architecture.
- **Sharing posture** — "presentable" means a README and code presentation the maintainer is
  happy with. The submodule already splits card *data* from card *art*: `cards/index.json` is
  tracked, the 70 PNGs are gitignored and pulled locally by `scripts/pull_cards.py`. The
  project will not be widely advertised; if the art ever becomes a problem, the submodule can
  be shed while the server logic survives. The tracked card data doubles as a gift to creators
  wanting machine-readable cards.

**Rules reference** — the source rulebook is copyrighted and deliberately untracked at
`valeofeternity_cardDB/docs/VoE_Rulebook_WEB.pdf`, with a full text extraction beside it at
`VoE_Rulebook_TEXT.txt`. Commit neither. The design reference distilled from them lands at
`docs/rules/base-game.md` and is owned by *Write the base-game rules into a design reference*;
until that closes, the rules facts on this map's *Decisions so far* line are the record. An
earlier attempt sits unmerged on branch `research/base-game-rules` — its findings hold and are
carried forward, its form does not. Do not merge it.

**Working agreements** — Wayfinder's plan-don't-do default is **on**: tickets resolve
decisions; the build happens once the way is clear. Every session invokes `/grilling` and
`/domain-modeling`; `/research` for facts outside the working directory, `/prototype` where
"how should it look or behave" is the question.

**Side quests** — optional exploration lives on the `side-quest` label, deliberately *not* as
children of this map. Nothing on the route waits for them, and keeping them off the map keeps
the frontier honest. Find them among the tickets in `issues/` that carry `Type: side-quest`.

## Decisions so far

<!-- one line per closed ticket -->

- [Pull the base-game rules into reusable context](issues/01-pull-the-base-game-rules-into-reusable-context.md) — three turn-ordered phases (Hunting / Action / Resolution), nothing simultaneous; the only player-count variance is pool size (`2×N`) and starting score by seat rank; exactly one randomness source (shuffle + reshuffle), so replay is cheap to make reproducible; `end_of_round` is a misnomer for the rulebook's *Active* timing; `[family:X:N]` tokens are magic stones, not families.
- [Survey MCP primitives and patterns for stateful servers](issues/07-survey-mcp-primitives-and-patterns-for-stateful-servers.md) — the blocking-call "hibernate inside a pending tool call" pattern does **not** hold up (spec-mandated max timeouts, Claude Code auto-backgrounds at ~2 min, `2026-07-28` removed sessions and stream resumability); use explicit state handles plus bounded poll/long-poll, with push as an opportunistic fast-path.

## Not yet specified

<!-- Cleared 2026-08-10 by 01 and 07: effect-timing semantics (now documented in
     docs/rules/, with the engine design owned by 02), player-count variance (answered
     outright — only pool size and starting score vary), and agent turn timing (now
     inside 06's scope, given 07's findings). -->

- The human UI
- Persistence, save, resume
- Transport protocol between core and clients
- Agents overfitting a fixed 70-card pool

## Out of scope

- The evaluation harness itself — batch match running, metrics, model comparison. Whether the
  engine must nonetheless be built instrumentable is decided by *What the rules imply for the
  core state and effect model*.
- Artifacts and Curse expansion modules
- Agent-as-advisor: a read-only observer attached to a human seat, suggesting plays
