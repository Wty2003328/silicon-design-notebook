# Graphics Processing Unit (GPU) Architecture › Core Architecture

> **Abbreviation key — skim now and return as needed:** single instruction, multiple threads (SIMT); streaming multiprocessor (SM).

**Plain-language purpose:** Explain how a graphics processing unit (GPU) uses many resident thread groups to maximize throughput.

## Terms introduced here

| Term | Meaning |
|---|---|
| single instruction, multiple threads (SIMT) | one issued instruction controls many lanes with per-thread state |
| streaming multiprocessor (SM) | repeated GPU core containing schedulers, lanes, registers, and local storage |
| warp | group of threads scheduled together |
| occupancy | fraction of warp capacity resident on an SM |
| divergence | lanes follow different control-flow paths and execute in subsets |

## Reading order

1. [GPU Architecture](01_GPU_Architecture.md) — throughput-machine overview.
2. [SIMT Scheduling and Occupancy](02_SIMT_Scheduling_and_Occupancy.md) — residency, eligibility, scoreboards, stalls, divergence.

**Hands off to:** [GPU Memory System](../02_Memory_System/00_Index.md).

---

[GPU Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
