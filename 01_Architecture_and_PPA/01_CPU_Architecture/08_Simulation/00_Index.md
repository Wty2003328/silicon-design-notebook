# Central Processing Unit (CPU) Architecture › Simulation

> **Abbreviation key — skim now and return as needed:** network on chip (NoC).

**Plain-language purpose:** Turn CPU pipeline, cache, and coherence hypotheses into controlled executable experiments.

## Terms introduced here

| Term | Meaning |
|---|---|
| ELF executable | target-specific machine instructions, data, loader metadata, and entry point |
| dynamic instruction | one runtime execution of a static machine instruction |
| cycle model | estimates when each modeled event occurs |
| execution-driven | later requests change when simulated timing changes |
| trace-driven | replays a fixed recorded stream |
| Ruby | gem5 subsystem for detailed memory/coherence modeling |
| Garnet | gem5 timing model for a packet network on chip |

## Reading order

1. Read the shared [Benchmark-to-Results Workflow](../../05_Architecture_Foundations_and_Methods/05_Simulation_Methodology/03_Benchmark_to_Results_End_to_End.md) for source → compiler/assembly → ELF → loader → dynamic instructions → events → counters.
2. [gem5](01_gem5.md) — concrete target build, ELF loading, StaticInst/dynamic records, CPU models, full-system mode, checkpoints, Ruby memory, statistics, and calibration.
3. [NoC and Coherence Simulation](02_NoC_and_Coherence_Simulation.md) — load/store → cache transaction → coherence messages → packets/flits → completion, plus liveness and tail latency.

**Prerequisite:** [Shared Simulation Methodology](../../05_Architecture_Foundations_and_Methods/05_Simulation_Methodology/00_Index.md).

---

[CPU Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
