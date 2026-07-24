# 03 · Frontend RTL and Verification — Folder Index

*Write synthesizable RTL; prove it correct (dynamic + static).*

### Design

| # | Page | Coverage |
|---|------|----------|
| 01 | [RTL Design Methodology](01_RTL_Design_Methodology.md) | synchronous discipline, three reset styles in RTL, clocking, datapath/control split, 1/2/3-process FSM coding, ICG, synthesis-safe coding |
| 02 | [Data Types and Basics](02_Data_Types_and_Basics.md) | 2/4-state, X-optimism vs X-pessimism, nets vs variables, arrays, structs, enums, casting |
| 03 | [Procedural, Processes, and IPC](03_Procedural_Processes_and_IPC.md) | event regions, blocking vs non-blocking, always blocks, fork/join, scheduling, mailbox/semaphore/events |
| 04 | [Clock Division and Switching](04_Clock_Division_and_Switching.md) | even/odd/fractional dividers (div-3 RTL), glitch-free clock mux, ICG clock gating |
| 05 | [PLL, DLL, and Clock Distribution](05_PLL_DLL_and_Clock_Distribution.md) | PFD/charge pump/VCO, lock, jitter, DLL vs PLL, H-tree/mesh distribution |
| 06 | [Async Design and CDC](06_Async_Design_and_CDC.md) | metastability/MTBF, synchronizers, gray code, handshakes, async FIFO (full RTL), CDC |
| 07 | [Lint, CDC & RDC Signoff](07_Lint_CDC_RDC_Signoff.md) | static lint (good/bad RTL), structural+functional CDC, reset-domain crossing |
| 14 | [RTL Design Patterns](14_RTL_Design_Patterns.md) | pipelining & retiming (breaking a long combinational path), FSMD, parameterization/generate, case coding, interfaces/modports, cookbook (counter, shift, LFSR, priority encoder, round-robin arbiter) |
| 15 | [Flow Control and FIFOs](15_Flow_Control_and_FIFOs.md) | valid/ready handshake, skid buffer (register slice), synchronous FIFO (+FWFT), credit-based flow control, back-pressure through a pipeline |
| 16 | [Arithmetic and Memory RTL](16_Arithmetic_and_Memory_RTL.md) | fixed-point/Q-format, saturating arithmetic, rounding (round-to-even), overflow, RAM inference + read-during-write, register file, memory-mapped CSR (RW/RO/W1C) |

### Verification

| # | Page | Coverage |
|---|------|----------|
| 08 | [OOP and Randomization](08_OOP_and_Randomization.md) | classes, polymorphism, constraint randomization, rand vs randc, the factory |
| 09 | [Assertions and Coverage](09_Assertions_and_Coverage.md) | immediate vs concurrent SVA, assert/assume/cover, functional vs code coverage |
| 10 | [UVM Methodology](10_UVM_Methodology.md) | components, phasing, sequences, factory, config_db, TLM, RAL, AXI4-Lite agent skeleton |
| 11 | [Verification Planning & Coverage Closure](11_Verification_Planning_and_Coverage_Closure.md) | vplan, coverage taxonomy, directed vs constrained-random, closure loop, sign-off criteria |
| 12 | [Formal Verification](12_Formal_Verification.md) | SAT/BDD, BMC vs k-induction vs IC3/PDR, LEC, model checking, CDC formal, connectivity |
| 13 | [Gate-Level Sim & Emulation](13_Gate_Level_Sim_and_Emulation.md) | GLS (zero-delay/unit-delay/SDF, X-prop), scan/DFT, emulation, FPGA prototyping |

> Files 14–16 are design-side pattern references (they follow 07 in reading order; the higher numbers just reflect that they were added later). Read the Design group top-to-bottom, then the Verification group.

---

⬅ prev [02 · Power and Low-Power](../02_Power_and_Low_Power/00_Index.md) · [Root Index](../Index.md) · [Flow Overview](../Chip_Design_Flow_Overview.md) · next ➡ [04 · Synthesis](../04_Synthesis/00_Index.md)
