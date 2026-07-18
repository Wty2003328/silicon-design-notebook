# Neural Processing Unit (NPU) Architecture › Mapping and Memory

**Plain-language purpose:** Translate a tensor operation into tiles that fit local storage and expose enough reuse and parallel work.

## Terms introduced here

| Term | Meaning |
|---|---|
| tile | smaller block of a tensor operation processed at one time |
| scratchpad | software/compiler-managed on-chip memory |
| double buffering | one buffer computes while another transfers next data |
| sparsity | some represented values are zero and may be skipped |
| quantization | represents values with fewer or different numeric levels |
| metadata | extra information describing compressed or sparse data |

## Reading order

1. [Tensor Tiling and Data Movement](01_Tensor_Tiling_and_Data_Movement.md).
2. [Sparsity, Quantization, and Compression](02_Sparsity_Quantization_and_Compression.md).

**Hands off to:** [System Integration](../03_System_Integration/00_Index.md).

---

[NPU Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
