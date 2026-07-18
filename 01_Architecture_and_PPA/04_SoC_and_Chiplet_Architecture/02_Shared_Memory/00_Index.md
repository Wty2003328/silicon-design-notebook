# System-on-Chip (SoC) and Chiplet Architecture › Shared Memory

> **Abbreviation key — skim now and return as needed:** dynamic random-access memory (DRAM); double data rate (DDR).

**Plain-language purpose:** Explain how many agents share dense off-chip main memory whose electrical rules require command timing and scheduling.

## Terms introduced here

| Term | Meaning |
|---|---|
| double data rate (DDR) | transfers data on both clock edges |
| dynamic random-access memory (DRAM) | dense memory cell that leaks charge and needs refresh |
| bank | independently activated portion of DRAM |
| row buffer | sense-amplifier state holding one open row |
| refresh | periodic restoration of stored charge |
| memory controller | schedules legal commands and returns request data |

## Reading order

1. [DDR Memory Controller](01_DDR_Controller.md) — commands, timing, row policy, scheduling, refresh, errors, bandwidth.

**Hands off to:** [DRAM Simulation](../06_Simulation/00_Index.md).

---

[SoC and Chiplet Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
