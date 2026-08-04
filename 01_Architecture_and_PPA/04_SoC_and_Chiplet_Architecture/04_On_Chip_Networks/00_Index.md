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
| allocator | the block that matches waiting flits to scarce output resources each cycle |
| channel load | the number of packets crossing one channel per injected packet, under a stated traffic pattern |
| bisection | the minimum channel count crossing a cut that splits the network in half |
| radix | the number of ports on a router |

## Reading order

1. [Network on Chip](01_Network_on_Chip.md) — the framing: why a bus and a crossbar both stop scaling, the diameter-bisection-cost trade, flow-control families, the routing/deadlock theorems in outline, the router pipeline as a concept, latency under load, and the coherent mesh in practice.
2. [Routing, Flow Control, and Deadlock](02_Routing_Flow_Control_and_Deadlock.md) — the theory, proved: wormhole dependency, credit flow control, virtual channels as waiting classes, the channel-dependency graph, adaptive routing with escape resources, protocol versus network deadlock, livelock and starvation, multicast dependencies, congestion control, and fault-aware routing.
3. [Router Microarchitecture](03_Router_Microarchitecture.md) — the block you would actually build. The BW/RC/VA/SA/ST/LT pipeline stage by stage; buffer organization and the credit-round-trip depth equation; lookahead route computation; VC allocation; switch allocation with separable, wavefront, iSLIP, and matrix arbiters compared on delay, area, fairness, and matching quality; speculation and pipeline collapsing; crossbar cost and speedup; credit accounting; the network interface; a full FO4 critical-path budget that predicts what closes timing; fabric floorplanning, link pipelining, and power; a complete parameterized SystemVerilog router; and the verification invariants.
4. [Topology Selection and Performance Analysis](04_Topology_Selection_and_Performance_Analysis.md) — predicting throughput before building. The topology design space as closed-form parameters; why bisection is a wire-budget constraint rather than a knob; **channel load and ideal throughput derived** for mesh and torus under uniform and adversarial traffic; traffic patterns as theorem tests; the latency-throughput curve and how to read it; the low-radix families; the high-radix families and the pin-economics argument that produced them (flattened butterfly, fat tree, dragonfly); hierarchical and heterogeneous fabrics; chiplet topology; mapping and interleaving as traffic shaping; energy per bit; the research frontier surveyed honestly; and a decision procedure.

**Reading routes.** For the theory, read 1 → 2. To *implement* a fabric, read 1 → 2 → 3. To *size or choose* one, read 1 → 4, and use 3 for the router parameters that 4's models assume. The analytical results in 4 are what the simulation methodology in [NoC and Coherence Simulation](../../01_CPU_Architecture/08_Simulation/02_NoC_and_Coherence_Simulation.md) validates against.

**Hands off to:** [I/O and Chiplets](../05_IO_and_Chiplets/00_Index.md), [NoC, QoS, I/O and Chiplet Integration Blueprint](../08_Implementation_Blueprints/02_NoC_QoS_IO_and_Chiplet_Integration_Blueprint.md).

---

[SoC and Chiplet Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
