<!-- was GitHub issue #4 (Pfeif-Master/valeofeternity_MCP) -->

# Language and runtime for the core server

Type: grilling

## Question

The core is a server; the MCP service and an Electron app attach to it as clients. What does
the core run on?

- **TypeScript** shares a type system with the Electron client and has a first-party MCP SDK.
- **Python** also has a first-party MCP SDK, and matches the existing `scripts/pull_cards.py`
  tooling in the card-data submodule.
- Mixing means maintaining card-data types in two places.

Small ticket — likely minutes rather than a full session — but it should be decided
deliberately rather than settled by whoever happens to write the first file.
