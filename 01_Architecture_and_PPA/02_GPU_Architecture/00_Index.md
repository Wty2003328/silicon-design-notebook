# Graphics Processing Unit (GPU) Architecture

> **Abbreviation key — skim now and return as needed:** central processing unit (CPU); neural processing unit (NPU); system on chip (SoC); single instruction, multiple threads (SIMT); high-bandwidth memory (HBM); key-value (KV); time to first token (TTFT); time per output token (TPOT).

A GPU is a throughput-oriented processor: it keeps many threads resident so another ready group can run while one waits. This book keeps the core, high-bandwidth memory, multi-GPU communication, and GPU simulation together because their limits are coupled.

> **First-time reader:** Start with [GPU Workloads, Performance Modeling, and Design-Space Exploration](00_Design_Methodology/01_GPU_Workloads_Performance_and_DSE.md). “Warp,” “occupancy,” and “coalescing” are GPU-specific terms and are introduced from first principles in the core and memory chapters.

~~~mermaid
flowchart LR
    Kernel["software kernel"] --> Core["SIMT core + warp scheduler"]
    Core --> Mem["coalescing / shared memory / caches"]
    Mem --> HBM["high-bandwidth memory"]
    HBM --> Scale["multi-GPU links + collectives"]
    Scale --> AI["AI operators + serving SLOs"]
    AI -. "shapes / schedules" .-> Core
    Sim["GPU simulation"] -. validates .-> Core
    Sim -. validates .-> Mem
~~~

## Subdomains

| Order | Subdomain | Chapters | What it owns |
|---:|---|---:|---|
| 0 | [GPU Design Methodology](00_Design_Methodology/00_Index.md) | 3 | GPU workload contracts, occupancy/roofline/coalescing DSE, replicated physical costs, and CUDA/PTX/SASS evidence |
| 1 | [Core Architecture](01_Core_Architecture/00_Index.md) | 4 | SIMT execution, scheduling, operand delivery, independent threads, asynchronous pipelines |
| 2 | [Memory System](02_Memory_System/00_Index.md) | 2 | coalescing, shared memory, caches, translation, partitions, HBM |
| 3 | [Scale-Up](03_Scale_Up/00_Index.md) | 1 | peer memory, topology, collectives, placement, communication overlap |
| 4 | [GPU Simulation](04_Simulation/00_Index.md) | 1 | execution/trace models for GPU timing and memory behavior |
| 5 | [AI Workloads and Serving](05_AI_Workloads_and_Serving/00_Index.md) | 3 | operator-to-microarchitecture mapping, end-to-end inference, KV state, batching, SLOs, profiling, and research methods |

## Reading order

[GPU Design Methodology](00_Design_Methodology/00_Index.md) → [GPU Architecture](01_Core_Architecture/01_GPU_Architecture.md) → [SIMT Scheduling](01_Core_Architecture/02_SIMT_Scheduling_and_Occupancy.md) → [Operand Delivery](01_Core_Architecture/03_Operand_Collectors_Register_Files_and_Scoreboards.md) → [Independent Threads and Asynchronous Pipelines](01_Core_Architecture/04_Independent_Thread_Scheduling_and_Asynchronous_Pipelines.md) → [GPU Memory](02_Memory_System/01_Coalescing_Caches_and_Shared_Memory.md) → [HBM](02_Memory_System/02_HBM_and_Advanced_Memory_Systems.md) → [Multi-GPU](03_Scale_Up/01_Multi_GPU_Interconnect_and_Execution.md) → [AI Workloads and Serving](05_AI_Workloads_and_Serving/00_Index.md) → [GPU Simulation](04_Simulation/00_Index.md).

**Hands off to:** [SoC and Chiplet Architecture](../04_SoC_and_Chiplet_Architecture/00_Index.md) for shared on-chip fabrics and die/package composition.

---

← [CPU Architecture](../01_CPU_Architecture/00_Index.md) · [Architecture book](../00_Index.md) · [NPU Architecture](../03_NPU_Architecture/00_Index.md) →
