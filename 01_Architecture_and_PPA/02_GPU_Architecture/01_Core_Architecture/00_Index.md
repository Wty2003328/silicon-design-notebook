# Graphics Processing Unit (GPU) Architecture › Core Architecture

> **Abbreviation key — skim now and return as needed:** single instruction, multiple threads (SIMT); streaming multiprocessor (SM); register file (RF); Tensor Memory Accelerator (TMA).

**Plain-language purpose:** Explain how a graphics processing unit (GPU) uses many resident thread groups to maximize throughput.

## Terms introduced here

| Term | Meaning |
|---|---|
| single instruction, multiple threads (SIMT) | one issued instruction controls many lanes with per-thread state |
| streaming multiprocessor (SM) | repeated GPU core containing schedulers, lanes, registers, and local storage |
| warp | group of threads scheduled together |
| occupancy | fraction of warp capacity resident on an SM |
| divergence | lanes follow different control-flow paths and execute in subsets |
| operand collector | gathers a warp instruction's register operands across banked RF cycles |
| independent thread scheduling | maintains finer-grained thread control state while still issuing SIMT groups |
| warp specialization | assigns producer, compute, or epilogue roles to different warps |
| matrix multiply–accumulate (MMA) | cooperative matrix operation executed by a dedicated matrix pipeline |

## Reading order

1. [GPU Architecture](01_GPU_Architecture.md) — throughput-machine overview, including the matrix-instruction data path used by AI workloads.
2. [SIMT Scheduling and Occupancy](02_SIMT_Scheduling_and_Occupancy.md) — residency, eligibility, scoreboards, stalls, divergence.
3. [Operand Collectors, Register Files, and Scoreboards](03_Operand_Collectors_Register_Files_and_Scoreboards.md) — banked operand delivery, dispatch, writeback, and replay.
4. [Independent Threads and Asynchronous Pipelines](04_Independent_Thread_Scheduling_and_Asynchronous_Pipelines.md) — sub-warp control, TMA, transaction barriers, warp specialization, clusters, and distributed shared memory.

**Hands off to:** [GPU Memory System](../02_Memory_System/00_Index.md), then [AI Workloads and Serving](../05_AI_Workloads_and_Serving/00_Index.md) for model/operator mapping and end-to-end inference.

---

[GPU Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
