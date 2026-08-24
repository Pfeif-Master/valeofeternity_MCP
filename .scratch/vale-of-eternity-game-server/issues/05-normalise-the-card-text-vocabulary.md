<!-- was GitHub issue #6 (Pfeif-Master/valeofeternity_MCP) -->

# Normalise the card text vocabulary

Type: prototype
Blocked by: 01, 10

## Question

The 70 cards carry effect text in a half-formed bracket language. Some tokens are semantic and
machine-readable — `[points:4]`, `[family:wind]`, `[stone:1]`. Others are literal descriptions of
printed icons that nothing can execute — `[lightning bolt]` (6 uses), `[hourglass icon]` (5),
`[infinity symbol]` (3).

A third group is semantic but overloaded: `[family:X:N]` reads like a family and is actually a
magic stone (Red 1 / Blue 3 / Purple 6), because the extractor read the coloured stone icons as
family icons. `[family:X]` means a real family only when it modifies "card". Eight
`[glyph:immediate|end_of_round|passive]` tokens are already semantic and already grounded — they
name the rulebook's Instant / Permanent / Active timings.

Decide the token language the card text compiles into.

Keep the language **engine-agnostic**: it should describe what is *printed on the card*, not how
the engine executes it. That decoupling is what lets *What the rules imply for the core state and
effect model* be decided independently, instead of the two designs deadlocking on each other.

Resolved as a prototype — settle the vocabulary against a handful of the gnarliest cards and react
to it. The bulk rewrite of all 70 entries follows afterward as execution, not as part of this
ticket.

Blocked by the rules reference, because the remaining icon tokens cannot be named without knowing
what they mean.

## Comments

**Pfeif-Master:**
stones don't have family. the color just confused the scraping agent. they come in the form of 1, 3, 6

**Pfeif-Master:**
Context from *Pull the base-game rules into reusable context* (#2), verified directly against `cards/index.json` — this ticket does not need to re-derive it:

**`[family:X:N]` tokens are magic stones, not families.** The card data encodes stones three different ways:

| Encoding | Count | Meaning |
|---|---|---|
| `[stone:1]` / `[stone:3]` / `[stone:6]` | 3 (all on *Willow*) | correct |
| `[family:fire:1]` / `[family:water:3]` / `[family:dragon:6]` | 7 / 12 / 6 | Red / Blue / Purple stone |
| `[family:fire] [points:1]` / `[family:water] [points:3]` / `[family:dragon] [points:6]` | 3 / 1 / 1 | the same three stones, split in two |

The values match the rulebook's denominations exactly (Red 1, Blue 3, Purple 6), and there are **no `earth` or `wind` compound tokens** — there are only three stone colours. The extractor read the coloured stone icons as family icons.

**Disambiguation rule:** `[family:X]` is a genuine family reference when followed by "card" (or in a taming/summoning context); it denotes a stone when it carries a value — compounded, or paired with an adjacent `[points:N]`.

Also for this ticket: **`end_of_round` is a misnomer.** The rulebook's three types are Instant / Permanent / Active; `immediate`=Instant, `passive`=Permanent, `end_of_round`=**Active** — resolved in the Resolution Phase, a distinct turn-ordered phase before the round-end check, in player-chosen order, and mandatory.

Full token inventory (24 distinct kinds) and the rules reference are on branch `research/base-game-rules` in `docs/rules/`.
