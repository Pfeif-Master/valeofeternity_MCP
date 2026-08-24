<!-- was GitHub issue #2 (Pfeif-Master/valeofeternity_MCP) -->

# Pull the base-game rules into reusable context

Type: research
Status: resolved

## Question

The whole map waits on this. We have card data (name, family, cost, type, text for 70 cards)
and no rules.

Produce a reusable in-repo rules document covering:

- Player counts, and what varies by count (setup, pool size, round count)
- The round/phase sequence — name each phase, and state whether it is simultaneous or
  turn-ordered, and how turn order is determined and changes
- The snake-draft acquisition mechanism, exactly
- The stone economy — denominations, how earned, how spent, limits, any conversion
- Summoning: how a card goes from hand to play area, its cost, and any area or per-turn limits
- Precise definitions of the three effect timings present in the card data — `immediate` (41
  cards), `end_of_round` (22), `passive` (7) — and exactly when each resolves
- Whether effects target or harm other players, and whether any reaction/priority window exists
  or effects resolve atomically
- What information is hidden from whom, at each point — this determines what a per-seat view
  must redact
- Every source of randomness and when it is consumed — this determines what must be seeded for
  reproducible replay
- The end trigger and every scoring source
- Known errata, official FAQ, and community-disputed rulings an engine would have to adjudicate

**No local rulebook exists.** The submodule's `.gitignore` reserves `docs/artifacts_rulebook.pdf`
and its README advertises `docs/recon_report.md`, but there is no `docs/` directory at all. This
needs the network.

Resolved by a `/research` subagent. Output is a document committed to the repo, not a comment.

## Answer

Rules pulled from the **official English rulebook** (Renegade Game Studios edition; designed by Eric Hong, © 2024 Mandoo Games), 12 pages. The PDF is copyrighted, so it is **not committed** — it lives untracked at `valeofeternity_cardDB/docs/VoE_Rulebook_WEB.pdf` (the submodule's `.gitignore` now covers `docs/*.pdf` and `docs/*.txt`) and can be re-pulled with:

```
curl -sSL -o valeofeternity_cardDB/docs/VoE_Rulebook_WEB.pdf \
  "https://renegadegamestudios.com/content/File%20Storage%20for%20site/Rulebooks/Vale%20of%20Eternity/VoE_Rulebook_WEB.pdf"
```

The tracked output is a **restatement as an engineering spec**, not a reproduction, with `[p.N]` citations throughout.

### Output — branch `research/base-game-rules` (not merged)

- `docs/rules/base-game.md` — prose rules reference, organised by this ticket's question list
- `docs/rules/rules.json` — machine-readable game facts (player-count table, phases, denominations, caps, end trigger, scoring)
- `docs/rules/open-questions.md` — 9 rulebook ambiguities + 6 community-sourced items

Note: the research agent initially committed these into the `valeofeternity_cardDB` submodule. They have been relocated to the main repo — the submodule is card *assets*; the rules reference is project context.

### Answers

**Phases** — three, all **turn-ordered**; nothing in this game is simultaneous.
1. **Hunting** — reveal `2 × players` cards sorted into family zones. Snake draft: first player clockwise picks one each, then from the *last* player counterclockwise picks a second each. Consequence worth stating: markers exactly equal cards, so **every revealed card is claimed** — the draft allocates priority, not scarcity.
2. **Action** — clockwise from the first player, one complete turn each. Four actions (Sell, Summon, Tame, Remove), any number, any order, repeatable. A player **cannot end their turn while a marker is still on a card** — it must be cleared by selling or taming, which makes those two forced rather than optional.
3. **Resolution** — clockwise from the first player; each player activates **every** available Active effect once, in an order of their choosing.

Round ends after Resolution; the first-player marker passes to the left.

**Player-count variance is much thinner than this ticket assumed.** The *only* things that vary are the reveal pool (`2 × N`) and the starting score by seat rank (1st player 1 pt, 2nd 2, 3rd 3, 4th 4 — turn-order compensation). Round cap (10), areas-per-round, stone cap, and remove cost are all count-independent.

**Economy** — Red 1 / Blue 3 / Purple 6 (42 stones total). Hard cap of **4 stones by count, not by value**; over the cap you keep 4 and discard the rest *before any other action*. Stones are **public**. **No change is given** when overpaying. Selling pays by **family**, not by card cost.

**Areas** — the number of areas equals the current round number. Remove (discarding your own summoned card) costs the current round number.

**Effect timings** — the rulebook's names are Instant / Permanent / Active, and our card data's names map onto them: `immediate`(41)=Instant, `passive`(7)=Permanent, `end_of_round`(22)=Active. **`end_of_round` is a misnomer** — Active effects resolve in the Resolution Phase, a distinct turn-ordered phase *before* the round-end check, in player-chosen order, and they are **mandatory**.

**Interaction** — effects do target other players, and the *targeted* player chooses which of their cards to discard `[p.12]`. So effect resolution contains decision points belonging to a seat other than the active one. No reaction or priority windows otherwise.

**Randomness — exactly one source**: the initial shuffle, plus the reshuffle of the discard pile when the draw pile empties. No dice anywhere. First-player selection is out-of-band ("most recently seen a reptile") and an engine must take it as a parameter. This is very good news for reproducible replay.

**End trigger** — score ≥ 60 at end of round, or the 10th round ends. Tiebreak: most points → most summoned cards → shared victory. Score can be reduced but never below zero.

**Adjudication rules `[p.12]`** — these are the heart of the effect engine: card effect beats game rule; a card cannot be summoned if any part of its Instant effect cannot resolve; `Recover` defaults to the card itself; every effect is mandatory; an untriggered trigger does not fire; a Permanent effect that modifies summoning does not apply to its own summoning; during Instant resolution the card counts as already in the area and not in hand.

### Bonus finding — `[family:X:N]` tokens are **magic stones**, not families

Verified directly against `cards/index.json`. The card data encodes magic stones **three different ways**:

| Encoding | Occurrences | Meaning |
|---|---|---|
| `[stone:1]` `[stone:3]` `[stone:6]` | 3 (all on *Willow*) | correct |
| `[family:fire:1]` / `[family:water:3]` / `[family:dragon:6]` | 7 / 12 / 6 | Red / Blue / Purple stone |
| `[family:fire] [points:1]` / `[family:water] [points:3]` / `[family:dragon] [points:6]` | 3 / 1 / 1 | same three stones, split into two tokens |

The mapping is exactly the rulebook's denominations (Red 1, Blue 3, Purple 6), and **no `earth` or `wind` compound tokens exist** — because there are only three stone colours. The extractor evidently read the coloured stone icons as family icons.

The card text confirms it: *Water Giant* — "Values of your `[family:water:3]` and `[family:dragon:6]` are each increased by 1" (families have no value); *Hae-tae* — "Value of your `[family:water:3]` counts as `[family:dragon:6]`"; *Snail Maiden* — "Exchange one of your `[family:water:3]` with `[family:dragon:6]`". Meanwhile *Poseidon* — "for each `[family:water]` **card** in your area" — is a genuine family reference.

**Disambiguation rule:** `[family:X]` is a real family reference when followed by the word "card" (or a taming/summoning context); it denotes a stone when it carries a value, either compounded (`[family:X:N]`) or paired with an adjacent `[points:N]`. This resolves the highest-priority item the research agent flagged and is handed to *Normalise the card text vocabulary*.

### Not delivered — one gap, ticketed separately

**Sell values per family are not in the rulebook.** Selling pays "depending on the family of the card", and the rates are printed on the **game board** `[p.7]`, not in the text. The extracted text cannot supply them and no source consulted confirmed them. This is load-bearing for the economy, so it is split out as its own ticket rather than left as a footnote.

## Comments

Resolved — see the resolution comment above. Rules reference on branch `research/base-game-rules` (`docs/rules/`). One gap split out as its own ticket: family sell values.
