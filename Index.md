# Hardware Design — Index

> A senior-level technical notebook for digital IC design, CPU architecture, and SystemVerilog verification interviews.
> Target roles: RTL design engineer, physical design engineer, STA/signoff engineer, DFT engineer, verification engineer, CPU architect.
> Style: deep theory + gate-level detail + worked examples + interview-ready tradeoffs.
>
> **27 pages** organized in 6 thematic sections.

---

## Section 1 — Fundamentals

Device physics and basic building blocks. These pages establish the transistor-level foundation that everything else builds on.

| Page | Coverage |
|------|----------|
| [CMOS Fundamentals](Fundamentals/CMOS_Fundamentals.md) | MOSFET operation, CMOS inverter VTC, noise margins, propagation delay, latch-up, ESD, FinFET |
| [Fabrication Process](Fundamentals/Fabrication_Process.md) | Wafer prep, lithography (DUV/EUV), etching, ion implantation, CMP, FinFET, BEOL, yield/DFM |
| [Basic Knowledge](Fundamentals/Basic_Knowledge.md) | Combinational/sequential logic: MUX, Shannon expansion, encoders, decoders, basic building blocks |
| [Adders](Fundamentals/Adders.md) | Half adder through Kogge-Stone, carry-lookahead, carry-select, gate-level architectures |
| [Floating Point](Fundamentals/Floating_Point.md) | IEEE 754, bias encoding, rounding modes, FPU arithmetic operations |

## Section 2 — Architecture

System-level design: CPU pipelines, memory hierarchies, and on-chip bus protocols.

| Page | Coverage |
|------|----------|
| [CPU Architecture](Architecture/CPU_Architecture.md) | 5-stage pipeline, hazards, forwarding, branch prediction, Tomasulo, superscalar, MESI, virtual memory |
| [Memory](Architecture/Memory.md) | 6T SRAM transistor-level, DRAM, ROM, flash, memory controllers |
| [AHB / AXI / APB](Architecture/AHB_AXI_APB.md) | AMBA bus protocols: APB state machines, AHB pipelining/bursts, AXI4 complete spec, interconnect |

## Section 3 — Implementation

The RTL-to-GDSII EDA flow: synthesis, place-and-route, timing signoff, testing, and formal verification.

| Page | Coverage |
|------|----------|
| [Synthesis and Optimization](Implementation/Synthesis_and_Optimization.md) | RTL-to-gates, SDC constraints, technology mapping, timing closure, area optimization |
| [Physical Design (PnR)](Implementation/Physical_Design.md) | Floorplanning, placement, CTS, routing, physical verification signoff |
| [Static Timing Analysis](Implementation/STA.md) | Cell delay modeling, setup/hold derivation, CRPR, OCV/AOCV/POCV, SDC walkthrough |
| [DFT and ATPG](Implementation/DFT_and_ATPG.md) | Scan chains, fault models, ATPG algorithms, at-speed testing, test compression, logic BIST |
| [Formal Verification](Implementation/Formal_Verification.md) | SAT/BDD, LEC equivalence checking, model checking, CDC formal, connectivity verification |
| [IC Packaging](Implementation/IC_Packaging.md) | Wire bonding, flip chip, WLP, 2.5D/3D stacking, chiplets, HBM, UCIe |

## Section 4 — Clocking and Signals

Clock generation, clock domain crossing, and signal integrity — the cross-cutting concerns that span implementation and signoff.

| Page | Coverage |
|------|----------|
| [Clock Division](Clocking_and_Signals/Clock_Division.md) | Even/odd division, non-integer dividers, fractional techniques, gate-level circuits |
| [Asynchronous Circuit Design](Clocking_and_Signals/Async_Circuit_Design.md) | Metastability physics, synchronizers, async FIFO design, handshake protocols, CDC verification |
| [Signal Integrity and Reliability](Clocking_and_Signals/Signal_Integrity_Reliability.md) | Crosstalk, EM, IR drop, power grid design, thermal analysis, reliability signoff |

## Section 5 — Power

Power analysis, reduction, and signoff across all abstraction levels. Note: `Low_Power_Design.md` covers architecture-level power techniques that complement the analysis/signoff pages.

| Page | Coverage |
|------|----------|
| [Power Fundamentals](Power/Power_Fundamentals.md) | Switching/short-circuit/leakage physics, TDP, power density, technology scaling |
| [Power Reduction Techniques](Power/Power_Reduction_Techniques.md) | Optimization across system, micro-arch, RTL, gate, circuit, physical levels |
| [Power Analysis and Signoff](Power/Power_Analysis_and_Signoff.md) | RTL/gate/transistor-level analysis, PrimeTime PX, Voltus, activity estimation, signoff flow |
| [UPF Power Intent](Power/UPF_Power_Intent.md) | IEEE 1801: power domains, retention, isolation, level shifters, power switches |
| [Low Power Design](Power/Low_Power_Design.md) | Clock gating, DVFS, power gating, multi-Vt, body biasing — system-level techniques |

## Section 6 — SystemVerilog

The language and methodology for design and verification: types, processes, OOP, assertions, and testbench architecture.

| Page | Coverage |
|------|----------|
| [Data Types and Basics](SystemVerilog/Data_Types_and_Basics.md) | 2-state vs 4-state, packed/unpacked arrays, structs, unions, enums, casting |
| [Procedural and Processes](SystemVerilog/Procedural_and_Processes.md) | Event regions, always blocks, fork/join variants, simulation scheduling |
| [OOP and Randomization](SystemVerilog/OOP_and_Randomization.md) | Classes, inheritance, polymorphism, constraint-based randomization, coverage-driven |
| [Assertions and Coverage](SystemVerilog/Assertions_and_Coverage.md) | Immediate/concurrent SVA, properties, functional and code coverage strategies |
| [IPC and Verification](SystemVerilog/IPC_and_Verification.md) | Mailbox, semaphores, testbench architecture, UVM-style patterns |

---

## Reading Order

**For a digital design / RTL interview:**

CMOS Fundamentals → Basic Knowledge → Adders → Floating Point → CPU Architecture → Synthesis → STA → SystemVerilog basics.

**For a physical design / STA interview:**

CMOS Fundamentals → Synthesis → Physical Design → STA → Signal Integrity → Power Analysis → Clock Division.

**For a verification / SystemVerilog interview:**

Data Types → Procedural → OOP → Assertions → IPC and Verification → Formal Verification → Async/CDC.

**For a DFT / test engineer interview:**

CMOS Fundamentals → Basic Knowledge → DFT and ATPG → STA → Formal Verification.

**For a power-aware design interview:**

Power Fundamentals → Low Power Design → Power Reduction Techniques → UPF → Power Analysis and Signoff → STA.

---

**Up:** [../README.md](../README.md)
