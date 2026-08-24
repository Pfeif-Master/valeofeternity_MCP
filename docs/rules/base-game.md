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
  Each card has a name, family, summoning cost, and one effect with a type (see
  Effects below).
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
- Exceeding the cap forces an immediate discard down to 4, chosen by the owner, before
  any other action.
- Stones are public information at all times. No change on overpay; no exchange between
  denominations except by a specific card effect.

## Effects: types and resolution

| Rulebook name | card-data `type` | Fires |
|---|---|---|
| Instant | `immediate` | once, at the moment the card is summoned — part of the summon action itself |
| Permanent | `passive` | continuously, for as long as the card stays in the area |
| Active | `end_of_round` | once per round, during the Resolution Phase, activated by the owner |

`end_of_round` is a misnomer worth flagging: Active effects fire during the Resolution
Phase, which precedes the end-of-round check — not literally at the end of the round.

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

(subject to change as cards db is not finalized)

Card text in `cards/index.json` uses bracket tokens:

- `[points:N]` — fixed point award or loss of N. Bare `[points]`, `[points:?]`, and
  `[points:UNKNOWN]` all mean a variable amount computed at resolution (e.g. "equal to
  its cost") — three spellings of the same thing, an inconsistency in the data rather
  than a rules distinction.
- `[stone:1]` / `[stone:3]` / `[stone:6]` — a Red / Blue / Purple stone.
- `[family:fire]` / `water` / `earth` / `wind` / `dragon` — a genuine family reference,
  used when it modifies "card" (targeting or counting cards of that family).
- `[family:fire:1]`, `[family:water:3]`, `[family:dragon:6]` — **not families**, despite
  the token shape: these are stones (Red / Blue / Purple respectively). The source data
  extractor read the colored stone icon as a family icon. Occasionally split into an
  adjacent pair instead — `[family:fire] [points:1]` means the same thing.
- `[glyph:immediate]` / `[glyph:end_of_round]` / `[glyph:passive]`, equivalently
  `[lightning bolt]` / `[hourglass icon]` / `[infinity symbol]` — mark which
  effect-type clause of a multi-clause card the following text belongs to. Two
  tokenizations of the same three icons occur across different cards.

## Visibility

Hidden: hand contents, and only from everyone but the owner. Everything else — areas,
stone counts and denominations, discard pile, scores, pending marker claims, and the
cards currently revealed for drafting — is public. The undrawn draw pile's contents are
unknown to everyone.

## Randomness

Exactly two sources: the initial full-deck shuffle at setup, and reshuffling the
discard pile into a new draw pile whenever the draw pile empties during a Hunting Phase
reveal. First-player selection is an external input, not in-engine randomness. For
reproducible replay, seed once at setup — reshuffle events depend on play (how many
cards get bought and discarded), not just the seed, so a replay must feed decisions back
in lockstep with RNG draws.

## End of game and scoring

Checked once, right after the Resolution Phase, before a new round would start: the
game ends if any player's score is ≥ 60, or round 10 has just completed. Winner is
highest score; ties break first by most summoned cards (area size), then share the
victory if still tied. Score never drops below 0.

## Open questions for the engine

- Several card effects say "draw a card" (Griffon, Sylph, Pegasus, Hippogriff,
  Dandelion Spirit, Willow, Hydra) — moving a card from the draw pile straight to hand.
  The rulebook never names this as a distinct action, and doesn't say whether it
  reshuffles-on-empty the same way a Hunting Phase reveal does.
    - yes reshuffle on empty
- Stacking/associativity for non-additive Permanent effects (e.g. two different "value
  of X counts as Y" or "value increased by" modifiers on the same stone) isn't
  addressed beyond the additive cost-reduction case.
    - permanents are considered in effect. example value of 3 and 6 stones swapping effect cards that score those stone by value.
        - the blue stone is now value 6 and the purple stone is now value 3. scoring total value of purple(6) stones is now scoring total value of purple(3) stones.
- Whether discarded excess stones (over the 4-stone cap) and sold or removed cards
  return to a shared supply or are simply gone: no card effect reads this, so it's an
  implementation choice, not a rules gap.
    - returned to supply. assume unlimited number of stones
