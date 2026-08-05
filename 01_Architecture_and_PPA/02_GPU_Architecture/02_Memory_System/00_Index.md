# Graphics Processing Unit (GPU) Architecture › Memory System

> **Abbreviation key — skim now and return as needed:** high-bandwidth memory (HBM).

**Plain-language purpose:** Follow one GPU memory instruction from lane addresses through on-chip storage and caches to stacked high-bandwidth memory.

## Terms introduced here

| Term | Meaning |
|---|---|
| coalescing | combines lane accesses into fewer memory transactions |
| shared memory | software-managed low-latency storage shared by threads in a block |
| bank conflict | multiple accesses need the same independently serviced memory bank |
| memory partition | controller slice responsible for part of the address space |
| high-bandwidth memory (HBM) | stacked memory connected through many package wires |
| GPU atomic | a read-modify-write performed at the shared-memory unit or an L2 slice, not in the thread |
| warp-aggregated atomic | one atomic issued per warp after the lanes combine their updates |

## Reading order

1. [Coalescing, Caches, and Shared Memory](01_Coalescing_Caches_and_Shared_Memory.md).
2. [HBM and Advanced Memory Systems](02_HBM_and_Advanced_Memory_Systems.md).
3. [GPU Atomics and Synchronization](03_GPU_Atomics_and_Synchronization.md) — why the CPU answer does not transfer, where a GPU atomic executes, intra-warp conflict and the replay cost model, `atom` versus `red`, warp aggregation, shared-memory atomics and privatized histograms, scopes, the SIMT spin-lock deadlock, grid-scale synchronization and cooperative groups, floating-point non-determinism, system-scope and multi-GPU atomics, and diagnosis.

**Hands off to:** [GPU Scale-Up](../03_Scale_Up/00_Index.md) and [AI Workloads and Serving](../05_AI_Workloads_and_Serving/00_Index.md), where HBM traffic becomes weight, activation, and KV-cache limits. The single-threaded-machine version of 03 is [CPU Atomic Operations](../../01_CPU_Architecture/06_Coherence_and_Consistency/04_Atomic_Operations.md), and the scoped consistency model that 03 assumes stays in [Coalescing, Caches, and Shared Memory §14](01_Coalescing_Caches_and_Shared_Memory.md).

---

[GPU Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
