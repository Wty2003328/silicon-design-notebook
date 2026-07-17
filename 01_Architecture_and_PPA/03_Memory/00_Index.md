# Part 3 · Architecture › Memory — Chapter Index

*Shared part: the storage hierarchy every machine reuses — from the SRAM/DRAM bit cell up through caches, address translation, and the DRAM scheduler.*

Each chapter is concept-first — why the structure must exist, the mechanism, the trade-offs, then the derivations and worked numbers — closing with **Numbers to memorize** and **Cross-references**.

### [Cache Microarchitecture — Locality, AMAT, and Controller Design](01_Cache_Microarchitecture.md)

- §1  The organizing theory: locality, the memory wall, and AMAT
- §2  Associativity: trading conflict misses against hit cost
- §3  Non-blocking caches and the MSHR: buying memory-level parallelism
- §4  Write policy: when the cached and backing copies may diverge
- §5  Replacement: predicting re-reference distance
- §6  The cache hierarchy: recursive AMAT and inclusion policy
- §7  Prefetching: converting misses to hits before they stall
- §8  Coherence: the conceptual story
- §9  Cache power: reducing the energy of the hit path
- §10  Quality of service: partitioning a shared last level
- §11  Numbers to memorize
- §12  Worked problems

### [TLB and Virtual Memory — Address Translation on the Critical Path](02_TLB_and_Virtual_Memory.md)

- §1  Translation is a memory access before every memory access
- §2  What a TLB entry must hold — derived from three jobs
- §3  Sizing and organization — the hot structure on the critical path
- §4  ASIDs and the global bit — a tag that buys out a flush
- §5  The page walk and the page-walk cache
- §6  VIPT — overlapping translation with the cache
- §7  Superpages — buying reach against fragmentation
- §8  TLB shootdown — paying for absent coherence
- §9  Worked problems

### [Memory Circuits — the Bit-Storage Trade Surface](03_Memory.md)

- §1  The trade surface: store, read, hold
- §2  The 6T SRAM cell: six transistors, three conflicting jobs
- §3  Breaking the 6T conflict: 8T, 10T, and assist circuits
- §4  From cell to array: yield, soft errors, retention
- §5  DRAM 1T1C: trading persistence for density
- §6  The refresh tax: the price of volatility
- §7  Multi-port memories and register files: why ports cost quadratically
- §8  The memory compiler: SRAM as generated IP
- §9  ECC: coding theory for memory
- §10  CAM and TCAM: memory that compares itself
- §11  Compute-in-memory: moving the ALU onto the bitline
- §12  Scaling limits and emerging non-volatile memories

### [DDR Memory Controller — Why DRAM Needs a Scheduler](04_DDR_Controller.md)

- §1  Why DRAM is not RAM: four broken clauses, one controller
- §2  The bank as a state machine, and the row buffer's three cases
- §3  JEDEC timing constraints as physics, not parameters
- §4  Row-buffer policy: a spatial-locality predictor
- §5  FR-FCFS: scheduling from the locality-vs-fairness trade
- §6  Refresh: the mandatory background tax
- §7  The achieved-bandwidth model and latency under load
- §8  ECC and RAS as concepts
- §9  DDR5 and LPDDR5X as concepts

---
⬅ [Architecture Book Contents](../00_Index.md) · [Root Index](../../Index.md) · [← Part 2 · CPU](../02_CPU/00_Index.md) · [Part 4 · Interconnect →](../04_Interconnect/00_Index.md)
