# Hardware Design — Index (chip-design-flow order)

> A senior-level technical notebook for digital IC design, CPU architecture, and verification.
> Target roles: RTL design, physical design, STA/signoff, DFT, verification, CPU architect.
> Style: deep theory + gate-level detail + worked examples + interview-ready tradeoffs.
>
> **Organized by the chip design flow.** The folders are numbered `00 → 07` in flow order — early PPA/architecture, frontend RTL + verification, synthesis, backend, signoff, manufacturing. Power is kept together as a cross-cutting track (02). **Start with the [Chip Design Flow Overview](Chip_Design_Flow_Overview.md)** — it maps every stage, hand-off, and iteration loop.
>
> **48 pages.** New pages added in the flow reorg are marked ⭐.

---

## ▶ [Chip Design Flow Overview](Chip_Design_Flow_Overview.md) ⭐ — read this first

The spine: spec → architecture → RTL → synthesis → backend → signoff → silicon, with the hand-off contract and cost-of-late-change at each boundary.

---

## 00 · Fundamentals
*Device physics, logic, and arithmetic the rest of the flow assumes.*

| Page | Coverage |
|------|----------|
| [CMOS Fundamentals](00_Fundamentals/CMOS_Fundamentals.md) | MOSFET, inverter VTC, noise margins, delay, logic families, **I/O signaling standards (LVCMOS/SSTL/POD/LVDS/CML)**, latch-up, ESD, FinFET, 6T SRAM, leakage |
| [Basic Knowledge (Logic Building Blocks)](00_Fundamentals/Basic_Knowledge.md) | MUX, Shannon expansion, encoders/decoders, latch vs FF (transistor-level), metastability, FSMs, gray code, hazards, FIFO depth |
| [Adders](00_Fundamentals/Adders.md) | Half/full adder → CLA, carry-select, carry-skip, prefix (Kogge-Stone), CSA, Booth, Wallace/Dadda |
| [Floating Point](00_Fundamentals/Floating_Point.md) | IEEE-754, add/mul pipelines, GRS rounding, SRT division, FMA microarchitecture |

## 01 · Architecture and PPA
*Explore the microarchitecture; model performance; budget power and area — before RTL.*

| Page | Coverage |
|------|----------|
| [Performance Modeling & DSE](01_Architecture_and_PPA/Performance_Modeling_and_DSE.md) ⭐ | modeling-fidelity ladder, CPI stack, gem5/SystemC-TLM, SimPoint, design-space exploration, PPA tradeoffs |
| [Full-Chip Modeling](01_Architecture_and_PPA/Full_Chip_Modeling.md) ⭐ | full-chip hierarchical power+perf modeling: CPU core→cluster→SoC/DDR, GPU SM→GPC→chip, NPU PE→array→core→chip→pod, contention/overlap/DVFS/thermal coupling |
| [CPU Architecture](01_Architecture_and_PPA/CPU_Architecture.md) | 5-stage pipeline, hazards, forwarding, Tomasulo, superscalar, MESI, SMT, Spectre/Meltdown |
| [OoO Execution](01_Architecture_and_PPA/OoO_Execution.md) | rename, ROB, issue queue, LSQ, misprediction recovery, wakeup/select |
| [Branch Prediction Deep Dive](01_Architecture_and_PPA/Branch_Prediction_Deep_Dive.md) | BTB, gshare, TAGE-SC-L, perceptron, RAS, ITTAGE |
| [Cache Microarchitecture](01_Architecture_and_PPA/Cache_Microarchitecture.md) | pipeline, MSHR, write policy, prefetch, replacement, coherence |
| [TLB and Virtual Memory](01_Architecture_and_PPA/TLB_and_Virtual_Memory.md) | TLB format, page-table walker, VIPT, shootdown, superpages, 5-level |
| [RISC-V ISA](01_Architecture_and_PPA/RISC_V_ISA.md) | RV64GC, privilege modes, Sv39/48, Vector, Bitmanip, Hypervisor |
| [Xiangshan CPU Design](01_Architecture_and_PPA/Xiangshan_CPU_Design.md) | open-source RISC-V OoO core case study (Nanhu/Kunminghu) |
| [Memory](01_Architecture_and_PPA/Memory.md) | 6T/8T/10T SRAM (transistor-level), DRAM, async FIFO, multi-port, register file, CAM/TCAM, CIM |
| [DDR Controller](01_Architecture_and_PPA/DDR_Controller.md) | commands, timing, row-buffer, FR-FCFS, ECC, DDR5, LPDDR5X |
| [Network-on-Chip](01_Architecture_and_PPA/Network_on_Chip.md) | topology/bisection math, wormhole/VC flow control, **router datapath + allocators**, deadlock theory, CHI-over-mesh |
| [AHB / AXI / APB](01_Architecture_and_PPA/AHB_AXI_APB.md) | AMBA bus protocols, AXI4 spec, CDC bridges, ATOP |
| [ACE and CHI](01_Architecture_and_PPA/ACE_and_CHI.md) | cache-coherence protocols, snoop, CXL/PCIe |

## 02 · Power and Low-Power *(cross-cutting track)*
*Power is specified early, intended in RTL, implemented in synth/backend, signed off at the end — kept together here.*

| Page | Coverage |
|------|----------|
| [Power Fundamentals](02_Power_and_Low_Power/Power_Fundamentals.md) | switching/short-circuit/leakage physics, scaling, sub-threshold swing |
| [Block Activity and Power](02_Power_and_Low_Power/Block_Activity_and_Power.md) | per-block/per-mode modeling, RTL power, glitch, emulation power, on-die telemetry |
| [Power Reduction Techniques](02_Power_and_Low_Power/Power_Reduction_Techniques.md) | clock gating, DVFS, power gating, multi-Vt, body biasing, operand isolation |
| [Low Power Design](02_Power_and_Low_Power/Low_Power_Design.md) | the interview bible: ICG, power-gating switches/retention, debug drills |
| [UPF Power Intent](02_Power_and_Low_Power/UPF_Power_Intent.md) | IEEE 1801, domains, retention, isolation, level shifters, switches, PST |
| [Power Analysis and Signoff](02_Power_and_Low_Power/Power_Analysis_and_Signoff.md) | PrimeTime PX/Voltus, IR/EM, glitch, peak/di-dt, backside power |

## 03 · Frontend RTL and Verification
*Write synthesizable RTL; prove it correct (dynamic + static).*

| Page | Coverage |
|------|----------|
| [RTL Design Methodology](03_Frontend_RTL_and_Verification/RTL_Design_Methodology.md) ⭐ | synchronous discipline, reset architecture, clocking, datapath/control split, synthesis-safe coding |
| [Data Types and Basics](03_Frontend_RTL_and_Verification/Data_Types_and_Basics.md) | 2/4-state, arrays, structs, enums, casting |
| [Procedural and Processes](03_Frontend_RTL_and_Verification/Procedural_and_Processes.md) | event regions, always blocks, fork/join, scheduling |
| [OOP and Randomization](03_Frontend_RTL_and_Verification/OOP_and_Randomization.md) | classes, polymorphism, constraint randomization |
| [Clock Division](03_Frontend_RTL_and_Verification/Clock_Division.md) | even/odd/fractional dividers, PLL/DLL, glitch-free switching |
| [Asynchronous Circuit Design](03_Frontend_RTL_and_Verification/Async_Circuit_Design.md) | metastability/MTBF, synchronizers, async FIFO, handshakes, CDC |
| [Assertions and Coverage](03_Frontend_RTL_and_Verification/Assertions_and_Coverage.md) | immediate/concurrent SVA, functional+code coverage |
| [IPC and Verification](03_Frontend_RTL_and_Verification/IPC_and_Verification.md) | mailbox, semaphore, testbench architecture |
| [UVM Methodology](03_Frontend_RTL_and_Verification/UVM_Methodology.md) | components, phasing, sequences, factory, config_db, RAL |
| [Lint, CDC & RDC Signoff](03_Frontend_RTL_and_Verification/Lint_CDC_RDC_Signoff.md) ⭐ | static lint, structural+functional CDC, reset-domain crossing |
| [Gate-Level Sim & Emulation](03_Frontend_RTL_and_Verification/Gate_Level_Sim_and_Emulation.md) ⭐ | GLS (zero-delay/SDF, X-prop), emulation, FPGA prototyping |
| [Verification Planning & Coverage Closure](03_Frontend_RTL_and_Verification/Verification_Planning_and_Coverage_Closure.md) ⭐ | vplan, coverage taxonomy, closure loop, sign-off criteria |
| [Formal Verification](03_Frontend_RTL_and_Verification/Formal_Verification.md) | SAT/BDD, LEC, model checking, CDC formal, connectivity |

## 04 · Synthesis
*RTL → gate netlist under constraints.*

| Page | Coverage |
|------|----------|
| [Synthesis and Optimization](04_Synthesis/Synthesis_and_Optimization.md) | RTL-to-gates, mapping, timing closure, area/power opt |
| [Constraints (SDC)](04_Synthesis/Constraints_SDC.md) ⭐ | clocks, I/O delay, the four exceptions, MCMM, SDC discipline |

## 05 · Backend (Physical Design)
*Gate netlist → layout.*

| Page | Coverage |
|------|----------|
| [Physical Design (PnR)](05_Backend_Physical_Design/Physical_Design.md) | floorplan, placement, CTS, routing, physical signoff |
| [Signal Integrity and Reliability](05_Backend_Physical_Design/Signal_Integrity_Reliability.md) | crosstalk, EM, IR drop, power grid, thermal, aging, antenna |

## 06 · Signoff
*Prove it's correct before tape-out.*

| Page | Coverage |
|------|----------|
| [Static Timing Analysis](06_Signoff/STA.md) | cell delay, setup/hold, CRPR, OCV/AOCV/POCV, SDC walkthrough |
| [DFT and ATPG](06_Signoff/DFT_and_ATPG.md) | scan, fault models, ATPG, at-speed, compression, BIST, AI-accelerator DFT |
| [Physical Verification (DRC/LVS)](06_Signoff/Physical_Verification_DRC_LVS.md) ⭐ | DRC, LVS graph-compare, antenna, DFM/fill |

## 07 · Manufacturing and Bring-up
*Fab, package, tape-out, first silicon.*

| Page | Coverage |
|------|----------|
| [Fabrication Process](07_Manufacturing_and_Bringup/Fabrication_Process.md) | wafer→litho (DUV/EUV)→etch→implant→CMP→FinFET/GAA→BEOL, yield/DFM |
| [IC Packaging](07_Manufacturing_and_Bringup/IC_Packaging.md) | wire bond, flip chip, WLP, 2.5D/3D, chiplets, HBM, UCIe, Foveros/SoIC |
| [Tapeout & Post-Silicon Bring-up](07_Manufacturing_and_Bringup/Tapeout_and_Post_Silicon_Bringup.md) ⭐ | GDSII hand-off, mask/OPC, respin economics, bring-up, shmoo, debug, yield ramp |

## Interview Prep

| Page | Coverage |
|------|----------|
| [RTL Coding Questions](interview_prep/RTL_Coding_Questions.md) | whiteboard canon with full SystemVerilog solutions |
| [Hardware Interview Questions](interview_prep/Hardware_Interview_Questions.md) | worked numeric problems + snap answers per domain |

---

> *Reorg note: the old topic folders (Architecture/, Power/, Implementation/, SystemVerilog/, Clocking_and_Signals/, Fundamentals/) are now empty and can be deleted in your file explorer. See [REORG_PLAN](REORG_PLAN.md) for the full mapping.*
