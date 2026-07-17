# Part 2 · Architecture › CPU — Chapter Index

*Chip type: the general-purpose latency machine — a complete core, from the pipelined contract through out-of-order speculation to a real open-source design.*

Each chapter is concept-first — why the structure must exist, the mechanism, the trade-offs, then the derivations and worked numbers — closing with **Numbers to memorize** and **Cross-references**.

### [CPU Architecture — The Pipelined Machine and Its Contract](01_CPU_Architecture.md)

- §1  Pipelining: why overlap buys throughput, and how deep to go
- §2  Hazards: the three ways overlap breaks, and what each costs
- §3  Forwarding: closing one timing gap
- §4  Control hazards and branch prediction — the concept
- §5  Beyond in-order: the scalar ceiling and the four ideas that break it
- §6  The memory hierarchy the pipeline must hide
- §7  Virtual memory and the TLB — the concept
- §8  Cache coherence: the single-writer invariant
- §9  Memory consistency: the ordering contract
- §10  Simultaneous multithreading: replicate state, share the engine
- §11  Speculative-execution security: the invariant speculation breaks
- §12  Numbers to memorize
- §13  Worked problems

### [RISC-V ISA — Design Rationale of a Modular Load-Store Contract](02_RISC_V_ISA.md)

- §1  The ISA as a contract: a small base plus orthogonal extensions
- §2  Load-store and fixed width: designing the encoding for the decoder
- §3  The compressed (C) extension: buying code density back as an option
- §4  The modular extensions: capability paid for in area, not mandatory complexity
- §5  Privilege and virtual memory: the mechanism system software needs
- §6  The vector (V) extension: length-agnostic as a portability contract
- §7  Why RISC-V matters for hardware design

### [Out-of-Order Execution — Datapath Design Deep Dive](03_OoO_Execution.md)

- §1  The core idea: a dataflow engine behind an in-order façade
- §2  Register renaming: mapping names to erase false dependencies
- §3  The reorder buffer: where speculation becomes fact
- §4  The issue queue: dataflow scheduling and the wakeup–select recurrence
- §5  The load-store queue: disambiguating a name space that binds too late
- §6  Branch misprediction: the dominant limiter
- §7  Execution units as a latency/throughput menu
- §8  Simultaneous multithreading (SMT)
- §9  Precise exceptions and interrupts
- §10  Value prediction: probing the ILP ceiling
- §11  Real cores: Golden Cove vs Zen 4
- §12  Numbers to memorize
- §13  Worked problems

### [Branch Prediction — the Speculative Front End](04_Branch_Prediction_Deep_Dive.md)

- §1  What must be predicted, and why one structure cannot do it
- §2  The BTB: "is this a branch, and where?" from the PC alone
- §3  Direction prediction: bias first, then correlation
- §4  TAGE: letting each branch pick its own history length
- §5  Indirect branches and the perceptron alternative
- §6  The RAS: returns are context-determined, not PC-determined
- §7  The decoupled front end: FTQ, and the fetch-width wall
- §8  Real cores: where the trade-offs land
- §9  Numbers to memorize
- §10  Worked problems

### [Xiangshan (香山) — Reading an Open OoO Core as a Sequence of Design Decisions](05_Xiangshan_CPU_Design.md)

- §1  The design as a set of constraints
- §2  Pipeline depth and width: the first two bets
- §3  The decoupled front end: the FTQ as the front-end analogue of the ROB
- §4  Branch prediction: an accuracy machine that buys down the tax
- §5  The out-of-order window: three structures, three different theory curves
- §6  The load-store unit: the late-binding name space in practice
- §7  The memory hierarchy and interconnect: from private cache to coherent fabric
- §8  Reading the evolution as a trade-off trajectory
- §9  Numbers to memorize
- §10  Cross-references
- §11  References

---
⬅ [Architecture Book Contents](../00_Index.md) · [Root Index](../../Index.md) · [← Part 1 · Modeling](../01_Modeling/00_Index.md) · [Part 3 · Memory →](../03_Memory/00_Index.md)
