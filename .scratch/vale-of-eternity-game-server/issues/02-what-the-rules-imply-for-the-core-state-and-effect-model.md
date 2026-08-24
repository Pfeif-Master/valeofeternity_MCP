<!-- was GitHub issue #3 (Pfeif-Master/valeofeternity_MCP) -->

# What the rules imply for the core state and effect model

Type: grilling
Blocked by: 01, 10

## Question

Given the rules, decide the shape of the core's game state and how effects resolve against it.

- What is the state model?
- How are the three effect timings (`immediate`, `end_of_round`, `passive`) sequenced?
- Where do player choices interrupt resolution?
- What must a per-seat view redact?

Folded in from a collapsed ticket: **must the engine be built instrumentable** — deterministic
given a seed, and emitting a complete ordered event log sufficient to replay any game exactly?
The evaluation harness itself is out of scope for this map, but instrumentability is cheap to
design in and brutal to retrofit, so the decision belongs here rather than later.
