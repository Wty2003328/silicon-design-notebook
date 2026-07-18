# Architecture and Power, Performance, and Area (PPA) — Chip-Architecture Books

> **Abbreviation key — skim now and return as needed:** central processing unit (CPU); graphics processing unit (GPU); neural processing unit (NPU); system on chip (SoC); register-transfer level (RTL);
> single instruction, multiple threads (SIMT); high-bandwidth memory (HBM); double data rate (DDR); network on chip (NoC); Advanced Microcontroller Bus Architecture (AMBA);
> tera operations per second (TOPS); input/output (I/O).

Architecture turns software needs into hardware structure before register-transfer level (RTL) design begins. The material is organized by the kind of chip being designed—not by detached subsystem names. It contains **53 substantive chapters in 28 focused subdomains**.

> **New to computer architecture?** Begin with [Architecture Primer — A First Map of the Machine](05_Architecture_Foundations_and_Methods/01_Reader_Foundations/01_Architecture_Primer_and_Glossary.md). It explains the basic ideas, units, and abbreviations assumed by the later books.

## Ownership rule

A topic lives with the architecture whose design decisions give it meaning:

- CPU cache hierarchy, virtual memory, cache coherence, consistency, and coherent CPU fabrics live in **CPU Architecture**.
- GPU high-bandwidth memory, multi-GPU links, and GPU simulators live in **GPU Architecture**.
- NPU dataflow, mapping, integration, and accelerator simulators live in **NPU Architecture**.
- Shared buses, on-chip networks, DDR memory, I/O policy, and chiplets live in **SoC and Chiplet Architecture**.
- Only concepts reused without changing ownership—reader foundations, performance/PPA methods, circuit structures, and simulation methodology—live in **Architecture Foundations and Methods**.

~~~mermaid
flowchart LR
    Need["workload + product goal"] --> Primer["reader foundations"]
    Primer --> CPU["CPU architecture"]
    Primer --> GPU["GPU architecture"]
    Primer --> NPU["NPU architecture"]
    CPU --> SoC["SoC + chiplet composition"]
    GPU --> SoC
    NPU --> SoC
    Methods["modeling + PPA + simulation methods"] --> CPU
    Methods --> GPU
    Methods --> NPU
    Methods --> SoC
~~~

## Five books

| Book | Subdomains | Chapters | What it owns |
|---|---:|---:|---|
| [CPU Architecture](01_CPU_Architecture/00_Index.md) | 8 | 20 | CPU contract, prediction/speculation, out-of-order scheduling/replay, cache/VM, coherence, case studies, simulation |
| [GPU Architecture](02_GPU_Architecture/00_Index.md) | 4 | 8 | SIMT core, operand delivery, independent-thread/asynchronous pipelines, memory/HBM, scale-up, simulation |
| [NPU Architecture](03_NPU_Architecture/00_Index.md) | 4 | 9 | dense/Transformer/sparse dataflows, mapping, decoupled access/execute, host integration, simulation |
| [SoC and Chiplet Architecture](04_SoC_and_Chiplet_Architecture/00_Index.md) | 6 | 8 | system composition, DDR, AMBA protocols, NoC, I/O policy, chiplets |
| [Architecture Foundations and Methods](05_Architecture_Foundations_and_Methods/00_Index.md) | 6 | 8 | primer, performance/PPA, memory structures, simulation method/tool selection |

## Folder tree

~~~text
01_Architecture_and_PPA/
├── 01_CPU_Architecture/
│   ├── 01_Core_Foundations/
│   ├── 02_Frontend_and_Prediction/
│   ├── 03_Out_of_Order_Backend/
│   ├── 04_Cache_Hierarchy/
│   ├── 05_Virtual_Memory/
│   ├── 06_Coherence_and_Consistency/
│   ├── 07_Core_Case_Studies/
│   └── 08_Simulation/
├── 02_GPU_Architecture/
│   ├── 01_Core_Architecture/
│   ├── 02_Memory_System/
│   ├── 03_Scale_Up/
│   └── 04_Simulation/
├── 03_NPU_Architecture/
│   ├── 01_Compute_Dataflows/
│   ├── 02_Mapping_and_Memory/
│   ├── 03_System_Integration/
│   └── 04_Simulation/
├── 04_SoC_and_Chiplet_Architecture/
│   ├── 01_System_Modeling/
│   ├── 02_Shared_Memory/
│   ├── 03_Transaction_Protocols/
│   ├── 04_On_Chip_Networks/
│   ├── 05_IO_and_Chiplets/
│   └── 06_Simulation/
└── 05_Architecture_Foundations_and_Methods/
    ├── 01_Reader_Foundations/
    ├── 02_Performance_Analysis/
    ├── 03_PPA_Estimation/
    ├── 04_Hardware_Structures/
    ├── 05_Simulation_Methodology/
    └── 06_Tool_Landscape/
~~~

## First-time-reader paths

| Goal | Reading path |
|---|---|
| Understand what the words mean | [Primer](05_Architecture_Foundations_and_Methods/01_Reader_Foundations/01_Architecture_Primer_and_Glossary.md) → choose one architecture book |
| Learn a CPU from software to memory | [CPU foundations](01_CPU_Architecture/01_Core_Foundations/00_Index.md) → [frontend](01_CPU_Architecture/02_Frontend_and_Prediction/00_Index.md) → [backend](01_CPU_Architecture/03_Out_of_Order_Backend/00_Index.md) → [cache](01_CPU_Architecture/04_Cache_Hierarchy/00_Index.md) → [coherence](01_CPU_Architecture/06_Coherence_and_Consistency/00_Index.md) |
| Understand a GPU below CUDA | [GPU core](02_GPU_Architecture/01_Core_Architecture/00_Index.md) → [GPU memory](02_GPU_Architecture/02_Memory_System/00_Index.md) → [scale-up](02_GPU_Architecture/03_Scale_Up/00_Index.md) |
| Understand an NPU below TOPS | [NPU compute](03_NPU_Architecture/01_Compute_Dataflows/00_Index.md) → [mapping/memory](03_NPU_Architecture/02_Mapping_and_Memory/00_Index.md) → [integration](03_NPU_Architecture/03_System_Integration/00_Index.md) |
| Compose a complete chip | one compute architecture → [SoC system modeling](04_SoC_and_Chiplet_Architecture/01_System_Modeling/00_Index.md) → [memory](04_SoC_and_Chiplet_Architecture/02_Shared_Memory/00_Index.md) → [NoC](04_SoC_and_Chiplet_Architecture/04_On_Chip_Networks/00_Index.md) |
| Evaluate a design honestly | [workloads/performance](05_Architecture_Foundations_and_Methods/02_Performance_Analysis/00_Index.md) → [PPA uncertainty](05_Architecture_Foundations_and_Methods/03_PPA_Estimation/00_Index.md) → [simulation methodology](05_Architecture_Foundations_and_Methods/05_Simulation_Methodology/00_Index.md) |

---

← [Fundamentals](../00_Fundamentals/00_Index.md) · [Root Index](../Index.md) · [Flow Overview](../Chip_Design_Flow_Overview.md) · [Power and Low-Power](../02_Power_and_Low_Power/00_Index.md) →
