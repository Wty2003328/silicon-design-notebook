# Graphics Processing Unit (GPU) Architecture › Simulation

> **Abbreviation key — skim now and return as needed:** general-purpose computing on graphics processing units (GPGPU).

**Plain-language purpose:** Model GPU thread scheduling, memory transactions, high-bandwidth memory, and tensor pipelines before hardware exists.

## Terms introduced here

| Term | Meaning |
|---|---|
| functional model | computes correct results without detailed time |
| timing model | estimates when operations and resource conflicts occur |
| trace | recorded instruction or request stream |
| calibration | tunes model parameters against a trusted reference |
| validation | tests whether the model predicts observations it was not tuned to |

## Reading order

1. [GPU Simulators](01_GPU_Simulators.md) — GPGPU-Sim, Accel-Sim, trace/execution tradeoffs, validation, statistics.

**Prerequisite:** [Shared Simulation Methodology](../../05_Architecture_Foundations_and_Methods/05_Simulation_Methodology/00_Index.md).

---

[GPU Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
