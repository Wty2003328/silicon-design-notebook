# NPU Mapping and Memory

This subdomain owns the compiler–hardware contract that turns tensor operators into tile schedules and compressed data movement.

| Chapter | Owns |
|---|---|
| [Tensor Tiling and Data Movement](01_Tensor_Tiling_and_Data_Movement.md) | loop nests, reuse, buffer capacity, double buffering, NoC scheduling |
| [Sparsity, Quantization, and Compression](02_Sparsity_Quantization_and_Compression.md) | metadata, zero skipping, precision scaling, accumulator sizing, accuracy/performance contracts |

**Up:** [NPU](../00_Index.md) · [Architecture + PPA](../../00_Index.md)
