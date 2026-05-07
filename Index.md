# Hardware Design — Index

> A senior-level technical notebook for digital IC design, CPU architecture, and SystemVerilog verification interviews.
> Target roles: RTL design engineer, physical design engineer, STA/signoff engineer, DFT engineer, verification engineer, CPU architect.
> Style: deep theory + gate-level detail + worked examples + interview-ready tradeoffs.
>
> **35 pages** organized in 6 thematic sections.

---

## Section 1 — Fundamentals

Device physics and basic building blocks. These pages establish the transistor-level foundation that everything else builds on.

| Page | Lines | Coverage |
|------|-------|----------|
| [CMOS Fundamentals](Fundamentals/CMOS_Fundamentals.md) | 1,079 | MOSFET operation, CMOS inverter VTC, noise margins, propagation delay, latch-up, ESD, FinFET |
| [Fabrication Process](Fundamentals/Fabrication_Process.md) | 1,127 | Wafer prep, lithography (DUV/EUV), etching, ion implantation, CMP, FinFET, BEOL, yield/DFM |
| [Basic Knowledge](Fundamentals/Basic_Knowledge.md) | 999 | Combinational/sequential logic: MUX, Shannon expansion, encoders, decoders, basic building blocks |
| [Adders](Fundamentals/Adders.md) | 878 | Half adder through Kogge-Stone, carry-lookahead, carry-select, gate-level architectures |
| [Floating Point](Fundamentals/Floating_Point.md) | 1,686 | IEEE 754, bias encoding, rounding modes, FPU pipeline, division algorithms |

## Section 2 — Architecture

CPU microarchitecture, ISA, memory subsystem, and SoC interconnect. This is the largest section, covering everything from RISC-V ISA to out-of-order execution to cache and DDR design.

| Page | Lines | Coverage |
|------|-------|----------|
| [RISC-V ISA](Architecture/RISC_V_ISA.md) | 1,223 | RV64G (RV64IMAFdc) instruction set, encoding formats, privilege modes, Sv39/Sv48 virtual memory |
| [CPU Architecture](Architecture/CPU_Architecture.md) | 1,731 | 5-stage pipeline, hazards, forwarding, branch prediction, Tomasulo, superscalar, MESI, fetch unit, SMT, exceptions |
| [OoO Execution](Architecture/OoO_Execution.md) | 905 | Full OoO datapath: rename, ROB, issue queue, LSQ, misprediction recovery, execution units, SMT |
| [Branch Prediction Deep Dive](Architecture/Branch_Prediction_Deep_Dive.md) | 986 | BTB, gshare, TAGE/TAGE-SC, perceptron, tournament, RAS, ITTAGE, fetch unit architecture |
| [Cache Microarchitecture](Architecture/Cache_Microarchitecture.md) | 966 | Cache pipeline, MSHR, write policy, prefetch engines, replacement policies, MESI/MOESI |
| [TLB and Virtual Memory](Architecture/TLB_and_Virtual_Memory.md) | 1,450 | TLB entry format, multi-level TLB, hardware page walker, VIPT, shootdown, superpages |
| [Xiangshan CPU Design](Architecture/Xiangshan_CPU_Design.md) | 1,165 | Open-source RISC-V OoO processor: Nanhu/Kunminghu, TAGE-SC, distributed IQ, LSU, cache hierarchy |
| [DDR Controller](Architecture/DDR_Controller.md) | 1,176 | DDR commands, timing parameters, row buffer management, FR-FCFS scheduling, ECC, DDR5 |
| [ACE and CHI](Architecture/ACE_and_CHI.md) | 971 | AMBA ACE/CHI coherence protocols, snoop channels, TrustZone, AXI ATOP |
| [Memory](Architecture/Memory.md) | 1,560 | 6T SRAM, DRAM, async FIFO, multi-port SRAM, register files, ECC, CAM/TCAM |
| [AHB / AXI / APB](Architecture/AHB_AXI_APB.md) | 1,538 | AMBA bus protocols, AXI4 complete spec, CDC bridges, TrustZone, AXI ATOP |

## Section 3 — Implementation

The RTL-to-GDSII EDA flow: synthesis, place-and-route, timing signoff, testing, and formal verification.

| Page | Lines | Coverage |
|------|-------|----------|
| [Synthesis and Optimization](Implementation/Synthesis_and_Optimization.md) | 1,918 | RTL-to-gates, SDC constraints, technology mapping, timing closure, area optimization |
| [Physical Design (PnR)](Implementation/Physical_Design.md) | 1,920 | Floorplanning, placement, CTS, routing, physical verification signoff |
| [Static Timing Analysis](Implementation/STA.md) | 1,341 | Cell delay modeling, setup/hold derivation, CRPR, OCV/AOCV/POCV, SDC walkthrough |
| [DFT and ATPG](Implementation/DFT_and_ATPG.md) | 1,890 | Scan chains, fault models, ATPG algorithms, at-speed testing, test compression, logic BIST |
| [Formal Verification](Implementation/Formal_Verification.md) | 1,065 | SAT/BDD, LEC equivalence checking, model checking, CDC formal, connectivity verification |
| [IC Packaging](Implementation/IC_Packaging.md) | 786 | Wire bonding, flip chip, WLP, 2.5D/3D stacking, chiplets, HBM, UCIe |

## Section 4 — Clocking and Signals

Clock generation, clock domain crossing, and signal integrity — the cross-cutting concerns that span implementation and signoff.

| Page | Lines | Coverage |
|------|-------|----------|
| [Clock Division](Clocking_and_Signals/Clock_Division.md) | 1,987 | Even/odd division, non-integer dividers, fractional techniques, gate-level circuits |
| [Asynchronous Circuit Design](Clocking_and_Signals/Async_Circuit_Design.md) | 1,501 | Metastability physics, synchronizers, async FIFO design, handshake protocols, CDC verification |
| [Signal Integrity and Reliability](Clocking_and_Signals/Signal_Integrity_Reliability.md) | 840 | Crosstalk, EM, IR drop, power grid design, thermal analysis, reliability signoff |

## Section 5 — Power

Power analysis, reduction, and signoff across all abstraction levels.

| Page | Lines | Coverage |
|------|-------|----------|
| [Power Fundamentals](Power/Power_Fundamentals.md) | 994 | Switching/short-circuit/leakage physics, TDP, power density, technology scaling |
| [Power Reduction Techniques](Power/Power_Reduction_Techniques.md) | 1,144 | Optimization across system, micro-arch, RTL, gate, circuit, physical levels |
| [Power Analysis and Signoff](Power/Power_Analysis_and_Signoff.md) | 1,249 | RTL/gate/transistor-level analysis, PrimeTime PX, Voltus, activity estimation, signoff flow |
| [UPF Power Intent](Power/UPF_Power_Intent.md) | 1,155 | IEEE 1801: power domains, retention, isolation, level shifters, power switches |
| [Low Power Design](Power/Low_Power_Design.md) | 1,155 | Clock gating, DVFS, power gating, multi-Vt, body biasing — system-level techniques |

## Section 6 — SystemVerilog

The language and methodology for design and verification: types, processes, OOP, assertions, and testbench architecture.

| Page | Lines | Coverage |
|------|-------|----------|
| [Data Types and Basics](SystemVerilog/Data_Types_and_Basics.md) | 928 | 2-state vs 4-state, packed/unpacked arrays, structs, unions, enums, casting |
| [Procedural and Processes](SystemVerilog/Procedural_and_Processes.md) | 820 | Event regions, always blocks, fork/join variants, simulation scheduling |
| [OOP and Randomization](SystemVerilog/OOP_and_Randomization.md) | 1,094 | Classes, inheritance, polymorphism, constraint-based randomization, coverage-driven |
| [Assertions and Coverage](SystemVerilog/Assertions_and_Coverage.md) | 963 | Immediate/concurrent SVA, properties, functional and code coverage strategies |
| [IPC and Verification](SystemVerilog/IPC_and_Verification.md) | 1,531 | Mailbox, semaphores, testbench architecture, UVM-style patterns |

---

## Reading Order

**For a CPU design / microarchitecture interview:**

RISC-V ISA → CPU Architecture → OoO Execution → Branch Prediction Deep Dive → Cache Microarchitecture → TLB and Virtual Memory → Xiangshan CPU Design

**For a digital design / RTL interview:**

CMOS Fundamentals → Basic Knowledge → Adders → Floating Point → CPU Architecture → RISC-V ISA → Synthesis → STA → SystemVerilog basics

**For a physical design / STA interview:**

CMOS Fundamentals → Synthesis → Physical Design → STA → Signal Integrity → Power Analysis → Clock Division

**For a verification / SystemVerilog interview:**

Data Types → Procedural → OOP → Assertions → IPC and Verification → Formal Verification → Async/CDC

**For a memory subsystem / SoC interconnect interview:**

Memory → Cache Microarchitecture → TLB and Virtual Memory → DDR Controller → AHB/AXI/APB → ACE and CHI

**For a DFT / test engineer interview:**

CMOS Fundamentals → Basic Knowledge → DFT and ATPG → STA → Formal Verification

**For a power-aware design interview:**

Power Fundamentals → Low Power Design → Power Reduction Techniques → UPF → Power Analysis and Signoff → STA

---

**Up:** [../README.md](../README.md)
