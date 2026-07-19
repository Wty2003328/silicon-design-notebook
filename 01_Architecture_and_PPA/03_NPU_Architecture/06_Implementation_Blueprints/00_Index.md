# NPU Implementation Blueprints

A neural processing unit (NPU) is not only a multiply-accumulate array. It is a compiler-visible execution machine whose performance depends on graph legality, tiling, dataflow, local-memory lifetime, direct-memory-access scheduling, and runtime admission. These blueprints define those layers well enough to reconstruct an original design without prescribing source code.

| Order | Blueprint | Reconstruction outcome |
|---:|---|---|
| 1 | [Graph Compiler and Execution Array](01_Graph_Compiler_and_Execution_Array_Implementation_Blueprint.md) | supported graph contract, target intermediate representation, command semantics, processing-element/array state, numerical rules, and dataflow sizing |
| 2 | [Scratchpad, DMA, Runtime, and Serving](02_Scratchpad_DMA_Runtime_and_Serving_Implementation_Blueprint.md) | bank/lifetime allocation, descriptor engines, dependency events, host queues, dynamic-work scheduling, and serving-capacity specification |
| 3 | [Verification, Performance, and Bring-up](03_NPU_Verification_Performance_and_Bringup_Blueprint.md) | cross-layer reference models, simulator calibration, PPA evidence, fault tests, counters, staged enablement, and research reproducibility plan |

Prerequisites: [NPU Accelerators](../01_Compute_Dataflows/01_NPU_Accelerators.md), [Tensor Tiling](../02_Mapping_and_Memory/01_Tensor_Tiling_and_Data_Movement.md), [Decoupled Access–Execute](../02_Mapping_and_Memory/03_Decoupled_Access_Execute_and_Scratchpad_Scheduling.md), and [NPU Serving](../05_AI_Workloads_and_Serving/02_End_to_End_AI_Inference_and_Serving_on_NPUs.md).

---

← [NPU Architecture](../00_Index.md)
