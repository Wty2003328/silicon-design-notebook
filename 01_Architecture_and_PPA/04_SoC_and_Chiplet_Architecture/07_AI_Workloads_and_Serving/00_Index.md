# System-on-Chip and Chiplet Architecture › Artificial Intelligence Workloads and Serving

> **Abbreviation key:** artificial intelligence (AI); central processing unit (CPU); graphics processing unit (GPU); neural processing unit (NPU); high-bandwidth memory (HBM); key-value (KV) cache; direct memory access (DMA); remote direct memory access (RDMA); network interface controller (NIC); service-level objective (SLO); time to first token (TTFT); time per output token (TPOT); network on chip (NoC).

This section owns the system behavior that appears only after CPU orchestration, accelerator execution, memory, storage, and communication are composed. The device books explain how an operator executes *inside* a CPU, GPU, or NPU; these chapters explain how a complete AI request moves *between* them and how to attribute end-to-end performance.

The section follows the notebook-wide [Research-Depth and Evidence Standard](../../../Research_Depth_and_Evidence_Standard.md): terminology, boundary, mechanism, theory, observables, validation, worked reasoning, failure modes, and research questions must remain explicit.

## Reading order

1. [End-to-End AI Serving on SoCs and Chiplet Systems](01_End_to_End_AI_Serving_on_SoCs_and_Chiplets.md) — model provisioning, request admission, prefill, decode, KV state, sampling, and disaggregated execution.
2. [AI Workload Mapping to SoC, Memory, NoC, and Chiplets](02_AI_Workload_Mapping_to_SoC_Memory_NoC_and_Chiplets.md) — operator traffic signatures, parallelism, collectives, memory ownership, QoS, and chiplet partition choices.
3. [AI Serving Performance Analysis and Research Methodology](03_AI_Serving_Performance_Analysis_and_Research_Methodology.md) — TTFT/TPOT/goodput, capacity and communication equations, queueing, measurement, simulation, and research questions.
4. [Heterogeneous AI Platform and Control-Plane Blueprint](04_Heterogeneous_AI_Platform_and_Control_Plane_Implementation_Blueprint.md) — artifact registry, topology inventory, placement solver, worker-group specifications, model residency, unified runtime, routing, autoscaling, and security.
5. [Distributed AI Serving Data-Plane and State Blueprint](05_Distributed_AI_Serving_Data_Plane_and_State_Implementation_Blueprint.md) — request/commit state machine, reservations, KV transfer, collectives, memory tiers, backpressure, failure fencing, and recovery.
6. [AI Platform Verification, Operations, and Deployment](06_AI_Platform_Verification_Operations_and_Deployment_Blueprint.md) — composed validation, chaos/fencing, end-to-end telemetry, capacity/resilience, progressive delivery, rollback, disaster recovery, and governance.

## Ownership boundary

- CPU/GPU/NPU chapters own execution resources, instruction/dataflow mechanisms, kernel mapping, and device-local counters.
- This section owns the host-device-storage-network path, cross-device placement, system scheduling, contention, communication, tail latency, and SLO-constrained capacity.
- Production software names are examples of policies and mechanisms, not architecture definitions. Claims must be reproducible from an explicit model, trace, counter set, or experiment.
- The implementation target includes the AI control plane and data plane: schemas, APIs, placement, state authority, scheduler and transfer protocols, failure fencing, validation, observability, deployment, and recovery must be reconstructable.

---

← [SoC and Chiplet Architecture](../00_Index.md) · [Simulation](../06_Simulation/00_Index.md) · [Architecture book](../../00_Index.md)
