# GPU Implementation Blueprints

These blueprints convert the graphics processing unit (GPU) book into a buildable specification. They describe state and interfaces inside a streaming multiprocessor, the memory and scale-up transaction paths, and the software/simulator/verification chain that makes the hardware usable. No particular RTL language or vendor implementation is assumed.

| Order | Blueprint | Reconstruction outcome |
|---:|---|---|
| 1 | [SIMT Core and Tensor Pipeline](01_SIMT_Core_and_Tensor_Pipeline_Implementation_Blueprint.md) | streaming-multiprocessor block specification, warp-state schema, scheduling rules, operand/tensor pipeline sizing, and correctness invariants |
| 2 | [GPU Memory and Scale-Up](02_GPU_Memory_and_Scale_Up_Implementation_Blueprint.md) | coalescer/cache/shared-memory/HBM path, translation and partition contracts, peer-memory rules, and forward-progress plan |
| 3 | [Software, Simulator, Verification, and Bring-up](03_GPU_Software_Simulator_Verification_and_Bringup_Blueprint.md) | compiler/runtime-to-command contract, executable simulator pipeline, validation suite, performance counters, and staged enablement plan |

Read the mechanism chapters first: [GPU Architecture](../01_Core_Architecture/01_GPU_Architecture.md), [Operand Delivery](../01_Core_Architecture/03_Operand_Collectors_Register_Files_and_Scoreboards.md), [GPU Memory](../02_Memory_System/01_Coalescing_Caches_and_Shared_Memory.md), and [GPU Simulators](../04_Simulation/01_GPU_Simulators.md).

---

← [GPU Architecture](../00_Index.md)
