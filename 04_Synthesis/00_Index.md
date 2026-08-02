# 04 · Synthesis — Folder Index

*RTL → gate netlist under constraints.*

| # | Page | Coverage |
|---|------|----------|
| 01 | [Synthesis and Optimization](01_Synthesis_and_Optimization.md) | RTL-to-gates, technology mapping, timing closure, area/power optimization |
| 02 | [Constraints (SDC)](02_Constraints_SDC.md) | clocks, generated clocks, I/O delay, the four exceptions, DRV, complete worked SDC, MCMM discipline |
| 03 | [Standard-Cell Libraries and Characterization](03_Standard_Cell_Libraries_and_Characterization.md) | what a cell physically is; `.lib`/LEF/GDS/CDL views and the one-cell invariant; how a delay number is characterized; NLDM interpolation, CCS/ECSM; timing arcs and constraint arcs; PVT corners, temperature inversion, derating; Vt flavors; library QA |
| 04 | [Synthesis Flow and QoR Closure](04_Synthesis_Flow_and_QoR_Closure.md) | the run as a data flow; a real commented TCL script (and its open-source equivalent); elaboration warnings; compile strategies and ungrouping; hierarchical budgeting and ETM/ILM; DFT-aware and power-aware compile; **reading the reports**; the QoR triage playbook; LEC as a gate; release criteria |
| 05 | [Physical Synthesis and Design Planning](05_Physical_Synthesis_and_Design_Planning.md) | the death of the wireload model; topographical synthesis; partitioning and hierarchy choice; die-size and utilization estimation; partition interfaces and feedthroughs; timing budgets across partitions; macro planning; congestion prediction and Rent's rule; the hand-off package |

**Reading order.** 01 gives the theory (synthesis as a compiler), 02 the specification it optimizes against, 03 the instruction set it compiles to, 04 the run you actually drive, 05 the bridge to the backend. A reader who has never run a tool should read 02 → 01 → 03 → 04.

---

⬅ prev [03 · Frontend RTL and Verification](../03_Frontend_RTL_and_Verification/00_Index.md) · [Root Index](../Index.md) · [Flow Overview](../Chip_Design_Flow_Overview.md) · next ➡ [05 · Backend (Physical Design)](../05_Backend_Physical_Design/00_Index.md)
