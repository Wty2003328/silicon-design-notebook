# Other Architecture Simulators — A Surveyed Catalog

> **Prerequisites:** [Simulation_Methodology](01_Simulation_Methodology.md) (the paradigm vocabulary — functional/timing/physical, the fidelity ladder, the discrete-event engine, trace- vs execution-driven — that every tool below instantiates).
> **Hands off to:** [Full_Chip_Modeling](../02_Full_Chip_Modeling.md) (composing these into a chip/system model with the perf→power→thermal loop and the tool map).

---

## 0. Why this page exists

The gem5 / DRAM / GPU / accelerator pages in this folder cover the tools an architect reaches for most, but the field is wider: manycore-CPU studies, on-chip networks, compute-in-memory, datacenter fabrics, chiplet integration, and RISC-V core bring-up each have their own established simulators, built at different rungs of the fidelity ladder for different reasons. This page is a **surveyed catalog** — one table plus short conceptual notes per category — that places each tool in the [Simulation_Methodology](01_Simulation_Methodology.md) taxonomy (which paradigm, what it models, what it is blind to) and says *when* to use it. It is deliberately breadth-first: the goal is to know what exists, what class it belongs to, and therefore how far to trust its numbers, not to re-derive the mechanisms already covered on the methodology page.

---

## 1. The catalog

Read a row as *"this tool is a [paradigm] model of [what], reach for it when [job]."* The **paradigm** column uses the vocabulary of [Simulation_Methodology §1–4](01_Simulation_Methodology.md): *functional* (correctness only), *analytical/mechanistic* (closed-form), *event-driven cycle-approximate*, *cycle-accurate*, *RTL*; and *trace-* vs *execution-driven*.

| Tool | Category | Paradigm | What it models | When to use |
|---|---|---|---|---|
| **Sniper** | Manycore CPU | Analytical **interval** core in an event-driven multicore engine; execution-driven (Pin) | x86 multicore: interval core model, cache hierarchy, coherence, NoC; McPAT power | Fast, reasonably accurate **manycore** studies where you need trends over 10s–100s of cores without cycle-accurate cost |
| **ZSim** | Manycore CPU | Event-driven cycle-approximate; **DBT** (Pin); **bound-weave** parallel; execution-driven | OoO x86 cores, caches, coherence at **1000-core** scale (user-level) | Thousand-core / large-cache-hierarchy DSE where simulation speed is the binding constraint |
| **MARSSx86** | Manycore CPU | Cycle-accurate, **full-system** (PTLsim OoO + QEMU); execution-driven | Cycle-accurate x86 core + memory, boots a real OS | Full-system x86 studies needing OS effects (largely superseded by gem5-x86, but historically the reference) |
| **BookSim 2.0** | NoC / interconnect | Cycle-accurate, standalone; synthetic-traffic or trace-driven | Topology, routing, VC flow control, allocators, router pipeline; latency-throughput curves | Pure **network** studies: comparing topologies/routers under synthetic load, isolated from a core model |
| **Garnet (3.0 / HeteroGarnet)** | NoC / interconnect | Cycle-accurate, **inside gem5-Ruby**; execution-driven | Detailed router microarchitecture carrying **real coherence traffic**; heterogeneous/clocked links | NoC studies that must see the *actual* coherence/message traffic of a running workload |
| **DSENT** | NoC / interconnect | Analytical **energy/area** model (not timing) | Per-component NoC energy & area (electrical + photonic) from technology + activity | The **power overlay** for BookSim/Garnet — pJ/flit-hop, not cycles |
| **DNN+NeuroSim** | Compute-in-memory | Analytical **circuit-level** (macro models, not event-driven) | ReRAM/SRAM/FeFET CIM: device→array→PE→tile→chip area/latency/energy + inference accuracy with device non-idealities | Evaluating a **CIM** accelerator's PPA *and* accuracy across device technologies and nodes |
| **NeuroXplorer / SANA-FE** | Neuromorphic | Event-driven (spike-driven) architecture models | Mapping SNNs onto tiled, NoC-connected neuromorphic chips; spike traffic, energy, latency | Architecture-level **neuromorphic** DSE (distinct from neuroscience SNN sims like NEST/Brian) |
| **SST** | Datacenter / HPC | **Parallel discrete-event (PDES)**, MPI-scalable; component/link/event | Whole HPC systems: network (Merlin), MPI motifs (Ember), memory (memHierarchy, Miranda), CPU (Vanadis), DRAM backends | **Scale-out** system co-design — millions of components across hundreds of CPUs |
| **ns-3** | Datacenter / network | Discrete-event, **packet-level**; execution/trace-driven | Full network stacks: TCP/IP, wireless, datacenter fabrics, congestion control | Network-protocol and datacenter-**fabric** studies (packets, not cycles) |
| **OMNeT++** | Datacenter / network | Component-based discrete-event framework (+ INET) | General DES; with INET, full network models; also buses/queues | A general **DES framework** when you build the model; networks via INET |
| **CHIPSIM** | Chiplet / 2.5D-3D | Event-driven **co-simulation** (perf + power + transient thermal) | Chiplet DNN systems: compute chiplets + **Network-on-Interposer** contention/pipelining, µs-granularity power → transient thermal | **Chiplet/NoI** architecture DSE where communication and thermal transients dominate |
| **HotSpot** | Thermal (all) | Analytical **compact RC thermal** solve (steady + transient) | Temperature map from floorplan + power map; grid/block model | The **thermal** stage co-simulated with any perf/power tool ([Full_Chip §5.4](../02_Full_Chip_Modeling.md)); 3D-ICE/PACT for 2.5D/3D |
| **Spike** | RISC-V core | **Functional** golden ISA model | Architectural state of RV binaries (no timing) | The **reference** for correctness / co-simulation tandem-checking against RTL |
| **Dromajo** | RISC-V core | Functional, built for **RTL co-simulation** | RV64GC state + commit log, checkpoint/restore, fast-forward | Co-verifying a core (BOOM/other) against a golden trace; long-run checkpoints |
| **BOOM / Rocket (Verilator)** | RISC-V core | **RTL** (cycle-exact) | The actual Chisel-generated core, gate-for-gate | Golden timing/power for a specific RISC-V core (slow; the signoff-ish rung) |
| **gem5 (RISC-V)** | RISC-V core | Functional (Atomic) + event-driven cycle-approximate (O3) | RV cores + full memory system, SE or FS mode | Microarchitecture **exploration** of a RISC-V design before/without RTL |

The rest of the page is the "conceptual note per category" the table abbreviates.

---

## 2. Manycore / CPU — the speed-vs-fidelity split, made for many cores

The core-simulation ideas of [Simulation_Methodology §6](01_Simulation_Methodology.md) (in-order vs O3 timing, the interval/mechanistic dual) reappear here, but the binding constraint changes: at 100–1000 cores, an event-driven O3 model per core is too slow, so these tools each make a different speed bet.

- **Sniper** (Ghent, Carlson/Heirman/Eeckhout, SC 2011) is the **interval model** ([Simulation_Methodology §6](01_Simulation_Methodology.md), Karkhanis–Smith) made into a parallel multicore simulator. Instead of stepping a pipeline, it treats *miss events* (branch mispredicts, cache/TLB misses, serialization) as the things that punch holes in a smooth issue stream, and computes the CPI *between* misses analytically; it keeps a per-core instruction window (a proxy ROB) so it still captures memory-level parallelism — overlapping misses — the effect a naive analytical model misses. The payoff is ~1–2 MIPS/core with parallel host threads and ~10–20% accuracy, i.e. **the mechanistic rung's error bar at close to functional-emulation speed**, which is exactly right for multicore *trend* studies. It ships with McPAT for power. A newer "instruction-window-centric" core model trades some speed for more detail when the interval abstraction is too coarse.

- **ZSim** (MIT/Stanford, Sanchez & Kozyrakis, ISCA 2013) makes the opposite bet: keep a genuine event-driven OoO core model but *parallelize aggressively* to reach 1000 cores. Its trick is **bound-weave** two-phase parallelization. In the **bound** phase, all cores run in parallel on the host using Pin-based dynamic binary translation ([Simulation_Methodology §4](01_Simulation_Methodology.md), execution-driven) with *approximate* memory latencies, gathering per-core access traces; in the **weave** phase, those traces are replayed over a short interval to resolve the true contention and interleaving — also in parallel. This sidesteps the classic parallel-simulation hazard (the shared event queue serializes) by confining cross-core interaction to the bounded weave window. Validated within ~10% of a real Westmere on average, at aggregate speeds far beyond a serial O3 model. It is user-level (syscall emulation, no OS), which is the deliberate cost of that speed.

- **MARSSx86** (Patel et al.) is the **cycle-accurate, full-system** x86 point: it bolts the PTLsim out-of-order model onto QEMU for functional/OS support, so it boots a real operating system and times it. It is the heaviest and slowest of the three and has been largely superseded by gem5's x86 support, but it anchors the top of this category's ladder — when you genuinely need OS-in-the-loop x86 timing, this is the class of tool.

**The selection rule** is the §2 fidelity rule specialized for core count: Sniper for the most cores and the fastest turnaround at trend-level accuracy; ZSim when you still want an explicit OoO model at thousand-core scale; MARSSx86/gem5-FS when OS effects must be in the timing.

---

## 3. NoC / interconnect — cycle-accurate transport, and a separate energy model

On-chip networks are where the queueing law of [Simulation_Methodology §7](01_Simulation_Methodology.md) ($\text{latency}\sim 1/(1-\rho)$) is the whole story, so their simulators are built to produce **latency-throughput curves** — latency flat at low load, then a knee as offered load $\rho$ approaches saturation. The router-microarchitecture and topology math these tools evaluate is the subject of [Network_on_Chip](../13_Network_on_Chip.md); here they are as *simulators*:

- **BookSim 2.0** (Stanford, Jiang et al., ISPASS 2013) is the standalone, **cycle-accurate** NoC reference. You give it a topology (mesh, torus, flattened butterfly, …), a routing algorithm, a virtual-channel flow-control and allocator configuration, and a *synthetic traffic pattern* (uniform-random, transpose, hotspot) or an injected trace; it steps the router pipeline flit by flit and reports the latency-throughput curve and where saturation sets in. Being standalone and traffic-driven makes it ideal for studying the network *in isolation* — the address/traffic stream is a faithful stimulus for a network the same way it is for a DRAM channel ([Simulation_Methodology §4](01_Simulation_Methodology.md), trace-driven is sound when the studied thing does not feed back into the instruction path).

- **Garnet** (Georgia Tech; Garnet 2.0, then **HeteroGarnet/Garnet 3.0**, 2020) is the cycle-accurate NoC that lives **inside gem5's Ruby** coherence subsystem, so it carries the *real* coherence message traffic of an executing workload rather than a synthetic pattern — execution-driven transport. HeteroGarnet adds heterogeneous and independently-clocked links (useful for chiplet/2.5D links). Use Garnet when the interconnect study must reflect the actual read/snoop/writeback mix a program generates; use BookSim when you want a controlled synthetic sweep.

- **DSENT** (MIT, Sun et al., NOCS 2012) is **not a timing simulator at all** — it is the *energy and area* model that overlays one. It computes per-component NoC energy (buffers, crossbars, links; electrical *and* photonic) from technology parameters and activity, yielding the pJ/flit-hop figure that a BookSim or Garnet activity count multiplies against. It is the NoC entry in the "timing tool vs power tool" distinction the [Full_Chip_Modeling tool map](../02_Full_Chip_Modeling.md) insists on: Garnet gives latency-under-load, DSENT gives energy — never conflate them.

---

## 4. Compute-in-memory & neuromorphic — modeling analog physics and spikes

This category departs from the digital-timing paradigm entirely, because the device physics *is* the computation.

- **DNN+NeuroSim** (Georgia Tech, Shimeng Yu group; **V1.4**, 2024) is the standard benchmarking framework for **compute-in-memory (CIM)** accelerators — the crossbar substrate covered device-side in [Memory §12b](../09_Memory.md). Its paradigm is **analytical circuit-level**: a hierarchy of *macro models* (device → synaptic array → PE → tile → chip) calibrated to circuit simulation rather than a discrete-event engine. A PyTorch wrapper captures a network's per-layer activity and feeds a C++ estimation engine that reports area, latency, energy, and leakage for the mapped hardware — but its distinguishing output is **inference accuracy under device non-idealities**: conductance variation, stuck-at faults, limited ON/OFF ratio, and ADC (analog-to-digital converter) quantization all degrade the analog dot-product, so PPA and *accuracy* must be co-reported. V1.4 extends technology support toward the 1 nm node (nanosheet/CFET devices) and adds digital-CIM alongside analog. This is the tool when the question is "what is the PPA *and* the accuracy of this ReRAM/SRAM CIM design," a trade no digital simulator expresses. Circuit-level results are validated against SPICE-class macro characterization rather than against a cycle count.

- **Neuromorphic** architecture simulators — e.g. **NeuroXplorer** (Drexel) and **SANA-FE** (UT Austin) — model spiking neural networks (SNNs) mapped onto *tiled, NoC-connected* neuromorphic chips (Loihi/TrueNorth-class): they are **spike-event-driven**, tracking spike traffic across the on-chip network to estimate energy and latency of a placement. Keep two things distinct: these are *hardware-architecture* simulators, unlike neuroscience SNN simulators (NEST, Brian) that model biological dynamics functionally with no hardware, and unlike vendor SDK/functional stacks (e.g. Intel Lava for Loihi). For a digital notebook this is a "know it exists and what class it is" entry rather than a primary tool.

---

## 5. Datacenter & network — parallel discrete-event at system scale

When the unit of study grows from a chip to a rack or a cluster, one host thread cannot hold the model, so these tools are built on **parallel discrete-event simulation (PDES)** — the §3 engine, sharded across many hosts with a conservative/optimistic synchronization protocol keeping the distributed event queues causally consistent.

- **SST — the Structural Simulation Toolkit** (Sandia/NTESS) is the HPC-scale framework: a PDES core (MPI-parallel, scaling to *millions* of components across hundreds of CPUs) plus a library of interchangeable **elements** connected by *links* that exchange timed *events*. The libraries cover a whole system — **Merlin** (network/interconnect), **Ember** (MPI communication *motifs* — skeleton traffic that reproduces an HPC app's messaging without running it), **memHierarchy** and **Miranda** (caches and a memory-access generator), **Vanadis** (a CPU core), and DRAM back-ends (DRAMSim3, Ramulator). The modular element/link/event structure is the §3 discrete-event engine turned into a component framework, and its purpose is **scale-out co-design**: interconnect, memory, and compute of a supercomputer together, at a fidelity you dial per component.

- **ns-3** is the open-source, **packet-level** discrete-event network simulator: full TCP/IP and wireless stacks, congestion-control algorithms, and datacenter fabric models, where each packet is an event. It answers fabric- and protocol-level questions (incast, ECN, load balancing) that a cycle-level on-chip NoC tool (§3) is the wrong scale for.

- **OMNeT++** is a general **component-based DES framework** (not a network simulator per se); with the **INET** framework it becomes a full network simulator, and it is also used for buses, storage, and queueing systems. Choose it when you are *building* a custom system model and want a mature DES kernel and GUI; choose ns-3 when you want batteries-included network protocol models.

The through-line: at datacenter scale the paradigm is always discrete-event; what differs is the *component library* and the *synchronization* that lets it run in parallel.

---

## 6. Chiplet / 2.5D-3D — where communication and heat become the model

Disaggregating a monolithic die into chiplets on an interposer moves the bottleneck from on-die logic to the **die-to-die interconnect** and to **thermal coupling** between stacked/adjacent dies, so this category's tools are inherently *co-simulations* of performance, power, and thermal — the coupled loop of [Full_Chip_Modeling §5](../02_Full_Chip_Modeling.md), specialized to a package.

- **CHIPSIM** (Pfromm et al., 2025, [arXiv:2510.25958](https://arxiv.org/abs/2510.25958)) is a co-simulation framework for **DNN execution on chiplet-based systems**. It concurrently models compute chiplets *and* the **Network-on-Interposer (NoI)** — capturing inter-chiplet network contention and pipelining that per-chiplet models miss — and profiles chiplet and NoI power at *microsecond* granularity to drive **transient** thermal analysis. Modeling communication and heat *together* (rather than perf then power then thermal in isolation) is the point: the paper reports large accuracy improvements over decoupled approaches precisely because the transient interaction is where chiplet designs live. Reach for it (or its class) when the design question is chiplet count/placement and NoI topology under a thermal budget.

- **HotSpot** (Virginia, Skadron et al.) is the thermal engine underneath most of this: a **compact RC thermal-network** solver that takes a floorplan and a per-block power map and returns steady-state and transient temperatures ($\tau_\theta = R_\theta C_\theta$; die time-constant ~ms, heatsink ~s — [Full_Chip §5.4](../02_Full_Chip_Modeling.md)). It is the standard thermal stage co-simulated with any perf/power tool; **3D-ICE, PACT, and MFIT** extend the same compact-model idea to 2.5D/3D stacks with inter-layer conduction and microfluidic cooling. Thermal is not a paradigm of its own so much as the *physical* question ([Simulation_Methodology §1](01_Simulation_Methodology.md)) overlaid on the activity a timing tool produced.

---

## 7. RISC-V cores — the four rungs of the ladder in one ecosystem

RISC-V is the cleanest place to see all four fidelity rungs of [Simulation_Methodology §2](01_Simulation_Methodology.md) coexisting for one ISA, because the open ecosystem provides a tool at each rung and *uses them together* in a verification/exploration flow.

- **Spike** (`riscv-isa-sim`, RISC-V International) is the **functional golden model**: it executes RV binaries so that architectural state ends up correct, with *no timing*. Its role is to be *right* — it is the reference against which everything else is checked, and the canonical partner for **tandem co-simulation**, where a core's RTL commit log is compared instruction-by-instruction against Spike's to catch functional bugs ([Simulation_Methodology §1](01_Simulation_Methodology.md), the functional question).

- **Dromajo** (CHIPS Alliance / riscv-boom) is a functional RV64GC emulator engineered specifically for **RTL co-simulation** at scale: it supports checkpoint/restore and fast-forward so you can boot to a region of interest, then run the RTL in *lockstep* against Dromajo's golden trace for long workloads. Same functional rung as Spike, tuned for the co-verification job.

- **BOOM / Rocket under Verilator** is the **RTL rung**: the actual Chisel-generated core, compiled by Verilator into a cycle-exact software model (part of the Chipyard flow). It *is* the design — golden timing and, with power tools, golden energy — at RTL-simulation cost (§2's $10^6$–$10^7\times$ slowdown, KHz–low-MHz). This is where a specific core's real IPC comes from.

- **gem5 (RISC-V)** is the **exploration rung**: functional `AtomicSimpleCPU` for fast bring-up and the event-driven cycle-approximate `O3CPU` for microarchitecture studies, with the full gem5 memory system, in SE or FS mode. Use it to explore a RISC-V design's microarchitecture *before* (or without) writing RTL — the same role gem5 plays for any ISA, detailed on the gem5 page in this folder.

The lesson is the ladder itself: **you do not pick one RISC-V simulator, you use the whole ladder** — Spike/Dromajo to be *correct*, gem5 to *explore*, Verilator-on-RTL to be *exact* — matching each question to the cheapest rung that can answer it.

---

## Numbers to memorize

| Quantity | Value | Why it matters |
|---|---|---|
| Sniper (interval) speed / error | ~1–2 MIPS/core, ~10–20% | mechanistic-rung accuracy at near-functional speed → manycore trends |
| ZSim scale / validation | ~1000 cores, ~10% vs Westmere | bound-weave parallelism buys scale without abandoning an OoO model |
| NoC latency law | $\sim 1/(1-\rho)$ | why BookSim/Garnet output a latency-throughput *knee* |
| DSENT is energy, not timing | pJ/flit-hop | never conflate the NoC timing tool with its power model |
| CIM must co-report accuracy | PPA **and** inference accuracy | analog non-idealities (variation, ADC) degrade the dot-product |
| SST scale | millions of components, 100s of CPUs (PDES) | datacenter/HPC co-design needs parallel discrete-event |
| Thermal time constants | die ~ms, heatsink ~s | why chiplet/3D tools need *transient* (not just steady) thermal |
| RISC-V ladder | Spike (functional) → gem5 (cycle-approx) → Verilator-RTL (exact) | use the whole ladder, cheapest rung per question |

---

## Cross-references

- **Down the stack:** [Simulation_Methodology](01_Simulation_Methodology.md) (the paradigm/ladder/trace-vs-execution vocabulary this catalog classifies against), [Network_on_Chip](../13_Network_on_Chip.md) (router microarchitecture, topology math, and the queueing the NoC simulators evaluate), [Memory §12b](../09_Memory.md) (compute-in-memory devices NeuroSim models), [OoO_Execution](../05_OoO_Execution.md) (the OoO structures Sniper/ZSim/gem5 abstract).
- **Up the stack:** [Full_Chip_Modeling](../02_Full_Chip_Modeling.md) (the perf→power→thermal co-simulation loop and tool map these fit into; §5.4 thermal, §5.5 co-modeling), [Performance_Modeling_and_DSE](../01_Performance_Modeling_and_DSE.md) (the fidelity ladder and DSE these tools serve).
- **Sibling pages (this folder):** [Accelerator_and_NPU_Simulators](05_Accelerator_and_NPU_Simulators.md) (DNN/NPU tools — where NeuroSim's CIM and CHIPSIM's DNN-on-chiplet context connect), and the gem5 / DRAM / GPU per-tool pages.

## References

- T. E. Carlson, W. Heirman, L. Eeckhout, "Sniper: Exploring the Level of Abstraction for Scalable and Accurate Parallel Multi-Core Simulation," SC 2011 — [snipersim.org](https://snipersim.org/w/The_Sniper_Multi-Core_Simulator), [Interval Simulation](https://snipersim.org/w/Interval_Simulation).
- D. Sanchez, C. Kozyrakis, "ZSim: Fast and Accurate Microarchitectural Simulation of Thousand-Core Systems," ISCA 2013 — [paper PDF](https://people.csail.mit.edu/sanchez/papers/2013.zsim.isca.pdf).
- A. Patel et al., "MARSSx86: A Full System Simulator for x86 CPUs," DAC 2011.
- N. Jiang et al., "A Detailed and Flexible Cycle-Accurate Network-on-Chip Simulator (BookSim 2.0)," ISPASS 2013.
- N. Agarwal et al., "GARNET: A Detailed On-Chip Network Model inside a Full-System Simulator," ISPASS 2009; HeteroGarnet — [gem5.org HeteroGarnet](https://www.gem5.org/documentation/general_docs/ruby/heterogarnet/).
- C. Sun et al., "DSENT — A Tool Connecting Emerging Photonics with Electronics for Opto-Electronic NoC Modeling," NOCS 2012.
- X. Peng et al., "DNN+NeuroSim V1.4," Georgia Tech — [github.com/neurosim/DNN_NeuroSim_V1.4](https://github.com/neurosim/DNN_NeuroSim_V1.4); "NeuroSim Simulator for CIM: Validation and Benchmark," Frontiers in AI 2021.
- A. Balaji et al., "NeuroXplorer 1.0: An Extensible Framework for Architectural Exploration with Spiking Neural Networks," [arXiv:2105.01795](https://arxiv.org/abs/2105.01795); "SANA-FE," IEEE TCAD 2025.
- A. F. Rodrigues et al., "The Structural Simulation Toolkit (SST)," Sandia — [sst-simulator.org](http://sst-simulator.org/sst-docs), [github.com/sstsimulator/sst-core](https://github.com/sstsimulator/sst-core).
- ns-3 — [nsnam.org](https://www.nsnam.org/); OMNeT++ / INET — [omnetpp.org](https://omnetpp.org/).
- L. Pfromm et al., "CHIPSIM: A Co-Simulation Framework for Deep Learning on Chiplet-Based Systems," 2025 — [arXiv:2510.25958](https://arxiv.org/abs/2510.25958), [github.com/LukasPfromm/CHIPSIM](https://github.com/LukasPfromm/CHIPSIM).
- W. Huang et al., "HotSpot: A Compact Thermal Modeling Methodology," IEEE TVLSI 2006; 3D-ICE, PACT, MFIT for 2.5D/3D extensions.
- Spike (`riscv-isa-sim`) — [github.com/riscv-software-src/riscv-isa-sim](https://github.com/riscv-software-src/riscv-isa-sim); Dromajo — [github.com/chipsalliance/dromajo](https://github.com/chipsalliance/dromajo); BOOM/Rocket + Verilator via [Chipyard](https://chipyard.readthedocs.io/).
