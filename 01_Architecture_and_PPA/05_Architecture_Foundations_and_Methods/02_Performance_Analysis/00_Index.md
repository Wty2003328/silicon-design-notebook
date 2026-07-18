# Architecture Foundations and Methods › Performance Analysis

> **Abbreviation key — skim now and return as needed:** power, performance, and area (PPA); instructions per cycle (IPC); cycles per instruction (CPI); design-space exploration (DSE); service-level objective (SLO).

**Plain-language purpose:** Define the workload being optimized, select representative measurements, and compare architecture choices at appropriate model fidelity.

## Terms introduced here

| Term | Meaning |
|---|---|
| service-level objective (SLO) | measurable product target such as tail latency |
| cycles per instruction (CPI) | average cycles spent per retired instruction |
| instructions per cycle (IPC) | average retired instructions per cycle |
| design-space exploration (DSE) | systematic comparison of architecture parameter choices |
| phase | interval of execution with relatively stable behavior |
| confidence interval | range expressing uncertainty in an estimated metric |

## Reading order

1. [Performance Modeling and DSE](01_Performance_Modeling_and_DSE.md).
2. [Workload Characterization and Sampling](02_Workload_Characterization_and_Sampling.md).

**Hands off to:** [PPA Estimation](../03_PPA_Estimation/00_Index.md) and architecture-specific simulators.

---

[Foundations and Methods](../00_Index.md) · [Architecture book](../../00_Index.md)
