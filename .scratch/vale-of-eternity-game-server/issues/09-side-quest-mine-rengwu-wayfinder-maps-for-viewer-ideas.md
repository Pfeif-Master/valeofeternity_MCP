<!-- was GitHub issue #10 (Pfeif-Master/valeofeternity_MCP) -->

# Side quest: mine rengwu/wayfinder-maps for viewer ideas

Type: side-quest

## Side quest

Off the route. Pick up when the mood strikes; nothing on the map waits for it.

**`rengwu/wayfinder-maps`** — despite the name, this does **not** visualize what we have. It renders the *other* wayfinder backend: maps kept as local markdown files under `.plan/`, not as GitHub issues. Pointed at this repo it would find nothing.

**Why it's still worth a look**

This repo's issue tracker is configured for GitHub (see `docs/agents/issue-tracker.md`), but wayfinder also supports a **local-markdown** tracker as a first-class mode. This tool is the clearest available picture of what that mode looks like when someone actually builds for it.

**What to find out**

- What does its visualization actually show — a dependency graph, a list, something else? Is it better than what a Mermaid flowchart would give us?
- What does the local `.plan/` map format look like in practice, and is anything about it *better* than issues — faster to edit, easier to diff, reviewable in a PR?
- Is there anything worth stealing outright for the homegrown viewer — layout, how it distinguishes the frontier, how it renders fog?

**Not a proposal to switch trackers.** GitHub issues are the right call here: native sub-issues, native dependencies, and a UI that renders blocking without any tooling at all. This is idea-mining, not migration.
