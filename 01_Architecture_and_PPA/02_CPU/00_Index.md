# 02 · Architecture › CPU — Subfolder Index

*Chip type: the general-purpose core. Read top to bottom for a complete latency-optimized machine — from the ISA contract, through the in-order pipeline, into out-of-order execution and speculation, and finishing on a real open-source core.*

| # | Page | Coverage |
|---|------|----------|
| 01 | [CPU Architecture](01_CPU_Architecture.md) | 5-stage pipeline, hazards, forwarding network, Tomasulo, superscalar, MESI, SMT, µop fusion, Spectre/Meltdown |
| 02 | [RISC-V ISA](02_RISC_V_ISA.md) | RV64GC, privilege modes, Sv39/48, Vector, Bitmanip, Hypervisor |
| 03 | [OoO Execution](03_OoO_Execution.md) | rename/RAT, ROB, issue-queue wakeup-select, LSQ, misprediction recovery, exception pipeline, execution units |
| 04 | [Branch Prediction Deep Dive](04_Branch_Prediction_Deep_Dive.md) | BTB, gshare, TAGE-SC-L, perceptron, RAS, ITTAGE, fetch unit + FTQ |
| 05 | [Xiangshan CPU Design](05_Xiangshan_CPU_Design.md) | open-source RISC-V OoO core case study (Nanhu/Kunminghu) |

---

⬅ [Architecture & PPA index](../00_Index.md) · [Root Index](../../Index.md)
