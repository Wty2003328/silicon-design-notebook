# Neural Processing Unit (NPU) Architecture › Compute Dataflows

> **Abbreviation key — skim now and return as needed:** processing element (PE); multiply-accumulate (MAC).

**Plain-language purpose:** Explain how a neural processing unit (NPU) maps tensor arithmetic across repeated compute units and local communication.

## Terms introduced here

| Term | Meaning |
|---|---|
| tensor | multidimensional array of numbers |
| multiply-accumulate (MAC) | multiply two values and add the product to a running sum |
| processing element (PE) | repeated arithmetic/storage unit in an accelerator array |
| systolic array | neighboring PEs pass data rhythmically through a regular grid |
| dataflow | choice of what stays local and what moves through space/time |

## Reading order

1. [NPU Accelerators](01_NPU_Accelerators.md) — why spatial hardware helps tensor workloads.
2. [Systolic, Spatial, and Vector Dataflows](02_Systolic_Spatial_and_Vector_Dataflows.md) — loop mapping, reuse, fill/drain, reductions, utilization.

**Hands off to:** [Mapping and Memory](../02_Mapping_and_Memory/00_Index.md).

---

[NPU Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
