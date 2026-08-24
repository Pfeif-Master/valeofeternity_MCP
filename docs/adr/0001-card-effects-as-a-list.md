# Card data models effects as a list, not a single type/text pair

`cards/index.json` originally gave every card a single `type` + `text` field, matching
the rulebook's assumption that a card carries one effect. A card-by-card QA pass against
the printed art (see wayfinder ticket *Normalise the card text vocabulary*) found 8 of
the 70 cards actually print two effect clauses (e.g. an Instant "draw a card" plus a
Permanent cost reduction), and that the old shape wasn't just awkward for them — Mud
Slime's `type: "immediate"` silently dropped its second, Active clause entirely.

Decided: every card's data is `effects: [{type, text}]`, an ordered, non-empty list —
always a list, even for the 62 cards with exactly one entry, so there's one shape and no
special-cased scalar path. At the same time, `type` values were renamed from the
extractor's `immediate`/`passive`/`end_of_round` to the rulebook-grounded
`instant`/`permanent`/`active` (see `CONTEXT.md`), since every card record was being
rewritten anyway and the old names were already flagged in the domain glossary as a
misnomer to avoid.

Considered keeping the scalar fields for the common single-effect case and adding an
optional `extra_effects` list for the 8 exceptions; rejected because it means two code
paths reading the same concept, for a distinction (1 vs 2 effects) with no game-rule
significance.
