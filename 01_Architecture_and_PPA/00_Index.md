# 01 · Architecture and PPA — Folder Index

*Explore the microarchitecture; model performance; budget power and area — before RTL.*

| # | Page | Coverage |
|---|------|----------|
| 01 | [Performance Modeling & DSE](01_Performance_Modeling_and_DSE.md) | modeling-fidelity ladder, CPI stack, gem5/SystemC-TLM, SimPoint, design-space exploration, PPA tradeoffs |
| 02 | [Full-Chip Modeling](02_Full_Chip_Modeling.md) | full-chip hierarchical power+perf modeling: CPU core→cluster→SoC/DDR, GPU SM→GPC→chip, NPU PE→array→core→chip→pod, contention/overlap/DVFS/thermal coupling |
| 03 | [CPU Architecture](03_CPU_Architecture.md) | 5-stage pipeline, hazards, forwarding network, Tomasulo, superscalar, MESI, SMT, µop fusion, Spectre/Meltdown |
| 04 | [RISC-V ISA](04_RISC_V_ISA.md) | RV64GC, privilege modes, Sv39/48, Vector, Bitmanip, Hypervisor |
| 05 | [OoO Execution](05_OoO_Execution.md) | rename/RAT, ROB, issue queue wakeup-select, LSQ, misprediction recovery, exception pipeline, execution units |
| 06 | [Branch Prediction Deep Dive](06_Branch_Prediction_Deep_Dive.md) | BTB, gshare, TAGE-SC-L, perceptron, RAS, ITTAGE, fetch unit + FTQ |
| 07 | [Cache Microarchitecture](07_Cache_Microarchitecture.md) | pipeline, MSHR, write policy, prefetch, replacement, coherence implementation |
| 08 | [TLB and Virtual Memory](08_TLB_and_Virtual_Memory.md) | TLB format, page-table walker, VIPT, shootdown, superpages, 5-level |
| 09 | [Memory](09_Memory.md) | 6T/8T/10T SRAM (transistor-level), DRAM cell + refresh, memory compiler, multi-port SRAM, register files, ECC, CAM/TCAM, CIM |
| 10 | [DDR Controller](10_DDR_Controller.md) | commands, timing, row-buffer, FR-FCFS, ECC, DDR5, LPDDR5X |
| 11 | [AHB / AXI / APB](11_AHB_AXI_APB.md) | AMBA bus protocols, AXI4 spec, CDC bridges, ATOP |
| 12 | [ACE and CHI](12_ACE_and_CHI.md) | cache-coherence protocols, snoop, CXL/PCIe |
| 13 | [Network-on-Chip](13_Network_on_Chip.md) | topology/bisection math, wormhole/VC flow control, router datapath + allocators, deadlock theory, CHI-over-mesh |
| 14 | [Xiangshan CPU Design](14_Xiangshan_CPU_Design.md) | open-source RISC-V OoO core case study (Nanhu/Kunminghu) |

---

⬅ prev [00 · Fundamentals](../00_Fundamentals/00_Index.md) · [Root Index](../Index.md) · [Flow Overview](../Chip_Design_Flow_Overview.md) · next ➡ [02 · Power and Low-Power](../02_Power_and_Low_Power/00_Index.md)
