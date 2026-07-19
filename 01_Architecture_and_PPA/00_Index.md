# Architecture and Power, Performance, and Area (PPA) — Chip-Architecture Books

> **Abbreviation key — skim now and return as needed:** central processing unit (CPU); graphics processing unit (GPU); neural processing unit (NPU); system on chip (SoC); register-transfer level (RTL);
> single instruction, multiple threads (SIMT); high-bandwidth memory (HBM); double data rate (DDR); network on chip (NoC); Advanced Microcontroller Bus Architecture (AMBA);
> tera operations per second (TOPS); input/output (I/O).

Architecture and AI-stack co-design turn software needs into hardware structure, executable work, runtime policy, and an operable service. The material is organized by the kind of chip being designed—not by detached subsystem names or a shared-method book. It contains **93 substantive chapters in 34 focused subdomains**.

Every research claim should follow the notebook-wide [Research-Depth and Evidence Standard](../Research_Depth_and_Evidence_Standard.md): define terminology and boundaries, derive the mechanism and assumptions, connect it to measurable evidence, validate intermediate observables, and state failure conditions and open questions. Its implementation-reconstruction contract additionally requires enough state, interface, sizing, policy, invariant, physical, verification, and bring-up detail to derive an original block specification without copying code.

> **New to computer architecture?** Choose the processor/system you want to understand and begin with its architecture-owned methodology: [CPU](01_CPU_Architecture/00_Design_Methodology/00_Index.md), [GPU](02_GPU_Architecture/00_Design_Methodology/00_Index.md), [NPU](03_NPU_Architecture/00_Design_Methodology/00_Index.md), or [SoC/chiplet](04_SoC_and_Chiplet_Architecture/00_Design_Methodology/00_Index.md). Each introduces terminology through that architecture's own workloads, structures, PPA constraints, and simulator flow.

## Ownership rule

A topic lives with the architecture whose design decisions give it meaning:

- CPU cache hierarchy, virtual memory, cache coherence, consistency, and coherent CPU fabrics live in **CPU Architecture**.
- GPU high-bandwidth memory, multi-GPU links, and GPU simulators live in **GPU Architecture**.
- NPU dataflow, mapping, integration, and accelerator simulators live in **NPU Architecture**.
- Shared buses, on-chip networks, DDR memory, I/O policy, and chiplets live in **SoC and Chiplet Architecture**.
- Workload selection, performance modeling, PPA, physical structures, and simulation methodology live **inside every architecture book**, because their equations, counters, bottlenecks, input artifacts, and implementation costs change with the architecture.
- AI operator mapping and serving analysis also live **inside every architecture book**: CPU owns host and CPU execution, GPU owns SIMT/matrix/HBM execution, NPU owns graph-to-dataflow lowering, and SoC/chiplet owns heterogeneous composition and end-to-end traffic.
- Implementation blueprints live **inside every architecture book**. They integrate that architecture's mechanisms into buildable contracts rather than creating a detached generic methodology.
- AI-stack implementation blueprints live **inside each architecture's AI subdomain**. They reconstruct framework/compiler/runtime, serving state, distributed execution, validation, observability, deployment, and operations with architecture-specific contracts.

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
| [CPU Architecture](01_CPU_Architecture/00_Index.md) | 11 | 32 | CPU-owned workloads/DSE/PPA/simulation, core/memory/integration blueprints, and reconstructable CPU AI compiler/runtime/serving/operations stack |
| [GPU Architecture](02_GPU_Architecture/00_Index.md) | 7 | 20 | GPU-owned SIMT/tensor/memory/scale-up and hardware blueprints plus reconstructable framework-to-kernel, serving/KV, and deployment stack |
| [NPU Architecture](03_NPU_Architecture/00_Index.md) | 7 | 21 | NPU-owned dataflow/mapping/integration and hardware blueprints plus reconstructable compiler/executable, serving/profile, and deployment stack |
| [SoC and Chiplet Architecture](04_SoC_and_Chiplet_Architecture/00_Index.md) | 9 | 20 | SoC composition and full-chip blueprints plus reconstructable heterogeneous AI control plane, distributed data/state plane, and operations stack |

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
│   ├── 08_Simulation/
│   ├── 09_AI_Workloads_and_Serving/
│   └── 10_Implementation_Blueprints/
├── 02_GPU_Architecture/
│   ├── 00_Design_Methodology/
│   ├── 01_Core_Architecture/
│   ├── 02_Memory_System/
│   ├── 03_Scale_Up/
│   ├── 04_Simulation/
│   ├── 05_AI_Workloads_and_Serving/
│   └── 06_Implementation_Blueprints/
├── 03_NPU_Architecture/
│   ├── 00_Design_Methodology/
│   ├── 01_Compute_Dataflows/
│   ├── 02_Mapping_and_Memory/
│   ├── 03_System_Integration/
│   ├── 04_Simulation/
│   ├── 05_AI_Workloads_and_Serving/
│   └── 06_Implementation_Blueprints/
└── 04_SoC_and_Chiplet_Architecture/
    ├── 00_Design_Methodology/
    ├── 01_System_Modeling/
    ├── 02_Shared_Memory/
    ├── 03_Transaction_Protocols/
    ├── 04_On_Chip_Networks/
    ├── 05_IO_and_Chiplets/
    ├── 06_Simulation/
    ├── 07_AI_Workloads_and_Serving/
    └── 08_Implementation_Blueprints/
~~~

## First-time-reader paths

| Goal | Reading path |
|---|---|
| Learn a CPU from software to memory | [CPU methods](01_CPU_Architecture/00_Design_Methodology/00_Index.md) → [core](01_CPU_Architecture/01_Core_Foundations/00_Index.md) → [frontend](01_CPU_Architecture/02_Frontend_and_Prediction/00_Index.md) → [backend](01_CPU_Architecture/03_Out_of_Order_Backend/00_Index.md) → [cache/coherence](01_CPU_Architecture/04_Cache_Hierarchy/00_Index.md) → [CPU blueprints](01_CPU_Architecture/10_Implementation_Blueprints/00_Index.md) |
| Understand a GPU below CUDA | [GPU methods](02_GPU_Architecture/00_Design_Methodology/00_Index.md) → [GPU core](02_GPU_Architecture/01_Core_Architecture/00_Index.md) → [GPU memory](02_GPU_Architecture/02_Memory_System/00_Index.md) → [scale-up](02_GPU_Architecture/03_Scale_Up/00_Index.md) → [GPU blueprints](02_GPU_Architecture/06_Implementation_Blueprints/00_Index.md) |
| Understand an NPU below TOPS | [NPU methods](03_NPU_Architecture/00_Design_Methodology/00_Index.md) → [NPU compute](03_NPU_Architecture/01_Compute_Dataflows/00_Index.md) → [mapping/memory](03_NPU_Architecture/02_Mapping_and_Memory/00_Index.md) → [integration](03_NPU_Architecture/03_System_Integration/00_Index.md) → [NPU blueprints](03_NPU_Architecture/06_Implementation_Blueprints/00_Index.md) |
| Compose a complete chip | one compute architecture → [SoC/chiplet methods](04_SoC_and_Chiplet_Architecture/00_Design_Methodology/00_Index.md) → [system modeling](04_SoC_and_Chiplet_Architecture/01_System_Modeling/00_Index.md) → [memory](04_SoC_and_Chiplet_Architecture/02_Shared_Memory/00_Index.md) → [NoC](04_SoC_and_Chiplet_Architecture/04_On_Chip_Networks/00_Index.md) → [SoC/chiplet blueprints](04_SoC_and_Chiplet_Architecture/08_Implementation_Blueprints/00_Index.md) |
| Evaluate a design honestly | use that architecture's `00_Design_Methodology`: workload/performance/DSE → PPA/physical implementation → simulation/evidence |
| Analyze AI workloads by architecture | [CPU AI](01_CPU_Architecture/09_AI_Workloads_and_Serving/00_Index.md) · [GPU AI](02_GPU_Architecture/05_AI_Workloads_and_Serving/00_Index.md) · [NPU AI](03_NPU_Architecture/05_AI_Workloads_and_Serving/00_Index.md) · [SoC/chiplet AI](04_SoC_and_Chiplet_Architecture/07_AI_Workloads_and_Serving/00_Index.md) |
| Reconstruct an implementation | [CPU blueprints](01_CPU_Architecture/10_Implementation_Blueprints/00_Index.md) · [GPU blueprints](02_GPU_Architecture/06_Implementation_Blueprints/00_Index.md) · [NPU blueprints](03_NPU_Architecture/06_Implementation_Blueprints/00_Index.md) · [SoC/chiplet blueprints](04_SoC_and_Chiplet_Architecture/08_Implementation_Blueprints/00_Index.md) |
| Reconstruct an AI software stack | [CPU AI stack](01_CPU_Architecture/09_AI_Workloads_and_Serving/04_CPU_AI_Software_Stack_Implementation_Blueprint.md) · [GPU AI stack](02_GPU_Architecture/05_AI_Workloads_and_Serving/04_GPU_Framework_Compiler_Kernel_and_Runtime_Implementation_Blueprint.md) · [NPU AI stack](03_NPU_Architecture/05_AI_Workloads_and_Serving/04_NPU_Compiler_Runtime_and_Executable_Implementation_Blueprint.md) · [heterogeneous AI platform](04_SoC_and_Chiplet_Architecture/07_AI_Workloads_and_Serving/04_Heterogeneous_AI_Platform_and_Control_Plane_Implementation_Blueprint.md) |

---

← [Fundamentals](../00_Fundamentals/00_Index.md) · [Root Index](../Index.md) · [Flow Overview](../Chip_Design_Flow_Overview.md) · [Power and Low-Power](../02_Power_and_Low_Power/00_Index.md) →
