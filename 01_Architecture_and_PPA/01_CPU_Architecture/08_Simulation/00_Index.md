# Central Processing Unit (CPU) Architecture › Simulation

> **Abbreviation key — skim now and return as needed:** network on chip (NoC).

**Plain-language purpose:** Turn CPU pipeline, cache, and coherence hypotheses into controlled executable experiments.

## Terms introduced here

| Term | Meaning |
|---|---|
| cycle model | estimates when each modeled event occurs |
| execution-driven | later requests change when simulated timing changes |
| trace-driven | replays a fixed recorded stream |
| Ruby | gem5 subsystem for detailed memory/coherence modeling |
| Garnet | gem5 timing model for a packet network on chip |

## Reading order

1. [gem5](01_gem5.md) — CPU models, full-system mode, checkpoints, Ruby memory, statistics, calibration.
2. [NoC and Coherence Simulation](02_NoC_and_Coherence_Simulation.md) — coupled protocol/network state, open-loop trace limits, liveness, tail latency.

**Prerequisite:** [Shared Simulation Methodology](../../05_Architecture_Foundations_and_Methods/05_Simulation_Methodology/00_Index.md).

---

[CPU Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
