# 03 · Architecture › Memory — Subfolder Index

*Shared part: the storage hierarchy every chip type reuses. From the SRAM caches nearest the core, through address translation, down to the bit-cell trade surface and the DRAM controller that feeds the whole machine.*

| # | Page | Coverage |
|---|------|----------|
| 01 | [Cache Microarchitecture](01_Cache_Microarchitecture.md) | pipeline, MSHR, write policy, prefetch, replacement, coherence implementation |
| 02 | [TLB and Virtual Memory](02_TLB_and_Virtual_Memory.md) | TLB format, page-table walker, VIPT, shootdown, superpages, 5-level |
| 03 | [Memory](03_Memory.md) | the bit-storage trade surface: 6T/8T/10T SRAM (SNM vs area vs speed), DRAM 1T1C density-vs-refresh, memory compiler, multi-port/register-file port cost, ECC coding theory, CAM/TCAM, compute-in-memory |
| 04 | [DDR Controller](04_DDR_Controller.md) | commands, timing, row-buffer, FR-FCFS, ECC, DDR5, LPDDR5X |

---

⬅ [Architecture & PPA index](../00_Index.md) · [Root Index](../../Index.md)
