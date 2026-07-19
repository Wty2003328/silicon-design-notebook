# GPU Framework, Compiler, Kernel, and Runtime Implementation Blueprint

> **Abbreviation key:** graphics processing unit (GPU); artificial intelligence (AI); intermediate representation (IR); single instruction, multiple threads (SIMT); general matrix multiplication (GEMM); high-bandwidth memory (HBM); application binary interface (ABI); just-in-time compilation (JIT); key-value (KV) cache.

## 0. Purpose and design ideology

This chapter reconstructs the path from a framework model to GPU kernels and command streams. The design ideology is **compile semantics into explicit work, then make locality and asynchrony first-class**. A GPU stack earns performance by choosing graph boundaries, layouts, fusion, tiles, memory lifetimes, streams, and kernel variants; it preserves correctness through typed IR, guards, dependency events, and reference fallbacks.

Read [AI Workload and Operator Mapping](01_AI_Workload_and_Operator_Mapping.md) for hardware mapping and [GPU Software, Simulator, Verification, and Bring-up](../06_Implementation_Blueprints/03_GPU_Software_Simulator_Verification_and_Bringup_Blueprint.md) for the lower driver/hardware contract.

## 1. Layer map and artifacts

~~~mermaid
flowchart LR
    FW["framework graph + parameters"] --> CAP["capture/export + guards"]
    CAP --> IR["semantic tensor IR"]
    IR --> OPT["decompose / fuse / layout / precision"]
    OPT --> LOW["GPU schedule + kernel IR"]
    LOW --> BIN["kernel code objects + engine plan"]
    BIN --> RT["runtime: allocator / streams / events / graphs"]
    RT --> DRV["driver context + command queues"]
    DRV --> GPU["SIMT / tensor / memory / collectives"]
~~~

The release contains a source-model manifest, captured graph and guard set, compiler configuration, optimized IR, engine/execution plan, kernel code objects, constant/packed weights, shape profiles, workspace bound, target GPU/driver requirements, numerical contract, and validation evidence. Hash every input that changes generated work.

## 2. Tensor and IR contract

Each IR value stores element/accumulator type, logical shape or symbolic bounds, strides/layout, address space/device, quantization parameters, alias/lifetime, alignment, mutability, sharding, and source identity. Operations store semantic attributes, side effects, reduction order constraints, randomness, collective group, and shape guards.

Dynamic dimensions are not “unknown integers.” Represent symbols, bounds, divisibility/alignment constraints, and relationships. A specialized engine is selected only if guards hold; otherwise choose another profile, compile a new variant under policy, or fall back. Unbounded JIT on a request path creates latency and denial-of-service risk.

## 3. Compiler pass order and legality

1. capture/import framework operations and constants;
2. infer shapes/types/aliases and functionalize mutations where possible;
3. decompose unsupported operations into a target-independent core;
4. partition CPU/GPU regions and insert explicit transfers/synchronization;
5. choose precision/quantization and conversion boundaries from quality policy;
6. fuse producer/consumer regions under numerical, alias, resource, and dynamic-shape legality;
7. propagate layouts and select tensor-core-compatible forms;
8. plan parallelism and collectives across devices;
9. lower fused regions to library calls or generated kernel IR;
10. schedule tiles, warps, asynchronous copies, reductions, and epilogues;
11. allocate temporary buffers using dependency/lifetime intervals;
12. emit code, plan, guards, diagnostics, and reference mapping.

Passes must explain rejected fusion or tensor-core mapping: unsupported type/shape, alias, register/shared-memory pressure, numerical order, layout conversion, dynamic guard, or target limitation. This feedback lets hardware/software research distinguish an unavailable mechanism from a compiler miss.

## 4. Kernel contract and code generation

A kernel descriptor records function/code-object identity, parameter ABI, grid/block/cluster mapping, dynamic shared memory, registers/thread estimate, tensor/memory layouts, supported shape/stride/alignment/precision, workspace, stream/event dependencies, determinism, numerical tolerance, and target architecture.

Generated kernels choose:

- program/thread-block mapping of output tiles;
- warp/lane roles and divergence;
- global-memory transaction and shared-memory bank layouts;
- register tile and accumulator placement;
- asynchronous load stages and producer-consumer barriers;
- tensor versus SIMT instructions;
- reduction order and epilogue fusion;
- edge masks and bounds safety.

The compiler cost model estimates operations, useful/physical bytes by level, occupancy resource limits, instruction/array utilization, bank/coalescing efficiency, pipeline stages, launch count, and communication. It rejects candidates exceeding register/shared-memory/workspace or incompatible with shape guards.

## 5. Library, generated, persistent, and graph choices

Vendor/library kernels give broad optimized coverage and stable validation. Generated kernels specialize fusion/layout but expand compilation and test space. Persistent kernels retain scheduling/state on the GPU and reduce launches, but consume resident resources and require a command protocol. Captured GPU command graphs amortize repeated launch setup but need stable topology/addresses or update rules.

Execution-plan nodes therefore have one of: library call with algorithm ID, generated kernel, memory/collective operation, host callback, or persistent-worker command. Each node names inputs/outputs, dependencies, stream, workspace slice, fallback, and profiling identity. The plan executor must not infer dependencies from stream order when cross-stream events are required.

## 6. Autotuning system

The candidate key includes operation/fusion graph, shapes/strides, types, layouts, target GPU and clocks/power class, driver/compiler versions, workspace limit, determinism, and relevant topology. Candidate generation is bounded by legal tile, warp, stage, split/reduction, and algorithm choices.

For each candidate: allocate isolated workspace; initialize inputs; warm; verify output/reference/tolerance; measure repeated execution with device events plus system checks; reject outliers or instability; record work/bytes/resource metadata; and store best result with confidence and validity range. Tuning time and memory are budgets. Production may use offline tuning, a curated database, or guarded background tuning—not unlimited synchronous search.

## 7. Runtime memory, streams, events, and graphs

A device buffer record stores allocation/generation, virtual address, size/alignment, memory kind, owner context/model/request, tensor layout, readiness event, last-use dependencies, reference count, and free state. Use pool/arena allocators to avoid synchronization-heavy allocation, but generations detect stale handles after reuse.

A task/command node stores plan/node identity, kernel/collective/copy, parameter buffer, stream, dependencies, produced event, workspace, request/batch identity, cancel epoch, submission/completion status, and error. Event objects express cross-stream visibility. Free or reuse occurs only after all recorded last-use events complete.

Graph capture records a dependency DAG and command parameters. State which pointers/shapes may update, how updates are validated, and whether a new graph variant is built. A graph cannot safely reuse KV or batch pointers whose lifetimes/generations changed unless parameter update and dependency rules cover them.

## 8. Multi-GPU plan and collectives

The compiler/runtime creates device mesh, tensor sharding, stage/expert ownership, process/rank mapping, and collective groups. Each collective node specifies operation, group/version, tensor layout/count/type, stream dependencies, algorithm/topology hint, timeout, and failure semantics. All ranks must execute compatible collective sequence; divergent control causes hangs.

Overlap requires chunk dependencies: computation publishes a chunk event, collective consumes it, and downstream compute waits on completion. Measure exposed rather than total communication. Keep control-plane membership/version separate from fast data-plane commands; reconfiguration drains or aborts the old group before IDs are reused.

## 9. Numerical, performance, and failure contracts

Precision choice includes input/storage/compute/accumulator/output types, scaling, rounding, saturation, special values, reduction order, and quality threshold. Fusion or split reductions can change bits. Define bit-exact, deterministic-tolerance, or statistically bounded behavior.

Runtime faults include compilation/guard failure, illegal memory, launch/resource failure, out of memory, device lost/reset, collective timeout, poisoned data, and canceled epoch. Each plan node reports one terminal state; later dependent nodes do not run on invalid input unless a defined recovery/fallback reconstructs it.

Performance is bounded by plan critical path, not sum of kernels. Track launch/queue gaps, kernel time, transfers, collectives, allocator/compilation time, and overlap. A fused kernel wins only if removed traffic/launch exceeds reduced occupancy, recomputation, or longer critical-path effects.

## 10. Invariants, tests, and staged construction

Invariants: every generated kernel maps to source semantic nodes; guards dominate specialization; task dependencies cover every producer/consumer and mutable alias; device allocation generations reject stale commands; workspace slices do not overlap live uses; collective sequence/group matches all ranks; and each submitted task completes/faults/cancels once.

Build one static graph using validated library calls; add multi-stream dependencies and pooled memory; add generated kernels with differential tests; add fusion/layout/precision; add offline autotuning; add dynamic profiles/JIT with bounded cache; add command graphs/persistent kernels; then multi-GPU collectives. Preserve node/kernel/transaction identities through profiler and hardware counters.

---

← [GPU AI Performance and Research](03_GPU_AI_Performance_Analysis_and_Research_Methods.md) · next → [GPU Serving Engine, Scheduler, and KV State](05_GPU_Serving_Engine_Scheduler_and_KV_Implementation_Blueprint.md)
