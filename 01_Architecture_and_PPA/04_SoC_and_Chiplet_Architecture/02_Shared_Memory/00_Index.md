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
| mode register (MR) | on-device configuration written by an MRS command |
| training | the calibration that finds a sampling point the signal actually has |
| DDR PHY interface (DFI) | the standard boundary between a memory controller and its PHY |
| bank group | a set of banks sharing prefetch resources, so back-to-back access to one is slower |

## Reading order

1. [DDR Memory Controller](01_DDR_Controller.md) — the controller's job: the bank state machine, JEDEC timing derived as physics, row-buffer policy, FR-FCFS scheduling, refresh, and the achieved-bandwidth model.
2. [The DRAM Device Protocol and Training](02_DRAM_Device_Protocol_and_Training.md) — the wire-level layer the controller drives: the pin list, the command truth table, bank groups and $t_{CCD\_S}$ vs $t_{CCD\_L}$, prefetch and burst arithmetic, mode registers, the initialization sequence, **training in depth** (why it is mandatory, and every step in the order it must run), ZQ and ODT, DBI and CRC, command-level refresh, what DDR5 and LPDDR5X actually changed, the DFI controller-PHY boundary, and bring-up triage.

**Which first?** Read 01 if you want to know how memory requests are *scheduled*; read 02 first if you want to know what the pins actually do, or if you are bringing up a memory interface. 02 assumes the cell physics in [Memory Circuits and Technologies §10](../../../00_Fundamentals/06_Memory_Circuits_and_Technologies.md).

**Hands off to:** [DRAM Simulation](../06_Simulation/00_Index.md), [High-Speed I/O §8](../05_IO_and_Chiplets/03_High_Speed_IO_and_Peripheral_Protocols.md) for the PHY at circuit level.

---

[SoC and Chiplet Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
