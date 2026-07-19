# Graphics Processing Unit (GPU) Architecture › Scale-Up

**Plain-language purpose:** Explain when several GPUs behave like one communication-limited system.

## Terms introduced here

| Term | Meaning |
|---|---|
| topology | physical/logical pattern of device links |
| collective | coordinated communication such as broadcast or all-reduce |
| peer memory | one GPU accesses memory owned by another |
| all-reduce | combines values from all participants and returns the result to all |
| overlap | communication proceeds while independent computation executes |

## Reading order

1. [Multi-GPU Interconnect and Execution](01_Multi_GPU_Interconnect_and_Execution.md) — links, rings/trees, placement, remote memory, failures, counters.

**Hands off to:** [AI Workloads and Serving](../05_AI_Workloads_and_Serving/00_Index.md) for tensor/expert parallelism, MoE all-to-all, KV transfer, and disaggregated inference.

---

[GPU Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
