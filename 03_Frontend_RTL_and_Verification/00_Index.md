# 03 · Frontend RTL and Verification — Folder Index

*Write synthesizable RTL; prove it correct (dynamic + static).*

| # | Page | Coverage |
|---|------|----------|
| 01 | [RTL Design Methodology](01_RTL_Design_Methodology.md) | synchronous discipline, reset architecture, clocking, datapath/control split, synthesis-safe coding |
| 02 | [Data Types and Basics](02_Data_Types_and_Basics.md) | 2/4-state, arrays, structs, enums, casting |
| 03 | [Procedural, Processes, and IPC](03_Procedural_Processes_and_IPC.md) | event regions, always blocks, fork/join, scheduling, mailbox/semaphore/events |
| 04 | [Clock Division and Switching](04_Clock_Division_and_Switching.md) | even/odd/fractional dividers, div-3 FSM, programmable divider RTL, glitch-free switching, clock MUX |
| 05 | [PLL, DLL, and Clock Distribution](05_PLL_DLL_and_Clock_Distribution.md) | PFD/charge pump/VCO, lock, jitter, DLL vs PLL, H-tree/mesh distribution |
| 06 | [Async Design and CDC](06_Async_Design_and_CDC.md) | metastability/MTBF, synchronizers, async FIFO, handshakes, CDC |
| 07 | [Lint, CDC & RDC Signoff](07_Lint_CDC_RDC_Signoff.md) | static lint, structural+functional CDC, reset-domain crossing |
| 08 | [OOP and Randomization](08_OOP_and_Randomization.md) | classes, polymorphism, constraint randomization |
| 09 | [Assertions and Coverage](09_Assertions_and_Coverage.md) | immediate/concurrent SVA, functional coverage mechanics, code coverage |
| 10 | [UVM Methodology](10_UVM_Methodology.md) | components, phasing, sequences, factory, config_db, TLM, RAL, complete AXI4-Lite testbench |
| 11 | [Verification Planning & Coverage Closure](11_Verification_Planning_and_Coverage_Closure.md) | vplan, coverage taxonomy, CDV cycle, closure loop, sign-off criteria |
| 12 | [Formal Verification](12_Formal_Verification.md) | SAT/BDD, LEC, model checking, CDC formal, connectivity |
| 13 | [Gate-Level Sim & Emulation](13_Gate_Level_Sim_and_Emulation.md) | GLS (zero-delay/SDF, X-prop), emulation, FPGA prototyping |

---

⬅ prev [02 · Power and Low-Power](../02_Power_and_Low_Power/00_Index.md) · [Root Index](../Index.md) · [Flow Overview](../Chip_Design_Flow_Overview.md) · next ➡ [04 · Synthesis](../04_Synthesis/00_Index.md)
