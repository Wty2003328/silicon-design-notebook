# Central Processing Unit (CPU) Architecture › Core Foundations

> **Abbreviation key — skim now and return as needed:** instruction set architecture (ISA); reduced instruction set computer (RISC); single instruction, multiple data (SIMD); simultaneous multithreading (SMT).

**Plain-language purpose:** Establish what a central processing unit (CPU) promises software and the basic machinery used to keep several operations in flight.

## Terms introduced here

| Term | Meaning |
|---|---|
| instruction set architecture (ISA) | the operations and visible state software may rely on |
| microarchitecture | the hidden pipeline, queues, predictors, and memories implementing an ISA |
| pipeline | staged assembly line that overlaps different instructions |
| superscalar | able to start more than one instruction per cycle |
| vector execution | one instruction operates on multiple data elements |
| privilege level | the mode that decides which state and instructions are reachable |
| control and status register (CSR) | architected core state read and written by dedicated instructions, not by loads and stores |
| trap | the controlled transfer to a handler taken on an exception or an interrupt |

## Reading order

1. [CPU Architecture](01_CPU_Architecture.md) — whole-machine map, pipeline, hazards, memory, and multicore context.
2. [RISC-V Instruction Set Architecture](02_RISC_V_ISA.md) — modular software-visible contract and implementation consequences.
3. [SMT, SIMD, Vector, and Matrix Execution](03_SMT_SIMD_and_Vector_Execution.md) — thread, lane, and tile-level data-parallel execution choices.
4. [Privileged Architecture, CSRs, and Traps](04_Privileged_Architecture_CSRs_and_Traps.md) — the other half of the software contract: why privilege levels exist at all, derived from what breaks without them; the CSR address space and why its read-modify-write must be atomic; **WPRI, WLRL, and WARL field semantics**, which is what a product specification is actually made of; the register groups a trap uses; trap delivery cycle by cycle and exactly what the return instruction undoes; architected exception priority; delegation; PMP and MPU region protection; counters and debug as architected state; the virtualization layer; Arm's banked system-register model compared; and how a real register map is specified, generated, and verified.

**Comes from:** [Architecture Primer](../00_Design_Methodology/01_CPU_Workloads_Performance_and_DSE.md).
**Hands off to:** [Frontend](../02_Frontend_and_Prediction/00_Index.md), [Out-of-Order Backend](../03_Out_of_Order_Backend/00_Index.md), and [AI Workloads and Serving](../09_AI_Workloads_and_Serving/00_Index.md).

---

[CPU Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
