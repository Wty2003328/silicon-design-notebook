# Central Processing Unit (CPU) Architecture

> **Abbreviation key — skim now and return as needed:** graphics processing unit (GPU); system on chip (SoC); instruction set architecture (ISA); reduced instruction set computer (RISC); translation lookaside buffer (TLB);
> AXI Coherency Extensions (ACE); Coherent Hub Interface (CHI).

A CPU is a general-purpose, latency-oriented machine that must preserve a software-visible instruction contract while internally predicting, reordering, caching, and overlapping work. This book follows one instruction from fetch to retirement, then follows its memory access through translation, cache, coherence, and the system fabric.

> **First-time reader:** Read the [Architecture Primer](../05_Architecture_Foundations_and_Methods/01_Reader_Foundations/01_Architecture_Primer_and_Glossary.md) first if terms such as pipeline, cache line, virtual address, or outstanding request are new.

~~~mermaid
flowchart LR
    ISA["ISA contract"] --> FE["predict / fetch / decode"]
    FE --> BE["rename / schedule / execute"]
    BE --> Retire["retire precise state"]
    BE --> Cache["CPU cache hierarchy"]
    Cache --> VM["virtual-to-physical translation"]
    Cache --> Coh["coherence + consistency"]
    Coh --> Fabric["ACE / CHI coherent fabric"]
    Sim["CPU simulation"] -. tests .-> FE
    Sim -. tests .-> Coh
~~~

## Subdomains

| Order | Subdomain | Chapters | Question answered |
|---:|---|---:|---|
| 1 | [Core Foundations](01_Core_Foundations/00_Index.md) | 3 | What contract does a CPU implement, and where does parallel work come from? |
| 2 | [Frontend and Prediction](02_Frontend_and_Prediction/00_Index.md) | 2 | How are useful instructions delivered before their control path is known? |
| 3 | [Out-of-Order Backend](03_Out_of_Order_Backend/00_Index.md) | 3 | How does the core execute early but still commit in order? |
| 4 | [Cache Hierarchy](04_Cache_Hierarchy/00_Index.md) | 2 | How does the CPU turn locality into low average access time? |
| 5 | [Virtual Memory](05_Virtual_Memory/00_Index.md) | 2 | How are software addresses translated and protected? |
| 6 | [Coherence and Consistency](06_Coherence_and_Consistency/00_Index.md) | 3 | How do CPU cores share memory without observing illegal values? |
| 7 | [Core Case Studies](07_Core_Case_Studies/00_Index.md) | 1 | How do these choices compose in a real open CPU? |
| 8 | [CPU Simulation](08_Simulation/00_Index.md) | 2 | How are CPU timing and coherence hypotheses tested? |

## Beginner reading order

1. [CPU Architecture](01_Core_Foundations/01_CPU_Architecture.md) for the whole-machine map.
2. [RISC-V ISA](01_Core_Foundations/02_RISC_V_ISA.md) for the software-visible contract.
3. [Fetch and Decode](02_Frontend_and_Prediction/02_Fetch_Decode_and_Uop_Delivery.md) and [Out-of-Order Execution](03_Out_of_Order_Backend/01_OoO_Execution.md).
4. [Cache Microarchitecture](04_Cache_Hierarchy/01_Cache_Microarchitecture.md), then [TLB and Virtual Memory](05_Virtual_Memory/01_TLB_and_Virtual_Memory.md).
5. [Cache Coherence](06_Coherence_and_Consistency/01_Cache_Coherence.md), then [Memory Consistency](06_Coherence_and_Consistency/02_Memory_Consistency_and_Atomics.md).

**Hands off to:** [SoC and Chiplet Architecture](../04_SoC_and_Chiplet_Architecture/00_Index.md) when requests leave the CPU-owned coherent subsystem.

---

[Architecture book](../00_Index.md) · next → [GPU Architecture](../02_GPU_Architecture/00_Index.md)
