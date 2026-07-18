# System-on-Chip (SoC) and Chiplet Architecture › Shared-Memory Simulation

> **Abbreviation key — skim now and return as needed:** dynamic random-access memory (DRAM); double data rate (DDR).

**Plain-language purpose:** Execute DRAM timing and scheduling rules to estimate delivered latency, bandwidth, queueing, and power.

## Terms introduced here

| Term | Meaning |
|---|---|
| timing guard | minimum legal delay between command types |
| row hit | request uses the row already open in a bank |
| scheduler | selects the next legal request/command |
| trace-driven | replays a fixed stream of memory requests |
| warm-up | establishes representative controller and memory state before measuring |

## Reading order

1. [DRAM Simulators](01_DRAM_Simulators.md) — Ramulator, DRAMSim3, DRAMPower, USIMM, coupling and validation.

**Prerequisite:** [DDR Controller](../02_Shared_Memory/01_DDR_Controller.md) and [Simulation Methodology](../../05_Architecture_Foundations_and_Methods/05_Simulation_Methodology/00_Index.md).

---

[SoC and Chiplet Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
