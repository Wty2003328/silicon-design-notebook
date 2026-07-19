# NPU Architecture › Design Methodology

> **Purpose:** Convert model semantics into operator/data-movement requirements, explore array/vector/memory mappings, price accelerator structures, and establish evidence from framework graph to validated latency, throughput, and energy.

## Terms introduced here

| Term | Meaning in NPU design |
|---|---|
| model contract | exact graph, weights, input shapes, batch/sequence, precision, sparsity, preprocessing, and accuracy target |
| lowering | conversion of framework/graph operators into supported tensor loops, commands, or fallback work |
| mapping | assignment of tensor-loop iterations to processing elements, time, and memory levels |
| dataflow | rule describing which tensor stays stationary and how operands/results move through the accelerator |
| coverage | fraction of graph operations and reference time represented by the NPU model |
| critical path | longest dependency/resource-constrained path that sets graph completion time |

## Reading order

1. [NPU Workloads, Performance Modeling, and DSE](01_NPU_Workloads_Performance_and_DSE.md).
2. [NPU PPA and Physical Implementation](02_NPU_PPA_and_Physical_Implementation.md).
3. [NPU Simulation Methodology and Evidence](03_NPU_Simulation_Methodology_and_Evidence.md).

Then continue to [Compute Dataflows](../01_Compute_Dataflows/00_Index.md).

After the mechanism chapters, use [AI Workloads and Serving](../05_AI_Workloads_and_Serving/00_Index.md) to connect the model compiler, deployment path, online scheduler, multi-NPU system, profiling evidence, and research methodology.

---

← [NPU book index](../00_Index.md) · next → [NPU Workloads, Performance Modeling, and DSE](01_NPU_Workloads_Performance_and_DSE.md)
