# Architecture and Power, Performance, and Area (PPA) — Chip-Architecture Books

> **Abbreviation key — skim now and return as needed:** central processing unit (CPU); graphics processing unit (GPU); neural processing unit (NPU); system on chip (SoC); register-transfer level (RTL);
> single instruction, multiple threads (SIMT); high-bandwidth memory (HBM); double data rate (DDR); network on chip (NoC); Advanced Microcontroller Bus Architecture (AMBA);
> tera operations per second (TOPS); input/output (I/O).

Architecture turns software needs into hardware structure before register-transfer level (RTL) design begins. The material is organized by the kind of chip being designed—not by detached subsystem names or a shared-method book. It contains **57 substantive chapters in 26 focused subdomains**.

> **New to computer architecture?** Choose the processor/system you want to understand and begin with its architecture-owned methodology: [CPU](01_CPU_Architecture/00_Design_Methodology/00_Index.md), [GPU](02_GPU_Architecture/00_Design_Methodology/00_Index.md), [NPU](03_NPU_Architecture/00_Design_Methodology/00_Index.md), or [SoC/chiplet](04_SoC_and_Chiplet_Architecture/00_Design_Methodology/00_Index.md). Each introduces terminology through that architecture's own workloads, structures, PPA constraints, and simulator flow.

## Ownership rule

A topic lives with the architecture whose design decisions give it meaning:

- CPU cache hierarchy, virtual memory, cache coherence, consistency, and coherent CPU fabrics live in **CPU Architecture**.
- GPU high-bandwidth memory, multi-GPU links, and GPU simulators live in **GPU Architecture**.
- NPU dataflow, mapping, integration, and accelerator simulators live in **NPU Architecture**.
- Shared buses, on-chip networks, DDR memory, I/O policy, and chiplets live in **SoC and Chiplet Architecture**.
- Workload selection, performance modeling, PPA, physical structures, and simulation methodology live **inside every architecture book**, because their equations, counters, bottlenecks, input artifacts, and implementation costs change with the architecture.

~~~mermaid
flowchart LR
    Need["workload + product goal"] --> CM["CPU methods"] --> CPU["CPU architecture"]
    Need --> GM["GPU methods"] --> GPU["GPU architecture"]
    Need --> NM["NPU methods"] --> NPU["NPU architecture"]
    CPU --> SM["SoC/chiplet methods"]
    GPU --> SM
    NPU --> SM
    SM --> SoC["SoC + chiplet composition"]
~~~

## Four architecture books

| Book | Subdomains | Chapters | What it owns |
|---|---:|---:|---|
| [CPU Architecture](01_CPU_Architecture/00_Index.md) | 9 | 23 | CPU-owned workloads/DSE/PPA/simulation method, prediction/speculation, out-of-order scheduling/replay, cache/VM, coherence, case studies |
| [GPU Architecture](02_GPU_Architecture/00_Index.md) | 5 | 11 | GPU-owned workloads/DSE/PPA/simulation method, SIMT scheduling, operand delivery, memory/HBM, scale-up |
| [NPU Architecture](03_NPU_Architecture/00_Index.md) | 5 | 12 | NPU-owned graph/mapping/DSE/PPA/simulation method, dense/Transformer/sparse dataflows, integration |
| [SoC and Chiplet Architecture](04_SoC_and_Chiplet_Architecture/00_Index.md) | 7 | 11 | SoC-owned use cases/DSE/PPA/simulation method, system composition, DDR, AMBA, NoC, I/O, chiplets |

## Folder tree

~~~text
01_Architecture_and_PPA/
├── 01_CPU_Architecture/
│   ├── 00_Design_Methodology/
│   ├── 01_Core_Foundations/
│   ├── 02_Frontend_and_Prediction/
│   ├── 03_Out_of_Order_Backend/
│   ├── 04_Cache_Hierarchy/
│   ├── 05_Virtual_Memory/
│   ├── 06_Coherence_and_Consistency/
│   ├── 07_Core_Case_Studies/
│   └── 08_Simulation/
├── 02_GPU_Architecture/
│   ├── 00_Design_Methodology/
│   ├── 01_Core_Architecture/
│   ├── 02_Memory_System/
│   ├── 03_Scale_Up/
│   └── 04_Simulation/
├── 03_NPU_Architecture/
│   ├── 00_Design_Methodology/
│   ├── 01_Compute_Dataflows/
│   ├── 02_Mapping_and_Memory/
│   ├── 03_System_Integration/
│   └── 04_Simulation/
└── 04_SoC_and_Chiplet_Architecture/
    ├── 00_Design_Methodology/
    ├── 01_System_Modeling/
    ├── 02_Shared_Memory/
    ├── 03_Transaction_Protocols/
    ├── 04_On_Chip_Networks/
    ├── 05_IO_and_Chiplets/
    └── 06_Simulation/
~~~

## First-time-reader paths

| Goal | Reading path |
|---|---|
| Learn a CPU from software to memory | [CPU methods](01_CPU_Architecture/00_Design_Methodology/00_Index.md) → [core](01_CPU_Architecture/01_Core_Foundations/00_Index.md) → [frontend](01_CPU_Architecture/02_Frontend_and_Prediction/00_Index.md) → [backend](01_CPU_Architecture/03_Out_of_Order_Backend/00_Index.md) → [cache/coherence](01_CPU_Architecture/04_Cache_Hierarchy/00_Index.md) |
| Understand a GPU below CUDA | [GPU methods](02_GPU_Architecture/00_Design_Methodology/00_Index.md) → [GPU core](02_GPU_Architecture/01_Core_Architecture/00_Index.md) → [GPU memory](02_GPU_Architecture/02_Memory_System/00_Index.md) → [scale-up](02_GPU_Architecture/03_Scale_Up/00_Index.md) |
| Understand an NPU below TOPS | [NPU methods](03_NPU_Architecture/00_Design_Methodology/00_Index.md) → [NPU compute](03_NPU_Architecture/01_Compute_Dataflows/00_Index.md) → [mapping/memory](03_NPU_Architecture/02_Mapping_and_Memory/00_Index.md) → [integration](03_NPU_Architecture/03_System_Integration/00_Index.md) |
| Compose a complete chip | one compute architecture → [SoC/chiplet methods](04_SoC_and_Chiplet_Architecture/00_Design_Methodology/00_Index.md) → [system modeling](04_SoC_and_Chiplet_Architecture/01_System_Modeling/00_Index.md) → [memory](04_SoC_and_Chiplet_Architecture/02_Shared_Memory/00_Index.md) → [NoC](04_SoC_and_Chiplet_Architecture/04_On_Chip_Networks/00_Index.md) |
| Evaluate a design honestly | use that architecture's `00_Design_Methodology`: workload/performance/DSE → PPA/physical implementation → simulation/evidence |

---

← [Fundamentals](../00_Fundamentals/00_Index.md) · [Root Index](../Index.md) · [Flow Overview](../Chip_Design_Flow_Overview.md) · [Power and Low-Power](../02_Power_and_Low_Power/00_Index.md) →
