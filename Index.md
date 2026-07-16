# Hardware Design — Index (chip-design-flow order)

> A senior-level technical notebook for digital IC design, CPU architecture, and verification.
> Target roles: RTL design, physical design, STA/signoff, DFT, verification, CPU architect.
> Style: deep theory + gate-level detail + worked examples + interview-ready tradeoffs.
>
> **Organized by the chip design flow.** The folders are numbered `00 → 07` in flow order — early PPA/architecture, frontend RTL + verification, synthesis, backend, signoff, manufacturing. Power is kept together as a cross-cutting track (02). Pages inside each folder are numbered in reading order, and every folder has its own `00_Index`. **Start with the [Chip Design Flow Overview](Chip_Design_Flow_Overview.md)** — it maps every stage, hand-off, and iteration loop.
>
> **47 flow pages** (+10 interview-prep banks).

---

## ▶ [Chip Design Flow Overview](Chip_Design_Flow_Overview.md) — read this first

The spine: spec → architecture → RTL → synthesis → backend → signoff → silicon, with the hand-off contract and cost-of-late-change at each boundary.

---

## [00 · Fundamentals](00_Fundamentals/00_Index.md)
*Device physics, logic, and arithmetic the rest of the flow assumes.*

| Page | Coverage |
|------|----------|
| [01 · CMOS Fundamentals](00_Fundamentals/01_CMOS_Fundamentals.md) | MOSFET, inverter VTC, noise margins, delay, logic families, I/O signaling standards (LVCMOS/SSTL/POD/LVDS/CML), latch-up, ESD, FinFET, 6T SRAM, leakage |
| [02 · Logic Building Blocks](00_Fundamentals/02_Logic_Building_Blocks.md) | MUX, Shannon expansion, encoders/decoders, latch vs FF (transistor-level), metastability, FSMs, gray code, hazards, FIFO depth |
| [03 · Adders and Multipliers](00_Fundamentals/03_Adders_and_Multipliers.md) | half/full adder → CLA, carry-select, carry-skip, prefix (Kogge-Stone), CSA, Booth, Wallace/Dadda |
| [04 · Floating Point](00_Fundamentals/04_Floating_Point.md) | IEEE-754, add/mul pipelines, GRS + rounding modes, SRT/Newton-Raphson/Goldschmidt division, FMA microarchitecture, AI formats (BF16/FP8/MX) |

---

## [01 · Architecture and PPA](01_Architecture_and_PPA/00_Index.md)
*Explore the microarchitecture; model performance; budget power and area — before RTL.*

| Page | Coverage |
|------|----------|
| [01 · Performance Modeling & DSE](01_Architecture_and_PPA/01_Performance_Modeling_and_DSE.md) | modeling-fidelity ladder, CPI stack, gem5/SystemC-TLM, SimPoint, design-space exploration, PPA tradeoffs |
| [02 · Full-Chip Modeling](01_Architecture_and_PPA/02_Full_Chip_Modeling.md) | full-chip hierarchical power+perf modeling: CPU core→cluster→SoC/DDR, GPU SM→GPC→chip, NPU PE→array→core→chip→pod, contention/overlap/DVFS/thermal coupling |
| [03 · CPU Architecture](01_Architecture_and_PPA/03_CPU_Architecture.md) | 5-stage pipeline, hazards, forwarding network, Tomasulo, superscalar, MESI, SMT, µop fusion, Spectre/Meltdown |
| [04 · RISC-V ISA](01_Architecture_and_PPA/04_RISC_V_ISA.md) | RV64GC, privilege modes, Sv39/48, Vector, Bitmanip, Hypervisor |
| [05 · OoO Execution](01_Architecture_and_PPA/05_OoO_Execution.md) | rename/RAT, ROB, issue queue wakeup-select, LSQ, misprediction recovery, exception pipeline, execution units |
| [06 · Branch Prediction Deep Dive](01_Architecture_and_PPA/06_Branch_Prediction_Deep_Dive.md) | BTB, gshare, TAGE-SC-L, perceptron, RAS, ITTAGE, fetch unit + FTQ |
| [07 · Cache Microarchitecture](01_Architecture_and_PPA/07_Cache_Microarchitecture.md) | pipeline, MSHR, write policy, prefetch, replacement, coherence implementation |
| [08 · TLB and Virtual Memory](01_Architecture_and_PPA/08_TLB_and_Virtual_Memory.md) | TLB format, page-table walker, VIPT, shootdown, superpages, 5-level |
| [09 · Memory](01_Architecture_and_PPA/09_Memory.md) | the bit-storage trade surface: 6T/8T/10T SRAM (SNM vs area vs speed), DRAM 1T1C density-vs-refresh, memory compiler, multi-port/register-file port cost, ECC coding theory, CAM/TCAM, compute-in-memory |
| [10 · DDR Controller](01_Architecture_and_PPA/10_DDR_Controller.md) | commands, timing, row-buffer, FR-FCFS, ECC, DDR5, LPDDR5X |
| [11 · AHB / AXI / APB](01_Architecture_and_PPA/11_AHB_AXI_APB.md) | AMBA bus protocols, AXI4 spec, CDC bridges, ATOP |
| [12 · ACE and CHI](01_Architecture_and_PPA/12_ACE_and_CHI.md) | cache-coherence protocols, snoop, CXL/PCIe |
| [13 · Network-on-Chip](01_Architecture_and_PPA/13_Network_on_Chip.md) | topology/bisection math, wormhole/VC flow control, router datapath + allocators, deadlock theory, CHI-over-mesh |
| [14 · Xiangshan CPU Design](01_Architecture_and_PPA/14_Xiangshan_CPU_Design.md) | open-source RISC-V OoO core case study (Nanhu/Kunminghu) |

---

## [02 · Power and Low-Power](02_Power_and_Low_Power/00_Index.md)
*Cross-cutting track: power is specified early, intended in RTL, implemented in synth/backend, signed off at the end.*

| Page | Coverage |
|------|----------|
| [01 · Power Fundamentals](02_Power_and_Low_Power/01_Power_Fundamentals.md) | switching/short-circuit/leakage physics, leakage-by-node breakdown, scaling/Dennard, sub-threshold swing |
| [02 · Block Activity and Power](02_Power_and_Low_Power/02_Block_Activity_and_Power.md) | per-block/per-mode modeling, RTL power, glitch, emulation power, on-die telemetry |
| [03 · Power Reduction Techniques](02_Power_and_Low_Power/03_Power_Reduction_Techniques.md) | low-power flow, clock gating, DVFS, power gating + retention, multi-Vt, body biasing, operand isolation |
| [04 · UPF Power Intent](02_Power_and_Low_Power/04_UPF_Power_Intent.md) | IEEE 1801, domains, retention, isolation, level shifters, switches, PST |
| [05 · Power Analysis and Signoff](02_Power_and_Low_Power/05_Power_Analysis_and_Signoff.md) | PrimeTime PX/Voltus flows, activity annotation, IR/EM, glitch, peak/di-dt, thermal, backside power |

---

## [03 · Frontend RTL and Verification](03_Frontend_RTL_and_Verification/00_Index.md)
*Write synthesizable RTL; prove it correct (dynamic + static).*

| Page | Coverage |
|------|----------|
| [01 · RTL Design Methodology](03_Frontend_RTL_and_Verification/01_RTL_Design_Methodology.md) | synchronous discipline, reset architecture, clocking, datapath/control split, synthesis-safe coding |
| [02 · Data Types and Basics](03_Frontend_RTL_and_Verification/02_Data_Types_and_Basics.md) | 2/4-state, arrays, structs, enums, casting |
| [03 · Procedural, Processes, and IPC](03_Frontend_RTL_and_Verification/03_Procedural_Processes_and_IPC.md) | event regions, always blocks, fork/join, scheduling, mailbox/semaphore/events |
| [04 · Clock Division and Switching](03_Frontend_RTL_and_Verification/04_Clock_Division_and_Switching.md) | even/odd/fractional dividers, div-3 FSM, programmable divider RTL, glitch-free switching, clock MUX |
| [05 · PLL, DLL, and Clock Distribution](03_Frontend_RTL_and_Verification/05_PLL_DLL_and_Clock_Distribution.md) | PFD/charge pump/VCO, lock, jitter, DLL vs PLL, H-tree/mesh distribution |
| [06 · Async Design and CDC](03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md) | metastability/MTBF, synchronizers, async FIFO, handshakes, CDC |
| [07 · Lint, CDC & RDC Signoff](03_Frontend_RTL_and_Verification/07_Lint_CDC_RDC_Signoff.md) | static lint, structural+functional CDC, reset-domain crossing |
| [08 · OOP and Randomization](03_Frontend_RTL_and_Verification/08_OOP_and_Randomization.md) | classes, polymorphism, constraint randomization |
| [09 · Assertions and Coverage](03_Frontend_RTL_and_Verification/09_Assertions_and_Coverage.md) | immediate/concurrent SVA, functional coverage mechanics, code coverage |
| [10 · UVM Methodology](03_Frontend_RTL_and_Verification/10_UVM_Methodology.md) | components, phasing, sequences, factory, config_db, TLM, RAL, complete AXI4-Lite testbench |
| [11 · Verification Planning & Coverage Closure](03_Frontend_RTL_and_Verification/11_Verification_Planning_and_Coverage_Closure.md) | vplan, coverage taxonomy, CDV cycle, closure loop, sign-off criteria |
| [12 · Formal Verification](03_Frontend_RTL_and_Verification/12_Formal_Verification.md) | SAT/BDD, LEC, model checking, CDC formal, connectivity |
| [13 · Gate-Level Sim & Emulation](03_Frontend_RTL_and_Verification/13_Gate_Level_Sim_and_Emulation.md) | GLS (zero-delay/SDF, X-prop), emulation, FPGA prototyping |

---

## [04 · Synthesis](04_Synthesis/00_Index.md)
*RTL → gate netlist under constraints.*

| Page | Coverage |
|------|----------|
| [01 · Synthesis and Optimization](04_Synthesis/01_Synthesis_and_Optimization.md) | RTL-to-gates, technology mapping, timing closure, area/power optimization |
| [02 · Constraints (SDC)](04_Synthesis/02_Constraints_SDC.md) | clocks, generated clocks, I/O delay, the four exceptions, DRV, complete worked SDC, MCMM discipline |

---

## [05 · Backend (Physical Design)](05_Backend_Physical_Design/00_Index.md)
*Gate netlist → layout.*

| Page | Coverage |
|------|----------|
| [01 · Physical Design (PnR)](05_Backend_Physical_Design/01_Physical_Design.md) | floorplan, placement, CTS, routing, ECO, advanced nodes; hand-off to PV/SI signoff |
| [02 · Signal Integrity and Reliability](05_Backend_Physical_Design/02_Signal_Integrity_Reliability.md) | crosstalk, EM, IR drop + grid models, power grid design, thermal, aging, antenna |

---

## [06 · Signoff](06_Signoff/00_Index.md)
*Prove it's correct before tape-out.*

| Page | Coverage |
|------|----------|
| [01 · Static Timing Analysis](06_Signoff/01_STA.md) | cell delay, setup/hold, CRPR, OCV/AOCV/POCV, clock-gating checks, SDC walkthrough |
| [02 · DFT and ATPG](06_Signoff/02_DFT_and_ATPG.md) | scan, fault models, ATPG, at-speed, compression, memory BIST + March, AI-accelerator DFT |
| [03 · Physical Verification (DRC/LVS)](06_Signoff/03_Physical_Verification_DRC_LVS.md) | DRC, LVS graph-compare, ERC, antenna, density/fill, DFM |

---

## [07 · Manufacturing and Bring-up](07_Manufacturing_and_Bringup/00_Index.md)
*Fab, package, tape-out, first silicon.*

| Page | Coverage |
|------|----------|
| [01 · Fabrication Process](07_Manufacturing_and_Bringup/01_Fabrication_Process.md) | wafer→litho (DUV/EUV)→etch→implant→CMP→FinFET/GAA→BEOL, yield/DFM |
| [02 · IC Packaging](07_Manufacturing_and_Bringup/02_IC_Packaging.md) | wire bond, flip chip, WLP, 2.5D/3D, chiplets, HBM, UCIe, Foveros/SoIC |
| [03 · Tapeout & Post-Silicon Bring-up](07_Manufacturing_and_Bringup/03_Tapeout_and_Post_Silicon_Bringup.md) | GDSII hand-off, mask/OPC, respin economics, bring-up, shmoo, debug, yield ramp |

---

## [Interview Prep](interview_prep/00_Index.md)

Per-folder Q&A consolidated out of the topic pages above, plus two cross-cutting banks.

| Page | Coverage |
|------|----------|
| [00 — Fundamentals Questions](interview_prep/00_Fundamentals_Questions.md) | CMOS/logic/adders/FP interview Q&A moved out of 00_Fundamentals/ |
| [01 — Architecture and PPA Questions](interview_prep/01_Architecture_and_PPA_Questions.md) | CPU/cache/memory/NoC/bus interview Q&A moved out of 01_Architecture_and_PPA/ |
| [02 — Power and Low-Power Questions](interview_prep/02_Power_and_Low_Power_Questions.md) | power/UPF/DVFS interview Q&A + low-power scenario drills |
| [03 — Frontend RTL and Verification Questions](interview_prep/03_Frontend_RTL_and_Verification_Questions.md) | RTL/UVM/CDC/formal interview Q&A moved out of 03_Frontend_RTL_and_Verification/ |
| [04 — Synthesis Questions](interview_prep/04_Synthesis_Questions.md) | synthesis/SDC interview Q&A moved out of 04_Synthesis/ |
| [05 — Backend Physical Design Questions](interview_prep/05_Backend_Physical_Design_Questions.md) | PnR/SI interview Q&A moved out of 05_Backend_Physical_Design/ |
| [06 — Signoff Questions](interview_prep/06_Signoff_Questions.md) | STA/DFT interview Q&A moved out of 06_Signoff/ |
| [07 — Manufacturing and Bring-up Questions](interview_prep/07_Manufacturing_and_Bringup_Questions.md) | fab/packaging/bring-up interview Q&A moved out of 07_Manufacturing_and_Bringup/ |
| [RTL Coding Questions](interview_prep/08_RTL_Coding_Questions.md) | whiteboard canon with full SystemVerilog solutions |
| [Hardware Interview Questions](interview_prep/09_Hardware_Interview_Questions.md) | worked numeric problems + snap answers per domain |
