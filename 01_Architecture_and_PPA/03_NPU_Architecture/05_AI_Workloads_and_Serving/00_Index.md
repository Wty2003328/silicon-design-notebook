# Neural Processing Unit (NPU) Architecture › Artificial Intelligence Workloads and Serving

> **Purpose:** Follow an artificial intelligence (AI) workload from model semantics to an NPU executable, then through a production serving system, and finally to defensible latency, throughput, utilization, memory, communication, and energy conclusions.

> **First-time reader:** An **operator** is one graph-level tensor operation, such as matrix multiplication or softmax. A **kernel** is an implementation of one operator or a fused group. A **serving engine** admits requests, batches them, manages persistent state, launches compiled programs, and returns results. These layers are related, but they are not interchangeable.

## Terms introduced here

| Term | Meaning |
|---|---|
| intermediate representation (IR) | compiler data structure between a framework graph and device commands |
| graph partition | division between NPU-supported regions and CPU/GPU/custom fallbacks |
| shape specialization | compiling a program for a particular tensor-shape class |
| prefill | processes all prompt tokens and creates the initial key–value cache |
| decode | repeatedly generates one or a few new tokens using prior key–value state |
| time to first token (TTFT) | request arrival to first generated token |
| time per output token (TPOT) | average time between generated tokens after the first |
| service-level objective (SLO) | required latency, throughput, availability, or quality target |
| attribution | connecting measured time or traffic to a causal compiler, architecture, runtime, or system event |

## Reading order

1. [AI Workload and Graph Mapping to NPUs](01_AI_Workload_and_Graph_Mapping_to_NPUs.md) — model artifact, graph capture, lowering, fusion, tiling, dataflow, quantization, fallbacks, and mappings for dense, sparse, convolutional, attention, embedding, and mixture-of-experts work.
2. [End-to-End AI Inference and Serving on NPUs](02_End_to_End_AI_Inference_and_Serving_on_NPUs.md) — storage-to-device deployment, request admission, prefill/decode, batching, key–value management, host/device queues, multi-NPU sharding, collectives, disaggregation, networking, failures, and tail latency.
3. [Performance, Compiler, Profiling, and Research Methodology](03_Performance_Compiler_Profiling_and_Research_Methodology.md) — analytical bounds, systolic utilization, hierarchical rooflines, capacity and communication models, compiler evidence, counters, traces, simulation, experiments, and claim validation.
4. [NPU Compiler, Runtime, and Executable Blueprint](04_NPU_Compiler_Runtime_and_Executable_Implementation_Blueprint.md) — capability/IR schemas, partition/fallback, compiler passes, executable package, runtime API/state, engine cache, and model residency.
5. [NPU Serving Engine, Scheduler, and State Blueprint](05_NPU_Serving_Engine_Scheduler_and_State_Implementation_Blueprint.md) — profile dispatch, admission/resource ledger, scheduling, KV layout compatibility, fallback, MoE, and multi-NPU transfer.
6. [NPU AI-Stack Verification, Operations, and Deployment](06_NPU_AI_Stack_Verification_Operations_and_Deployment_Blueprint.md) — compiler/numerical/runtime validation, telemetry, capacity, firmware/executable compatibility, canary/drain/rollback, and incident recovery.

## How this section connects to the microarchitecture book

The chapters here own the **end-to-end workload story**. They deliberately link down to the chapters that own each mechanism:

- [Compute Dataflows](../01_Compute_Dataflows/00_Index.md) owns matrix, vector, attention, sparse, and irregular execution structures.
- [Mapping and Memory](../02_Mapping_and_Memory/00_Index.md) owns legal tiling, scratchpad allocation, direct memory access, and event dependencies.
- [System Integration](../03_System_Integration/00_Index.md) owns protected submission, address translation, completion, scheduling, and faults.
- [Simulation](../04_Simulation/00_Index.md) owns mapping/cycle/energy models and their fidelity limits.

All six chapters apply the notebook-wide [Research-Depth and Evidence Standard](../../../Research_Depth_and_Evidence_Standard.md): every conclusion must connect workload, causal mechanism, theoretical assumptions, measurable evidence, validation, and failure boundary.

The three implementation blueprints extend that chain into buildable AI software: artifacts and IR, compiler/executable/runtime contracts, serving state and policy, distributed ownership, validation, observability, deployment, and operations.

---

[NPU Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
