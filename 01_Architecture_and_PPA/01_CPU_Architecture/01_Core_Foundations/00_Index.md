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

## Reading order

1. [CPU Architecture](01_CPU_Architecture.md) — whole-machine map, pipeline, hazards, memory, and multicore context.
2. [RISC-V Instruction Set Architecture](02_RISC_V_ISA.md) — modular software-visible contract and implementation consequences.
3. [SMT, SIMD, and Vector Execution](03_SMT_SIMD_and_Vector_Execution.md) — thread and data parallelism choices.

**Comes from:** [Architecture Primer](../../05_Architecture_Foundations_and_Methods/01_Reader_Foundations/01_Architecture_Primer_and_Glossary.md).
**Hands off to:** [Frontend](../02_Frontend_and_Prediction/00_Index.md) and [Out-of-Order Backend](../03_Out_of_Order_Backend/00_Index.md).

---

[CPU Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
