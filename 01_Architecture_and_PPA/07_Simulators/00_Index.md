# Part 7 · Architecture › Simulators — Chapter Index

*Shared part: how every non-silicon number in this book is produced — methodology, the tools tier by tier, and their closed-form duals.*

Each chapter is concept-first — why the structure must exist, the mechanism, the trade-offs, then the derivations and worked numbers — closing with **Numbers to memorize** and **Cross-references**.

### [Simulation Methodology — How Architectural Simulators Actually Work](01_Simulation_Methodology.md)

- §1  The three questions every simulator answers
- §2  The fidelity–speed tradeoff (the master curve)
- §3  The event-driven engine — the heart of a timing simulator
- §4  Trace-driven vs execution-driven — where the instruction stream comes from
- §5  How a workload becomes a number — the part people get wrong
- §6  How compute is modeled — the timing of a core
- §7  How the memory system is modeled — usually the dominant term
- §8  Validation, calibration, and the error budget
- §9  Numbers to memorize

### [gem5 — the configurable full-system architectural simulator](02_gem5.md)

- §1  What gem5 is — one functional core, many timing models
- §2  SE vs FS — the "is there an operating system?" axis
- §3  The CPU timing models — a fidelity ladder inside one tool
- §4  Classic vs Ruby — two memory systems with different coherence fidelity
- §5  The discrete-event engine, SimObjects, and ports
- §6  How a workload becomes a statistic
- §7  Fast-forwarding, checkpoints, and KVM warm-up
- §8  Validation and the error vs real silicon
- §9  Numbers to memorize

### [DRAM Simulators — Ramulator, DRAMSim3, DRAMPower, USIMM](03_DRAM_Simulators.md)

- §1  What a DRAM simulator models — and what it deliberately doesn't
- §2  The bank/rank/channel hierarchy as nested state machines
- §3  JEDEC timing constraints as the transition guards
- §4  Row-buffer management — open vs closed page
- §5  Address mapping — physical → (channel, rank, bank, row, col)
- §6  The request scheduler — FR-FCFS and its variants
- §7  Driving the model — trace-driven or coupled to a core
- §8  How bandwidth and latency are computed — the queueing intuition
- §9  DRAMPower — energy as Σ(time-in-state × IDD current × voltage)
- §10  The four tools at a glance

### [GPU Simulators — GPGPU-Sim and Accel-Sim](04_GPU_Simulators.md)

- §1  Two simulators, two paradigms
- §2  The SIMT core timing model
- §3  Coalescing and the cache / HBM / interconnect contention layer
- §4  Tensor Core modeling
- §5  Trace-driven vs execution-driven — Accel-Sim's key design point
- §6  How a CUDA kernel or SASS trace becomes IPC and achieved bandwidth
- §7  Power — GPUWattch and AccelWattch
- §8  Supported GPUs and the validation story

### [Accelerator & DNN/NPU Simulators — Mapping, Dataflow, and Cycles](05_Accelerator_and_NPU_Simulators.md)

- §1  The three paradigms and when to use each
- §2  From a layer + mapping to latency and energy — the shared mechanism
- §3  The dataflow taxonomy and why it is mostly an energy lever
- §4  Analytical mapping + energy — Timeloop and Accelergy
- §5  Analytical data-centric reuse — MAESTRO
- §6  Cycle-accurate systolic — SCALE-Sim v3
- §7  Operator-level, graph-scale — NeuSim and ONNXim

### [Other Architecture Simulators — A Surveyed Catalog](06_Other_Architecture_Simulators.md)

- §1  The catalog
- §2  Manycore / CPU — the speed-vs-fidelity split, made for many cores
- §3  NoC / interconnect — cycle-accurate transport, and a separate energy model
- §4  Compute-in-memory & neuromorphic — modeling analog physics and spikes
- §5  Datacenter & network — parallel discrete-event at system scale
- §6  Chiplet / 2.5D-3D — where communication and heat become the model
- §7  RISC-V cores — the four rungs of the ladder in one ecosystem

### [Analytical models — the closed-form duals of the simulators](07_Analytical_Models.md)

- §1  What the analytical layer is for — bound, decompose, size, escalate
- §2  Roofline — the throughput bound (extends DSE §2.3)
- §3  The interval / mechanistic model — core CPI in closed form
- §4  Amdahl, Gustafson, and the contention term Amdahl omits (USL)
- §5  Little's law and M/M/1 — the backbone of every contention model
- §6  LogCA / LogGP — the offload and communication cost model
- §7  When analytical suffices, and when you must escalate
- §8  Numbers to memorize

---
⬅ [Architecture Book Contents](../00_Index.md) · [Root Index](../../Index.md) · [← Part 6 · NPU](../06_NPU/00_Index.md)
