# Neural Processing Unit (NPU) Architecture › Compute Dataflows

> **Abbreviation key — skim now and return as needed:** processing element (PE); multiply-accumulate (MAC); key–value (KV); mixture of experts (MoE).

**Plain-language purpose:** Explain how a neural processing unit (NPU) maps tensor arithmetic across repeated compute units and local communication.

## Terms introduced here

| Term | Meaning |
|---|---|
| tensor | multidimensional array of numbers |
| multiply-accumulate (MAC) | multiply two values and add the product to a running sum |
| processing element (PE) | repeated arithmetic/storage unit in an accelerator array |
| systolic array | neighboring PEs pass data rhythmically through a regular grid |
| dataflow | choice of what stays local and what moves through space/time |
| attention engine | matrix, reduction, vector, scratchpad, and KV-cache pipeline for Transformer attention |
| mixture of experts (MoE) | routes each token to a small dynamically selected subset of expert networks |

## Reading order

1. [NPU Accelerators](01_NPU_Accelerators.md) — why spatial hardware helps tensor workloads.
2. [Systolic, Spatial, and Vector Dataflows](02_Systolic_Spatial_and_Vector_Dataflows.md) — loop mapping, reuse, fill/drain, reductions, utilization.
3. [Transformer and Attention Engines](03_Transformer_and_Attention_Engine_Microarchitecture.md) — prefill/decode shapes, online softmax, KV cache, heterogeneous engines, and token scheduling.
4. [Dynamic Sparsity, MoE, and Irregular Execution](04_Dynamic_Sparsity_MoE_and_Irregular_Execution.md) — metadata decode, matching, load balance, token routing, expert queues, and dense fallback.

**Hands off to:** [Mapping and Memory](../02_Mapping_and_Memory/00_Index.md), then [AI Workload and Graph Mapping](../05_AI_Workloads_and_Serving/01_AI_Workload_and_Graph_Mapping_to_NPUs.md) for complete model/operator coverage and compiler lowering.

---

[NPU Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
