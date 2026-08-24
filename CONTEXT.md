# The Vale of Eternity

A 70-card drafting and point-engine game for 2–4 seats. Each round, seats draft cards
via a snake draft, resolve their claims by selling for stones or taming to hand, spend
stones to summon cards into a scoring area, and activate that area's once-per-round
effects — racing to 60 points or round 10.

## Language

**Seat**:
A player slot in game state — one of 2 to 4 — independent of whether a human or an AI
agent drives it.
_Avoid_: player

**Controller**:
Whatever drives a Seat's decisions — a human or an AI agent. A detail the core game
state doesn't model.
_Avoid_: player, agent (as a stand-in for the whole seat)

**Hand**:
The set of cards a player has taken into their possession, via Tame or a card effect,
but not yet summoned. Secret from every other player; no size limit.

**Area**:
The zone where a player's summoned cards are displayed, face-up and public. Capacity
equals the current round number.
_Avoid_: board, tableau, field

**Claim**:
A drafted card a player has placed a marker on during the Hunting Phase, not yet
resolved. Resolved by Sell or Tame; a player cannot end their Action Phase turn with an
unresolved claim.
_Avoid_: marker (the physical component representing a claim; not a separate concept)

**Family**:
One of the five categories printed on every card — Fire, Water, Earth, Wind, or Dragon —
used for family-scoped counting and targeting effects. An extraction bug had scattered
Stone references across `[family:X]`/`[family:X:N]` bracket tokens throughout the card
data (not just the compound shape — bare tokens too, wherever context showed a stone
rather than a card); normalized as of the card-vocabulary QA pass. See Stone.
_Avoid_: type (ambiguous with Effect Type)

**Effect**:
One printed instant/permanent/active clause on a card — a type paired with its text.
Card data models a card's effects as an ordered, non-empty list; today every card has
exactly one, except 8 cards with two (always pairing Instant with one other type).
_Avoid_: action, ability

**Stone**:
A unit of currency in one of three denominations — Red, Blue, Purple — each carrying a
value (1/3/6 by default) that a Permanent effect can override; other effects reading a
denomination's "total value" read the current, possibly-overridden value, not the
printed one. Identity is the color; card data's `[stone:red]`/`[stone:blue]`/
`[stone:purple]` tokens name it that way, never by value.
_Avoid_: currency

**Hunting Phase**:
The round's first phase: reveals `2 × seat count` cards and runs the Snake Draft,
letting every seat place both of its markers.

**Snake Draft**:
The two-pass claiming pattern used by the Hunting Phase — pick pass one runs seat order
forward, pick pass two reverses, so the seat that picks last in pass one picks first (and
twice in a row) in pass two.
_Avoid_: draft (ambiguous between the whole Hunting Phase and just this pattern)

**Action Phase**:
The round's second phase: each seat, in turn, performs any number of Sell, Tame,
Summon, and Remove actions, then passes.

**Resolution Phase**:
The round's third phase: each seat activates every available Active effect on their
summoned cards exactly once, then passes.

**Sell**:
The action of discarding a Claim to gain Stones by its card's Family. Applies only to
Claims — not Hand or Area cards.

**Tame**:
The action of moving a Claim into Hand. Applies only to Claims — not Hand or Area cards.
_Avoid_: draft (names the Hunting Phase mechanic that produces the claim, not this action)

**Summon**:
The action of paying Stones to move a card from Hand into Area, resolving its Instant
effect if it has one.
_Avoid_: play, cast

**Remove**:
The action of discarding a card from your own Area, paying Stones equal to the current
round number, to free an Area slot.

**Discard**:
The general mechanic of moving a card out of Hand or Area into the discard pile.
Triggered by Sell, Remove, or specific card effects — those define what additionally
happens (stones gained, cost paid); Discard itself is just the move.

**Instant**:
An effect type that resolves once, at the moment its card is Summoned — a precondition
of the summon succeeding, not just a consequence of it. Card data's `effects[].type`
uses this name directly.
_Avoid_: immediate (card data's name for this before the schema normalization)

**Permanent**:
An effect type that stays continuously active for as long as its card remains in the
Area. Card data's `effects[].type` uses this name directly.
_Avoid_: passive (card data's name for this before the schema normalization)

**Active**:
An effect type that fires once per round, during the Resolution Phase, in an order its
owner chooses among their own Active effects. Card data's `effects[].type` uses this
name directly.
_Avoid_: end_of_round (card data's name for this before the schema normalization — a
misnomer, since Active effects resolve during the Resolution Phase, before the round
actually ends)

**Recover**:
The keyword that moves a summoned card from Area back to Hand; recovers itself if no
other target is named.
