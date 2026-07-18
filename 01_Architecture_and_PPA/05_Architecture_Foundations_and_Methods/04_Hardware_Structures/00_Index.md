# Architecture Foundations and Methods › Hardware Structures

> **Abbreviation key — skim now and return as needed:** central processing unit (CPU); graphics processing unit (GPU); neural processing unit (NPU); system on chip (SoC); translation lookaside buffer (TLB);
> static random-access memory (SRAM); dynamic random-access memory (DRAM); high-bandwidth memory (HBM); double data rate (DDR); content-addressable memory (CAM);
> error-correcting code (ECC).

**Plain-language purpose:** Explain the reusable circuit arrays that hold registers, cache data/tags, translations, directories, queues, and main-memory bits.

## Terms introduced here

| Term | Meaning |
|---|---|
| static random-access memory (SRAM) | fast on-chip bit storage that holds state while powered |
| dynamic random-access memory (DRAM) | dense charge-based storage requiring refresh |
| content-addressable memory (CAM) | searches stored entries by content rather than numeric address |
| error-correcting code (ECC) | redundant bits used to detect/correct data corruption |
| memory compiler | tool generating physical memory macros from parameters |

## Reading order

1. [Memory Arrays and Technologies](01_Memory_Arrays_and_Technologies.md).

**Hands off to:** CPU caches/TLBs, SoC DDR control, GPU HBM, and NPU scratchpads.

---

[Foundations and Methods](../00_Index.md) · [Architecture book](../../00_Index.md)
