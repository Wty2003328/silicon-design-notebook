# Neural Processing Unit (NPU) Architecture

> **Abbreviation key — skim now and return as needed:** graphics processing unit (GPU); system on chip (SoC); input-output memory management unit (IOMMU); direct memory access (DMA); key–value (KV) cache; time to first token (TTFT); time per output token (TPOT).

An NPU accelerates neural-network tensor operations by placing many arithmetic units beside explicitly managed local memory. Its useful performance depends as much on mapping and data movement as on the number of multiply-accumulate units.

> **First-time reader:** Begin with [NPU Workloads, Performance Modeling, and Design-Space Exploration](00_Design_Methodology/01_NPU_Workloads_Performance_and_DSE.md). The compute chapter introduces tensors, processing elements, multiply-accumulate operations, and dataflow before using their abbreviations.

~~~mermaid
flowchart LR
    Graph["neural-network graph"] --> Map["tile / layout / precision / sparsity"]
    Map --> Array["systolic / spatial / vector array"]
    Array --> Local["scratchpads + on-chip movement"]
    Local --> Host["host queues / DMA / IOMMU"]
    Host --> Serve["admission / batching / prefill / decode"]
    Serve --> Metric["TTFT / TPOT / throughput / energy"]
    Sim["NPU simulation"] -. evaluates .-> Map
    Sim -. evaluates .-> Array
    Sim -. calibrates .-> Metric
    Build["implementation blueprints"] -. specifies .-> Map
    Build -. specifies .-> Array
~~~

## Subdomains

| Order | Subdomain | Chapters | What it owns |
|---:|---|---:|---|
| 0 | [NPU Design Methodology](00_Design_Methodology/00_Index.md) | 3 | graph/shape workload contracts, mapping/DSE, accelerator physical costs, and framework-to-results evidence |
| 1 | [Compute Dataflows](01_Compute_Dataflows/00_Index.md) | 4 | arrays, attention engines, Transformer prefill/decode, sparsity, MoE, irregular execution |
| 2 | [Mapping and Memory](02_Mapping_and_Memory/00_Index.md) | 3 | tiling, compression, descriptor DMA, scratchpad lifetime, event-driven scheduling |
| 3 | [System Integration](03_System_Integration/00_Index.md) | 1 | host queues, direct memory access, address translation, scheduling, errors |
| 4 | [NPU Simulation](04_Simulation/00_Index.md) | 1 | mapping, cycle, energy, and area models for accelerator arrays |
| 5 | [AI Workloads and Serving](05_AI_Workloads_and_Serving/00_Index.md) | 6 | mapping/serving/research plus reconstructable compiler/executable/runtime, profile scheduler/state, validation, and deployment stacks |
| 6 | [Implementation Blueprints](06_Implementation_Blueprints/00_Index.md) | 3 | graph/compiler and array contracts, scratchpad/DMA/runtime state, verification, performance evidence, and bring-up |

## Reading order

[NPU Design Methodology](00_Design_Methodology/00_Index.md) → [NPU Accelerators](01_Compute_Dataflows/01_NPU_Accelerators.md) → [Dataflows](01_Compute_Dataflows/02_Systolic_Spatial_and_Vector_Dataflows.md) → [Transformer and Attention Engines](01_Compute_Dataflows/03_Transformer_and_Attention_Engine_Microarchitecture.md) → [Dynamic Sparsity and MoE](01_Compute_Dataflows/04_Dynamic_Sparsity_MoE_and_Irregular_Execution.md) → [Tensor Tiling](02_Mapping_and_Memory/01_Tensor_Tiling_and_Data_Movement.md) → [Sparsity and Quantization](02_Mapping_and_Memory/02_Sparsity_Quantization_and_Compression.md) → [Decoupled Access–Execute](02_Mapping_and_Memory/03_Decoupled_Access_Execute_and_Scratchpad_Scheduling.md) → [System Integration](03_System_Integration/01_Host_Interface_Memory_Visibility_and_Scheduling.md) → [AI Workload and Graph Mapping](05_AI_Workloads_and_Serving/01_AI_Workload_and_Graph_Mapping_to_NPUs.md) → [End-to-End NPU Serving](05_AI_Workloads_and_Serving/02_End_to_End_AI_Inference_and_Serving_on_NPUs.md) → [Performance and Research Methodology](05_AI_Workloads_and_Serving/03_Performance_Compiler_Profiling_and_Research_Methodology.md) → [Simulation](04_Simulation/00_Index.md) → [Implementation Blueprints](06_Implementation_Blueprints/00_Index.md).

**Hands off to:** [SoC and Chiplet Architecture](../04_SoC_and_Chiplet_Architecture/00_Index.md) when the NPU becomes one agent in a larger chip.

---

← [GPU Architecture](../02_GPU_Architecture/00_Index.md) · [Architecture book](../00_Index.md) · [SoC and Chiplet Architecture](../04_SoC_and_Chiplet_Architecture/00_Index.md) →
