# Central Processing Unit (CPU) Architecture › Coherence and Consistency

> **Abbreviation key — skim now and return as needed:** system on chip (SoC); Advanced eXtensible Interface (AXI); Advanced Microcontroller Bus Architecture (AMBA); AXI Coherency Extensions (ACE); Coherent Hub Interface (CHI);
> Modified, Exclusive, Shared, Invalid (MESI).

**Plain-language purpose:** Define what values CPU cores may observe when private caches and speculative pipelines access shared memory.

## Terms introduced here

| Term | Meaning |
|---|---|
| cache coherence | keeps cached copies of the same address compatible |
| memory consistency | defines legal observation order across different addresses |
| MESI | Modified, Exclusive, Shared, Invalid cache-line permission states |
| atomic operation | indivisible read-modify-write used for synchronization |
| AXI Coherency Extensions (ACE) | AMBA transactions for snoop-based coherent systems |
| Coherent Hub Interface (CHI) | packetized request/home/subordinate coherent protocol |

## Reading order

1. [Cache Coherence](01_Cache_Coherence.md) — permissions, transient states, directories, races, safety, liveness.
2. [Memory Consistency and Atomics](02_Memory_Consistency_and_Atomics.md) — ordering models, litmus tests, fences, atomic serialization.
3. [ACE and CHI](03_ACE_and_CHI.md) — CPU coherence messages on a real scalable interconnect.

**Hands off to:** [SoC On-Chip Networks](../../04_SoC_and_Chiplet_Architecture/04_On_Chip_Networks/00_Index.md) for generic transport.

---

[CPU Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
