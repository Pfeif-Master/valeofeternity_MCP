<!-- was GitHub issue #5 (Pfeif-Master/valeofeternity_MCP) -->

# Does the core publish a legal-action set?

Type: grilling
Blocked by: 02

## Question

Settled already: the core is authoritative and rejects invalid actions from players.

Open: does it also hand a seat the set of currently **legal** actions on request?

Publishing legal actions collapses a large class of agent failure — an agent misreading state
and submitting nonsense — and makes any client surface far easier to drive. The cost depends on
how expensive enumeration is against the real state and effect model, which is why this waits on
*What the rules imply for the core state and effect model*.
