# System-on-Chip (SoC) and Chiplet Architecture › On-Chip Networks

> **Abbreviation key — skim now and return as needed:** network on chip (NoC); virtual channel (VC).

**Plain-language purpose:** Explain how packet transport replaces long shared buses when many chip blocks must communicate concurrently.

## Terms introduced here

| Term | Meaning |
|---|---|
| network on chip (NoC) | packet network connecting blocks within a chip |
| packet / flit | complete message / smaller flow-control unit |
| router | chooses output direction and moves flits between links |
| virtual channel (VC) | logically separate queue sharing a physical link |
| credit | downstream promise that buffer space is available |
| deadlock | cyclic waiting state in which no participant can progress |

## Reading order

1. [Network on Chip](01_Network_on_Chip.md) — topology, routers, links, latency, load, physical effects.
2. [Routing, Flow Control, and Deadlock](02_Routing_Flow_Control_and_Deadlock.md) — dependency proofs, credits, VCs, escape routes, liveness.

**Hands off to:** [I/O and Chiplets](../05_IO_and_Chiplets/00_Index.md).

---

[SoC and Chiplet Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
