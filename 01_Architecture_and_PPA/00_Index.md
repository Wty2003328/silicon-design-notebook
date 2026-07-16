# 01 · Architecture and PPA — Folder Index

*Explore the microarchitecture; model performance; budget power and area — before RTL (register-transfer level).*

This folder is organized as a **hybrid of chip types and shared parts**. The *chip types* — [CPU](02_CPU/00_Index.md), [GPU](05_GPU/00_Index.md), [NPU](06_NPU/00_Index.md) — are the complete machines, each with its own microarchitecture. The *shared parts* — [Modeling](01_Modeling/00_Index.md), [Memory](03_Memory/00_Index.md), [Interconnect](04_Interconnect/00_Index.md), [Simulators](07_Simulators/00_Index.md) — are the building blocks and tools every chip type reuses. Read a chip-type folder to understand a machine end to end; read a shared-part folder to understand a block that appears in all of them.

---

## [01 · Modeling](01_Modeling/00_Index.md) — the PPA methodology *(shared part)*

| # | Page | Coverage |
|---|------|----------|
| 01 | [Performance Modeling & DSE](01_Modeling/01_Performance_Modeling_and_DSE.md) | modeling-fidelity ladder, CPI stack, gem5/SystemC-TLM, SimPoint, design-space exploration, PPA tradeoffs |
| 02 | [Full-Chip Modeling](01_Modeling/02_Full_Chip_Modeling.md) | full-chip hierarchical power+perf modeling: CPU core→cluster→SoC/DDR, GPU SM→GPC→chip, NPU PE→array→core→chip→pod, contention/overlap/DVFS/thermal coupling |

## [02 · CPU](02_CPU/00_Index.md) — the general-purpose core *(chip type)*

| # | Page | Coverage |
|---|------|----------|
| 01 | [CPU Architecture](02_CPU/01_CPU_Architecture.md) | 5-stage pipeline, hazards, forwarding network, Tomasulo, superscalar, MESI, SMT, µop fusion, Spectre/Meltdown |
| 02 | [RISC-V ISA](02_CPU/02_RISC_V_ISA.md) | RV64GC, privilege modes, Sv39/48, Vector, Bitmanip, Hypervisor |
| 03 | [OoO Execution](02_CPU/03_OoO_Execution.md) | rename/RAT, ROB, issue queue wakeup-select, LSQ, misprediction recovery, exception pipeline, execution units |
| 04 | [Branch Prediction Deep Dive](02_CPU/04_Branch_Prediction_Deep_Dive.md) | BTB, gshare, TAGE-SC-L, perceptron, RAS, ITTAGE, fetch unit + FTQ |
| 05 | [Xiangshan CPU Design](02_CPU/05_Xiangshan_CPU_Design.md) | open-source RISC-V OoO core case study (Nanhu/Kunminghu) |

## [03 · Memory](03_Memory/00_Index.md) — the on-/off-chip storage hierarchy *(shared part)*

| # | Page | Coverage |
|---|------|----------|
| 01 | [Cache Microarchitecture](03_Memory/01_Cache_Microarchitecture.md) | pipeline, MSHR, write policy, prefetch, replacement, coherence implementation |
| 02 | [TLB and Virtual Memory](03_Memory/02_TLB_and_Virtual_Memory.md) | TLB format, page-table walker, VIPT, shootdown, superpages, 5-level |
| 03 | [Memory](03_Memory/03_Memory.md) | the bit-storage trade surface: 6T/8T/10T SRAM (SNM vs area vs speed), DRAM 1T1C density-vs-refresh, memory compiler, multi-port/register-file port cost, ECC coding theory, CAM/TCAM, compute-in-memory |
| 04 | [DDR Controller](03_Memory/04_DDR_Controller.md) | commands, timing, row-buffer, FR-FCFS, ECC, DDR5, LPDDR5X |

## [04 · Interconnect](04_Interconnect/00_Index.md) — moving data on and off chip *(shared part)*

| # | Page | Coverage |
|---|------|----------|
| 01 | [AHB / AXI / APB](04_Interconnect/01_AHB_AXI_APB.md) | AMBA bus protocols, AXI4 spec, CDC bridges, ATOP |
| 02 | [ACE and CHI](04_Interconnect/02_ACE_and_CHI.md) | cache-coherence protocols, snoop, CXL/PCIe |
| 03 | [Network-on-Chip](04_Interconnect/03_Network_on_Chip.md) | topology/bisection math, wormhole/VC flow control, router datapath + allocators, deadlock theory, CHI-over-mesh |

## [05 · GPU](05_GPU/00_Index.md) — the throughput / latency-hiding machine *(chip type)*

| # | Page | Coverage |
|---|------|----------|
| 01 | [GPU Architecture](05_GPU/01_GPU_Architecture.md) | SIMT + massive multithreading, the SM as a hardware block, warp divergence, memory coalescing, occupancy, HBM hierarchy (hardware-design lens; cross-links the AI-infra notebook) |

## [06 · NPU](06_NPU/00_Index.md) — dataflow / spatial accelerators *(chip type)*

| # | Page | Coverage |
|---|------|----------|
| 01 | [NPU / Dataflow Accelerators](06_NPU/01_NPU_Accelerators.md) | why spatial/dataflow beats von-Neumann for GEMM: the systolic array as hardware, WS/OS/RS dataflow as an energy lever, scratchpad + mapping, roofline, scale-out |

## [07 · Simulators](07_Simulators/00_Index.md) — how every non-silicon number is produced *(shared part)*

| # | Page | Coverage |
|---|------|----------|
| — | [Simulators folder](07_Simulators/00_Index.md) | methodology, gem5, DRAM (DRAMSim/Ramulator), GPU (GPGPU-Sim/Accel-Sim), accelerator/NPU (SCALE-Sim/Timeloop/NeuSim), other-architecture simulators, analytical/roofline models |

---

⬅ prev [00 · Fundamentals](../00_Fundamentals/00_Index.md) · [Root Index](../Index.md) · [Flow Overview](../Chip_Design_Flow_Overview.md) · next ➡ [02 · Power and Low-Power](../02_Power_and_Low_Power/00_Index.md)
