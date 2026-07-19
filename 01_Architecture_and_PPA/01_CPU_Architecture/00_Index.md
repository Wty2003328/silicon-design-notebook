# Central Processing Unit (CPU) Architecture

> **Abbreviation key — skim now and return as needed:** graphics processing unit (GPU); system on chip (SoC); instruction set architecture (ISA); reduced instruction set computer (RISC); translation lookaside buffer (TLB);
> AXI Coherency Extensions (ACE); Coherent Hub Interface (CHI).

A CPU is a general-purpose, latency-oriented machine that must preserve a software-visible instruction contract while internally predicting, reordering, caching, and overlapping work. This book follows one instruction from fetch to retirement, then follows its memory access through translation, cache, coherence, and the system fabric.

> **First-time reader:** Read [CPU Workloads, Performance Modeling, and Design-Space Exploration](00_Design_Methodology/01_CPU_Workloads_Performance_and_DSE.md) first if terms such as pipeline, cache line, virtual address, or outstanding request are new.

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
| 0 | [CPU Design Methodology](00_Design_Methodology/00_Index.md) | 3 | Which workloads matter, what bottleneck is active, what does it cost physically, and what evidence supports the result? |
| 1 | [Core Foundations](01_Core_Foundations/00_Index.md) | 3 | What contract does a CPU implement, and where does parallel work come from? |
| 2 | [Frontend and Prediction](02_Frontend_and_Prediction/00_Index.md) | 3 | How are instructions predicted, delivered, validated, and recovered? |
| 3 | [Out-of-Order Backend](03_Out_of_Order_Backend/00_Index.md) | 4 | How does the core schedule and replay early work while still committing in order? |
| 4 | [Cache Hierarchy](04_Cache_Hierarchy/00_Index.md) | 2 | How does the CPU turn locality into low average access time? |
| 5 | [Virtual Memory](05_Virtual_Memory/00_Index.md) | 2 | How are software addresses translated and protected? |
| 6 | [Coherence and Consistency](06_Coherence_and_Consistency/00_Index.md) | 3 | How do CPU cores share memory without observing illegal values? |
| 7 | [Core Case Studies](07_Core_Case_Studies/00_Index.md) | 1 | How do these choices compose in a real open CPU? |
| 8 | [CPU Simulation](08_Simulation/00_Index.md) | 2 | How are CPU timing and coherence hypotheses tested? |
| 9 | [AI Workloads and Serving](09_AI_Workloads_and_Serving/00_Index.md) | 3 | How do training and inference workloads map onto CPU execution, memory, and serving systems? |

## Beginner reading order

1. [CPU Design Methodology](00_Design_Methodology/00_Index.md) for CPU terminology, workloads, performance equations, physical costs, and evidence.
2. [CPU Architecture](01_Core_Foundations/01_CPU_Architecture.md) for the whole-machine map.
3. [RISC-V ISA](01_Core_Foundations/02_RISC_V_ISA.md) for the software-visible contract.
4. [Fetch and Decode](02_Frontend_and_Prediction/02_Fetch_Decode_and_Uop_Delivery.md), [Speculative Execution](02_Frontend_and_Prediction/03_Speculative_Execution.md), and [Out-of-Order Execution](03_Out_of_Order_Backend/01_OoO_Execution.md).
5. [Advanced Scheduling, Wakeup, and Replay](03_Out_of_Order_Backend/04_Advanced_Scheduling_Wakeup_and_Replay.md) when the baseline pipeline is clear.
6. [Cache Microarchitecture](04_Cache_Hierarchy/01_Cache_Microarchitecture.md), then [TLB and Virtual Memory](05_Virtual_Memory/01_TLB_and_Virtual_Memory.md).
7. [Cache Coherence](06_Coherence_and_Consistency/01_Cache_Coherence.md), then [Memory Consistency](06_Coherence_and_Consistency/02_Memory_Consistency_and_Atomics.md).
8. [AI Workloads and Serving](09_AI_Workloads_and_Serving/00_Index.md) to connect model loading, tokenization, retrieval, tensor kernels, memory placement, and service-level objectives to the CPU mechanisms above.

**Hands off to:** [SoC and Chiplet Architecture](../04_SoC_and_Chiplet_Architecture/00_Index.md) when requests leave the CPU-owned coherent subsystem.

---

[Architecture book](../00_Index.md) · next → [GPU Architecture](../02_GPU_Architecture/00_Index.md)
