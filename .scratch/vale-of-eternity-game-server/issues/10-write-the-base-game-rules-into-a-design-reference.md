<!-- was GitHub issue #11 (Pfeif-Master/valeofeternity_MCP) -->

# Write the base-game rules into a design reference

Type: task

## Question

Produce the one document an engine implementer works from. Everything needed is already on
disk — this is transcription and structuring, not investigation.

**Inputs, all local:**

- `valeofeternity_cardDB/docs/VoE_Rulebook_WEB.pdf` — the official English rulebook
  (Renegade edition, 12 pages), already pulled. Untracked and copyrighted.
- `valeofeternity_cardDB/docs/VoE_Rulebook_TEXT.txt` — full text extraction of it, all 12
  pages, `===== PAGE N =====` separators. Page 1 is the cover and carries no text. Read this
  rather than the PDF; there is no PDF renderer on this machine.
- `valeofeternity_cardDB/cards/index.json` — the 70 base-game cards.

**Facts already established — use them, don't re-derive:**

- Sell values, confirmed off the physical board: Fire 3×Red, Earth 4×Red, Water 1×Blue, Wind
  1×Red + 1×Blue, Dragon 1×Purple. These are printed on the board, not in the rulebook.
- `[family:fire:1]` / `[family:water:3]` / `[family:dragon:6]` in the card data are magic
  stones (Red 1 / Blue 3 / Purple 6), not families — the extractor read the coloured stone
  icons as family icons. `[family:X]` means a real family only when it modifies "card".
- The rulebook's effect types are Instant / Permanent / Active, which the card data calls
  `immediate` / `passive` / `end_of_round`.

**Output:** one file, `docs/rules/base-game.md`. Cite pages as `[p.N]`. Restate the rules in
your own words — the PDF is copyrighted and stays untracked, as does its text extraction.

Write it for someone about to implement the engine: the sequence of play, the economy, how
effects resolve and when, what each seat can and cannot see, where randomness enters, and how
the game ends. Give each rule the space its substance needs and no more. Where the rulebook
genuinely leaves something an engine must decide, note it in a short closing section rather
than inline.

Anything the rulebook doesn't cover because it's printed on a component — ask the maintainer,
who owns the game. Don't hunt for it.
