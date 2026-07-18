# Neural Processing Unit (NPU) Architecture

> **Abbreviation key — skim now and return as needed:** graphics processing unit (GPU); system on chip (SoC); input-output memory management unit (IOMMU); direct memory access (DMA).

An NPU accelerates neural-network tensor operations by placing many arithmetic units beside explicitly managed local memory. Its useful performance depends as much on mapping and data movement as on the number of multiply-accumulate units.

> **First-time reader:** Begin with the [Architecture Primer](../05_Architecture_Foundations_and_Methods/01_Reader_Foundations/01_Architecture_Primer_and_Glossary.md). The compute chapter introduces tensors, processing elements, multiply-accumulate operations, and dataflow before using their abbreviations.

~~~mermaid
flowchart LR
    Graph["neural-network graph"] --> Map["tile / layout / precision / sparsity"]
    Map --> Array["systolic / spatial / vector array"]
    Array --> Local["scratchpads + on-chip movement"]
    Local --> Host["host queues / DMA / IOMMU"]
    Sim["NPU simulation"] -. evaluates .-> Map
    Sim -. evaluates .-> Array
~~~

## Subdomains

| Order | Subdomain | Chapters | What it owns |
|---:|---|---:|---|
| 1 | [Compute Dataflows](01_Compute_Dataflows/00_Index.md) | 2 | processing-element arrays, tensor loops, spatial/temporal mapping, utilization |
| 2 | [Mapping and Memory](02_Mapping_and_Memory/00_Index.md) | 2 | tensor tiling, data movement, sparsity, quantization, compression |
| 3 | [System Integration](03_System_Integration/00_Index.md) | 1 | host queues, direct memory access, address translation, scheduling, errors |
| 4 | [NPU Simulation](04_Simulation/00_Index.md) | 1 | mapping, cycle, energy, and area models for accelerator arrays |

## Reading order

[NPU Accelerators](01_Compute_Dataflows/01_NPU_Accelerators.md) → [Dataflows](01_Compute_Dataflows/02_Systolic_Spatial_and_Vector_Dataflows.md) → [Tensor Tiling](02_Mapping_and_Memory/01_Tensor_Tiling_and_Data_Movement.md) → [Sparsity and Quantization](02_Mapping_and_Memory/02_Sparsity_Quantization_and_Compression.md) → [System Integration](03_System_Integration/01_Host_Interface_Memory_Visibility_and_Scheduling.md).

**Hands off to:** [SoC and Chiplet Architecture](../04_SoC_and_Chiplet_Architecture/00_Index.md) when the NPU becomes one agent in a larger chip.

---

← [GPU Architecture](../02_GPU_Architecture/00_Index.md) · [Architecture book](../00_Index.md) · [SoC and Chiplet Architecture](../04_SoC_and_Chiplet_Architecture/00_Index.md) →
