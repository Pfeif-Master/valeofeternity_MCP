# The Vale of Eternity — Base Game Rules

Restated from the official rulebook (Renegade Game Studios edition; design by Eric Hong,
© 2024 Mandoo Games) and the physical game board, in the author's own words. Written to
serve two readers at once: an agent that needs to know how to play, and an engine
implementer who needs to know what to build. The source PDF and its text extraction are
copyrighted and deliberately untracked (`valeofeternity_cardDB/docs/`); this document is
the durable reference.

## Objective

Score the most points by drafting, taming, and summoning creatures over up to 10 rounds.
The game ends at the first end-of-round check where any player's score is ≥ 60, or
after round 10 completes — whichever comes first.

## Components

- **70 cards**, five families: Fire (15), Water (15), Earth (15), Wind (15), Dragon (10).
  Each card has a name, family, summoning cost, and one or more effects, each with a
  type (see Effects below); all but 8 cards have exactly one.
- **42 magic stones**, three denominations: Red = 1 (×16), Blue = 3 (×14), Purple = 6
  (×12). The supply is a soft cap only — if it runs out, substitute any other item.
- Per player: a **hand** (secret), an **area** (public; capacity equals the current
  round number), a **score** (public; floors at 0), and 2 player markers.

## Setup

1. Each player picks a color, takes 2 markers and 1 player tile.
2. First player is chosen out-of-band (a real-world tiebreaker in the physical game —
   an engine takes this as an external input, not something it simulates).
3. Shuffle all 70 cards into a draw pile.
4. Starting score = seating rank from the first player, clockwise: 1st player 1pt, 2nd
   2pt, 3rd 3pt, 4th 4pt. Not a score of zero — easy to miss when implementing setup.
5. Round marker to round 1.

## Round structure

Each round: **Hunting Phase → Action Phase → Resolution Phase → end check**. All three
phases are turn-ordered, never simultaneous, starting from the current first player and
proceeding clockwise — except the Hunting Phase's second pick pass, which reverses (see
below). First player rotates clockwise (to the previous first player's left) between
rounds; it never changes mid-round.

### 1. Hunting Phase — snake draft

1. Reveal `2 × player count` cards from the draw pile, sorted into per-family zones. If
   the draw pile is empty, reshuffle the discard pile into a new draw pile first.
2. **Pick pass 1**: first player, then clockwise — each player places one marker on an
   unclaimed revealed card.
3. **Pick pass 2**: last player, then counterclockwise — each places their remaining
   marker on an unclaimed card. The last player thus picks twice in a row, which is
   what makes it a snake draft. The phase ends once the first player places their
   marker on the final card.

`2N` cards revealed, `2N` marker placements: every revealed card ends up claimed, none
left over. A marker placement is a claim only, not a resolution — the claimed card is
sold or taken to hand during the Action Phase that follows.

### 2. Action Phase

First player, then clockwise — each player takes one turn: any number of actions, in
any order, repeats allowed, then passes. A player cannot end their turn while a marker
they placed during the Hunting Phase is still on a card — it must be retrieved by
selling or taming that card, in the same turn. Four actions, each usable any number of
times:

- **Sell a card** — retrieve a marker, discard the matching card, gain stones by the
  card's *family* (not its cost) — see Magic stones below.
- **Tame a card** — retrieve a marker, take the matching card into hand.
- **Summon a card** — from hand, only if the area isn't full: pay stones ≥ the card's
  (reduced) cost, no change given; place it face-up in the area; resolve its Instant
  effect, if any (see Effects below).
- **Remove a card** — discard one of your own summoned cards, paying stones equal to
  the current round number. Frees an area slot.

### 3. Resolution Phase

First player, then clockwise — each player activates every Active effect available
among their summoned cards, exactly once each, choosing the order if they have more
than one, then passes. Every available Active effect must be used; none are optional.

### End of round

Run the end check (see End of game below). If not met: advance the round marker, first
player passes clockwise, next round begins.

## Magic stones

- **Cap**: 4 stones per player, counted by quantity, not value. Sell values, confirmed
  against the physical board:

  | Family | Stones received | Value |
  |---|---|---|
  | Fire | 3×Red | 3 |
  | Earth | 4×Red | 4 |
  | Water | 1×Blue | 3 |
  | Wind | 1×Red + 1×Blue | 4 |
  | Dragon | 1×Purple | 6 |

  Denomination matters as much as value: Fire and Water are both worth 3, but selling
  Fire fills 3 of the 4 slots while Water fills 1. Model stones as a multiset of
  denominations, never a single integer.
- **Value is a property of the denomination, not a fixed constant.** A Permanent effect
  can override it — Hae-tae makes Blue's value count as Purple's and Purple's count as
  Blue's, so after it resolves Blue reads as 6 and Purple reads as 3. Any other effect
  that reads "total value of your X stones" (e.g. Rock Golem) uses the current,
  possibly-overridden value, never the printed default.
- Exceeding the cap forces an immediate discard down to 4, chosen by the owner, before
  any other action.
- Stones are public information at all times. No change on overpay; no exchange between
  denominations except by a specific card effect.
- **Supply is unlimited for engine purposes.** The physical 42-stone count is a
  component limit, not a gameplay one — the rulebook says to substitute any other item
  if it runs out. Discarded excess stones (from exceeding the cap) return to it.

## Effects: types and resolution

A card's `effects` field is an ordered, non-empty list of `{type, text}` — usually one
entry, occasionally two (an Instant clause plus one other). Each `type` is one of:

| Type | Fires |
|---|---|
| `instant` | once, at the moment the card is summoned — part of the summon action itself |
| `permanent` | continuously, for as long as the card stays in the area |
| `active` | once per round, during the Resolution Phase, activated by the owner (not literally at the end of the round, despite the name) |

Resolution rules that apply across all three types:

- **Card text overrides these general rules** wherever the two conflict.
- **Recover** = move a summoned card from the area back to hand; it recovers itself if
  no other target is named.
- **Every available effect must be used.** None are optional (e.g. Medusa's Active
  effect is mandatory unless its hand-discard cost genuinely can't be paid).
- **A failed trigger cancels its payoff**: if the cost/trigger half of an effect can't
  be resolved (nothing to discard, no qualifying target), the payoff half doesn't
  happen either.
- **Instant effects are a precondition of summoning**, not just a consequence: a card
  cannot be summoned at all if any part of its Instant effect can't legally resolve
  (e.g. Ember requires some player, possibly the summoner, with a Water card in their
  area to discard).
- While its own Instant effect resolves, a card counts as already in the area and no
  longer in hand — self-referential counts (Eternity's "each card family in your area")
  include the card itself. A Permanent effect that would modify summoning does not
  apply to the summoning of the card that carries it (Kappa doesn't trigger off its own
  summon).
- Cost-reduction Permanent effects stack additively; cost floors at zero.
- Effects can target other players: a Dragon-family Instant may force a chosen player
  to discard one of their own cards of a named family — the acting player picks *who*,
  the targeted player then picks *which* of their own qualifying cards to lose. No
  response or reaction window exists anywhere; effects resolve as a single atomic step
  at the moment they trigger.

## Card text vocabulary

Effect text in `cards/index.json` uses bracket tokens:

- `[points:N]` — fixed point award or loss of N. Bare `[points]`, `[points:?]`, and
  `[points:UNKNOWN]` all mean a variable amount computed at resolution (e.g. "equal to
  its cost") — three spellings of the same thing, an inconsistency in the data rather
  than a rules distinction.
- `[stone:red]` / `[stone:blue]` / `[stone:purple]` — a stone of that color. Named by
  color, never by value, since a Permanent effect can change what a color's value reads
  as (see Magic stones above) without changing which stone it is.
- `[family:fire]` / `water` / `earth` / `wind` / `dragon` — a genuine family reference,
  used when it modifies "card" (targeting or counting cards of that family).
- `[glyph:instant]` / `[glyph:permanent]` / `[glyph:active]` — refers to another card's
  effect type, for cards whose own effect targets or filters by it (e.g. "recover a card
  with `[glyph:instant]`"). A card's *own* effect type is the `type` on its own entry in
  `effects`, never an inline token in its own text.

## Visibility

Hidden: hand contents, and only from everyone but the owner. Everything else — areas,
stone counts and denominations, discard pile, scores, pending marker claims, and the
cards currently revealed for drafting — is public. The undrawn draw pile's contents are
unknown to everyone.

## Randomness

Exactly two sources: the initial full-deck shuffle at setup, and reshuffling the
discard pile into a new draw pile whenever the draw pile empties — during a Hunting
Phase reveal, or when a card's "draw a card" effect resolves (Griffon, Sylph, Pegasus,
Hippogriff, Dandelion Spirit, Willow, Hydra — takes the top card of the draw pile
straight into hand; same reshuffle-on-empty rule applies). First-player selection is an
external input, not in-engine randomness. For reproducible replay, seed once at setup —
reshuffle events depend on play (how many cards get bought and discarded, and how many
draw effects fire), not just the seed, so a replay must feed decisions back in lockstep
with RNG draws.

## End of game and scoring

Checked once, right after the Resolution Phase, before a new round would start: the
game ends if any player's score is ≥ 60, or round 10 has just completed. Winner is
highest score; ties break first by most summoned cards (area size), then share the
victory if still tied. Score never drops below 0.
