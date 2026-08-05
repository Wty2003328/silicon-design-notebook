# Central Processing Unit (CPU) Architecture › Coherence and Consistency

> **Abbreviation key — skim now and return as needed:** system on chip (SoC); Advanced eXtensible Interface (AXI); Advanced Microcontroller Bus Architecture (AMBA); AXI Coherency Extensions (ACE); Coherent Hub Interface (CHI);
> Modified, Exclusive, Shared, Invalid (MESI); atomic memory operation (AMO); compare-and-swap (CAS); load-reserved/store-conditional (LR/SC); hardware transactional memory (HTM).

**Plain-language purpose:** Define what values CPU cores may observe when private caches and speculative pipelines access shared memory.

## Terms introduced here

| Term | Meaning |
|---|---|
| cache coherence | keeps cached copies of the same address compatible |
| memory consistency | defines legal observation order across different addresses |
| MESI | Modified, Exclusive, Shared, Invalid cache-line permission states |
| atomic operation | indivisible read-modify-write used for synchronization |
| serialization point | the one place in the machine that orders an atomic against every other agent |
| load-reserved / store-conditional | a paired sequence whose store fails if the reservation was lost |
| AXI Coherency Extensions (ACE) | AMBA transactions for snoop-based coherent systems |
| Coherent Hub Interface (CHI) | packetized request/home/subordinate coherent protocol |

## Reading order

1. [Cache Coherence](01_Cache_Coherence.md) — permissions, transient states, directories, races, safety, liveness.
2. [Memory Consistency](02_Memory_Consistency_and_Atomics.md) — which cross-core observations are legal: ordering models and litmus tests, fence decode and completion ledgers, store-buffer ordering points, scope and cumulativity, multi-copy atomicity, the compiler and device-ordering boundaries, and verification by litmus test and formal model. Its file name still ends in *and Atomics*, which is deliberate — the name was kept so that inbound links keep resolving — but the read-modify-write machinery now lives in 04.
3. [ACE and CHI](03_ACE_and_CHI.md) — one complete chapter from the broadcast-to-directory motivation through ACE channels and snoop-controller design, CHI nodes/flits/credits/home-node hardware, canonical flows, transient races, deadlock, verification, and debug.
4. [Atomic Operations](04_Atomic_Operations.md) — the read-modify-write made indivisible and what that costs: what indivisibility actually constrains, choosing the serialization point near the core or at the home node, the fetch-and-op family, compare-and-swap and ABA, load-reserved/store-conditional and the inside of the reservation monitor, ordering qualifiers, the three ISAs (RISC-V A/Zacas/Zabha, Arm exclusives versus LSE, x86 `LOCK`), the atomic inside an out-of-order core, precise faults and cancellation, contention arithmetic and lock algorithms, hardware transactional memory and lock elision, performance counters, and the verification plan.

**Which first?** Ordering questions — "is this observation legal?" — go 01 → 02. Synchronization questions — "what does this lock cost, and where does it serialize?" — go 01 → 04, returning to 02 §8 for the fences the atomic has to carry. Read 03 before 04 if your atomic has to cross a coherent fabric rather than stop at a shared cache.

**Hands off to:** [SoC On-Chip Networks](../../04_SoC_and_Chiplet_Architecture/04_On_Chip_Networks/00_Index.md) for generic transport, and — when the serialization point sits outside every core — [System Atomics and Exclusive Access](../../04_SoC_and_Chiplet_Architecture/02_Shared_Memory/04_System_Atomics_and_Exclusive_Access.md). The throughput-machine counterpart of 04 is [GPU Atomics and Synchronization](../../02_GPU_Architecture/02_Memory_System/03_GPU_Atomics_and_Synchronization.md).

---

[CPU Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
