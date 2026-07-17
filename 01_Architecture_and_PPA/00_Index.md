# 01 · Architecture and PPA — Book Contents

*Explore the microarchitecture; model performance; budget power and area — before RTL (register-transfer level).*

This folder is a **hierarchical book**. It pairs **chip types** — [CPU](02_CPU/00_Index.md), [GPU](05_GPU/00_Index.md), [NPU](06_NPU/00_Index.md) (complete machines) — with **shared parts** — [Modeling](01_Modeling/00_Index.md), [Memory](03_Memory/00_Index.md), [Interconnect](04_Interconnect/00_Index.md), [Simulators](07_Simulators/00_Index.md) (blocks and tools every machine reuses). Each *part* is a sub-folder; each *chapter* is a page built concept-first (why the structure must exist → the mechanism → the trade-offs → the derivations and worked numbers); each chapter's *sections* are listed below.

State-bearing protocols (MESI/MOESI, CHI, the branch-predictor counter, the DRAM bank FSM, the SIMT reconvergence stack) carry state diagrams; every chapter closes with **Numbers to memorize** and **Cross-references**.

---

## Part 1 · [Modeling](01_Modeling/00_Index.md) — the PPA methodology — how you model performance, power, and area, and search the design space, *before* committing RTL

**[Performance Modeling and Design-Space Exploration](01_Modeling/01_Performance_Modeling_and_DSE.md)**  
<br/><span>§1 The modeling-fidelity ladder  · §2 Analytical models — the back-of-envelope that decides the most  · §3 Climbing to cycle simulation — when the spreadsheet can't decide  · §4 System-level modeling and virtual platforms — a different question  · §5 Design-space exploration — optimization over the PPA surface  · §6 The PPA trade-off — the fundamental tension  · §7 Numbers to memorize  · §8 Worked problem — the MLP trap  · §9 GPU performance modeling — throughput, occupancy, per-kernel roofline  · §10 NPU / accelerator performance modeling — systolic arrays, dataflow, tiling  · §11 Worked example — building an industrial operator-level DSE model (NeuSim)</span>

**[Full-Chip Power and Performance Modeling](01_Modeling/02_Full_Chip_Modeling.md)**  
<br/><span>§1 Composition: push the physics down, keep the composition cheap  · §2 Why a chip is not the sum of its blocks: five coupling terms  · §3 The system loop: one fixed point, not two models  · §4 One equation, three architectures  · §5 Numbers to remember  · §6 The tool map</span>


## Part 2 · [CPU](02_CPU/00_Index.md) — the general-purpose latency machine — a complete core, from the pipelined contract through out-of-order speculation to a real open-source design

**[CPU Architecture — The Pipelined Machine and Its Contract](02_CPU/01_CPU_Architecture.md)**  
<br/><span>§1 Pipelining: why overlap buys throughput, and how deep to go  · §2 Hazards: the three ways overlap breaks, and what each costs  · §3 Forwarding: closing one timing gap  · §4 Control hazards and branch prediction — the concept  · §5 Beyond in-order: the scalar ceiling and the four ideas that break it  · §6 The memory hierarchy the pipeline must hide  · §7 Virtual memory and the TLB — the concept  · §8 Cache coherence: the single-writer invariant  · §9 Memory consistency: the ordering contract  · §10 Simultaneous multithreading: replicate state, share the engine  · §11 Speculative-execution security: the invariant speculation breaks  · §12 Numbers to memorize  · §13 Worked problems</span>

**[RISC-V ISA — Design Rationale of a Modular Load-Store Contract](02_CPU/02_RISC_V_ISA.md)**  
<br/><span>§1 The ISA as a contract: a small base plus orthogonal extensions  · §2 Load-store and fixed width: designing the encoding for the decoder  · §3 The compressed (C) extension: buying code density back as an option  · §4 The modular extensions: capability paid for in area, not mandatory complexity  · §5 Privilege and virtual memory: the mechanism system software needs  · §6 The vector (V) extension: length-agnostic as a portability contract  · §7 Why RISC-V matters for hardware design</span>

**[Out-of-Order Execution — Datapath Design Deep Dive](02_CPU/03_OoO_Execution.md)**  
<br/><span>§1 The core idea: a dataflow engine behind an in-order façade  · §2 Register renaming: mapping names to erase false dependencies  · §3 The reorder buffer: where speculation becomes fact  · §4 The issue queue: dataflow scheduling and the wakeup–select recurrence  · §5 The load-store queue: disambiguating a name space that binds too late  · §6 Branch misprediction: the dominant limiter  · §7 Execution units as a latency/throughput menu  · §8 Simultaneous multithreading (SMT)  · §9 Precise exceptions and interrupts  · §10 Value prediction: probing the ILP ceiling  · §11 Real cores: Golden Cove vs Zen 4  · §12 Numbers to memorize  · §13 Worked problems</span>

**[Branch Prediction — the Speculative Front End](02_CPU/04_Branch_Prediction_Deep_Dive.md)**  
<br/><span>§1 What must be predicted, and why one structure cannot do it  · §2 The BTB: "is this a branch, and where?" from the PC alone  · §3 Direction prediction: bias first, then correlation  · §4 TAGE: letting each branch pick its own history length  · §5 Indirect branches and the perceptron alternative  · §6 The RAS: returns are context-determined, not PC-determined  · §7 The decoupled front end: FTQ, and the fetch-width wall  · §8 Real cores: where the trade-offs land  · §9 Numbers to memorize  · §10 Worked problems</span>

**[Xiangshan (香山) — Reading an Open OoO Core as a Sequence of Design Decisions](02_CPU/05_Xiangshan_CPU_Design.md)**  
<br/><span>§1 The design as a set of constraints  · §2 Pipeline depth and width: the first two bets  · §3 The decoupled front end: the FTQ as the front-end analogue of the ROB  · §4 Branch prediction: an accuracy machine that buys down the tax  · §5 The out-of-order window: three structures, three different theory curves  · §6 The load-store unit: the late-binding name space in practice  · §7 The memory hierarchy and interconnect: from private cache to coherent fabric  · §8 Reading the evolution as a trade-off trajectory  · §9 Numbers to memorize  · §10 Cross-references  · §11 References</span>


## Part 3 · [Memory](03_Memory/00_Index.md) — the storage hierarchy every machine reuses — from the SRAM/DRAM bit cell up through caches, address translation, and the DRAM scheduler

**[Cache Microarchitecture — Locality, AMAT, and Controller Design](03_Memory/01_Cache_Microarchitecture.md)**  
<br/><span>§1 The organizing theory: locality, the memory wall, and AMAT  · §2 Associativity: trading conflict misses against hit cost  · §3 Non-blocking caches and the MSHR: buying memory-level parallelism  · §4 Write policy: when the cached and backing copies may diverge  · §5 Replacement: predicting re-reference distance  · §6 The cache hierarchy: recursive AMAT and inclusion policy  · §7 Prefetching: converting misses to hits before they stall  · §8 Coherence: the conceptual story  · §9 Cache power: reducing the energy of the hit path  · §10 Quality of service: partitioning a shared last level  · §11 Numbers to memorize  · §12 Worked problems</span>

**[TLB and Virtual Memory — Address Translation on the Critical Path](03_Memory/02_TLB_and_Virtual_Memory.md)**  
<br/><span>§1 Translation is a memory access before every memory access  · §2 What a TLB entry must hold — derived from three jobs  · §3 Sizing and organization — the hot structure on the critical path  · §4 ASIDs and the global bit — a tag that buys out a flush  · §5 The page walk and the page-walk cache  · §6 VIPT — overlapping translation with the cache  · §7 Superpages — buying reach against fragmentation  · §8 TLB shootdown — paying for absent coherence  · §9 Worked problems</span>

**[Memory Circuits — the Bit-Storage Trade Surface](03_Memory/03_Memory.md)**  
<br/><span>§1 The trade surface: store, read, hold  · §2 The 6T SRAM cell: six transistors, three conflicting jobs  · §3 Breaking the 6T conflict: 8T, 10T, and assist circuits  · §4 From cell to array: yield, soft errors, retention  · §5 DRAM 1T1C: trading persistence for density  · §6 The refresh tax: the price of volatility  · §7 Multi-port memories and register files: why ports cost quadratically  · §8 The memory compiler: SRAM as generated IP  · §9 ECC: coding theory for memory  · §10 CAM and TCAM: memory that compares itself  · §11 Compute-in-memory: moving the ALU onto the bitline  · §12 Scaling limits and emerging non-volatile memories</span>

**[DDR Memory Controller — Why DRAM Needs a Scheduler](03_Memory/04_DDR_Controller.md)**  
<br/><span>§1 Why DRAM is not RAM: four broken clauses, one controller  · §2 The bank as a state machine, and the row buffer's three cases  · §3 JEDEC timing constraints as physics, not parameters  · §4 Row-buffer policy: a spatial-locality predictor  · §5 FR-FCFS: scheduling from the locality-vs-fairness trade  · §6 Refresh: the mandatory background tax  · §7 The achieved-bandwidth model and latency under load  · §8 ECC and RAS as concepts  · §9 DDR5 and LPDDR5X as concepts</span>


## Part 4 · [Interconnect](04_Interconnect/00_Index.md) — how blocks, cores, and chips talk — the load/store fabric, the coherence protocols, and the on-chip network that scales past a bus

**[On-Chip Interconnect — AXI, AHB, APB as a Composition Contract](04_Interconnect/01_AHB_AXI_APB.md)**  
<br/><span>§1 The problem a standard interconnect solves: composing IP  · §2 The valid/ready handshake: flow control as *whether*, not *when*  · §3 Why AXI is five independent channels  · §4 Outstanding, ID-tagged, out-of-order transactions: hiding latency  · §5 Bursts: amortizing the address phase  · §6 The three-tier family: AXI vs AHB vs APB as a PPA trade  · §7 Topology: the fabric as a separate, scalable concern  · §8 Bridges: everything reduces to buffering behind a handshake  · §9 Policy sidebands: QoS, security, and atomics  · §10 Numbers to memorize  · §11 Worked problems</span>

**[ACE and CHI — Realizing Cache Coherence at the Interconnect](04_Interconnect/02_ACE_and_CHI.md)**  
<br/><span>§1 What the fabric must provide that a lone cache cannot  · §2 Snoop coherence (ACE): broadcast the question  · §3 The $O(N^2)$ wall: why broadcast cannot scale  · §4 Directory coherence (CHI): the home node  · §5 The price of the directory: storage and indirection  · §6 Why CHI is layered, and why it rides a mesh  · §7 Off-chip coherence: why the story left the die (CXL)  · §8 The ordering point's other duties  · §9 ACE vs CHI: where real silicon lands  · §10 Numbers to memorize  · §11 Worked problems</span>

**[Network-on-Chip — Why On-Chip Communication Becomes a Network](04_Interconnect/03_Network_on_Chip.md)**  
<br/><span>§1 Why a bus and a crossbar both stop scaling  · §2 Topology: the diameter–bisection–cost trade  · §3 Flow control: sharing buffers without dropping flits  · §4 Routing and deadlock — the part with theorems  · §5 The router pipeline as a concept  · §6 Latency under load  · §7 The coherent mesh in practice (Arm CMN-class)  · §8 Physical design of the fabric  · §9 Numbers to memorize  · §10 Worked problems</span>


## Part 5 · [GPU](05_GPU/00_Index.md) — the throughput / latency-hiding machine — SIMT, occupancy, and why it is shaped opposite to a CPU

**[GPU Architecture — The Throughput Machine and the Streaming Multiprocessor](05_GPU/01_GPU_Architecture.md)**  
<br/><span>§1 Throughput, not latency — why the machine is shaped differently  · §2 SIMT — amortizing the front end over 32 lanes  · §3 Warp divergence and reconvergence — the hardware cost of the SIMT bargain  · §4 The Streaming Multiprocessor as a hardware block  · §5 The register file and operand delivery — the SM's defining structure  · §6 Memory coalescing — why it exists and the hardware that does it  · §7 The memory hierarchy and occupancy — resource-limited concurrency  · §8 Clocking and power at throughput — why wide-and-slow wins  · §9 Numbers to memorize  · §10 Worked problems</span>


## Part 6 · [NPU](06_NPU/00_Index.md) — the spatial dataflow accelerator — the systolic array and the mapping problem

**[NPU Accelerators — Spatial Dataflow Hardware for Dense Linear Algebra](06_NPU/01_NPU_Accelerators.md)**  
<br/><span>§1 Why a von-Neumann core is the wrong shape for dense GEMM  · §2 The systolic array as hardware  · §3 Dataflow — which operand stays resident, and why that is the energy lever  · §4 Explicit on-chip memory and the mapping problem — a scratchpad, not a cache  · §5 The roofline view of an accelerator  · §6 Scaling out — multi-die and pods, where the interconnect is the roofline  · §7 Trade-offs — the efficiency–flexibility ledger, and where dataflow hardware stops winning  · §8 Worked problems</span>


## Part 7 · [Simulators](07_Simulators/00_Index.md) — how every non-silicon number in this book is produced — methodology, the tools tier by tier, and their closed-form duals

**[Simulation Methodology — How Architectural Simulators Actually Work](07_Simulators/01_Simulation_Methodology.md)**  
<br/><span>§1 The three questions every simulator answers  · §2 The fidelity–speed tradeoff (the master curve)  · §3 The event-driven engine — the heart of a timing simulator  · §4 Trace-driven vs execution-driven — where the instruction stream comes from  · §5 How a workload becomes a number — the part people get wrong  · §6 How compute is modeled — the timing of a core  · §7 How the memory system is modeled — usually the dominant term  · §8 Validation, calibration, and the error budget  · §9 Numbers to memorize</span>

**[gem5 — the configurable full-system architectural simulator](07_Simulators/02_gem5.md)**  
<br/><span>§1 What gem5 is — one functional core, many timing models  · §2 SE vs FS — the "is there an operating system?" axis  · §3 The CPU timing models — a fidelity ladder inside one tool  · §4 Classic vs Ruby — two memory systems with different coherence fidelity  · §5 The discrete-event engine, SimObjects, and ports  · §6 How a workload becomes a statistic  · §7 Fast-forwarding, checkpoints, and KVM warm-up  · §8 Validation and the error vs real silicon  · §9 Numbers to memorize</span>

**[DRAM Simulators — Ramulator, DRAMSim3, DRAMPower, USIMM](07_Simulators/03_DRAM_Simulators.md)**  
<br/><span>§1 What a DRAM simulator models — and what it deliberately doesn't  · §2 The bank/rank/channel hierarchy as nested state machines  · §3 JEDEC timing constraints as the transition guards  · §4 Row-buffer management — open vs closed page  · §5 Address mapping — physical → (channel, rank, bank, row, col)  · §6 The request scheduler — FR-FCFS and its variants  · §7 Driving the model — trace-driven or coupled to a core  · §8 How bandwidth and latency are computed — the queueing intuition  · §9 DRAMPower — energy as Σ(time-in-state × IDD current × voltage)  · §10 The four tools at a glance</span>

**[GPU Simulators — GPGPU-Sim and Accel-Sim](07_Simulators/04_GPU_Simulators.md)**  
<br/><span>§1 Two simulators, two paradigms  · §2 The SIMT core timing model  · §3 Coalescing and the cache / HBM / interconnect contention layer  · §4 Tensor Core modeling  · §5 Trace-driven vs execution-driven — Accel-Sim's key design point  · §6 How a CUDA kernel or SASS trace becomes IPC and achieved bandwidth  · §7 Power — GPUWattch and AccelWattch  · §8 Supported GPUs and the validation story</span>

**[Accelerator & DNN/NPU Simulators — Mapping, Dataflow, and Cycles](07_Simulators/05_Accelerator_and_NPU_Simulators.md)**  
<br/><span>§1 The three paradigms and when to use each  · §2 From a layer + mapping to latency and energy — the shared mechanism  · §3 The dataflow taxonomy and why it is mostly an energy lever  · §4 Analytical mapping + energy — Timeloop and Accelergy  · §5 Analytical data-centric reuse — MAESTRO  · §6 Cycle-accurate systolic — SCALE-Sim v3  · §7 Operator-level, graph-scale — NeuSim and ONNXim</span>

**[Other Architecture Simulators — A Surveyed Catalog](07_Simulators/06_Other_Architecture_Simulators.md)**  
<br/><span>§1 The catalog  · §2 Manycore / CPU — the speed-vs-fidelity split, made for many cores  · §3 NoC / interconnect — cycle-accurate transport, and a separate energy model  · §4 Compute-in-memory & neuromorphic — modeling analog physics and spikes  · §5 Datacenter & network — parallel discrete-event at system scale  · §6 Chiplet / 2.5D-3D — where communication and heat become the model  · §7 RISC-V cores — the four rungs of the ladder in one ecosystem</span>

**[Analytical models — the closed-form duals of the simulators](07_Simulators/07_Analytical_Models.md)**  
<br/><span>§1 What the analytical layer is for — bound, decompose, size, escalate  · §2 Roofline — the throughput bound (extends DSE §2.3)  · §3 The interval / mechanistic model — core CPI in closed form  · §4 Amdahl, Gustafson, and the contention term Amdahl omits (USL)  · §5 Little's law and M/M/1 — the backbone of every contention model  · §6 LogCA / LogGP — the offload and communication cost model  · §7 When analytical suffices, and when you must escalate  · §8 Numbers to memorize</span>


---
⬅ prev [00 · Fundamentals](../00_Fundamentals/00_Index.md) · [Root Index](../Index.md) · [Flow Overview](../Chip_Design_Flow_Overview.md) · next ➡ [02 · Power and Low-Power](../02_Power_and_Low_Power/00_Index.md)
