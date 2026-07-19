# Neural Processing Unit (NPU) Architecture › System Integration

> **Abbreviation key — skim now and return as needed:** central processing unit (CPU); input-output memory management unit (IOMMU); direct memory access (DMA).

**Plain-language purpose:** Make the NPU a safe, schedulable system agent rather than an isolated arithmetic array.

## Terms introduced here

| Term | Meaning |
|---|---|
| command queue | ordered descriptors telling an accelerator what work to run |
| direct memory access (DMA) | device moves data without CPU copying each word |
| input-output memory management unit (IOMMU) | device address translation and protection |
| doorbell | small host write notifying a device that commands are ready |
| completion | ordered record or interrupt saying command results are visible |
| virtualization | safely shares hardware among isolated software contexts |

## Reading order

1. [Host Interface, Memory Visibility, and Scheduling](01_Host_Interface_Memory_Visibility_and_Scheduling.md) — how the NPU participates as a client without relocating CPU coherence protocol design into this book.

**Hands off to:** [End-to-End AI Inference and Serving on NPUs](../05_AI_Workloads_and_Serving/02_End_to_End_AI_Inference_and_Serving_on_NPUs.md) for model loading, admission, batching, KV lifetime, prefill/decode, collectives, disaggregation, tail latency, and the way serving software uses these mechanisms; then to drivers, runtimes, firmware, register-transfer level command processors, and verification.

---

[NPU Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
