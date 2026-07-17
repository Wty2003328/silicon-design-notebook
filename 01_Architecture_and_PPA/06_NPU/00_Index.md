# Part 6 · Architecture › NPU — Chapter Index

*Chip type: the spatial dataflow accelerator — the systolic array and the mapping problem.*

Each chapter is concept-first — why the structure must exist, the mechanism, the trade-offs, then the derivations and worked numbers — closing with **Numbers to memorize** and **Cross-references**.

### [NPU Accelerators — Spatial Dataflow Hardware for Dense Linear Algebra](01_NPU_Accelerators.md)

- §1  Why a von-Neumann core is the wrong shape for dense GEMM
- §2  The systolic array as hardware
- §3  Dataflow — which operand stays resident, and why that is the energy lever
- §4  Explicit on-chip memory and the mapping problem — a scratchpad, not a cache
- §5  The roofline view of an accelerator
- §6  Scaling out — multi-die and pods, where the interconnect is the roofline
- §7  Trade-offs — the efficiency–flexibility ledger, and where dataflow hardware stops winning
- §8  Worked problems

---
⬅ [Architecture Book Contents](../00_Index.md) · [Root Index](../../Index.md) · [← Part 5 · GPU](../05_GPU/00_Index.md) · [Part 7 · Simulators →](../07_Simulators/00_Index.md)
