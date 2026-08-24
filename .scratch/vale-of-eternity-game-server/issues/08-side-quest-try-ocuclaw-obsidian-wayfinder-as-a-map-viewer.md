<!-- was GitHub issue #9 (Pfeif-Master/valeofeternity_MCP) -->

# Side quest: try ocuclaw/obsidian-wayfinder as a map viewer

Type: side-quest

## Side quest

Off the route. Pick up when the mood strikes; nothing on the map waits for it.

**`ocuclaw/obsidian-wayfinder`** — the only purpose-built wayfinder map visualizer found. An Obsidian plugin that reads GitHub's **native** sub-issues and `blocked_by` edges, and explicitly implements the frontier concept: open + verified-unblocked + unassigned gets flagged as FRONTIER. The design is right — it understands the actual data model rather than the legacy `Blocked by: #12` text convention that most tools stop at.

**What to find out**

- Does it actually render *Vale of Eternity game server* (issue #1) and its seven children correctly?
- Is its frontier logic right? It should surface exactly #2, #4, and #8 as of charting.
- Is the Obsidian dependency worth it, given nothing else here uses Obsidian?
- Does it survive the map growing — fog graduating into new tickets, tickets getting deleted when an upstream answer invalidates them?

**Caution before installing**

It stores its GitHub token in **plaintext**. It is also three weeks old, one star, single author — correct design, unproven execution. If you try it, use a **fine-grained PAT** scoped to these two repos only, read-only, with a short expiry. Don't hand it a classic token.

**Alternative already on the table:** a homegrown ~100–150 line script pulling `gh api` and emitting a Mermaid flowchart with the frontier styled. No new dependency, no token storage, uses existing `gh` auth. This side quest is worth doing anyway — if the plugin is good, it saves writing that script.
