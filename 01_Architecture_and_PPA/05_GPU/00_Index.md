# Part 5 · Architecture › GPU — Chapter Index

*Chip type: the throughput / latency-hiding machine — SIMT, occupancy, and why it is shaped opposite to a CPU.*

Each chapter is concept-first — why the structure must exist, the mechanism, the trade-offs, then the derivations and worked numbers — closing with **Numbers to memorize** and **Cross-references**.

### [GPU Architecture — The Throughput Machine and the Streaming Multiprocessor](01_GPU_Architecture.md)

- §1  Throughput, not latency — why the machine is shaped differently
- §2  SIMT — amortizing the front end over 32 lanes
- §3  Warp divergence and reconvergence — the hardware cost of the SIMT bargain
- §4  The Streaming Multiprocessor as a hardware block
- §5  The register file and operand delivery — the SM's defining structure
- §6  Memory coalescing — why it exists and the hardware that does it
- §7  The memory hierarchy and occupancy — resource-limited concurrency
- §8  Clocking and power at throughput — why wide-and-slow wins
- §9  Numbers to memorize
- §10  Worked problems

---
⬅ [Architecture Book Contents](../00_Index.md) · [Root Index](../../Index.md) · [← Part 4 · Interconnect](../04_Interconnect/00_Index.md) · [Part 6 · NPU →](../06_NPU/00_Index.md)
