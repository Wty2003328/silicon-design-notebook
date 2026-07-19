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

## Reading order

1. [Coalescing, Caches, and Shared Memory](01_Coalescing_Caches_and_Shared_Memory.md).
2. [HBM and Advanced Memory Systems](02_HBM_and_Advanced_Memory_Systems.md).

**Hands off to:** [GPU Scale-Up](../03_Scale_Up/00_Index.md) and [AI Workloads and Serving](../05_AI_Workloads_and_Serving/00_Index.md), where HBM traffic becomes weight, activation, and KV-cache limits.

---

[GPU Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
