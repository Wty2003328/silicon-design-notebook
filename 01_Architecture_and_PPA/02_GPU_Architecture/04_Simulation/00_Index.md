# Graphics Processing Unit (GPU) Architecture › Simulation

> **Abbreviation key — skim now and return as needed:** general-purpose computing on graphics processing units (GPGPU).

**Plain-language purpose:** Model GPU thread scheduling, memory transactions, high-bandwidth memory, and tensor pipelines before hardware exists.

## Terms introduced here

| Term | Meaning |
|---|---|
| functional model | computes correct results without detailed time |
| timing model | estimates when operations and resource conflicts occur |
| trace | recorded instruction or request stream |
| PTX / SASS | virtual GPU ISA versus target-specific GPU machine instructions |
| fat binary | executable container holding PTX and one or more target cubins |
| calibration | tunes model parameters against a trusted reference |
| validation | tests whether the model predicts observations it was not tuned to |

## Reading order

1. Read the GPU-specific [CUDA-to-Results Workflow](../00_Design_Methodology/03_GPU_Simulation_Methodology_and_Evidence.md).
2. [GPU Simulators](01_GPU_Simulators.md) — CUDA source → PTX/cubin/SASS → grid/warps or NVBit trace → timing events → validation and statistics in GPGPU-Sim/Accel-Sim.

**Prerequisite:** [GPU Simulation Methodology and Evidence](../00_Design_Methodology/03_GPU_Simulation_Methodology_and_Evidence.md).

---

[GPU Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
