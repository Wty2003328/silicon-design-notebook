# GPU Architecture › Design Methodology

> **Purpose:** Define GPU kernels and end-to-end workloads, model throughput and scaling, price the replicated SM/register/memory structures, and establish the evidence chain from CUDA artifacts to validated results.

## Terms introduced here

| Term | Meaning in GPU design |
|---|---|
| kernel contract | exact host/device program, input, grid/block shape, streams, precision, and measured boundary |
| warp instruction | one dynamic instruction issued for a warp under an active-lane mask |
| occupancy | resident warps relative to the SM's supported warp slots; a latency-hiding capacity, not utilization |
| coalescing | combining lane memory addresses into cache-sector or memory transactions |
| arithmetic intensity | useful operations divided by bytes crossing a named memory boundary |
| trace provenance | capture GPU, toolkit, driver, binary, input, tracer, and trace hash needed to interpret SASS replay |

## Reading order

1. [GPU Workloads, Performance Modeling, and DSE](01_GPU_Workloads_Performance_and_DSE.md).
2. [GPU PPA and Physical Implementation](02_GPU_PPA_and_Physical_Implementation.md).
3. [GPU Simulation Methodology and Evidence](03_GPU_Simulation_Methodology_and_Evidence.md).

Then continue to [Core Architecture](../01_Core_Architecture/00_Index.md).

---

← [GPU book index](../00_Index.md) · next → [GPU Workloads, Performance Modeling, and DSE](01_GPU_Workloads_Performance_and_DSE.md)
